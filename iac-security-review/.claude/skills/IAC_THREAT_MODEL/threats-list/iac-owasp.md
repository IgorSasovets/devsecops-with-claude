# IaC Security — OWASP Cheatsheet Distilled
# Reference for /iac-threat-model — IaC Security Review Suite
# Source: OWASP Infrastructure as Code Security Cheat Sheet
#         https://cheatsheetseries.owasp.org/cheatsheets/Infrastructure_as_Code_Security_Cheat_Sheet.html
#         Snyk Top 5 IaC Security Concerns
# Version: 1.0 | Scope: Cross-framework IaC threats (Terraform + CloudFormation)
# ─────────────────────────────────────────────────────────────────────────────
# This file captures framework-agnostic IaC security categories for use as
# a threat modeling lens when no specific Terraform or CFN catalog entry applies.
# ─────────────────────────────────────────────────────────────────────────────

## OWASP-1 — Insecure Defaults

**Risk:** Cloud providers and IaC frameworks ship with permissive defaults that
are safe for development but dangerous in production. IaC templates that omit
security properties implicitly accept these defaults.

**Threat scenarios:**
- EC2 IMDSv1 enabled by default → SSRF to credential theft
- S3 Block Public Access disabled by default (pre-2023 accounts) → data exposure
- RDS `PubliclyAccessible` defaults to `false` but is easily set to `true`
- Security Groups allow all egress by default → data exfiltration

**IaC mitigation:** Explicitly set all security-relevant properties — never rely
on cloud defaults. Use `aws_default_*` Terraform resources to harden defaults
in each VPC/account. Fail-closed: if a property is omitted, assume it is insecure.

---

## OWASP-2 — Hardcoded Secrets

**Risk:** Credentials, API keys, passwords, and tokens embedded directly in IaC
templates are committed to VCS, stored in state files, and visible in
deployment logs. They are effectively public.

**Threat scenarios:**
- Database passwords in `aws_db_instance.password` in `.tf` files
- API keys in `aws_lambda_function.environment.variables`
- SSH key material in `aws_instance.user_data`
- `Default:` values for CFN parameters containing secrets
- Secrets in Terraform outputs without `sensitive = true`

**IaC mitigation:**
- Terraform: `sensitive = true` on variables and outputs; use `random_password`
  + Secrets Manager for database passwords
- CFN: `NoEcho: true` on all sensitive parameters; use `{{resolve:secretsmanager:...}}`
  dynamic references
- Universal: scan templates with tools like `detect-secrets`, `truffleHog`,
  `checkov` (CKV_SECRET_*) before commit

---

## OWASP-3 — Insufficient Access Controls

**Risk:** Over-permissive IAM policies, broad trust relationships, and missing
resource-level policies create lateral movement and privilege escalation paths.

**Threat scenarios:**
- IAM policies with `Action: "*"` or `Resource: "*"` 
- Role trust policies with `Principal: "*"` (no condition)
- S3 bucket policies missing `aws:SourceVpc` or `aws:PrincipalOrgID` conditions
- Lambda resource policies open to any account
- Missing VPC endpoint policies (traffic filtered only by bucket policy)

**IaC mitigation:**
- Apply least privilege: enumerate specific actions and ARNs
- Use `aws:PrincipalOrgID` condition to scope cross-account access to org
- Define resource-level policies for S3, SQS, SNS, KMS, Lambda
- Use IAM permission boundaries for roles created by IaC pipelines

---

## OWASP-4 — Missing Encryption

**Risk:** Data stored or transmitted without encryption is exposed to
anyone with access to the underlying storage layer, AWS support staff,
or a storage-level breach.

**Threat scenarios:**
- Unencrypted EBS volumes, RDS instances, EFS file systems
- S3 buckets without default encryption — objects stored in plaintext
- SQS/SNS without KMS encryption — messages readable without KMS IAM
- ElastiCache without in-transit or at-rest encryption
- CloudTrail log files without KMS encryption
- ELB/ALB without HTTPS listener — traffic in plaintext

**IaC mitigation:**
- Universal encryption-at-rest: enable for all storage services (EBS, S3, RDS,
  EFS, DynamoDB, SQS, SNS, ElastiCache, Secrets Manager)
- Use customer-managed KMS keys (CMKs) for sensitive data; do not rely solely
  on AWS-managed keys for key rotation control
- Enforce HTTPS/TLS: S3 bucket policy deny on `aws:SecureTransport: false`;
  ALB HTTPS-only listeners; RDS `require_ssl`

---

## OWASP-5 — Inadequate Logging and Monitoring

**Risk:** Without comprehensive logging, attack activity cannot be detected,
investigated, or proven. Gaps in logging are themselves a compliance failure.

**Threat scenarios:**
- CloudTrail disabled or single-region only → API calls go unrecorded
- No VPC Flow Logs → network attacks invisible
- No S3 server-access logging on sensitive buckets
- CloudTrail log file validation disabled → tampered logs undetectable
- CloudTrail KMS key deletable → logs become unreadable
- No SNS/CloudWatch alarm on root account usage

**IaC mitigation:**
- Always provision CloudTrail as `IsMultiRegionTrail: true` with log file validation
- VPC Flow Logs on every VPC (log ALL traffic, not just REJECT)
- S3 access logging on sensitive and log-destination buckets
- CloudWatch metric filters + alarms for root usage, IAM changes, SG changes

---

## OWASP-6 — Unpatched / Outdated Components

**Risk:** Using outdated AMIs, database engine versions, or container images
exposes workloads to known CVEs. IaC templates that hardcode old versions
institutionalize the vulnerability.

**Threat scenarios:**
- `aws_db_instance.engine_version = "5.7"` (EOL MySQL)
- EC2 launch templates with hardcoded AMI IDs that age without update
- Lambda runtimes pinned to deprecated runtime versions
- EKS node groups with outdated Kubernetes version
- Auto Minor Version Upgrade disabled → CVE patches skipped

**IaC mitigation:**
- Enable `auto_minor_version_upgrade = true` for managed databases
- Use SSM Parameter Store AMI lookups (`data "aws_ssm_parameter" "ami"`) instead
  of hardcoded AMI IDs
- Pin to supported engine versions; document EOL dates in comments
- Enable EKS auto-upgrade or document manual upgrade process

---

## OWASP-7 — Insecure Network Configuration

**Risk:** Overly permissive network rules expose services to untargeted
internet scanning, lateral movement, and data exfiltration.

**Threat scenarios:**
- Security Groups with `0.0.0.0/0` ingress on non-public-service ports
- NACLs allowing all traffic (effectively disabled)
- VPC peering without route table least-access
- Resources deployed in default VPC (wide-open default SG)
- No VPC endpoints for S3/DynamoDB (traffic traverses internet gateway)
- Public subnets used for database tiers

**IaC mitigation:**
- Define Security Groups with minimum required ingress/egress rules
- Remove or harden default VPC and default Security Group
- Use private subnets for all non-public-facing resources
- Add VPC endpoints for AWS services accessed from private subnets
- Apply VPC peering route tables with specific CIDR prefixes only

---

## OWASP-8 — IaC Pipeline and Supply Chain

**Risk:** The IaC pipeline itself is an attack surface. Compromised pipeline
credentials, malicious modules, or unsigned templates can alter infrastructure
at scale.

**Threat scenarios:**
- Terraform modules from public registry without version pinning
- CFN templates fetched from public S3 URLs
- Terraform state in unprotected S3 bucket (full topology + secrets in state)
- `terraform plan` / `apply` credentials with broad permissions stored in CI secrets
- No approval gate before `terraform apply` in production

**IaC mitigation:**
- Pin all provider and module versions exactly (`= X.Y.Z`)
- Store Terraform state in private, versioned, KMS-encrypted S3 bucket
- Use DynamoDB state locking
- Scope CI/CD IaC role to minimum required permissions; use OIDC (no long-lived keys)
- Require human approval for production IaC applies; use Terraform Cloud / Atlantis

---

## Threat Modeling Quick-Reference: STRIDE × IaC Layer

| STRIDE Category | Typical IaC Layer | Key Controls |
|----------------|-------------------|--------------|
| Spoofing | IAM trust policies, STS | `ExternalId`, `aws:PrincipalOrgID`, MFA conditions |
| Tampering | State files, templates, resources | State backend access controls, DeletionPolicy, stack policies |
| Repudiation | CloudTrail, VPC Flow Logs, S3 access logs | Multi-region trail, log file validation, immutable log bucket |
| Information Disclosure | IAM policies, network config, encryption | Least privilege, encryption-at-rest/transit, no public exposure |
| Denial of Service | Resource quotas, DynamoDB lock, lifecycle | `prevent_destroy`, multi-AZ, deletion protection |
| Elevation of Privilege | IAM policies, role trust, instance metadata | Least privilege, IMDSv2-only, permission boundaries |
