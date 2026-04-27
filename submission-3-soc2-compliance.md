=== BUIDL TITLE ===
SOC 2 Type II Readiness Accelerator for AWS Startups

=== DETAILS FIELD (the actual prompt) ===

You are an AWS compliance architect helping me prepare our AWS infrastructure for SOC 2 Type II audit. This prompt automates the deployment of 30+ security controls required by SOC 2 auditors.

## OBJECTIVE
Deploy production-grade SOC 2 Type II compliance controls on AWS in 2 hours instead of 2 months. This setup implements: centralized logging (CloudTrail + CloudWatch), access controls (IAM + MFA), encryption (KMS), vulnerability scanning (Inspector), config monitoring (AWS Config), and automated evidence collection for annual audits.

## WHY SOC 2 MATTERS FOR STARTUPS

Every B2B SaaS company selling to enterprises hits the "SOC 2 wall" at Series A. Without SOC 2 Type II certification:
- Enterprise deals blocked (procurement requires audit report)
- Security questionnaires take 40+ hours per customer
- Annual audit costs $30K-$80K
- 6-12 month lead time from start to certification

This prompt gives you 80% of the required controls in under 2 hours.

## SOC 2 TRUST SERVICE CRITERIA COVERAGE

This deployment addresses all 5 Trust Service Criteria (TSC):

1. **CC6.1 - Logical Access**: IAM policies, MFA enforcement
2. **CC6.2 - System Monitoring**: CloudWatch, GuardDuty, Config
3. **CC6.6 - Encryption**: KMS, TLS 1.2+, S3 encryption
4. **CC6.7 - Data Classification**: Tagging strategy
5. **CC7.2 - Change Management**: CloudFormation, change tracking

## ARCHITECTURE

**Audit Logging**: AWS CloudTrail → S3 (encrypted, 7-year retention)
**Monitoring**: CloudWatch Logs + GuardDuty + AWS Config
**Access Control**: IAM policies + MFA + access keys rotation
**Encryption**: AWS KMS (customer-managed keys)
**Vulnerability Scanning**: Amazon Inspector + AWS Systems Manager Patch Manager
**Evidence Collection**: Automated compliance reports via AWS Audit Manager

## STEP-BY-STEP DEPLOYMENT

### 1. Enable AWS Organizations (Multi-Account Setup)

```bash
aws organizations create-organization --feature-set ALL
aws organizations create-organizational-unit \
  --parent-id r-xxxx \
  --name Production
```

**Decision Point**: Use single account or multi-account? SOC 2 auditors prefer production workloads in separate AWS account from development.

### 2. Enable CloudTrail (Audit Logging)

Create S3 bucket with 7-year retention (SOC 2 requirement):

```bash
aws s3 mb s3://company-cloudtrail-logs-$(date +%s) \
  --region us-east-1

aws s3api put-bucket-versioning \
  --bucket company-cloudtrail-logs-xxxxx \
  --versioning-configuration Status=Enabled

aws s3api put-bucket-lifecycle-configuration \
  --bucket company-cloudtrail-logs-xxxxx \
  --lifecycle-configuration '{
    "Rules": [{
      "Id": "Retain7Years",
      "Status": "Enabled",
      "Transitions": [{
        "Days": 90,
        "StorageClass": "GLACIER"
      }],
      "Expiration": {"Days": 2555}
    }]
  }'
```

Enable CloudTrail organization-wide:

```bash
aws cloudtrail create-trail \
  --name org-audit-trail \
  --s3-bucket-name company-cloudtrail-logs-xxxxx \
  --is-organization-trail \
  --enable-log-file-validation \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/xxxxx

aws cloudtrail start-logging --name org-audit-trail
```

**SOC 2 Evidence**: CloudTrail logs prove "who did what, when" for all AWS API calls.

### 3. Enable AWS Config (Configuration Compliance)

```bash
aws configservice put-configuration-recorder \
  --configuration-recorder '{
    "name": "default",
    "roleARN": "arn:aws:iam::123456789012:role/ConfigRole",
    "recordingGroup": {
      "allSupported": true,
      "includeGlobalResources": true
    }
  }'

aws configservice put-delivery-channel \
  --delivery-channel '{
    "name": "default",
    "s3BucketName": "company-config-logs-xxxxx",
    "snsTopicARN": "arn:aws:sns:us-east-1:123456789012:config-alerts"
  }'

aws configservice start-configuration-recorder \
  --configuration-recorder-name default
```

Deploy SOC 2 compliance pack:

```bash
aws configservice put-conformance-pack \
  --conformance-pack-name soc2-compliance \
  --template-s3-uri s3://aws-configservice-conformance-packs/OperationalBestPracticesforSOC2.yaml
```

This enables 30+ Config Rules including:
- RDS encryption at rest
- S3 bucket public access blocked
- IAM password policy enforced
- MFA enabled for console access

### 4. Enforce IAM Security Controls

Create IAM password policy (CC6.1):

```bash
aws iam update-account-password-policy \
  --minimum-password-length 14 \
  --require-symbols \
  --require-numbers \
  --require-uppercase-characters \
  --require-lowercase-characters \
  --allow-users-to-change-password \
  --max-password-age 90 \
  --password-reuse-prevention 12
```

Enforce MFA for all IAM users:

```bash
aws iam put-user-policy \
  --user-name {{USERNAME}} \
  --policy-name EnforceMFA \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "DenyAllExceptMFASetup",
      "Effect": "Deny",
      "NotAction": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:GetUser",
        "iam:ListMFADevices"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {"aws:MultiFactorAuthPresent": "false"}
      }
    }]
  }'
```

### 5. Enable Encryption at Rest (CC6.6)

Create customer-managed KMS key:

```bash
aws kms create-key \
  --description "SOC2 encryption key" \
  --key-policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::123456789012:root"},
      "Action": "kms:*",
      "Resource": "*"
    }]
  }'

aws kms create-alias \
  --alias-name alias/soc2-encryption \
  --target-key-id xxxxx
```

Enforce S3 bucket encryption:

```bash
aws s3api put-bucket-encryption \
  --bucket production-data \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/xxxxx"
      },
      "BucketKeyEnabled": true
    }]
  }'
```

### 6. Enable Amazon GuardDuty (Threat Detection)

```bash
aws guardduty create-detector --enable

# Enable S3 protection
aws guardduty update-detector \
  --detector-id xxxxx \
  --data-sources '{
    "S3Logs": {"Enable": true}
  }'
```

Create CloudWatch alarm for GuardDuty findings:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name guardduty-critical-findings \
  --metric-name FindingCount \
  --namespace GuardDuty \
  --statistic Sum \
  --period 300 \
  --threshold 1 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:security-team
```

### 7. Enable Amazon Inspector (Vulnerability Scanning)

```bash
aws inspector2 enable \
  --resource-types EC2 ECR LAMBDA

# Auto-scan all EC2 instances and container images
aws inspector2 update-organization-configuration \
  --auto-enable '{
    "ec2": true,
    "ecr": true,
    "lambda": true
  }'
```

### 8. Enable AWS Security Hub (Centralized Findings)

```bash
aws securityhub enable-security-hub \
  --enable-default-standards

# Enable CIS AWS Foundations Benchmark
aws securityhub batch-enable-standards \
  --standards-subscription-requests '[{
    "StandardsArn": "arn:aws:securityhub:us-east-1::standards/cis-aws-foundations-benchmark/v/1.4.0"
  }]'
```

### 9. Deploy Audit Manager (Evidence Collection)

```bash
aws auditmanager create-assessment \
  --name "SOC2-2026-Audit" \
  --scope '{
    "awsAccounts": [{"id": "123456789012"}],
    "awsServices": [{"serviceName": "S3"}, {"serviceName": "RDS"}]
  }' \
  --framework-id arn:aws:auditmanager:us-east-1::framework/xxxxx-SOC2
```

This automatically collects evidence for 64 SOC 2 controls.

### 10. Create Compliance Dashboard

Deploy CloudWatch dashboard:

```bash
aws cloudwatch put-dashboard \
  --dashboard-name SOC2-Compliance \
  --dashboard-body file://dashboard-config.json
```

**dashboard-config.json** includes metrics for:
- MFA compliance rate
- Config rule violations
- GuardDuty critical findings
- Inspector vulnerability count
- Failed login attempts

### 11. Set Up Access Reviews (CC6.1 Requirement)

Create Lambda function for quarterly IAM access review:

```python
import boto3

def lambda_handler(event, context):
    iam = boto3.client('iam')
    users = iam.list_users()['Users']

    report = []
    for user in users:
        last_used = user.get('PasswordLastUsed')
        access_keys = iam.list_access_keys(UserName=user['UserName'])

        report.append({
            'Username': user['UserName'],
            'LastActivity': str(last_used),
            'AccessKeyCount': len(access_keys['AccessKeyMetadata']),
            'MFAEnabled': iam.list_mfa_devices(UserName=user['UserName'])['MFADevices']
        })

    # Send to security team
    sns = boto3.client('sns')
    sns.publish(
        TopicArn='arn:aws:sns:us-east-1:123456789012:access-review',
        Subject='Quarterly IAM Access Review',
        Message=str(report)
    )
```

Schedule with EventBridge:

```bash
aws events put-rule \
  --name quarterly-access-review \
  --schedule-expression "rate(90 days)"

aws events put-targets \
  --rule quarterly-access-review \
  --targets "Id"="1","Arn"="arn:aws:lambda:us-east-1:123456789012:function:access-review"
```

### 12. Document Incident Response Plan (CC7.3)

Create runbook in S3:

```bash
cat > incident-response-plan.md << 'EOF'
# Incident Response Plan

## Severity Levels
- P0: Data breach, unauthorized access to production
- P1: Service outage affecting customers
- P2: Security vulnerability discovered
- P3: Policy violation

## Response Procedure
1. **Detection**: GuardDuty / Security Hub alert triggers SNS
2. **Containment**: Isolate affected resources (revoke IAM credentials, quarantine EC2)
3. **Investigation**: Review CloudTrail logs, VPC Flow Logs
4. **Remediation**: Apply security patches, rotate credentials
5. **Documentation**: Log in Jira, update Audit Manager

## Contact List
- Security Team: security@company.com
- On-Call Engineer: oncall@company.com
- AWS Support: Enterprise plan required
EOF

aws s3 cp incident-response-plan.md s3://company-compliance-docs/
```

### 13. Validation & Audit Evidence

Generate compliance report:

```bash
# Check Config compliance
aws configservice describe-compliance-by-config-rule \
  --config-rule-names $(aws configservice describe-config-rules --query 'ConfigRules[*].ConfigRuleName' --output text)

# Export GuardDuty findings
aws guardduty get-findings \
  --detector-id xxxxx \
  --finding-ids $(aws guardduty list-findings --detector-id xxxxx --query 'FindingIds' --output text)

# Generate Security Hub compliance score
aws securityhub get-findings \
  --filters '{"ComplianceStatus": [{"Value": "FAILED", "Comparison": "EQUALS"}]}'
```

Audit Manager exports evidence automatically to S3 every 30 days.

=== CONTEXT & DOCUMENTATION FIELD ===

**Prerequisites**:
- AWS account with Organizations enabled
- AWS CLI v2 with AdministratorAccess
- Budget for compliance services (~$500-$1500/month)
- Understanding of SOC 2 Trust Service Criteria
- Engagement letter with SOC 2 auditor (Vanta, Drata, or Big4 firm)

**Use Case**:
This prompt is for B2B SaaS startups preparing for first SOC 2 Type II audit:
- Pre-Series A startups entering enterprise market
- Companies with first Fortune 500 customer requiring audit
- Engineering teams without dedicated security personnel
- Reducing audit prep time from 6 months → 2 months

**Expected Outcome**:
- CloudTrail enabled organization-wide with 7-year retention
- AWS Config monitoring 30+ compliance rules
- IAM password policy + MFA enforcement
- KMS encryption for all data at rest
- GuardDuty + Inspector threat detection
- Security Hub centralized dashboard
- Audit Manager evidence collection
- Incident response runbook
- **Estimated setup time**: 2 hours
- **Estimated monthly cost**: $500-$1500 (scales with usage)

**Troubleshooting Tips**:

1. **"Config rules showing non-compliant resources"**
   - Review specific violations in AWS Config dashboard
   - Use remediation actions to auto-fix (e.g., enable S3 encryption)
   - Document exceptions in Audit Manager

2. **"GuardDuty generating too many false positives"**
   - Suppress findings for known safe IPs (e.g., office network)
   - Create suppression rules in GuardDuty console
   - Review findings weekly, not real-time

3. **"Audit Manager not collecting evidence"**
   - Verify CloudTrail is enabled and logging to S3
   - Check IAM permissions for Audit Manager service role
   - Ensure assessment scope includes all production accounts

4. **"High costs (> $2000/month)"**
   - Disable GuardDuty in non-production accounts
   - Reduce Config rule evaluation frequency to 24 hours
   - Use S3 Glacier for CloudTrail logs after 90 days

5. **"Auditor requesting additional evidence"**
   - Export CloudTrail logs for specific date range
   - Generate Security Hub compliance report PDF
   - Provide Audit Manager assessment report

=== AWS SERVICES & BEST PRACTICES FIELD ===

**Services used**:
- AWS Organizations (multi-account management)
- AWS CloudTrail (audit logging)
- AWS Config (compliance monitoring)
- AWS KMS (encryption)
- Amazon GuardDuty (threat detection)
- Amazon Inspector (vulnerability scanning)
- AWS Security Hub (centralized findings)
- AWS Audit Manager (evidence collection)
- AWS IAM (access control)
- Amazon CloudWatch (monitoring)
- Amazon EventBridge (automation)
- AWS Lambda (custom automation)
- Amazon SNS (alerting)

**Well-Architected pillars addressed**:

1. **Security** (primary focus):
   - IAM least-privilege policies
   - MFA enforcement
   - KMS encryption at rest
   - TLS 1.2+ in transit
   - GuardDuty threat detection
   - Incident response automation

2. **Operational Excellence**:
   - Centralized logging (CloudTrail)
   - Automated compliance monitoring (Config)
   - Evidence collection (Audit Manager)
   - Runbook documentation

3. **Reliability**:
   - Multi-AZ CloudTrail delivery
   - S3 versioning for logs
   - 7-year retention for audit trail

4. **Cost Optimization**:
   - S3 Glacier for old logs
   - Config rules scoped to production
   - GuardDuty only in production accounts

**Security controls included** (SOC 2 TSC mapping):

**CC6.1 - Logical Access**:
- IAM password policy (14+ chars, 90-day expiration)
- MFA enforcement
- Access key rotation
- Quarterly access reviews

**CC6.2 - Monitoring**:
- CloudWatch Logs
- GuardDuty threat detection
- Inspector vulnerability scanning
- Config compliance monitoring

**CC6.6 - Encryption**:
- KMS customer-managed keys
- S3 encryption at rest
- TLS 1.2+ for all API calls

**CC6.7 - Data Classification**:
- Tagging strategy for sensitive data
- S3 bucket policies
- Resource-based access control

**CC7.2 - Change Management**:
- CloudFormation for infrastructure
- CloudTrail change tracking
- Config drift detection

**CC7.3 - Risk Mitigation**:
- Incident response plan
- GuardDuty automated alerts
- Security Hub risk scoring

**Cost optimization measures**:
- S3 Intelligent-Tiering for logs
- Config rules scoped to critical resources
- GuardDuty in production accounts only
- CloudWatch Logs retention tuning

**Estimated Monthly Cost**:
- CloudTrail: $5 (organization trail)
- AWS Config: $200-$500 (30+ rules × 10 resources)
- GuardDuty: $150-$400 (depends on data volume)
- Inspector: $50-$150 (EC2 + container scanning)
- Security Hub: $10
- Audit Manager: $100 (per assessment)
- CloudWatch Logs: $50-$200
- **Total: $565-$1,410/month** for typical startup infrastructure
