# AWS Threat Technique Catalog
# Reference for /iac-threat-model — IaC Security Review Suite
# Source: AWS Samples Threat Technique Catalog for AWS + STRIDE mapping
# Version: 1.0 | Provider: AWS | Scope: IaC-reviewable threats only
#
# Format per entry:
#   ID, STRIDE category, Threat, Typical actor, Attack surface, Impacted asset,
#   IaC root cause, Detection signal
# ─────────────────────────────────────────────────────────────────────────────

## T-IAM — Identity and Access Management Threats

| ID | STRIDE | Threat | Actor | Attack Surface | Asset | IaC Root Cause |
|----|--------|--------|-------|---------------|-------|----------------|
| T-IAM-01 | Elevation of Privilege | Assume role via wildcard trust policy | Insider / external | IAM AssumeRole API | Any AWS resource in account | `assume_role_policy` with `Principal: "*"` or overly broad condition |
| T-IAM-02 | Elevation of Privilege | Escalate privileges via `iam:PassRole` | Compromised IAM user | IAM service | EC2, Lambda, ECS task | Role with `iam:PassRole` on `*` without boundary |
| T-IAM-03 | Elevation of Privilege | Abuse `iam:CreatePolicyVersion` to self-grant admin | Compromised IAM principal | IAM service | Entire AWS account | Managed policy allowing `iam:CreatePolicyVersion` |
| T-IAM-04 | Elevation of Privilege | Add attacker-controlled key to existing role trust | Compromised IAM admin | IAM trust policies | Role's assumed permissions | No SCP preventing trust policy modification |
| T-IAM-05 | Information Disclosure | Exfiltrate secrets via `iam:UpdateAssumeRolePolicy` cross-account | External attacker | IAM trust policies | Secrets / data accessed by the role | Role trust policy that allows external account IDs without `ExternalId` |
| T-IAM-06 | Elevation of Privilege | Attach `AdministratorAccess` managed policy | Compromised principal with `iam:AttachRolePolicy` | IAM service | Entire AWS account | IAM policy permitting `iam:AttachRolePolicy` on `*` |
| T-IAM-07 | Elevation of Privilege | Create new IAM user and add to admin group | Compromised principal | IAM service | Entire AWS account | Policy with `iam:CreateUser` + `iam:AddUserToGroup` |
| T-IAM-08 | Elevation of Privilege | Abuse Lambda resource-based policy for cross-account invocation | External attacker | Lambda resource policy | Lambda function and its role | `aws_lambda_permission` with `principal: "*"` |
| T-IAM-09 | Information Disclosure | Enumerate IAM roles and policies to map attack surface | Reconnaissance actor | IAM read APIs | Account structure | No SCPs blocking `iam:List*` / `iam:Get*` for untrusted identities |
| T-IAM-10 | Elevation of Privilege | IMDS v1 abuse — extract EC2 instance role credentials | Attacker with SSRF / code exec on EC2 | EC2 metadata service | Instance IAM role | `http_tokens = "optional"` (IMDSv1 enabled) |

---

## T-S3 — S3 Storage Threats

| ID | STRIDE | Threat | Actor | Attack Surface | Asset | IaC Root Cause |
|----|--------|--------|-------|---------------|-------|----------------|
| T-S3-01 | Information Disclosure | Public read access to sensitive bucket | Any internet user | S3 public access | Stored data | `acl = "public-read"` or Block Public Access disabled |
| T-S3-02 | Information Disclosure | Exfiltrate data via pre-signed URL from compromised role | Attacker with role credentials | S3 API | Bucket objects | No VPC endpoint + bucket policy allowing broad principals |
| T-S3-03 | Tampering | Overwrite / delete objects (data destruction or poisoning) | Compromised principal | S3 API | Bucket objects | No MFA delete, no versioning, no Object Lock |
| T-S3-04 | Information Disclosure | Server-side encryption bypass — data readable at storage layer | AWS insider / physical access | Storage layer | Objects at rest | `sse_algorithm` not set or `aws:kms` not enforced |
| T-S3-05 | Tampering | Bucket policy modification to exfiltrate data to attacker account | Compromised IAM admin | S3 bucket policy | Entire bucket | No SCP restricting `s3:PutBucketPolicy` cross-account |
| T-S3-06 | Repudiation | Disable access logging to cover exfiltration | Attacker with `s3:PutBucketLogging` | S3 logging config | Audit trail | `logging` block absent or `target_bucket` points to same bucket |
| T-S3-07 | Information Disclosure | Data transit interception — unencrypted S3 access over HTTP | Network-level attacker | HTTP endpoint | Objects in transit | Bucket policy missing `aws:SecureTransport` deny condition |
| T-S3-08 | Elevation of Privilege | Confused deputy via S3 bucket policy cross-service | Service principal abuser | S3 resource policy | Bucket contents | Bucket policy with `Principal: {"Service": "*"}` |

---

## T-NET — Networking Threats

| ID | STRIDE | Threat | Actor | Attack Surface | Asset | IaC Root Cause |
|----|--------|--------|-------|---------------|-------|----------------|
| T-NET-01 | Information Disclosure | Lateral movement via overly permissive Security Group ingress | Attacker with initial access | Security Group | All instances in SG | `cidr_blocks = ["0.0.0.0/0"]` on non-80/443 ports |
| T-NET-02 | Information Disclosure | Data exfiltration via overly permissive Security Group egress | Malware on instance | Security Group egress rules | Data on instance | Unrestricted egress (`0.0.0.0/0`) |
| T-NET-03 | Information Disclosure | Public RDS / ElastiCache reachable from internet | External attacker | RDS endpoint | Database contents | `publicly_accessible = true` without SG restriction |
| T-NET-04 | Tampering | Route table manipulation to redirect traffic | Compromised VPC admin | Route tables | All VPC traffic | No Network Firewall / no route table change monitoring |
| T-NET-05 | Denial of Service | NACL modification to block legitimate traffic | Insider threat | NACLs | VPC connectivity | No SCP restricting `ec2:ReplaceNetworkAclEntry` |
| T-NET-06 | Information Disclosure | VPC peering without least-access routing | Compromised peer VPC | Peering connection | All subnets in VPC | Route table allows full peer CIDR instead of specific prefixes |
| T-NET-07 | Information Disclosure | ELB/ALB exposed without WAF | External attacker | Load balancer | Web application | `aws_wafv2_web_acl_association` absent on internet-facing ALB |
| T-NET-08 | Denial of Service | Default VPC in use — wide-open default SG | Attacker with any IAM creds | Default security group | All resources using default SG | Default VPC not removed; default SG not restricted |

---

## T-LOG — Logging and Monitoring Threats

| ID | STRIDE | Threat | Actor | Attack Surface | Asset | IaC Root Cause |
|----|--------|--------|-------|---------------|-------|----------------|
| T-LOG-01 | Repudiation | Disable CloudTrail to evade detection | Attacker with CloudTrail admin | CloudTrail service | Audit trail | No SCP preventing `cloudtrail:StopLogging`; no trail deletion alarm |
| T-LOG-02 | Repudiation | Tamper with CloudTrail log files in S3 | Attacker with S3 write access | CloudTrail log bucket | Log integrity | `enable_log_file_validation = false` |
| T-LOG-03 | Repudiation | Delete VPC Flow Log config | Attacker with `ec2:DeleteFlowLogs` | VPC flow logs | Network audit trail | Flow logs not provisioned via IaC (no drift detection) |
| T-LOG-04 | Information Disclosure | CloudTrail logs stored in public S3 bucket | External attacker | Log bucket | Full API audit trail | Log bucket missing `block_public_acls = true` |
| T-LOG-05 | Repudiation | KMS key deletion disables CloudTrail log decryption | Insider with KMS admin | KMS key | Log readability | CloudTrail KMS key has no key deletion window protection |

---

## T-COMP — Compute Threats (EC2, Lambda, ECS, EKS)

| ID | STRIDE | Threat | Actor | Attack Surface | Asset | IaC Root Cause |
|----|--------|--------|-------|---------------|-------|----------------|
| T-COMP-01 | Elevation of Privilege | IMDSv1 SSRF — steal EC2 instance role tokens | Attacker exploiting SSRF in app | EC2 metadata service (169.254.169.254) | Instance IAM role | `metadata_options { http_tokens = "optional" }` |
| T-COMP-02 | Elevation of Privilege | Over-privileged Lambda execution role | Compromised Lambda | Lambda execution role | Any AWS resource role can access | Lambda role with `Action: "*"` or attached `AdministratorAccess` |
| T-COMP-03 | Information Disclosure | ECS task definition with hardcoded secrets in environment | Any user who can `DescribeTaskDefinition` | ECS task definition | Secrets / credentials | `environment` key in `container_definitions` holding secrets instead of `secrets` (SSM/Secrets Manager ref) |
| T-COMP-04 | Information Disclosure | EC2 user-data containing secrets | Any user who can `DescribeInstanceAttribute` | EC2 user data | Secrets in userdata | `user_data` block containing `password=`, `secret=`, etc. |
| T-COMP-05 | Elevation of Privilege | EKS node IAM role with excessive permissions | Compromised pod | EKS node role | All AWS resources node role can access | Node group IAM role policy broader than `AmazonEKSWorkerNodePolicy` |
| T-COMP-06 | Denial of Service | Lambda reserved concurrency set to 0 (accidental DoS) | IaC misconfiguration | Lambda service quotas | Function availability | `reserved_concurrent_executions = 0` unintentionally |

---

## T-DATA — Data and Key Management Threats

| ID | STRIDE | Threat | Actor | Attack Surface | Asset | IaC Root Cause |
|----|--------|--------|-------|---------------|-------|----------------|
| T-DATA-01 | Information Disclosure | Unencrypted EBS volume snapshot shared publicly | External attacker | EBS snapshot | Volume data | `encrypted = false` on EBS + no account-level snapshot policy |
| T-DATA-02 | Information Disclosure | RDS snapshot shared with another account | Attacker with target account access | RDS snapshot | Database contents | `publicly_accessible` or `final_snapshot_identifier` shared |
| T-DATA-03 | Tampering | KMS key rotation disabled — long-lived key compromise | Attacker with key material | KMS key | All data encrypted under key | `enable_key_rotation = false` |
| T-DATA-04 | Denial of Service | KMS key scheduled for deletion while still in use | Misconfigured IaC change | KMS key | All dependent encrypted resources | `deletion_window_in_days` too short without usage check |
| T-DATA-05 | Information Disclosure | Secrets Manager secret missing resource policy — broad access | Any IAM principal in account | Secrets Manager | Secret value | No `aws_secretsmanager_resource_policy` scoping access |
| T-DATA-06 | Information Disclosure | Parameter Store SecureString uses default KMS key | Attacker with `ssm:GetParameter` | SSM Parameter Store | Stored secrets | `type = "SecureString"` without explicit `key_id` (uses aws/ssm) |

---

## T-SUPPLY — Supply Chain and Pipeline Threats

| ID | STRIDE | Threat | Actor | Attack Surface | Asset | IaC Root Cause |
|----|--------|--------|-------|---------------|-------|----------------|
| T-SUPPLY-01 | Tampering | Terraform provider or module from untrusted registry | Supply chain attacker | Terraform registry | Entire IaC deployment | `source` referencing non-official registry without version pinning |
| T-SUPPLY-02 | Tampering | CloudFormation template fetched from public S3 URL | Attacker who can modify bucket | S3 template source | Deployed stack | `TemplateURL` pointing to public or non-versioned S3 object |
| T-SUPPLY-03 | Tampering | Terraform state file stored in unprotected backend | Attacker with S3 read access | Terraform state | Full infrastructure topology + secrets in state | S3 backend without encryption, versioning, or access logging |
| T-SUPPLY-04 | Elevation of Privilege | Terraform state lock disabled — concurrent apply race | Concurrent malicious apply | DynamoDB lock table | Infrastructure consistency | `dynamodb_table` not set in S3 backend config |
