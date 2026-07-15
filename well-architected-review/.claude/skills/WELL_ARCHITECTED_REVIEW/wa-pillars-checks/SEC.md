# Well-Architected Review — Security Pillar Checks
#
# Source:  AWS Well-Architected Framework, Security Pillar
#          https://docs.aws.amazon.com/wellarchitected/latest/framework/a-security.html
# Version: 2024 (WAF revision; check https://aws.amazon.com/architecture/well-architected
#          for the latest edition)
# Scope:   22 checks across SEC 1–8 (Security Foundations through Application Security).
#          Checks are limited to what is verifiable via read-only AWS CLI evidence.
#          SEC 9 (Incident Response process maturity) is excluded — requires human interview.
# Format:  Each check entry: ID | question reference | best practice title | evaluation logic
#          Evidence file references use paths relative to <output-dir>/raw/.

---

## SEC 1 — Security Foundations

### SEC01-BP01 — Separate workloads using accounts
**WAF Question:** SEC 1. How do you securely operate your workload?
**Evidence files:** `org_info.json`

Read `org_info.json`. PASS if an AWS Organization exists (`Organization` key present and
not `_error`) with a `MasterAccountId` — this confirms a multi-account structure.
WARNING if the file contains `_error` (single account or Orgs API denied — cannot confirm
account separation). FAIL is not applicable (cannot disprove separation from read-only data).

### SEC01-BP02 — Secure account root user and properties
**WAF Question:** SEC 1. How do you securely operate your workload?
**Evidence files:** `iam_account_summary.json`

Read `iam_account_summary.json`. Evaluate:
- FAIL if `AccountMFAEnabled` = 0 (root MFA not enabled).
- FAIL if `AccountAccessKeysPresent` = 1 (root access keys exist).
- PASS if both `AccountMFAEnabled` = 1 and `AccountAccessKeysPresent` = 0.
If the file is `_error`, mark SKIPPED: "access denied or service not enabled".

### SEC01-BP03 — Identify and validate control objectives — AWS Config enabled
**WAF Question:** SEC 1. How do you securely operate your workload?
**Evidence files:** `<REGION>/config_recorders.json`, `<REGION>/config_delivery_channels.json`
  (check all regions in scope)

For each region: PASS if both a configuration recorder and a delivery channel exist
and are not `_error`. WARNING if present in some regions but not all checked regions.
FAIL if no recorder or no delivery channel exists in any checked region.

### SEC01-BP04 — Stay up to date with security threats — Security Hub enabled
**WAF Question:** SEC 1. How do you securely operate your workload?
**Evidence files:** `<REGION>/securityhub_hub.json`, `<REGION>/securityhub_standards.json`

For each region: check `securityhub_hub.json`. FAIL if `_error` in all regions
(Security Hub not enabled anywhere). WARNING if enabled in some regions but not all.
If hub is present, read `securityhub_standards.json`: WARNING if `EnabledStandards`
list is empty (hub enabled but no security standards activated). PASS if hub enabled
in all regions with at least one standard active.

### SEC01-BP06 — Automate deployment of standard security controls — IaC in use
**WAF Question:** SEC 1. How do you securely operate your workload?
**Evidence files:** `<REGION>/cfn_stacks.json` (all regions)

Read `cfn_stacks.json` for each region. PASS if CloudFormation stacks exist in at
least one region (suggests infrastructure managed as code). WARNING if no stacks found
in any region (suggests manual deployments, which undermine consistent security baselines).

---

## SEC 2 — Identity and Access Management (Authentication)

### SEC02-BP01 — Use strong sign-in mechanisms — MFA for IAM users
**WAF Question:** SEC 2. How do you manage authentication for people and machines?
**Evidence files:** `iam_users.json`, `iam_mfa_devices.json`

Read `iam_users.json` to get all IAM users. Read `iam_mfa_devices.json` to get MFA
assignments. For each user in `iam_users.json`:
- If the user has a password (`PasswordLastUsed` field exists or `LoginProfile` can
  be inferred), check if an MFA device is assigned to that user's ARN in `iam_mfa_devices.json`.
- FAIL if any console-enabled IAM user has no MFA device assigned.
- WARNING if coverage is partial (some users have MFA, some do not).
- PASS if all IAM users with console access have MFA, or no IAM users exist (SSO-only).
- If either file is `_error`, mark SKIPPED: "access denied or service not enabled".

### SEC02-BP02 — Use temporary credentials — IAM Identity Center configured
**WAF Question:** SEC 2. How do you manage authentication for people and machines?
**Evidence files:** `iam_sso_instances.json`

Read `iam_sso_instances.json`. PASS if at least one SSO instance exists in `Instances`
list. WARNING if the list is empty or the file is `_error` — absence of SSO suggests
reliance on long-lived IAM user credentials.

### SEC02-BP04 — Rely on a centralized identity provider
**WAF Question:** SEC 2. How do you manage authentication for people and machines?
**Evidence files:** `iam_sso_instances.json`

Same evidence as SEC02-BP02. PASS if Identity Center instance exists (centralized IdP
configured). WARNING if absent (fragmented identity management likely).

### SEC02-BP05 — Audit and rotate credentials — access key ages
**WAF Question:** SEC 2. How do you manage authentication for people and machines?
**Evidence files:** `iam_users.json`

Read `iam_users.json`. For each user that has access keys (check `AccessKeyMetadata`
or infer from `CreateDate`):
- FAIL if any active access key `CreateDate` is older than 180 days.
- WARNING if any active access key `CreateDate` is between 90 and 180 days old.
- PASS if all active access keys are less than 90 days old, or no access keys exist.
Note: the 90-day threshold aligns with CIS AWS Foundations Benchmark v4.0.1.

---

## SEC 3 — Identity and Access Management (Permissions)

### SEC03-BP01 — Define access requirements — IAM Access Analyzer enabled
**WAF Question:** SEC 3. How do you manage permissions for people and machines?
**Evidence files:** `iam_access_analyzer.json`

Read `iam_access_analyzer.json`. FAIL if `Analyzers` list is empty or file is `_error`.
PASS if at least one analyzer with `Status: ACTIVE` exists.
WARNING if analyzers exist but none are `ACTIVE`.

### SEC03-BP02 — Grant least privilege — review for overly broad policies
**WAF Question:** SEC 3. How do you manage permissions for people and machines?
**Evidence files:** `iam_customer_policies.json`

Read `iam_customer_policies.json`. Scan policy names in the `Policies` list.
WARNING if any policy name contains substrings suggesting full access
(e.g. "FullAccess", "Admin", "PowerUser", "All") outside of a clearly named
admin or break-glass context.
Note: Full semantic analysis of every policy document's Action/Resource
wildcards requires fetching each policy version — flag overly named policies
for human review rather than attempting incomplete automated verdict.
Recommendation: use IAM Access Analyzer and AWS-managed policies where possible.

### SEC03-BP04 — Reduce permissions continuously — Permission Boundaries in use
**WAF Question:** SEC 3. How do you manage permissions for people and machines?
**Evidence files:** `iam_roles.json`

Read `iam_roles.json`. Inspect each role in `Roles` list for `PermissionsBoundary` field.
WARNING if zero roles have a `PermissionsBoundary` set (suggests no boundary guardrails
on developer or service roles). PASS if at least some roles — particularly any named
for developer use — have permission boundaries applied.

### SEC03-BP07 — Use resource-based policies and VPC Endpoints
**WAF Question:** SEC 3. How do you manage permissions for people and machines?
**Evidence files:** `<REGION>/ec2_vpc_endpoints.json` (all regions)

Read `ec2_vpc_endpoints.json` for each region. WARNING if no VPC endpoints found in
any region (AWS service traffic such as S3 or DynamoDB may traverse the public internet).
PASS if at least one endpoint exists (Gateway or Interface type).

---

## SEC 4 — Detection

### SEC04-BP01 — Configure service and application logging — CloudTrail enabled
**WAF Question:** SEC 4. How do you detect and investigate security events?
**Evidence files:** `<REGION>/cloudtrail_trails.json` (all regions)

Read `cloudtrail_trails.json` for each region. Evaluate across all trails:
- FAIL if no trail exists in any checked region.
- FAIL if any trail has `IsLogging: false`.
- WARNING if a trail exists but `LogFileValidationEnabled: false`
  (log integrity cannot be confirmed).
- PASS if a multi-region trail exists (`IsMultiRegionTrail: true`) with
  `IsLogging: true` and `LogFileValidationEnabled: true`.

### SEC04-BP02 — Capture logs — VPC Flow Logs enabled
**WAF Question:** SEC 4. How do you detect and investigate security events?
**Evidence files:** `<REGION>/ec2_flow_logs.json`, `<REGION>/ec2_vpcs.json` (all regions)

For each region: read `ec2_vpcs.json` to get VPC IDs. Read `ec2_flow_logs.json`
to get flow log resources. Cross-reference:
- FAIL if any VPC has no associated flow log (`ResourceId` not present in flow log list).
- WARNING if some VPCs have flow logs and some do not.
- PASS if all VPCs have an active flow log.

### SEC04-BP02b — Capture logs — S3 server access logging
**WAF Question:** SEC 4. How do you detect and investigate security events?
**Evidence files:** `s3_bucket_<NAME>.json` (all sampled buckets, `logging` field)

Read the `logging` section of each per-bucket file. Count buckets where
`LoggingEnabled` is absent or `_error`. WARNING if more than 20% of sampled
buckets lack server access logging. PASS if all checked buckets have logging enabled.

### SEC04-BP03 — Analyze logs — GuardDuty enabled
**WAF Question:** SEC 4. How do you detect and investigate security events?
**Evidence files:** `<REGION>/guardduty_detectors.json` (all regions)

Read `guardduty_detectors.json` for each region. Check `DetectorIds` list:
- FAIL if the list is empty or `_error` in all checked regions.
- WARNING if a detector exists in some regions but not all checked regions.
- PASS if an active detector exists in every checked region.

---

## SEC 5 — Infrastructure Protection (Network)

### SEC05-BP01 — Create network layers — VPC with multi-AZ subnets
**WAF Question:** SEC 5. How do you protect your network resources?
**Evidence files:** `<REGION>/ec2_vpcs.json`, `<REGION>/ec2_subnets.json` (all regions)

For each region: read `ec2_vpcs.json` and `ec2_subnets.json`. Group subnets by VPC.
- WARNING if any VPC has all subnets in a single Availability Zone
  (no network-layer separation).
- PASS if every VPC has subnets spanning at least two distinct AZs.

### SEC05-BP02 — Control traffic flow — Security Group hygiene
**WAF Question:** SEC 5. How do you protect your network resources?
**Evidence files:** `<REGION>/ec2_security_groups.json` (all regions)

For each region, read `ec2_security_groups.json`. For each group in `SecurityGroups`,
inspect `IpPermissions`:
- FAIL if any inbound rule allows `0.0.0.0/0` or `::/0` on port 22 (SSH)
  or port 3389 (RDP). Cite the GroupId and rule.
- WARNING if `0.0.0.0/0` or `::/0` is open on any other port except 80 or 443.
- PASS if no overly permissive SSH/RDP ingress rules found.

### SEC05-BP03 — Implement inspection-based protection — WAF in use
**WAF Question:** SEC 5. How do you protect your network resources?
**Evidence files:** `<REGION>/wafv2_regional_acls.json`, `wafv2_cloudfront_acls.json`

Read regional and CloudFront WAF ACL files across all regions.
WARNING if no WAF ACLs found in any scope. PASS if at least one ACL exists
(regional or CloudFront).

### SEC05-BP04 — Automate network protection — AWS Shield Advanced
**WAF Question:** SEC 5. How do you protect your network resources?
**Evidence files:** `shield_subscription.json`

Read `shield_subscription.json`. PASS if subscription is active
(`SubscriptionState: ACTIVE`). WARNING if file is `_error` or no subscription
(only Shield Standard, which provides no DDoS response team or cost protection).

---

## SEC 6 — Infrastructure Protection (Compute)

### SEC06-BP01 — Perform vulnerability management — Inspector enabled
**WAF Question:** SEC 6. How do you protect your compute resources?
**Evidence files:** `<REGION>/inspector2_permissions.json` (all regions)

Read `inspector2_permissions.json` for each region. WARNING if file is `_error`
or returns empty permissions (Inspector not enabled). PASS if at least one region
shows Inspector permissions active.

### SEC06-BP02 — Provision from hardened images — ECR image scanning
**WAF Question:** SEC 6. How do you protect your compute resources?
**Evidence files:** `<REGION>/ecr_repositories.json` (all regions)

For each region, read `ecr_repositories.json`. Inspect `repositories`:
- WARNING if any repository has `imageScanningConfiguration.scanOnPush: false`.
  Cite the repository name.
- PASS if all repositories have `scanOnPush: true`, or no repositories exist.

### SEC06-BP03 — Reduce attack surface — SSM over SSH
**WAF Question:** SEC 6. How do you protect your compute resources?
**Evidence files:** `<REGION>/ec2_security_groups.json`, `<REGION>/ssm_managed_instances.json`

For each region: check `ec2_security_groups.json` for any inbound rule allowing
port 22 on `0.0.0.0/0` (see SEC05-BP02). Also read `ssm_managed_instances.json`.
- FAIL if port 22 is open to `0.0.0.0/0` AND `ssm_managed_instances.json` has an
  empty `InstanceInformationList` (SSH exposed with no SSM alternative).
- WARNING if port 22 is open to `0.0.0.0/0` but SSM instances do exist
  (SSM available but SSH still exposed).
- PASS if port 22 is not open to `0.0.0.0/0` and SSM instances are present.

---

## SEC 7 — Data Protection

### SEC07-BP01 — Classify your data — S3 account-level public access block
**WAF Question:** SEC 7. How do you classify your data?
**Evidence files:** `s3_account_public_access.json`

Read `s3_account_public_access.json`. Check `PublicAccessBlockConfiguration`:
- FAIL if any of the four settings (`BlockPublicAcls`, `IgnorePublicAcls`,
  `BlockPublicPolicy`, `RestrictPublicBuckets`) is `false` or absent.
- PASS if all four settings are `true`.
- SKIPPED if file is `_error`.

### SEC07-BP02 — Protect data at rest — S3 bucket encryption
**WAF Question:** SEC 7. How do you classify your data?
**Evidence files:** `s3_bucket_<NAME>.json` (all sampled buckets, `encryption` field)

For each per-bucket file, read the `encryption` section:
- FAIL if `ServerSideEncryptionConfiguration` is absent or `_error` (no default encryption).
- WARNING if the rule uses `SSE-S3` (AES256) rather than `SSE-KMS` (weaker key management).
- PASS if all checked buckets have at least SSE-S3 default encryption.

### SEC07-BP03 — Protect data at rest — EBS volume encryption
**WAF Question:** SEC 7. How do you classify your data?
**Evidence files:** `<REGION>/ec2_volumes.json` (all regions)

For each region, read `ec2_volumes.json`. Check each volume in `Volumes`:
- FAIL if any volume has `Encrypted: false`. Cite VolumeId and region.
- WARNING if some volumes are unencrypted (partial coverage).
- PASS if all volumes are encrypted, or no volumes exist.

### SEC07-BP04 — Protect data at rest — RDS encryption
**WAF Question:** SEC 7. How do you classify your data?
**Evidence files:** `<REGION>/rds_instances.json` (all regions)

For each region, read `rds_instances.json`. Check each DB in `DBInstances`:
- FAIL if any instance has `StorageEncrypted: false`. Cite DBInstanceIdentifier.
- PASS if all instances are encrypted, or no RDS instances exist.

### SEC07-BP05 — Protect data at rest — KMS key rotation
**WAF Question:** SEC 7. How do you classify your data?
**Evidence files:** `<REGION>/kms_keys.json` (all regions)

Read `kms_keys.json` for each region. For each key in `Keys`:
- Skip AWS-managed keys (KeyManager: AWS) — rotation is automatic.
- For customer-managed keys (KeyManager: CUSTOMER): WARNING if
  `KeyRotationEnabled: false` is returned. Cite KeyId.
- PASS if all customer-managed keys have rotation enabled, or only AWS-managed
  keys exist.

### SEC07-BP07 — Protect data in transit — S3 enforce HTTPS
**WAF Question:** SEC 7. How do you classify your data?
**Evidence files:** `s3_bucket_<NAME>.json` (all sampled buckets, `policy` field)

For each per-bucket file, read the `policy` section. Parse the bucket policy JSON
and look for a `Deny` statement with `aws:SecureTransport: false` condition:
- WARNING if the bucket policy is absent (`_error`) or has no
  `aws:SecureTransport` deny condition (HTTP access permitted).
- PASS if a deny on `aws:SecureTransport: false` is present.
Note: absence of a policy (`_error` on get-bucket-policy) means no SSL enforcement —
treat as WARNING, not PASS.
