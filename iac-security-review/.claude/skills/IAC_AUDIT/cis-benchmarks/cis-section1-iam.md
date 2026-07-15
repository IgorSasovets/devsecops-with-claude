# CIS AWS Foundations Benchmark v4.0.1 — Section 1: Identity and Access Management
# Source: CIS Amazon Web Services Foundations Benchmark v4.0.1 (December 2024)
# Scope: Automated controls only | IaC-checkable only
# For /iac-audit — IaC Security Review Suite
# ─────────────────────────────────────────────────────────────────────────────
# Controls excluded from this file:
#   - Manual assessment controls (e.g. 1.1, 1.2, 1.3, 1.6, 1.7, 1.11, 1.21, 1.22)
#   - Automated controls that require live AWS API calls, not IaC review:
#     1.5 (root MFA - runtime state), 1.10 (MFA per user - runtime state),
#     1.12 (credential age - runtime), 1.13 (key count - runtime),
#     1.14 (key rotation age - runtime), 1.19 (cert expiry - runtime)
# ─────────────────────────────────────────────────────────────────────────────

## [1.4] Ensure no 'root' user account access key exists
**Level:** L1 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** The 'root' user account is the most privileged user in an AWS
account. AWS Access Keys provide programmatic access to a given AWS account. It
is recommended that all access keys associated with the 'root' user account be
deleted.

**Rationale:** Deleting root access keys limits vectors by which the account can
be compromised. Root access keys cannot be scoped — they grant unrestricted
access to all AWS services.

**IaC Check:** Verify no IaC resources create or reference root access keys.
Look for `aws_iam_access_key` resources where `user = "root"`, or any
`AWS::IAM::AccessKey` with `UserName: root`.

**Terraform pattern (FAIL):**
```hcl
resource "aws_iam_access_key" "root_key" {
  user = "root"  # VIOLATION
}
```
**CFN pattern (FAIL):**
```yaml
RootAccessKey:
  Type: AWS::IAM::AccessKey
  Properties:
    UserName: root  # VIOLATION
```
**Note:** This control is primarily runtime-verified. IaC check looks for
explicit creation of root keys. The absence of such a resource in IaC does
not guarantee root keys don't exist in the account.

---

## [1.8] Ensure IAM password policy requires minimum length of 14 or greater
**Level:** L1 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** Password policies are, in part, used to enforce password
complexity requirements. IAM password policies can be used to ensure passwords
are at least a given length. It is recommended that the password policy require
a minimum password length of 14.

**Rationale:** Setting a password complexity policy increases account resiliency
against brute force login attempts.

**IaC Check:** Look for `aws_iam_account_password_policy` in Terraform or
`AWS::IAM::AccountPasswordPolicy` in CFN. Verify `minimum_password_length >= 14`.

**Terraform pattern (PASS):**
```hcl
resource "aws_iam_account_password_policy" "default" {
  minimum_password_length = 14  # Must be >= 14
  require_uppercase_characters = true
  require_lowercase_characters = true
  require_numbers              = true
  require_symbols              = true
}
```
**Terraform pattern (FAIL):** `minimum_password_length = 8` or resource absent.

---

## [1.9] Ensure IAM password policy prevents password reuse
**Level:** L1 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** IAM password policies can prevent the reuse of a given
password by the same user. It is recommended that the password policy prevent
the reuse of passwords.

**Rationale:** Preventing password reuse increases account resiliency against
brute force login attempts by reducing the attack surface for credential
stuffing attacks.

**IaC Check:** In `aws_iam_account_password_policy`, verify
`password_reuse_prevention` is set and >= 24 (CIS recommendation).

**Terraform pattern (PASS):**
```hcl
resource "aws_iam_account_password_policy" "default" {
  password_reuse_prevention = 24  # Must be set; >= 24 recommended
}
```
**Terraform pattern (FAIL):** `password_reuse_prevention` absent or `= 0`.

---

## [1.15] Ensure IAM users receive permissions only through groups
**Level:** L1 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** IAM users are granted access to services, functions, and data
through IAM policies. There are three ways to define policies for a user: 1)
Edit the user policy directly, 2) attach a policy directly to a user, or
3) add the user to an IAM group that has an attached policy. It is recommended
to only use the third approach.

**Rationale:** Assigning IAM policy only through groups unifies permissions
management to a single, reviewable location. Reduces risk of ad-hoc permission
grants that are not audited.

**IaC Check:** In Terraform, flag any `aws_iam_user_policy` resources or
`aws_iam_user_policy_attachment` resources. In CFN, flag `AWS::IAM::User`
with inline `Policies` property or `ManagedPolicyArns` directly.

**Terraform pattern (FAIL):**
```hcl
resource "aws_iam_user_policy" "inline" {  # VIOLATION
  user   = aws_iam_user.example.name
  policy = data.aws_iam_policy_document.example.json
}
resource "aws_iam_user_policy_attachment" "direct" {  # VIOLATION
  user       = aws_iam_user.example.name
  policy_arn = "arn:aws:iam::aws:policy/ReadOnlyAccess"
}
```
**Terraform pattern (PASS):** Use `aws_iam_group_policy` and
`aws_iam_group_membership` instead.

---

## [1.16] Ensure IAM policies that allow full "*:*" administrative privileges are not attached
**Level:** L1 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** IAM policies are the means by which privileges are granted to
users, groups, or roles. It is recommended and considered a standard security
advice to grant least privilege — that is, granting only the permissions required
to perform a task. Policies that allow full `"*:*"` administrative privileges
create risk of unintended privilege escalation.

**Rationale:** Attaching policies that have `"Effect": "Allow"` with `"Action":
"*"` over `"Resource": "*"` effectively grants administrator access to any
principal using the policy.

**IaC Check:** Search all IAM policy documents for the combination of
`Effect: Allow`, `Action: *`, and `Resource: *`. Flag any such policy regardless
of whether it's inline or managed.

**Terraform pattern (FAIL):**
```hcl
data "aws_iam_policy_document" "admin" {
  statement {
    effect    = "Allow"
    actions   = ["*"]      # VIOLATION
    resources = ["*"]      # VIOLATION
  }
}
```
**CFN pattern (FAIL):**
```yaml
PolicyDocument:
  Statement:
    - Effect: Allow
      Action: "*"        # VIOLATION
      Resource: "*"      # VIOLATION
```

---

## [1.17] Ensure a support role has been created to manage incidents with AWS Support
**Level:** L1 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** AWS provides a support center that can be used for incident
notification and response, as well as technical support and customer services.
Create an IAM Role to allow authorized users to manage incidents with AWS Support.

**Rationale:** By implementing least privilege for access control, an IAM Role
will require an appropriate IAM Policy to allow Support Center Access in order
to manage incidents.

**IaC Check:** Verify existence of an IAM role or policy attaching the
`AWSSupportAccess` managed policy ARN. At least one such resource should be
present.

**Terraform pattern (PASS):**
```hcl
resource "aws_iam_role_policy_attachment" "support" {
  role       = aws_iam_role.support.name
  policy_arn = "arn:aws:iam::aws:policy/AWSSupportAccess"
}
```
**CFN pattern (PASS):**
```yaml
SupportRolePolicy:
  Type: AWS::IAM::ManagedPolicy
  Properties:
    ManagedPolicyArns:
      - arn:aws:iam::aws:policy/AWSSupportAccess
```

---

## [1.18] Ensure IAM instance roles are used for AWS resource access from instances
**Level:** L2 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** AWS access from within AWS instances can be done by either
encoding AWS keys into AWS API calls or by assigning the instance to a role
which has an appropriate permissions policy for the required access. "AWS Access
from an instance using an assigned IAM role" is preferable because: (a) the role
is easily managed from the console, (b) credentials are rotated automatically, and
(c) role-based access is more auditable.

**Rationale:** Instance roles provide dynamic, automatically-rotated credentials.
Hardcoded credentials in EC2 user data or environment variables create long-lived
secrets that are hard to rotate and easy to exfiltrate.

**IaC Check:** Verify all `aws_instance` resources reference an
`iam_instance_profile`. Flag any that do not. Also check for hardcoded
AWS credentials in `user_data`. For ECS, verify task definitions use
`task_role_arn` rather than environment variable credentials.

**Terraform pattern (PASS):**
```hcl
resource "aws_instance" "app" {
  iam_instance_profile = aws_iam_instance_profile.app.name  # Required
  # No credentials in user_data
}
```
**Terraform pattern (FAIL):**
```hcl
resource "aws_instance" "app" {
  # No iam_instance_profile  # VIOLATION
  user_data = "AWS_ACCESS_KEY_ID=AKIA..."  # VIOLATION
}
```

---

## [1.20] Ensure that IAM Access Analyzer is enabled for all regions
**Level:** L1 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** Enable IAM Access Analyzer for IAM policies about all resources
in each active region. IAM Access Analyzer is a technology introduced at AWS
re:Invent 2019. After the Analyzer is enabled in IAM, scan results are updated
daily or whenever IaC changes are made. Findings appear in the IAM console and
Amazon EventBridge.

**Rationale:** AWS IAM Access Analyzer helps you identify the resources in your
organization and accounts, such as Amazon S3 buckets or IAM roles, shared with
an external entity. This lets you identify unintended access to your resources
and data.

**IaC Check:** Verify `aws_accessanalyzer_analyzer` is provisioned. Ensure
analyzer type is `ACCOUNT` (or `ORGANIZATION` if applicable). One analyzer per
active region is required; check for regional provider configurations.

**Terraform pattern (PASS):**
```hcl
resource "aws_accessanalyzer_analyzer" "default" {
  analyzer_name = "account-analyzer"
  type          = "ACCOUNT"
}
```
**CFN pattern (PASS):**
```yaml
AccessAnalyzer:
  Type: AWS::AccessAnalyzer::Analyzer
  Properties:
    AnalyzerName: account-analyzer
    Type: ACCOUNT
```
