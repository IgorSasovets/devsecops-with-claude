# CloudFormation-Specific Security Threats
# Reference for /iac-threat-model — IaC Security Review Suite
# Sources: AWS CFN Security Best Practices, AWS CFN Security Docs,
#          Cycode CFN Best Practices, AWS Security Blog
# Version: 1.0 | Framework: AWS CloudFormation | Scope: IaC-reviewable only
# ─────────────────────────────────────────────────────────────────────────────

## CFN-PARAM — Parameter Handling

| ID | STRIDE | Threat | Impact | CFN Anti-Pattern | Secure Pattern |
|----|--------|--------|--------|-----------------|----------------|
| CFN-PARAM-01 | Information Disclosure | Sensitive parameter value exposed in console / CloudTrail | Password or API key visible in stack events and describe-stacks output | `Type: String` for passwords without `NoEcho: true` | `Type: String; NoEcho: true` for all sensitive parameters |
| CFN-PARAM-02 | Information Disclosure | Hardcoded secrets in parameter `Default` value | Secret committed to template and logged in all stack events | `Default: "MyP@ssword123"` on a password parameter | No `Default` on sensitive params; use SSM or Secrets Manager dynamic references |
| CFN-PARAM-03 | Information Disclosure | Parameter value logged in Metadata or Outputs section | Bypasses `NoEcho` protection | `Outputs: DbPass: Value: !Ref DbPassword` where `DbPassword` has `NoEcho: true` | Never reference `NoEcho` parameters in Outputs or Metadata |
| CFN-PARAM-04 | Information Disclosure | Plaintext secret passed as parameter instead of dynamic reference | Secret stored in parameter store of calling tool/pipeline | Any `Type: String` parameter holding a secret value | Use `{{resolve:ssm-secure:/path/to/param}}` or `{{resolve:secretsmanager:...}}` dynamic references |
| CFN-PARAM-05 | Tampering | No `AllowedValues` constraint on parameters used in ARN/URL construction | Attacker with stack update permission injects arbitrary paths | `Type: String` parameter used directly in resource ARN without `AllowedValues` | `AllowedValues` or `AllowedPattern` constraint on any parameter used in resource construction |

---

## CFN-IAM — IAM Resource Misconfiguration

| ID | STRIDE | Threat | Impact | CFN Anti-Pattern | Secure Pattern |
|----|--------|--------|--------|-----------------|----------------|
| CFN-IAM-01 | Elevation of Privilege | IAM policy with `Action: "*"` or `Resource: "*"` | Full privilege escalation | `Effect: Allow; Action: "*"; Resource: "*"` in any inline or managed policy | Explicit least-privilege actions scoped to specific resource ARNs |
| CFN-IAM-02 | Elevation of Privilege | Role trust policy with unrestricted `Principal` | Any AWS identity can assume the role | `Principal: "*"` in `AssumeRolePolicyDocument` | Specific service or account ARN in `Principal` |
| CFN-IAM-03 | Elevation of Privilege | `CAPABILITY_IAM` / `CAPABILITY_NAMED_IAM` acknowledged without review | Stack creates privileged roles without audit | Automatic acknowledgment in CI pipelines | Require manual approval for stacks using IAM capabilities in production |
| CFN-IAM-04 | Elevation of Privilege | Lambda function resource policy allows invocation from `*` | Any principal (including unauthenticated) can invoke Lambda | `AWS::Lambda::Permission` with `Principal: "*"` | Restrict to specific service principal or account ARN |
| CFN-IAM-05 | Information Disclosure | IAM role without `PermissionsBoundary` where boundary policies are enforced | Privilege escalation bypasses guardrails | `AWS::IAM::Role` without `PermissionsBoundary` in regulated environment | `PermissionsBoundary: !Sub arn:aws:iam::${AWS::AccountId}:policy/boundary` |

---

## CFN-STACK — Stack and Template Security

| ID | STRIDE | Threat | Impact | CFN Anti-Pattern | Secure Pattern |
|----|--------|--------|--------|-----------------|----------------|
| CFN-STACK-01 | Tampering | Nested stack template fetched from public / mutable S3 URL | Attacker modifies S3 object to change deployed infrastructure | `TemplateURL` pointing to `https://s3.amazonaws.com/public-bucket/template.yaml` | Private S3 bucket with versioning; use S3 object version in URL |
| CFN-STACK-02 | Tampering | Stack policy absent — any IAM admin can update/delete any resource | Accidental or malicious stack modification | No `StackPolicy` on production stacks | Apply stack policy denying `Update:*` on critical resources |
| CFN-STACK-03 | Repudiation | Stack termination protection disabled | Accidental stack deletion destroys all resources | `EnableTerminationProtection: false` (default) | `EnableTerminationProtection: true` for production stacks |
| CFN-STACK-04 | Information Disclosure | Stack Outputs expose sensitive values without `NoEcho` | Outputs visible to any principal with `cloudformation:DescribeStacks` | `Outputs: ApiKey: Value: !GetAtt ...` for a sensitive value | Use SSM/Secrets Manager for sensitive values; only output ARNs/IDs |
| CFN-STACK-05 | Tampering | Cross-stack export used as input without validation | Downstream stack uses an export that can be manipulated by another team | `Fn::ImportValue` used directly in sensitive resource property | Validate imported values; use Conditions to gate on expected format |
| CFN-STACK-06 | Denial of Service | DeletionPolicy not set on stateful resources | Resources deleted when stack is deleted or resource removed | No `DeletionPolicy: Retain` or `Snapshot` on RDS, S3, DynamoDB | `DeletionPolicy: Retain` on databases; `Snapshot` where supported |
| CFN-STACK-07 | Tampering | UpdateReplacePolicy not set — update causes replacement | Resource replacement on stack update destroys production data | No `UpdateReplacePolicy: Retain` on stateful resources | `UpdateReplacePolicy: Retain` on databases and critical storage |

---

## CFN-NET — Network and Exposure

| ID | STRIDE | Threat | Impact | CFN Anti-Pattern | Secure Pattern |
|----|--------|--------|--------|-----------------|----------------|
| CFN-NET-01 | Information Disclosure | Security Group with `CidrIp: 0.0.0.0/0` on admin ports | Remote access to admin services from any IP | `SecurityGroupIngress: CidrIp: 0.0.0.0/0; FromPort: 22` | Restrict to bastion/VPN CIDR or use Systems Manager |
| CFN-NET-02 | Information Disclosure | RDS instance with `PubliclyAccessible: true` | Database directly reachable from internet | `AWS::RDS::DBInstance: PubliclyAccessible: true` | `PubliclyAccessible: false`; place in private subnet |
| CFN-NET-03 | Information Disclosure | ELB/ALB without `AccessLoggingPolicy` | No visibility into load balancer traffic | `AWS::ElasticLoadBalancingV2::LoadBalancer` without access log attribute | Enable access logging to dedicated S3 bucket |
| CFN-NET-04 | Information Disclosure | CloudFront distribution without WAF association | Web application exposed without Layer 7 protection | `AWS::CloudFront::Distribution` without `WebACLId` | Associate `AWS::WAFv2::WebACL` with CloudFront distribution |

---

## CFN-CRYPTO — Encryption

| ID | STRIDE | Threat | Impact | CFN Anti-Pattern | Secure Pattern |
|----|--------|--------|--------|-----------------|----------------|
| CFN-CRYPTO-01 | Information Disclosure | S3 bucket without default encryption | Objects stored in plaintext | `AWS::S3::Bucket` without `BucketEncryption` property | `BucketEncryption: ServerSideEncryptionConfiguration` with `aws:kms` |
| CFN-CRYPTO-02 | Information Disclosure | RDS instance without `StorageEncrypted: true` | Database data at rest in plaintext | `AWS::RDS::DBInstance` with `StorageEncrypted: false` or absent | `StorageEncrypted: true; KmsKeyId: !Ref MyKmsKey` |
| CFN-CRYPTO-03 | Information Disclosure | EBS volume without `Encrypted: true` | Volume data at rest in plaintext | `AWS::EC2::Volume` with `Encrypted: false` or absent | `Encrypted: true; KmsKeyId: !Ref MyKmsKey` |
| CFN-CRYPTO-04 | Information Disclosure | SQS queue without `KmsMasterKeyId` | Queue messages stored in plaintext | `AWS::SQS::Queue` without `KmsMasterKeyId` | `KmsMasterKeyId: !Ref MyKmsKey` |
| CFN-CRYPTO-05 | Information Disclosure | SNS topic without `KmsMasterKeyId` | Topic messages stored in plaintext | `AWS::SNS::Topic` without `KmsMasterKeyId` | `KmsMasterKeyId: !Ref MyKmsKey` |
| CFN-CRYPTO-06 | Information Disclosure | CloudTrail without KMS encryption | API audit logs stored in plaintext S3 | `AWS::CloudTrail::Trail` without `KMSKeyId` | `KMSKeyId: !Ref CloudTrailKmsKey` |

---

## CFN-LOG — Logging

| ID | STRIDE | Threat | Impact | CFN Anti-Pattern | Secure Pattern |
|----|--------|--------|--------|-----------------|----------------|
| CFN-LOG-01 | Repudiation | CloudTrail not enabled for all regions | API calls in non-primary regions go unlogged | `AWS::CloudTrail::Trail` with `IsMultiRegionTrail: false` | `IsMultiRegionTrail: true; IncludeGlobalServiceEvents: true` |
| CFN-LOG-02 | Repudiation | CloudTrail log file validation disabled | Tampered log files cannot be detected | `AWS::CloudTrail::Trail` with `EnableLogFileValidation: false` | `EnableLogFileValidation: true` |
| CFN-LOG-03 | Repudiation | VPC Flow Logs not configured | Network traffic not logged | No `AWS::EC2::FlowLog` resource in VPC stack | `AWS::EC2::FlowLog` attached to each VPC with REJECT or ALL traffic |
| CFN-LOG-04 | Repudiation | S3 server access logging disabled on CloudTrail log bucket | Data access to logs not audited | `AWS::S3::Bucket` (log bucket) without `LoggingConfiguration` | Enable server access logging on the log bucket itself |
