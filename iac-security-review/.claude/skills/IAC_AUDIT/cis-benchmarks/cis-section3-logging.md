# CIS AWS Foundations Benchmark v4.0.1 — Section 3: Logging
# Source: CIS Amazon Web Services Foundations Benchmark v4.0.1 (December 2024)
# Scope: Automated controls only | IaC-checkable only
# For /iac-audit — IaC Security Review Suite
# ─────────────────────────────────────────────────────────────────────────────
# Controls excluded from this file:
#   3.8, 3.9 (S3 object-level logging - Automated but requires specific S3 event
#   selectors in CloudTrail; partially IaC-checkable; flagged in section notes)
# ─────────────────────────────────────────────────────────────────────────────

## [3.1] Ensure CloudTrail is enabled in all regions
**Level:** L1 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** AWS CloudTrail is a web service that records AWS API calls for
your account and delivers log files to you. The recorded information includes
the identity of the API caller, the time of the API call, the source IP address
of the API caller, the request parameters, and the response elements returned
by the AWS service. CloudTrail provides a history of AWS API calls for an
account, including API calls made via the Management Console, SDKs, CLI, and
other services.

**Rationale:** The AWS API call history produced by CloudTrail enables security
analysis, resource change tracking, and compliance auditing. A multi-region trail
ensures API calls in non-primary regions are captured, including global services
like IAM, STS, and CloudFront.

**IaC Check:** Verify at least one `aws_cloudtrail` resource with:
- `is_multi_region_trail = true`
- `include_global_service_events = true`
- `enable_logging = true`
- `s3_bucket_name` pointing to a dedicated, non-public bucket

**Terraform pattern (PASS):**
```hcl
resource "aws_cloudtrail" "main" {
  name                          = "main-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail_logs.id
  is_multi_region_trail         = true
  include_global_service_events = true
  enable_logging                = true
}
```
**Terraform pattern (FAIL):**
- `is_multi_region_trail = false`
- `enable_logging = false`
- No `aws_cloudtrail` resource present in any template

**CFN pattern (PASS):**
```yaml
CloudTrail:
  Type: AWS::CloudTrail::Trail
  Properties:
    TrailName: main-trail
    S3BucketName: !Ref CloudTrailLogsBucket
    IsMultiRegionTrail: true
    IncludeGlobalServiceEvents: true
    IsLogging: true
```

---

## [3.2] Ensure CloudTrail log file validation is enabled
**Level:** L2 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** CloudTrail log file validation creates a digitally signed
digest file containing a hash of each log that CloudTrail writes to S3. These
digest files can be used to determine whether a log file was changed, deleted,
or remained unchanged after CloudTrail delivered the log.

**Rationale:** Enabling log file validation provides additional integrity checks
for CloudTrail logs. It allows detection of log tampering, which an attacker
might attempt to cover their tracks after a compromise.

**IaC Check:** Verify all `aws_cloudtrail` resources have
`enable_log_file_validation = true`.

**Terraform pattern (PASS):**
```hcl
resource "aws_cloudtrail" "main" {
  enable_log_file_validation = true
}
```
**Terraform pattern (FAIL):** `enable_log_file_validation = false` or absent
(defaults to `false`).

**CFN pattern (PASS):**
```yaml
CloudTrail:
  Type: AWS::CloudTrail::Trail
  Properties:
    EnableLogFileValidation: true
```

---

## [3.3] Ensure AWS Config is enabled in all regions
**Level:** L2 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** AWS Config is a web service that performs configuration
management of supported AWS resources within your account and delivers log files
to you. AWS Config tracks the state of your AWS resources and their configuration
over time. It records configuration changes and evaluates them against desired
configurations.

**Rationale:** The AWS configuration item history captured by AWS Config enables
security analysis, resource change tracking, and compliance auditing. It also
feeds into Security Hub findings and provides the basis for config rules.

**IaC Check:** Verify `aws_config_configuration_recorder` with `recording = true`
and `aws_config_delivery_channel` are both present. Check recorder covers
`all_supported = true` and `include_global_resource_types = true`.

**Terraform pattern (PASS):**
```hcl
resource "aws_config_configuration_recorder" "main" {
  name     = "main"
  role_arn = aws_iam_role.config.arn
  recording_group {
    all_supported                 = true
    include_global_resource_types = true
  }
}
resource "aws_config_configuration_recorder_status" "main" {
  name       = aws_config_configuration_recorder.main.name
  is_enabled = true
}
resource "aws_config_delivery_channel" "main" {
  name           = "main"
  s3_bucket_name = aws_s3_bucket.config_logs.id
}
```
**Terraform pattern (FAIL):** Any of the three resources absent, or `is_enabled = false`.

**CFN pattern (PASS):**
```yaml
ConfigRecorder:
  Type: AWS::Config::ConfigurationRecorder
  Properties:
    RoleARN: !GetAtt ConfigRole.Arn
    RecordingGroup:
      AllSupported: true
      IncludeGlobalResourceTypes: true
ConfigDeliveryChannel:
  Type: AWS::Config::DeliveryChannel
  Properties:
    S3BucketName: !Ref ConfigLogsBucket
```

---

## [3.4] Ensure that server access logging is enabled on the S3 bucket used to store CloudTrail logs
**Level:** L1 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** S3 Bucket Access Logging generates a log that contains access
records for each request made to your S3 bucket. An access log record contains
details such as the request type, the resources that were specified in the
request, and the time and date that the request was processed. It is recommended
that bucket access logging be enabled on the S3 bucket where CloudTrail logs
are stored.

**Rationale:** By enabling S3 bucket logging on target S3 buckets, it is possible
to capture all events which may affect objects within any target buckets. Configuring
logs to be placed in a separate bucket allows access to log information which
can be useful in security and incident response workflows.

**IaC Check:** For the S3 bucket used as the CloudTrail log destination, verify
it has `aws_s3_bucket_logging` configured (with a separate target bucket). The
target bucket itself should also be a private bucket.

**Terraform pattern (PASS):**
```hcl
resource "aws_s3_bucket_logging" "cloudtrail_logs" {
  bucket        = aws_s3_bucket.cloudtrail_logs.id
  target_bucket = aws_s3_bucket.access_logs.id  # Separate bucket
  target_prefix = "cloudtrail-access-logs/"
}
```
**Terraform pattern (FAIL):** `aws_s3_bucket_logging` resource absent for the
CloudTrail log bucket, or `target_bucket` same as `bucket` (logs to itself).

**CFN pattern (PASS):**
```yaml
CloudTrailBucket:
  Type: AWS::S3::Bucket
  Properties:
    LoggingConfiguration:
      DestinationBucketName: !Ref AccessLogsBucket
      LogFilePrefix: cloudtrail-access-logs/
```

---

## [3.5] Ensure CloudTrail logs are encrypted at rest using KMS CMKs
**Level:** L2 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** AWS CloudTrail is a web service that records AWS API calls for
an account and makes those logs available to users and resources in accordance
with IAM policies. AWS Key Management Service (KMS) is a managed service that
helps create and control the encryption keys used to encrypt account data, and
uses Hardware Security Modules (HSMs) to protect the security of encryption
keys. CloudTrail logs can be configured to leverage server-side encryption (SSE)
and KMS customer-created master keys (CMK) to further protect CloudTrail logs.

**Rationale:** Configuring CloudTrail to use SSE-KMS provides additional
confidentiality controls on log data as a given user must have S3 read permission
and decrypt permission on the relevant KMS key to decrypt the logs.

**IaC Check:** Verify `aws_cloudtrail` resource has `kms_key_id` set to a
customer-managed KMS key ARN (not the default AWS-managed key).

**Terraform pattern (PASS):**
```hcl
resource "aws_cloudtrail" "main" {
  kms_key_id = aws_kms_key.cloudtrail.arn  # Must reference a CMK
}
resource "aws_kms_key" "cloudtrail" {
  description             = "CloudTrail log encryption key"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}
```
**Terraform pattern (FAIL):** `kms_key_id` absent (uses default S3 encryption
or no encryption).

**CFN pattern (PASS):**
```yaml
CloudTrail:
  Type: AWS::CloudTrail::Trail
  Properties:
    KMSKeyId: !Ref CloudTrailKmsKey
```

---

## [3.6] Ensure rotation for customer-created symmetric CMKs is enabled
**Level:** L2 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** AWS Key Management Service (KMS) allows customers to rotate
the backing key which is key material stored within the KMS which is tied to
the key ID of the customer-created customer master key (CMK). It is the backing
key that is used to perform cryptographic operations such as encryption and
decryption. Automated key rotation currently retains all prior backing keys so
that decryption of encrypted data can take place transparently.

**Rationale:** Rotating encryption keys helps reduce the potential impact of a
compromised key, as data encrypted with a new key cannot be accessed with an
older key.

**IaC Check:** Verify all `aws_kms_key` resources have `enable_key_rotation = true`.
This applies to all symmetric CMKs (rotation is not supported for asymmetric keys).

**Terraform pattern (PASS):**
```hcl
resource "aws_kms_key" "main" {
  description             = "Application encryption key"
  enable_key_rotation     = true  # Must be explicit
  deletion_window_in_days = 30
}
```
**Terraform pattern (FAIL):** `enable_key_rotation = false` or absent (defaults
to `false`).

**CFN pattern (PASS):**
```yaml
KmsKey:
  Type: AWS::KMS::Key
  Properties:
    EnableKeyRotation: true
    PendingWindowInDays: 30
```

---

## [3.7] Ensure VPC flow logging is enabled in all VPCs
**Level:** L2 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** VPC Flow Logs is a feature that enables you to capture
information about the IP traffic going to and from network interfaces in your
VPC. After you've created a flow log, you can view and retrieve its data in
Amazon CloudWatch Logs. It is recommended that VPC Flow Logs be enabled for
packet "Accepts" and "Rejects" for VPCs.

**Rationale:** VPC Flow Logs provide visibility into network traffic that
traverses the VPC and can be used to detect anomalous traffic or gain insights
during security incident response workflows. Logging all traffic (not just
rejects) enables post-incident forensics.

**IaC Check:** For every `aws_vpc` resource, verify there is a corresponding
`aws_flow_log` resource referencing it. The flow log should capture `ALL` traffic
(not just `REJECT`). Destination should be CloudWatch Logs or S3.

**Terraform pattern (PASS):**
```hcl
resource "aws_flow_log" "main" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"  # Capture both ACCEPT and REJECT
  iam_role_arn    = aws_iam_role.flow_logs.arn
  log_destination = aws_cloudwatch_log_group.flow_logs.arn
}
```
**Terraform pattern (FAIL):**
- No `aws_flow_log` for a VPC
- `traffic_type = "REJECT"` only (misses accepted traffic for forensics)

**CFN pattern (PASS):**
```yaml
VpcFlowLog:
  Type: AWS::EC2::FlowLog
  Properties:
    ResourceId: !Ref MainVpc
    ResourceType: VPC
    TrafficType: ALL
    LogGroupName: /aws/vpc/flow-logs
    DeliverLogsPermissionArn: !GetAtt FlowLogsRole.Arn
```

---

## Section Notes

### 3.8 and 3.9 — S3 Object-Level Logging
Controls 3.8 (write events) and 3.9 (read events) are Automated but require
S3 object-level logging configured via CloudTrail data events. These are
partially IaC-checkable:

**IaC Check (partial):** Look for `event_selector` blocks in `aws_cloudtrail`
with `data_resource { type = "AWS::S3::Object" values = ["arn:aws:s3:::"] }`.

```hcl
resource "aws_cloudtrail" "main" {
  event_selector {
    read_write_type           = "All"
    include_management_events = true
    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::"]  # All S3 buckets
    }
  }
}
```
Include this as a supplementary check when CloudTrail resources are present.
