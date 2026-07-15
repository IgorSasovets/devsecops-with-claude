# Terraform-Specific Security Threats
# Reference for /iac-threat-model — IaC Security Review Suite
# Sources: HashiCorp Security Blog, tfsec rules, Sysdig Terraform Best Practices,
#          OWASP IaC Cheatsheet, Snyk IaC Top 5
# Version: 1.0 | Framework: Terraform (AWS provider) | Scope: IaC-reviewable only
# ─────────────────────────────────────────────────────────────────────────────

## TF-STATE — State File Security

| ID | STRIDE | Threat | Impact | Terraform Anti-Pattern | Secure Pattern |
|----|--------|--------|--------|----------------------|----------------|
| TF-STATE-01 | Information Disclosure | Secrets leak via plaintext state file | All sensitive values managed by Terraform (passwords, keys, certs) are stored in state in plaintext | Local state (`terraform.tfstate`) or unencrypted S3 backend | Remote backend with SSE-KMS + access logging + bucket policy |
| TF-STATE-02 | Tampering | State manipulation — attacker modifies state to cause destructive apply | Full infrastructure destroy/alter on next apply | S3 backend without versioning or MFA delete | Versioned S3 backend + DynamoDB lock + restrictive bucket policy |
| TF-STATE-03 | Information Disclosure | State file publicly accessible in S3 | Full topology + secrets exposure | `backend "s3"` without `block_public_acls` on target bucket | Dedicated private S3 bucket with `block_public_acls = true` |
| TF-STATE-04 | Repudiation | No state lock — concurrent apply causes race condition | Infrastructure drift, duplicate resources | Missing `dynamodb_table` in S3 backend config | `dynamodb_table = "terraform-state-lock"` |

---

## TF-PROV — Provider and Module Supply Chain

| ID | STRIDE | Threat | Impact | Terraform Anti-Pattern | Secure Pattern |
|----|--------|--------|--------|----------------------|----------------|
| TF-PROV-01 | Tampering | Unpinned provider version — breaking or malicious provider update | Unexpected behavior on `terraform init` after provider release | `version = ">= 1.0"` or no version constraint | `version = "= X.Y.Z"` exact pin in `required_providers` |
| TF-PROV-02 | Tampering | Module sourced from public registry without version pin | Malicious module update silently changes infrastructure | `module "x" { source = "hashicorp/consul/aws" }` (no version) | `version = "X.Y.Z"` pinned in every module block |
| TF-PROV-03 | Tampering | Module sourced from non-authoritative GitHub ref | Branch/tag hijack replaces module code | `source = "github.com/user/module?ref=main"` (branch ref) | `ref=<commit-sha>` or use registry with pinned version |
| TF-PROV-04 | Information Disclosure | Provider credentials in `.tfvars` file committed to VCS | Credential exposure in git history | `aws_access_key`, `aws_secret_key` in any `.tfvars` file | IAM roles / environment variables / Vault; never in `.tfvars` |

---

## TF-CREDS — Credential and Secret Handling

| ID | STRIDE | Threat | Impact | Terraform Anti-Pattern | Secure Pattern |
|----|--------|--------|--------|----------------------|----------------|
| TF-CREDS-01 | Information Disclosure | Hardcoded credentials in resource arguments | Credential exposure in state + VCS | `password = "hardcoded"` in `aws_db_instance` or similar | `password = random_password.x.result` + Secrets Manager |
| TF-CREDS-02 | Information Disclosure | Sensitive variable without `sensitive = true` | Value printed in plan output and logs | `variable "db_password" {}` without `sensitive = true` | `variable "db_password" { sensitive = true }` |
| TF-CREDS-03 | Information Disclosure | `output` block exposing sensitive values | Secrets visible in `terraform output` and CI logs | `output "db_password" { value = aws_db_instance.x.password }` | `output "db_password" { value = ...; sensitive = true }` |
| TF-CREDS-04 | Information Disclosure | User data with embedded secrets | Secrets retrievable via `DescribeInstanceAttribute` API | `user_data = "...SECRET=abc..."` in `aws_instance` | Reference SSM Parameter Store or Secrets Manager at runtime |

---

## TF-IAM — IAM Resource Misconfiguration

| ID | STRIDE | Threat | Impact | Terraform Anti-Pattern | Secure Pattern |
|----|--------|--------|--------|----------------------|----------------|
| TF-IAM-01 | Elevation of Privilege | Wildcard Action in IAM policy | Full privilege escalation path | `actions = ["*"]` in `aws_iam_policy_document` | Explicit least-privilege action list |
| TF-IAM-02 | Elevation of Privilege | Wildcard Resource in IAM policy | Policy applies to all resources of the type | `resources = ["*"]` where specific ARN is possible | Specific resource ARN(s) |
| TF-IAM-03 | Elevation of Privilege | No IAM permission boundary on user/role created by Terraform | Privilege escalation via role assumption | `aws_iam_role` without `permissions_boundary` in environments where it is enforced | `permissions_boundary = "arn:aws:iam::ACCOUNT:policy/boundary"` |
| TF-IAM-04 | Elevation of Privilege | Overly broad `assume_role_policy` principal | Any AWS principal can assume the role | `principals { type = "AWS"; identifiers = ["*"] }` | Explicit account/role ARN + condition keys |
| TF-IAM-05 | Elevation of Privilege | `iam:PassRole` on `*` resource | Attacker can assign any role to any service | `actions = ["iam:PassRole"]` + `resources = ["*"]` | Scope `iam:PassRole` to specific role ARN(s) |

---

## TF-NET — Network Resource Misconfiguration

| ID | STRIDE | Threat | Impact | Terraform Anti-Pattern | Secure Pattern |
|----|--------|--------|--------|----------------------|----------------|
| TF-NET-01 | Information Disclosure | Security Group open ingress from `0.0.0.0/0` on admin ports | Remote admin access from any IP | `cidr_blocks = ["0.0.0.0/0"]` on port 22, 3389, 5432, etc. | Restrict to known CIDRs or use Systems Manager Session Manager |
| TF-NET-02 | Information Disclosure | Security Group open egress to `0.0.0.0/0` | Data exfiltration on any port | Default egress rule left unrestricted | Restrict egress to required destinations only |
| TF-NET-03 | Denial of Service | Default VPC not removed | Resources accidentally deployed into default VPC inherit permissive default SG | No `aws_default_vpc` resource removing default VPC | `aws_default_vpc { force_destroy = true }` or documented exception |
| TF-NET-04 | Information Disclosure | `aws_default_security_group` with unrestricted rules | All default-SG members communicate freely | Default SG rules not overridden | `aws_default_security_group` with empty `ingress`/`egress` |

---

## TF-MISC — Miscellaneous Terraform Security Patterns

| ID | STRIDE | Threat | Impact | Terraform Anti-Pattern | Secure Pattern |
|----|--------|--------|--------|----------------------|----------------|
| TF-MISC-01 | Information Disclosure | `terraform plan` output logged without masking in CI | Sensitive values in CI logs | `terraform plan` piped to stdout in CI without `-out=plan.tfplan` | Use `-out` to save binary plan; display with `terraform show -json` + mask secrets |
| TF-MISC-02 | Tampering | Unprotected Terraform workspace — any team member can apply | Unauthorized infrastructure change | No remote execution (Terraform Cloud/Enterprise) or branch protection | Enforce remote execution + approval workflow for production |
| TF-MISC-03 | Information Disclosure | `local-exec` / `remote-exec` provisioner with sensitive args | Arguments logged by Terraform | `provisioner "local-exec" { command = "... secret=..." }` | Pass secrets via environment variables, not command arguments |
| TF-MISC-04 | Tampering | Lifecycle `prevent_destroy = false` on critical resources | Accidental deletion of databases, KMS keys | Missing `lifecycle { prevent_destroy = true }` on stateful resources | Add `lifecycle { prevent_destroy = true }` to RDS, KMS, S3 state buckets |
| TF-MISC-05 | Denial of Service | Auto Minor Version Upgrade disabled for managed databases | Known CVEs remain unpatched indefinitely | `auto_minor_version_upgrade = false` on `aws_db_instance` | `auto_minor_version_upgrade = true` (default) unless explicit exception |
