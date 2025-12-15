# Landing Zone Accelerator on AWS for Healthcare - Comprehensive Deployment Guide

This document provides a complete, step-by-step guide for deploying the Landing Zone Accelerator (LZA) for Healthcare on AWS. It consolidates all prerequisites, installation steps, configuration details, and post-deployment activities into a single reference.

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Deployment Steps](#deployment-steps)
5. [Configuration Reference](#configuration-reference)
6. [Post-Deployment Configuration](#post-deployment-configuration)
7. [Validation and Testing](#validation-and-testing)
8. [Troubleshooting](#troubleshooting)
9. [Cost Considerations](#cost-considerations)
10. [Maintenance and Updates](#maintenance-and-updates)

---

## Overview

### What is the LZA for Healthcare?

The Landing Zone Accelerator (LZA) for Healthcare is an industry-specific deployment of the [Landing Zone Accelerator on AWS](https://aws.amazon.com/solutions/implementations/landing-zone-accelerator-on-aws/) solution. It provides a secure, compliant, multi-account AWS foundation tailored for healthcare organizations subject to HIPAA compliance requirements.

### Key Capabilities

The solution delivers the following capabilities out of the box:

**HIPAA Compliance Enforcement**: Service Control Policies (SCPs) restrict AWS accounts to only the 202 HIPAA-eligible services, preventing accidental use of non-compliant services in accounts processing Protected Health Information (PHI).

**Multi-Account Isolation**: Organizational Units (OUs) segregate healthcare workloads containing PHI (HIS - Healthcare Information System) from general business workloads (EIS - Executive Information System), enabling different security postures for different data sensitivity levels.

**Automated Security Controls**: Detective guardrails (Security Hub, GuardDuty, AWS Config), preventative controls (SCPs, S3 public access blocks), and automated remediation (SSM Automation documents) deploy across all accounts automatically.

**Centralized Logging and Monitoring**: All logs (CloudTrail, VPC Flow Logs, Session Manager) route to a dedicated LogArchive account with configurable retention policies and cost-optimized lifecycle management.

**Network Segmentation**: Hub-and-spoke Transit Gateway architecture with optional traffic inspection, centralized VPC endpoints for cost optimization, and VPC-level isolation.

**Compliance Reporting**: Integration with Security Hub standards (AWS Foundational Security Best Practices, CIS Benchmarks v1.4.0, NIST 800-53 Rev 5) and CloudWatch alarms aligned with CIS compliance requirements.

### Important Disclaimer

The Landing Zone Accelerator solution will not, by itself, make you compliant. It provides the foundational infrastructure from which additional complementary solutions can be integrated. You must review, evaluate, assess, and approve the solution in compliance with your organization's particular security features, tools, and configurations.

---

## Architecture

### Organizational Structure

The LZA for Healthcare creates the following organizational structure:

```
Root
├── Management (Account)
├── Security (OU)
│   ├── LogArchive (Account)
│   └── Audit (Account)
├── Infrastructure (OU)
│   ├── Infra-Dev (OU)
│   └── Infra-Prod (OU)
│       └── Network-Prod (Account)
├── HIS (OU) - Healthcare Information System
│   └── HIS-Non-Prod (OU)
│       └── HIS-Dev (Account)
└── EIS (OU) - Executive Information System
    └── EIS-Prod (OU)
        └── EIS-Prod (Account)
```

### Account Purposes

| Account | Purpose |
|---------|---------|
| **Management** | Root account controlling the AWS Organization. Hosts billing consolidation and global configuration management. |
| **LogArchive** | Centralized logging destination for CloudTrail, VPC Flow Logs, and other audit logs. |
| **Audit** | Security services hub hosting delegated administration for GuardDuty, Security Hub, and Config aggregator. |
| **Network-Prod** | Shared networking services including Transit Gateway, Network-Inspection VPC, and Network-Endpoints VPC. |
| **HIS-Dev** | Development environment for Healthcare Information System workloads processing PHI. |
| **EIS-Prod** | Production environment for Executive Information System (general business) workloads. |

### Network Architecture

The solution implements a hub-and-spoke Transit Gateway architecture:

```
                    ┌─────────────────────────────────────────┐
                    │           Network-Prod Account          │
                    │                                         │
                    │  ┌─────────────────────────────────┐   │
                    │  │      Network-Main (TGW)         │   │
                    │  │         ASN: 65521              │   │
                    │  └─────────────────────────────────┘   │
                    │           │              │              │
                    │  ┌────────┴────┐  ┌─────┴──────┐      │
                    │  │  Network-   │  │  Network-  │      │
                    │  │  Endpoints  │  │ Inspection │      │
                    │  │ 10.1.0.0/22 │  │ 10.2.0.0/22│      │
                    │  └─────────────┘  └────────────┘      │
                    └─────────────────────────────────────────┘
                                       │
            ┌──────────────────────────┼──────────────────────────┐
            │                          │                          │
    ┌───────┴───────┐          ┌───────┴───────┐          ┌───────┴───────┐
    │   HIS-Dev     │          │   EIS-Prod    │          │    Future     │
    │  Account      │          │   Account     │          │   Accounts    │
    │ 10.4.0.0/16   │          │ 10.3.0.0/16   │          │               │
    └───────────────┘          └───────────────┘          └───────────────┘
```

**Network-Endpoints VPC (10.1.0.0/22)**: Hosts centralized interface endpoints (EC2, SSM, KMS, CloudWatch Logs) shared across the organization via Transit Gateway.

**Network-Inspection VPC (10.2.0.0/22)**: Designed for traffic inspection with Internet Gateway and dedicated firewall subnets. Note: Actual inspection appliances are NOT provisioned by default and must be deployed separately.

**Workload VPCs**: HIS-Dev (10.4.0.0/16) and EIS-Prod (10.3.0.0/16) connect to the Transit Gateway and can be configured to route traffic through the inspection VPC.

### CIDR Allocation

| CIDR Range | Purpose | Notes |
|------------|---------|-------|
| 10.1.0.0/22 | Network-Endpoints | Centralized VPC interface endpoints |
| 10.2.0.0/22 | Network-Inspection | Traffic inspection VPC |
| 10.3.0.0/16 | EIS-Prod | Executive Information System production |
| 10.4.0.0/16 | HIS-Dev | Healthcare Information System development |
| 10.5.0.0 - 10.255.255.255 | Reserved | Available for additional workload VPCs |

---

## Prerequisites

### 1. Foundational Knowledge

Before deploying, ensure you have foundational knowledge of the Landing Zone Accelerator on AWS (LZA). If you are not familiar with LZA, complete the [LZA Immersion Day Workshop](https://catalog.workshops.aws/landing-zone-accelerator/en-US) prior to deploying this reference architecture.

### 2. Email Addresses

You need **11 unique email addresses** for the deployment:

#### Account Email Addresses (6 required)

| Account | Purpose | Example |
|---------|---------|---------|
| Management Account | Root account for AWS Organizations | lz-dev+management@example.com |
| Audit Account | Security services hub (delegated admin) | lz-dev+security@example.com |
| Log Archive Account | Centralized logging destination | lz-dev+log-archive@example.com |
| Network Prod Account | Shared networking services | lz-dev+network-prod@example.com |
| HIS Dev Account | Healthcare workloads (PHI) | lz-dev+his-dev@example.com |
| EIS Prod Account | Business workloads | lz-dev+eis-prod@example.com |

#### Notification Email Addresses (5 required)

| Name | Purpose | Example |
|------|---------|---------|
| High Security Events | CRITICAL/HIGH severity alerts | lz-dev+security-alerts-high@example.com |
| Medium Security Events | MEDIUM severity alerts | lz-dev+security-alerts-medium@example.com |
| Low Security Events | LOW severity alerts | lz-dev+security-alerts-low@example.com |
| AWS Budgets Alerts | Spending limit notifications | lz-dev+budget-alerts@example.com |
| Pipeline Approval | LZA pipeline approval notifications | lz-dev+pipeline-approval@example.com |

**Tip**: Many email providers support the "+" addressing format (e.g., yourname+alias@domain.com), allowing you to use a single mailbox for multiple addresses.

### 3. AWS Account Setup

#### 3.1 Create Management Account

Create a new AWS Account for the reference architecture deployment following the [AWS guidance on setting up a new AWS account](https://aws.amazon.com/resources/create-account/).

#### 3.2 Pre-warm the Account

Launch a small EC2 instance (t3.medium) in the management account for 15 minutes to trigger AWS to automatically set the correct service quotas. After 15 minutes, terminate the instance and delete its security group.

#### 3.3 Configure Account Contacts

Set security, operations, and billing contact information following the guidance on [configuring account level contacts](https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-startup-security-baseline/acct-01.html).

#### 3.4 Verify Root User Email

Confirm the root user email address by clicking the verification link sent during account creation.

#### 3.5 Enable MFA on Root User

Configure hardware MFA on the root user following the guidance on [requiring MFA to login](https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-startup-security-baseline/acct-05.html). Review guidance on [restricting the use of the root user](https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-startup-security-baseline/acct-02.html).

#### 3.6 Create IAM Deployment User

Create a local IAM user for deployment activities:
1. [Configure console access for each user](https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-startup-security-baseline/acct-03.html)
2. [Require MFA to login](https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-startup-security-baseline/acct-05.html)

### 4. Service Quotas

#### 4.1 AWS Organizations Account Limit

If your deployment will have more than 10 AWS accounts, request an increase via the [Service Quota Console for AWS Organizations](https://console.aws.amazon.com/servicequotas/home?region=us-east-1#!/services/organizations/quotas).

#### 4.2 Lambda Concurrent Executions

Verify that the Lambda concurrent executions quota is at least 1000 in the Service Quota Console (Service = Lambda). If it shows "not available," ensure you completed the EC2 pre-warming step. Request an increase if needed.

#### 4.3 CodeBuild Concurrency

Follow the prerequisite step to [update AWS CodeBuild concurrency quota](https://docs.aws.amazon.com/solutions/latest/landing-zone-accelerator-on-aws/prerequisites#update-codebuild-conncurrency-quota).

### 5. Region Accessibility

Ensure your home AWS Region is accessible by following the prerequisite step to [ensure your global region is accessible](https://docs.aws.amazon.com/solutions/latest/landing-zone-accelerator-on-aws/prerequisites#ensure-your-global-region-is-accessible).

### 6. Enable AWS Cost Explorer

Enable AWS Cost Explorer following the [AWS Cost Explorer documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-enable.html).

### 7. GitHub Token (Optional)

If using GitHub as the source for LZA code, [store a GitHub token in AWS Secrets Manager](https://docs.aws.amazon.com/solutions/latest/landing-zone-accelerator-on-aws/prerequisites.html#create-a-github-personal-access-token-and-store-in-secrets-manager).

---

## Deployment Steps

### Phase 1: Deploy Landing Zone Accelerator

#### Step 1.1: Enable AWS Organizations

Navigate to the [AWS Organizations console](https://console.aws.amazon.com/organizations/v2) in your home region and choose **Create an organization**. AWS automatically sends a verification email to the management account email address.

#### Step 1.2: Deploy the Installer CloudFormation Stack

1. Click the **Launch Solution** button on [Step 1. Launch the stack](https://docs.aws.amazon.com/solutions/latest/landing-zone-accelerator-on-aws/step-1.-launch-the-stack.html).

2. **Important**: Ensure the Region is set to your desired home Region (it typically defaults to US East N. Virginia).

3. Name the stack `AWSAccelerator-InstallerStack`.

4. Configure the following parameters:

| Parameter | Value |
|-----------|-------|
| Manual Approval Stage notification email list | Pipeline Approval email from prerequisites |
| Management Account Email | Management Account email from prerequisites |
| Log Archive Account Email | Log Archive Account email from prerequisites |
| Audit Account Email | Audit Account email from prerequisites |
| Control Tower Environment | Yes |
| Configuration Repository Location | S3 (recommended for new accounts) or CodeCommit |

**Note**: AWS CodeCommit is no longer available to new customers. If the management account is a new AWS account, select S3 as the configuration repository location.

5. Leave all other values as default unless you have specific customization requirements.

6. Acknowledge the IAM capabilities and create the stack.

#### Step 1.3: Wait for Initial Pipeline Completion

Monitor the `AWSAccelerator-Pipeline` in the AWS CodePipeline console. Wait for successful completion of the initial deployment. This typically takes 60-90 minutes.

#### Step 1.4: Verify Control Tower Deployment

After Control Tower deployment completes, verify you have:
- Security OU with LogArchive and Audit accounts
- Infrastructure OU

#### Step 1.5: Disable Control Tower VPC Creation

1. Navigate to Control Tower Account Factory
2. Edit the Network configuration:
   - Set Maximum number of private subnets to **0**
   - Uncheck all regions for VPC creation (VPC creation will be handled by LZA)

### Phase 2: Deploy Healthcare Reference Architecture

#### Step 2.1: Download Configuration Files

Clone or download the LZA for Healthcare configuration files from this repository.

#### Step 2.2: Customize Configuration Files

Using your preferred IDE, customize the following configuration files:

##### replacements-config.yaml (Critical)

This file contains global variables referenced by all other configuration files:

```yaml
globalReplacements:
  # Home Region - Update to your deployment region
  - key: AcceleratorHomeRegion
    type: String
    value: us-east-1  # Change to your home region
  
  # Security notification emails - Update all placeholders
  - key: SecurityHigh
    type: String
    value: your-security-high@example.com
  - key: SecurityMedium
    type: String
    value: your-security-medium@example.com
  - key: SecurityLow
    type: String
    value: your-security-low@example.com
  
  # Budget notification email
  - key: BudgetEmail
    type: String
    value: your-budget@example.com
```

##### accounts-config.yaml (Critical)

Update all email addresses to match your prerequisites:

```yaml
mandatoryAccounts:
  - name: Management
    email: your-management@example.com  # Update
    organizationalUnit: Root
  - name: LogArchive
    email: your-log-archive@example.com  # Update
    organizationalUnit: Security
  - name: Audit
    email: your-audit@example.com  # Update
    organizationalUnit: Security

workloadAccounts:
  - name: Network-Prod
    email: your-network-prod@example.com  # Update
    organizationalUnit: Infrastructure/Infra-Prod
  - name: HIS-Dev
    email: your-his-dev@example.com  # Update
    organizationalUnit: HIS/HIS-Non-Prod
  - name: EIS-Prod
    email: your-eis-prod@example.com  # Update
    organizationalUnit: EIS/EIS-Prod
```

##### Changing the Home Region

If deploying to a region other than us-east-1, update the following files:

**global-config.yaml**:
```yaml
homeRegion: &HOME_REGION us-west-2  # Your region
```

**network-config.yaml**:
```yaml
homeRegion: &HOME_REGION us-west-2  # Your region
```

**security-config.yaml**:
```yaml
homeRegion: &HOME_REGION us-west-2  # Your region
```

Additionally, comment out or remove these AWS Config rules that are only available in us-east-1:
- `cloudfront-origin-access-identity-enabled`
- `cloudfront-security-policy-check`
- `shield-drt-access`

##### Changing the Accelerator Prefix

If you changed the accelerator prefix from `AWSAccelerator` during deployment, update **dynamic-partitioning/log-filters.json**:

```json
[
  { "logGroupPattern": "/YOUR-PREFIX-SecurityHub", "s3Prefix": "security-hub" }
]
```

#### Step 2.3: Upload Configuration Files

**For S3 Configuration Repository**:
1. Compress all configuration files into `aws-accelerator-config.zip`
2. Upload to the S3 bucket: `aws-accelerator-config-<ACCOUNT_ID>-<REGION>`

**For CodeCommit Configuration Repository**:
1. Clone the `aws-accelerator-config` repository
2. Replace the default configuration files with the healthcare configuration files
3. Commit and push the changes

#### Step 2.4: Run the Pipeline

1. Navigate to the AWS CodePipeline console
2. Select `AWSAccelerator-Pipeline`
3. Click **Release change**
4. Wait for successful completion (typically 2-4 hours for full deployment)

---

## Configuration Reference

### Configuration File Overview

| File | Purpose |
|------|---------|
| **replacements-config.yaml** | Global variables injected into other configs |
| **accounts-config.yaml** | AWS account structure and OU assignments |
| **global-config.yaml** | Control Tower, CloudTrail, logging, budgets |
| **network-config.yaml** | Transit Gateway, VPCs, subnets, endpoints |
| **security-config.yaml** | Security Hub, GuardDuty, Config rules, alarms |
| **organization-config.yaml** | SCPs, tagging policies, backup policies |
| **iam-config.yaml** | IAM roles, policies, groups, users |

### Security Controls Deployed

#### Service Control Policies

| Policy | Purpose | Applied To |
|--------|---------|------------|
| scp-hlc-base-root.json | Prevents accounts from leaving organization, disabling S3 public access blocks | Root OU |
| scp-hlc-hipaa-service.json | Restricts to 202 HIPAA-eligible services only | HIS OU |
| quarantine.json | Blocks all actions in newly created accounts | New accounts |

#### Security Hub Standards

- AWS Foundational Security Best Practices v1.0.0
- CIS AWS Foundations Benchmark v1.4.0
- NIST Special Publication 800-53 Revision 5

#### AWS Config Rules

The deployment includes 60+ AWS Config rules covering:
- EC2 instance profile requirements
- S3 encryption and HTTPS enforcement
- ELB logging requirements
- Backup compliance
- IAM policy restrictions
- Network security

#### CloudWatch Alarms (CIS-Aligned)

| Alarm | Monitors |
|-------|----------|
| RootAccountUsage | Root account login activity |
| IAMPolicyChanges | IAM policy modifications |
| CloudTrailChanges | CloudTrail configuration changes |
| ConsoleAuthenticationFailure | Failed console login attempts |
| DisableOrDeleteCMK | KMS key deletion/disabling |
| S3BucketPolicyChanges | S3 bucket policy modifications |
| AWSConfigChanges | AWS Config configuration changes |
| SecurityGroupChanges | Security group modifications |
| NetworkACLChanges | Network ACL modifications |
| NetworkGatewayChanges | Internet/NAT gateway changes |
| RouteTableChanges | Route table modifications |
| VPCChanges | VPC configuration changes |
| BreakglassAccountUsage | Breakglass account activity |

### Network Configuration Details

#### Transit Gateway Route Tables

| Route Table | Purpose |
|-------------|---------|
| Network-Main-Core | Core infrastructure routing |
| Network-Main-Segregated | Forces traffic through inspection VPC (0.0.0.0/0 -> Network-Inspection) |
| Network-Main-Shared | Shared services routing |
| Network-Main-Standalone | Isolated workload routing |

#### VPC Endpoints (Centralized)

The Network-Endpoints VPC hosts centralized interface endpoints:
- ec2
- ec2messages
- ssm
- ssmmessages
- kms
- logs (CloudWatch Logs)

Workload VPCs consume these endpoints via Transit Gateway instead of creating duplicates.

---

## Post-Deployment Configuration

### 1. Configure AWS IAM Identity Center

#### 1.1 Delegate Administration (Optional)

To delegate Identity Center administration to a member account:

1. Update `iam-config.yaml` to include the `identityCenter` class:
```yaml
identityCenter:
  name: IdentityCenter
  delegatedAdminAccount: Audit  # Or your preferred account
```

2. Upload the updated configuration and release the pipeline
3. Wait for successful completion

#### 1.2 Configure Identity Source

1. Log into the Identity Center administration account
2. Choose your identity source:
   - **AWS Directory**: Follow [IAM Identity Center guidance to configure AD](https://docs.aws.amazon.com/singlesignon/latest/userguide/connectawsad.html)
   - **External IdP**: Follow [IAM Identity Center guidance to configure an external identity provider](https://docs.aws.amazon.com/singlesignon/latest/userguide/manage-your-identity-source-idp.html)

#### 1.3 Configure Permission Sets and Assignments

1. Update `iam-config.yaml` with `identityCenterPermissionSets` and `identityCenterAssignments`
2. Upload the updated configuration and release the pipeline

### 2. Configure Multi-Factor Authentication

#### 2.1 Identity Center MFA

Follow guidance on [Multi-factor authentication for Identity Center users](https://docs.aws.amazon.com/singlesignon/latest/userguide/enable-mfa.html).

Recommended settings:
- Every time they sign in (always-on)
- Security key and built-in authenticators
- Authenticator apps
- Require one-time password sent by email to sign in
- Users can add and manage their own MFA devices

#### 2.2 Breakglass User MFA

The breakglass users (breakGlassUser01, breakGlassUser02) are highly privileged accounts. Configure hardware MFA on both accounts following the [AWS IAM documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa_enable.html).

### 3. Enable Security Controls in All Regions

The initial installation only enables the home region. To deploy guardrails across all default AWS regions:

1. Edit `global-config.yaml`:
```yaml
enabledRegions:
  - *HOME_REGION
  - "ap-northeast-1"
  - "ap-northeast-2"
  - "ap-northeast-3"
  - "ap-south-1"
  - "ap-southeast-1"
  - "ap-southeast-2"
  - "eu-central-1"
  - "eu-north-1"
  - "eu-west-1"
  - "eu-west-2"
  - "eu-west-3"
  - "sa-east-1"
  - "us-east-2"
  - "us-west-1"
  - "us-west-2"
```

2. Upload the updated configuration and release the pipeline

### 4. Deploy Network Inspection Appliances

The Network-Inspection VPC is provisioned but does NOT include actual firewall/IDS appliances. To enable traffic inspection:

1. Deploy [AWS Network Firewall](https://aws.amazon.com/network-firewall/) or a third-party appliance in the Network-Inspection VPC
2. Update route tables to direct traffic through the inspection appliance
3. Configure firewall rules according to your security requirements

---

## Validation and Testing

### 1. Verify Account Creation

1. Navigate to AWS Organizations in the Management account
2. Verify all accounts are created and in the correct OUs:
   - Management (Root)
   - LogArchive (Security)
   - Audit (Security)
   - Network-Prod (Infrastructure/Infra-Prod)
   - HIS-Dev (HIS/HIS-Non-Prod)
   - EIS-Prod (EIS/EIS-Prod)

### 2. Verify Security Services

1. **Security Hub**: Navigate to Security Hub in the Audit account and verify:
   - All standards are enabled
   - Findings are being aggregated from member accounts

2. **GuardDuty**: Navigate to GuardDuty in the Audit account and verify:
   - GuardDuty is enabled across all accounts
   - S3 protection is enabled

3. **AWS Config**: Navigate to AWS Config in any account and verify:
   - Config rules are deployed
   - Compliance status is being evaluated

### 3. Verify Networking

1. **Transit Gateway**: Navigate to VPC > Transit Gateways in the Network-Prod account and verify:
   - Transit Gateway is created
   - VPC attachments are present for all VPCs

2. **VPC Endpoints**: Navigate to VPC > Endpoints in the Network-Prod account and verify:
   - Centralized endpoints are created in Network-Endpoints VPC

3. **Connectivity Test**: Launch an EC2 instance in HIS-Dev or EIS-Prod VPC and verify:
   - Instance can reach centralized endpoints via Transit Gateway
   - Systems Manager Session Manager connectivity works

### 4. Verify Logging

1. **CloudTrail**: Navigate to CloudTrail in the Management account and verify:
   - Organization trail is enabled
   - Logs are being delivered to the LogArchive account S3 bucket

2. **VPC Flow Logs**: Navigate to VPC > Flow Logs and verify:
   - Flow logs are enabled for all VPCs
   - Logs are being delivered to CloudWatch Logs and S3

### 5. Verify SCPs

1. Navigate to AWS Organizations > Policies > Service control policies
2. Verify the following SCPs are attached:
   - scp-hlc-base-root.json (Root OU)
   - scp-hlc-hipaa-service.json (HIS OU)

3. Test SCP enforcement by attempting to use a non-HIPAA-eligible service in an HIS account (should be denied)

---

## Troubleshooting

### Common Issues

#### Pipeline Fails at Account Creation Stage

**Symptom**: Pipeline fails when creating new AWS accounts.

**Possible Causes**:
- Email addresses already associated with existing AWS accounts
- AWS Organizations account limit reached
- Email verification pending

**Resolution**:
1. Verify all email addresses are unique and not associated with existing accounts
2. Check and increase AWS Organizations account quota if needed
3. Verify all email addresses have been confirmed

#### Pipeline Fails at Network Stage

**Symptom**: Pipeline fails during VPC or Transit Gateway creation.

**Possible Causes**:
- CIDR conflicts with existing VPCs
- VPC quota limits reached
- Elastic IP quota limits reached

**Resolution**:
1. Review CIDR ranges in network-config.yaml for conflicts
2. Check and increase VPC and Elastic IP quotas if needed

#### Security Hub Findings Not Aggregating

**Symptom**: Security Hub in Audit account doesn't show findings from member accounts.

**Possible Causes**:
- Delegated administrator not properly configured
- Member accounts not enabled

**Resolution**:
1. Verify Audit account is set as delegated administrator
2. Check that all member accounts are enabled in Security Hub

#### Config Rules Showing Non-Compliant

**Symptom**: AWS Config rules show resources as non-compliant.

**Possible Causes**:
- Resources created before LZA deployment
- Resources created outside of LZA management

**Resolution**:
1. Review non-compliant resources and remediate as needed
2. Use SSM Automation documents for automated remediation where available

### Pipeline Troubleshooting

1. **View Pipeline Logs**: Navigate to CodePipeline > AWSAccelerator-Pipeline > View details on failed stage

2. **View CodeBuild Logs**: Click on the failed action to view detailed CodeBuild logs

3. **Check CloudFormation Stacks**: Navigate to CloudFormation in the affected account and review stack events for failures

### Getting Help

1. Review the [LZA Troubleshooting Guide](https://docs.aws.amazon.com/solutions/latest/landing-zone-accelerator-on-aws/troubleshooting.html)
2. Check the [LZA GitHub Issues](https://github.com/awslabs/landing-zone-accelerator-on-aws/issues)
3. Contact AWS Support if you have a support plan

---

## Cost Considerations

### Estimated Monthly Costs

The following costs are estimates for running the LZA for Healthcare in US East (N. Virginia) with no active workloads:

| AWS Service | Estimated Monthly Cost |
|-------------|----------------------|
| Amazon VPC (Transit Gateway, NAT Gateways, Endpoints) | $301.56 |
| AWS Security Hub | $44.32 |
| Amazon Config | $35.55 |
| AWS KMS | $15.65 |
| Amazon CloudWatch | $8.75 |
| Amazon GuardDuty | $5.75 |
| AWS CloudTrail | $4.30 |
| Amazon Route 53 | $2.00 |
| Amazon S3 | $1.48 |
| Amazon Secrets Manager | $0.48 |
| Amazon SNS | $0.42 |
| **Total** | **~$420/month** |

### Cost Optimization Tips

1. **VPC Endpoints**: The centralized endpoint architecture reduces costs by sharing endpoints across accounts instead of creating duplicates in each VPC.

2. **Log Retention**: Review and adjust log retention periods in global-config.yaml based on your compliance requirements. Default retention is 3653 days (10 years) for CloudWatch Logs.

3. **S3 Lifecycle Policies**: Logs automatically transition to Glacier Instant Retrieval after 365 days and expire after 1000 days. Adjust these values based on your retention requirements.

4. **Enabled Regions**: Only enable regions you actually need to reduce the cost of deploying security controls across multiple regions.

5. **Reserved Capacity**: Consider Reserved Capacity for predictable workloads once your environment stabilizes.

---

## Maintenance and Updates

### Updating Configuration

1. Make changes to the appropriate configuration file(s)
2. For S3: Compress files into `aws-accelerator-config.zip` and upload
3. For CodeCommit: Commit and push changes
4. Navigate to CodePipeline and release the pipeline
5. Monitor pipeline execution and verify changes

### Updating LZA Version

1. Review the [LZA Release Notes](https://github.com/awslabs/landing-zone-accelerator-on-aws/releases) for breaking changes
2. Update the CloudFormation stack with the new version
3. Test in a non-production environment first if possible

### Updating HIPAA Eligible Services List

The HIPAA eligible services list in `service-control-policies/scp-hlc-hipaa-service.json` requires periodic updates as AWS adds new HIPAA-eligible services.

1. Review the current [AWS HIPAA Eligible Services](https://aws.amazon.com/compliance/hipaa-eligible-services-reference/)
2. Update the `NotAction` array in the SCP file
3. Update the `Sid` date suffix to reflect the review date
4. Upload configuration and release the pipeline

### Regular Maintenance Tasks

| Task | Frequency | Description |
|------|-----------|-------------|
| Review Security Hub Findings | Weekly | Address critical and high findings |
| Review GuardDuty Findings | Weekly | Investigate and remediate threats |
| Review Config Compliance | Monthly | Address non-compliant resources |
| Update HIPAA Services List | Quarterly | Add newly eligible services |
| Review CloudWatch Alarms | Monthly | Ensure alarms are actionable |
| Review IAM Access | Quarterly | Audit user access and permissions |
| Test Breakglass Access | Quarterly | Verify emergency access works |
| Review Costs | Monthly | Identify optimization opportunities |

---

## References

### AWS Documentation

- [Landing Zone Accelerator on AWS Implementation Guide](https://docs.aws.amazon.com/solutions/latest/landing-zone-accelerator-on-aws/landing-zone-accelerator-on-aws.pdf)
- [LZA Configuration Reference](https://awslabs.github.io/landing-zone-accelerator-on-aws/)
- [AWS HIPAA Eligible Services](https://aws.amazon.com/compliance/hipaa-eligible-services-reference/)
- [AWS Security Reference Architecture](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/welcome.html)

### GitHub Resources

- [LZA GitHub Repository](https://github.com/awslabs/landing-zone-accelerator-on-aws)
- [LZA for Healthcare Configuration Files](https://github.com/jcote5150/landing-zone-accelerator-on-aws-for-healthcare)

### Best Practices

- [Best Practices for Organizational Units with AWS Organizations](https://aws.amazon.com/blogs/mt/best-practices-for-organizational-units-with-aws-organizations/)
- [Recommended OUs and Accounts](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/recommended-ous-and-accounts.html)
- [Operational Best Practices for HIPAA Security](https://docs.aws.amazon.com/config/latest/developerguide/operational-best-practices-for-hipaa_security.html)

---

## Appendix A: Configuration File Templates

### Minimal replacements-config.yaml

```yaml
globalReplacements:
  - key: AcceleratorHomeRegion
    type: String
    value: us-east-1
  - key: SecurityHigh
    type: String
    value: security-high@example.com
  - key: SecurityMedium
    type: String
    value: security-medium@example.com
  - key: SecurityLow
    type: String
    value: security-low@example.com
  - key: BudgetEmail
    type: String
    value: budget@example.com
```

### Minimal accounts-config.yaml

```yaml
mandatoryAccounts:
  - name: Management
    description: The management (primary) account
    email: management@example.com
    organizationalUnit: Root
  - name: LogArchive
    description: The log archive account
    email: log-archive@example.com
    organizationalUnit: Security
  - name: Audit
    description: The security audit account
    email: audit@example.com
    organizationalUnit: Security

workloadAccounts:
  - name: Network-Prod
    description: Network Prod
    email: network-prod@example.com
    organizationalUnit: Infrastructure/Infra-Prod
  - name: HIS-Dev
    description: HIS Dev
    email: his-dev@example.com
    organizationalUnit: HIS/HIS-Non-Prod
  - name: EIS-Prod
    description: EIS Prod
    email: eis-prod@example.com
    organizationalUnit: EIS/EIS-Prod
```

---

## Appendix B: Checklist

### Pre-Deployment Checklist

- [ ] Completed LZA Immersion Day Workshop (recommended)
- [ ] Provisioned 6 unique email addresses for accounts
- [ ] Provisioned 5 unique email addresses for notifications
- [ ] Created new AWS Management account
- [ ] Pre-warmed account with EC2 instance
- [ ] Configured account contacts
- [ ] Verified root user email
- [ ] Enabled MFA on root user
- [ ] Created IAM deployment user with MFA
- [ ] Verified/increased AWS Organizations account quota
- [ ] Verified Lambda concurrent executions quota (1000+)
- [ ] Updated CodeBuild concurrency quota
- [ ] Verified home region accessibility
- [ ] Enabled AWS Cost Explorer
- [ ] (Optional) Configured GitHub token in Secrets Manager

### Deployment Checklist

- [ ] Enabled AWS Organizations
- [ ] Deployed AWSAccelerator-InstallerStack
- [ ] Initial pipeline completed successfully
- [ ] Verified Control Tower deployment
- [ ] Disabled Control Tower VPC creation
- [ ] Customized replacements-config.yaml
- [ ] Customized accounts-config.yaml
- [ ] (If needed) Updated home region in all config files
- [ ] Uploaded healthcare configuration files
- [ ] Released pipeline with healthcare configuration
- [ ] Pipeline completed successfully

### Post-Deployment Checklist

- [ ] Verified all accounts created in correct OUs
- [ ] Verified Security Hub enabled and aggregating
- [ ] Verified GuardDuty enabled across accounts
- [ ] Verified AWS Config rules deployed
- [ ] Verified Transit Gateway and VPC connectivity
- [ ] Verified centralized VPC endpoints
- [ ] Verified CloudTrail logging to LogArchive
- [ ] Verified VPC Flow Logs enabled
- [ ] Verified SCPs attached to correct OUs
- [ ] Configured Identity Center (optional)
- [ ] Configured MFA for Identity Center users
- [ ] Configured MFA for breakglass users
- [ ] Enabled security controls in all required regions
- [ ] (Optional) Deployed network inspection appliances

---

*Last Updated: December 2025*
*Compatible with LZA Version: 1.9.0*
