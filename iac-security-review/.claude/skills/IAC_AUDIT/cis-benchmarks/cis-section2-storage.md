# CIS AWS Foundations Benchmark v4.0.1 — Section 2: Storage
# Source: CIS Amazon Web Services Foundations Benchmark v4.0.1 (December 2024)
# Scope: Automated controls only | IaC-checkable only
# For /iac-audit — IaC Security Review Suite
# ─────────────────────────────────────────────────────────────────────────────
# Controls excluded from this file:
#   2.1.2 (Manual - MFA Delete), 2.1.3 (Manual - data classification),
#   2.2.4 (Manual - Multi-AZ decision)
# ─────────────────────────────────────────────────────────────────────────────

## [2.1.1] Ensure S3 Bucket Policy is set to deny HTTP requests
**Level:** L2 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** At the Amazon S3 bucket level, you can configure permissions
through a bucket policy making the objects accessible only through HTTPS.
Denying access over plain HTTP prevents data from being transmitted unencrypted
over the network.

**Rationale:** By default, Amazon S3 allows both HTTP and HTTPS requests. To
enforce HTTPS-only access, the bucket policy must include a `Deny` statement
on `s3:*` with a condition for `aws:SecureTransport: false`.

**IaC Check:** For every `aws_s3_bucket_policy` or `AWS::S3::BucketPolicy`,
verify the policy document contains a `Deny` effect with `aws:SecureTransport`
condition set to `false`. Flag any bucket that has a policy without this
condition, and any bucket that has no policy at all.

**Terraform pattern (PASS):**
```hcl
data "aws_iam_policy_document" "s3_https_only" {
  statement {
    sid     = "DenyHTTP"
    effect  = "Deny"
    actions = ["s3:*"]
    resources = [
      aws_s3_bucket.example.arn,
      "${aws_s3_bucket.example.arn}/*"
    ]
    principals { type = "*"; identifiers = ["*"] }
    condition {
      test     = "Bool"
      variable = "aws:SecureTransport"
      values   = ["false"]
    }
  }
}
resource "aws_s3_bucket_policy" "example" {
  bucket = aws_s3_bucket.example.id
  policy = data.aws_iam_policy_document.s3_https_only.json
}
```
**Terraform pattern (FAIL):** Bucket policy absent, or policy lacks
`aws:SecureTransport` deny condition.

**CFN pattern (PASS):**
```yaml
BucketPolicy:
  Type: AWS::S3::BucketPolicy
  Properties:
    Bucket: !Ref MyBucket
    PolicyDocument:
      Statement:
        - Sid: DenyHTTP
          Effect: Deny
          Principal: "*"
          Action: "s3:*"
          Resource:
            - !GetAtt MyBucket.Arn
            - !Sub "${MyBucket.Arn}/*"
          Condition:
            Bool:
              aws:SecureTransport: "false"
```

---

## [2.1.4] Ensure that S3 is configured with 'Block Public Access' settings
**Level:** L1 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** Amazon S3 Block Public Access provides settings for access
points, buckets, and accounts to help you manage public access to Amazon S3
resources. By default, new buckets, access points, and objects do not allow
public access. All four Block Public Access settings should be enabled.

**Rationale:** Amazon S3 Block Public Access settings override S3 permissions
to block public access to Amazon S3 resources. Enabling all four prevents
accidental or intentional public data exposure via bucket ACLs or policies.

**IaC Check:** Verify every `aws_s3_bucket_public_access_block` resource has
all four properties set to `true`. Flag any bucket without this resource
attached.

**Terraform pattern (PASS):**
```hcl
resource "aws_s3_bucket_public_access_block" "example" {
  bucket                  = aws_s3_bucket.example.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```
**Terraform pattern (FAIL):** Resource absent, or any property set to `false`.

**CFN pattern (PASS):**
```yaml
BucketPublicAccessBlock:
  Type: AWS::S3::Bucket
  Properties:
    PublicAccessBlockConfiguration:
      BlockPublicAcls: true
      BlockPublicPolicy: true
      IgnorePublicAcls: true
      RestrictPublicBuckets: true
```

---

## [2.2.1] Ensure that encryption-at-rest is enabled for RDS instances
**Level:** L1 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** Amazon RDS encrypted DB instances use the industry-standard
AES-256 encryption algorithm to encrypt your data on the server that hosts your
Amazon RDS DB instances. After your data is encrypted, Amazon RDS handles
authentication of access and decryption of your data transparently with a
minimal impact on performance.

**Rationale:** Databases often host sensitive, confidential, and personally
identifiable information (PII). Encryption at rest prevents data from being
read if the underlying storage layer is physically or logically accessed.

**IaC Check:** Verify all `aws_db_instance` and `aws_rds_cluster` resources have
`storage_encrypted = true`. Ideally also specify a `kms_key_id` (customer-managed
key) rather than relying on the AWS-managed default.

**Terraform pattern (PASS):**
```hcl
resource "aws_db_instance" "main" {
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn  # Preferred over default key
}
```
**Terraform pattern (FAIL):** `storage_encrypted = false` or property absent.

**CFN pattern (PASS):**
```yaml
DBInstance:
  Type: AWS::RDS::DBInstance
  Properties:
    StorageEncrypted: true
    KmsKeyId: !Ref RdsKmsKey
```

---

## [2.2.2] Ensure the Auto Minor Version Upgrade feature is enabled for RDS instances
**Level:** L1 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** Ensure that RDS database instances have the Auto Minor Version
Upgrade flag enabled in order to receive automatically minor engine upgrades
during the specified maintenance window. This way, RDS instances can get the
new features, bug fixes, and security patches for their database engines.

**Rationale:** AWS RDS will occasionally deprecate minor engine versions and
provide new ones for an upgrade. When the last version number within the release
is incremented, this is a minor upgrade. Minor upgrades often include security
patches. Disabling auto minor version upgrade creates an ongoing maintenance
burden and allows known CVEs to persist.

**IaC Check:** Verify all `aws_db_instance` resources have
`auto_minor_version_upgrade = true` (this is the default but should be
explicit). Flag any resource where it is explicitly set to `false`.

**Terraform pattern (PASS):**
```hcl
resource "aws_db_instance" "main" {
  auto_minor_version_upgrade = true  # Explicitly set; default is true
}
```
**Terraform pattern (FAIL):** `auto_minor_version_upgrade = false`.

**CFN pattern (PASS):**
```yaml
DBInstance:
  Type: AWS::RDS::DBInstance
  Properties:
    AutoMinorVersionUpgrade: true
```

---

## [2.2.3] Ensure that RDS instances are not publicly accessible
**Level:** L1 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** Ensure and verify that RDS database instances provisioned in
your AWS account do restrict unauthorized access in order to minimize security
risks. To restrict access to any publicly accessible RDS database instance,
you must disable the database Publicly Accessible flag and update the VPC
security group associated with the instance.

**Rationale:** Ensuring RDS instances are not publicly accessible prevents
direct database access from the internet. Databases should be accessible only
via application tier or bastion hosts within the VPC.

**IaC Check:** Verify all `aws_db_instance` and `aws_rds_cluster` resources
have `publicly_accessible = false`. Also check for placement in a private
subnet (via `db_subnet_group_name` referencing subnets with no IGW route).

**Terraform pattern (PASS):**
```hcl
resource "aws_db_instance" "main" {
  publicly_accessible = false  # Must be explicit
  db_subnet_group_name = aws_db_subnet_group.private.name
}
```
**Terraform pattern (FAIL):** `publicly_accessible = true` or absent without
subnet group verification.

**CFN pattern (PASS):**
```yaml
DBInstance:
  Type: AWS::RDS::DBInstance
  Properties:
    PubliclyAccessible: false
    DBSubnetGroupName: !Ref PrivateSubnetGroup
```

---

## [2.3.1] Ensure that encryption is enabled for EFS file systems
**Level:** L1 | **Assessment:** Automated | **IaC-checkable:** Yes

**Description:** Amazon Elastic File System (EFS) supports two forms of
encryption for file systems: encryption of data in transit, and encryption at
rest. It is recommended to configure both for all EFS file systems.

**Rationale:** Data should be encrypted at rest to reduce the risk of data
exposure in the event of unauthorized physical or logical access to the
underlying storage. EFS file systems are often mounted by multiple EC2
instances — plaintext storage means any instance with mount access can
read all data.

**IaC Check:** Verify all `aws_efs_file_system` resources have `encrypted = true`
and ideally specify a `kms_key_id`. For transit encryption, verify EFS mount
targets use `tls` in the mount options (this is typically enforced at the OS
level, not in IaC, but can be noted).

**Terraform pattern (PASS):**
```hcl
resource "aws_efs_file_system" "main" {
  encrypted  = true
  kms_key_id = aws_kms_key.efs.arn
}
```
**Terraform pattern (FAIL):** `encrypted = false` or property absent (defaults
to `false` if not set).

**CFN pattern (PASS):**
```yaml
EFSFileSystem:
  Type: AWS::EFS::FileSystem
  Properties:
    Encrypted: true
    KmsKeyId: !Ref EfsKmsKey
```
