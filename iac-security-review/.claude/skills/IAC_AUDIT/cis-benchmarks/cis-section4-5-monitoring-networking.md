# CIS AWS Foundations Benchmark v4.0.1 — Sections 4 & 5: Monitoring and Networking
# Source: CIS Amazon Web Services Foundations Benchmark v4.0.1 (December 2024)
# Scope: Automated controls only | IaC-checkable only
# For /iac-audit — IaC Security Review Suite
# ─────────────────────────────────────────────────────────────────────────────
# Section 4 note: Most section 4 controls (4.2–4.15) require CloudWatch metric
# filters + alarms. These are checkable in IaC but most are Manual assessment type.
# Only 4.1 and 4.16 are Automated. Section 4 IaC patterns are provided for
# completeness as supplementary checks even when assessment type is Manual.
#
# Controls excluded (Manual assessment type or not IaC-checkable):
#   4.2–4.15 (Manual - metric filter/alarm checks; IaC patterns provided as
#   supplementary reference), 5.1.2 (Manual - CIFS access)
# ─────────────────────────────────────────────────────────────────────────────

## SECTION 4: MONITORING

## [4.1] Ensure unauthorized API calls are monitored
**Level:** L2 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** Real-time monitoring of API calls can be achieved by directing
CloudTrail Logs to CloudWatch Logs and establishing corresponding metric filters
and alarms. It is recommended that a metric filter and alarm be established for
unauthorized API calls — specifically calls that are denied due to lack of
permissions.

**Rationale:** Monitoring unauthorized API calls will help reveal application
errors and may reduce time to detect malicious activity.

**IaC Check:** Verify the following chain exists in IaC:
1. `aws_cloudtrail` with `cloud_watch_logs_group_arn` set
2. `aws_cloudwatch_log_metric_filter` matching unauthorized API call pattern
3. `aws_cloudwatch_metric_alarm` on that metric with `sns_topic_arns` for notification

**Terraform pattern (PASS):**
```hcl
resource "aws_cloudwatch_log_metric_filter" "unauthorized_api" {
  name           = "UnauthorizedAPICalls"
  pattern        = "{ ($.errorCode = \"*UnauthorizedAccess*\") || ($.errorCode = \"AccessDenied*\") }"
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name

  metric_transformation {
    name      = "UnauthorizedAPICalls"
    namespace = "CISBenchmark"
    value     = "1"
  }
}
resource "aws_cloudwatch_metric_alarm" "unauthorized_api" {
  alarm_name          = "UnauthorizedAPICalls"
  metric_name         = "UnauthorizedAPICalls"
  namespace           = "CISBenchmark"
  statistic           = "Sum"
  period              = "300"
  evaluation_periods  = "1"
  threshold           = "1"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  alarm_actions       = [aws_sns_topic.security_alerts.arn]
}
```

---

## [4.16] Ensure AWS Security Hub is enabled
**Level:** L2 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** Security Hub collects security data from various AWS accounts,
services, and supported third-party partner products, helping you analyze your
security trends and identify the highest-priority security issues. When you enable
Security Hub, it begins to consume, aggregate, organize, and prioritize findings
from AWS services that you have enabled, such as Amazon GuardDuty, Amazon Inspector,
and Amazon Macie.

**Rationale:** AWS Security Hub provides a comprehensive view of your security
state in AWS and helps you check your environment against security industry
standards and best practices, enabling you to quickly assess the security posture
across your AWS accounts.

**IaC Check:** Verify `aws_securityhub_account` resource is present. Optionally
check for `aws_securityhub_standards_subscription` enabling the AWS Foundational
Security Best Practices or CIS Benchmark standard.

**Terraform pattern (PASS):**
```hcl
resource "aws_securityhub_account" "main" {}

resource "aws_securityhub_standards_subscription" "cis" {
  standards_arn = "arn:aws:securityhub:${var.region}::standards/cis-aws-foundations-benchmark/v/4.0.0"
  depends_on    = [aws_securityhub_account.main]
}
```
**Terraform pattern (FAIL):** No `aws_securityhub_account` resource present.

**CFN pattern (PASS):**
```yaml
SecurityHub:
  Type: AWS::SecurityHub::Hub
  Properties: {}
```

---

## Supplementary Section 4 Reference — CloudWatch Alarm Patterns
# These controls are Manual assessment type but IaC-checkable.
# Include as supplementary findings when CloudWatch/CloudTrail resources are present.

### [4.3] Root account usage monitoring
```hcl
resource "aws_cloudwatch_log_metric_filter" "root_usage" {
  pattern = "{ $.userIdentity.type = \"Root\" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != \"AwsServiceEvent\" }"
  # ... alarm on this metric
}
```

### [4.4] IAM policy changes
```hcl
resource "aws_cloudwatch_log_metric_filter" "iam_changes" {
  pattern = "{ ($.eventName=DeleteGroupPolicy) || ($.eventName=DeleteRolePolicy) || ($.eventName=DeleteUserPolicy) || ($.eventName=PutGroupPolicy) || ($.eventName=PutRolePolicy) || ($.eventName=PutUserPolicy) || ($.eventName=CreatePolicy) || ($.eventName=DeletePolicy) || ($.eventName=CreatePolicyVersion) || ($.eventName=DeletePolicyVersion) || ($.eventName=SetDefaultPolicyVersion) || ($.eventName=AttachRolePolicy) || ($.eventName=DetachRolePolicy) || ($.eventName=AttachUserPolicy) || ($.eventName=DetachUserPolicy) || ($.eventName=AttachGroupPolicy) || ($.eventName=DetachGroupPolicy) }"
}
```

### [4.5] CloudTrail configuration changes
```hcl
resource "aws_cloudwatch_log_metric_filter" "cloudtrail_changes" {
  pattern = "{ ($.eventName = CreateTrail) || ($.eventName = UpdateTrail) || ($.eventName = DeleteTrail) || ($.eventName = StartLogging) || ($.eventName = StopLogging) }"
}
```

### [4.10] Security group changes
```hcl
resource "aws_cloudwatch_log_metric_filter" "sg_changes" {
  pattern = "{ ($.eventName = AuthorizeSecurityGroupIngress) || ($.eventName = AuthorizeSecurityGroupEgress) || ($.eventName = RevokeSecurityGroupIngress) || ($.eventName = RevokeSecurityGroupEgress) || ($.eventName = CreateSecurityGroup) || ($.eventName = DeleteSecurityGroup) }"
}
```

---

## SECTION 5: NETWORKING

## [5.1.1] Ensure EBS volume encryption is enabled in all regions
**Level:** L1 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** Elastic Compute Cloud (EC2) supports encryption at rest when
using the Elastic Block Store (EBS) service. While disabled by default, forcing
encryption when creating EBS volumes is supported. It is recommended to configure
EBS volume encryption to be enabled for new EBS volumes.

**Rationale:** Encrypting EBS volumes protects data at rest from unauthorized
access. Without encryption, anyone with physical or logical access to the
underlying hardware can read the data.

**IaC Check (two layers):**
1. Check for `aws_ebs_encryption_by_default` resource with `enabled = true` (account-level default).
2. Check all `aws_ebs_volume`, `aws_instance.root_block_device`, and
   `aws_launch_template.block_device_mappings` have `encrypted = true`.

**Terraform pattern (PASS — account-level default):**
```hcl
resource "aws_ebs_encryption_by_default" "main" {
  enabled = true
}
```
**Terraform pattern (PASS — per-resource):**
```hcl
resource "aws_ebs_volume" "data" {
  encrypted  = true
  kms_key_id = aws_kms_key.ebs.arn
}
resource "aws_instance" "app" {
  root_block_device {
    encrypted  = true
    kms_key_id = aws_kms_key.ebs.arn
  }
}
```
**Terraform pattern (FAIL):** `encrypted = false` or neither account-level
default nor per-resource encryption set.

**CFN pattern (PASS):**
```yaml
EBSVolume:
  Type: AWS::EC2::Volume
  Properties:
    Encrypted: true
    KmsKeyId: !Ref EbsKmsKey
```

---

## [5.2] Ensure no Network ACLs allow ingress from 0.0.0.0/0 to remote server administration ports
**Level:** L1 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** The Network Access Control List (NACL) function provides
stateless filtering of ingress and egress network traffic to AWS resources. It
is recommended that no NACL allows unrestricted ingress access to remote server
administration ports, such as SSH to port 22 and RDP to port 3389.

**Rationale:** Public access to remote server administration ports, such as 22
and 3389, increases resource attack surface and unnecessarily raises the risk
of resource compromise.

**IaC Check:** Review all `aws_network_acl_rule` resources. Flag any rule with:
- `rule_action = "allow"` AND `egress = false` (ingress)
- `cidr_block = "0.0.0.0/0"` or `ipv6_cidr_block = "::/0"`
- `from_port` or `to_port` covering ports 22 or 3389

**Terraform pattern (FAIL):**
```hcl
resource "aws_network_acl_rule" "allow_ssh" {
  rule_action = "allow"
  egress      = false
  cidr_block  = "0.0.0.0/0"  # VIOLATION
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
}
```
**CFN pattern (FAIL):**
```yaml
NACLEntry:
  Type: AWS::EC2::NetworkAclEntry
  Properties:
    RuleAction: allow
    Egress: false
    CidrBlock: 0.0.0.0/0   # VIOLATION
    PortRange:
      From: 22
      To: 22
```

---

## [5.3] Ensure no security groups allow ingress from 0.0.0.0/0 to remote server administration ports
**Level:** L1 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** Security groups provide stateful filtering of ingress and egress
network traffic to AWS resources. It is recommended that no security group allow
unrestricted ingress access to remote server administration ports, such as SSH
to port 22 and RDP to port 3389, using IPv4.

**Rationale:** Removing unfettered connectivity to remote console services such
as SSH and RDP reduces a server's exposure to risk. Public access to SSH/RDP
enables brute-force, credential-stuffing, and zero-day exploitation attempts
from the entire internet.

**IaC Check:** In all `aws_security_group` and `aws_security_group_rule`
resources, flag any ingress rule where:
- `cidr_blocks` contains `"0.0.0.0/0"` AND
- `from_port` ≤ 22 ≤ `to_port` OR `from_port` ≤ 3389 ≤ `to_port`
- OR `protocol = "-1"` (all traffic) with `cidr_blocks = ["0.0.0.0/0"]`

**Terraform pattern (FAIL):**
```hcl
resource "aws_security_group" "web" {
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # VIOLATION
  }
}
```
**CFN pattern (FAIL):**
```yaml
WebSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 22
        ToPort: 22
        CidrIp: 0.0.0.0/0   # VIOLATION
```

---

## [5.4] Ensure no security groups allow ingress from ::/0 to remote server administration ports
**Level:** L1 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** Security groups provide stateful filtering of ingress and egress
network traffic to AWS resources. It is recommended that no security group allow
unrestricted ingress access to remote server administration ports (SSH port 22,
RDP port 3389) using IPv6.

**Rationale:** Same rationale as 5.3 — IPv6 ingress from `::/0` is equally
exposed to the entire internet and should be restricted to known CIDRs.

**IaC Check:** Same logic as 5.3 but check `ipv6_cidr_blocks` contains `"::/0"`.

**Terraform pattern (FAIL):**
```hcl
resource "aws_security_group" "web" {
  ingress {
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    ipv6_cidr_blocks = ["::/0"]  # VIOLATION
  }
}
```

---

## [5.5] Ensure the default security group of every VPC restricts all traffic
**Level:** L2 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** A VPC comes with a default security group whose initial settings
deny all inbound traffic, allow all outbound traffic, and allow all traffic
between instances assigned to the security group. If you don't specify a security
group when you launch an instance, the instance is automatically assigned to this
default security group. As a result, the instance may accidentally be exposed if
resources are launched without an explicit SG.

**Rationale:** Configuring all VPC default security groups to restrict all
traffic prevents resources from inadvertently using the default SG and being
exposed to broad network access.

**IaC Check:** For every `aws_vpc`, verify there is a corresponding
`aws_default_security_group` resource with empty `ingress` and `egress` blocks
(no rules at all).

**Terraform pattern (PASS):**
```hcl
resource "aws_default_security_group" "default" {
  vpc_id = aws_vpc.main.id
  # No ingress or egress rules — completely restricts traffic
}
```
**Terraform pattern (FAIL):** No `aws_default_security_group` resource (leaves
default SG with default rules allowing all inter-SG traffic).

**CFN note:** The default security group cannot be directly managed as a CFN
resource type. Flag this as a gap if only CFN templates are present — recommend
managing with Terraform or custom resource.

---

## [5.7] Ensure that the EC2 Metadata Service only allows IMDSv2
**Level:** L1 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** When enabling the Metadata Service on AWS EC2 instances, users
have the option of using either Instance Metadata Service Version 1 (IMDSv1;
a request/response method) or Instance Metadata Service Version 2 (IMDSv2; a
session-oriented method). It is recommended that IMDSv2 is enforced for all EC2
instances.

**Rationale:** IMDSv2 requires a session token, which prevents SSRF attacks from
being able to steal the EC2 instance's IAM role credentials. IMDSv1 is exploitable
via simple HTTP GET requests (e.g., curl from a vulnerable web application using
SSRF) and has been used in several high-profile cloud breaches.

**IaC Check:** Verify all `aws_instance`, `aws_launch_template`, and
`aws_launch_configuration` resources have `http_tokens = "required"` in their
`metadata_options` block.

**Terraform pattern (PASS):**
```hcl
resource "aws_instance" "app" {
  metadata_options {
    http_tokens                 = "required"   # Enforces IMDSv2
    http_endpoint               = "enabled"
    http_put_response_hop_limit = 1            # Prevent container metadata exfil
  }
}
resource "aws_launch_template" "app" {
  metadata_options {
    http_tokens                 = "required"
    http_put_response_hop_limit = 1
  }
}
```
**Terraform pattern (FAIL):** `http_tokens = "optional"` (IMDSv1 allowed) or
`metadata_options` block absent (defaults to `"optional"` = IMDSv1 enabled).

**CFN pattern (PASS):**
```yaml
EC2Instance:
  Type: AWS::EC2::Instance
  Properties:
    MetadataOptions:
      HttpTokens: required
      HttpEndpoint: enabled
      HttpPutResponseHopLimit: 1
```
