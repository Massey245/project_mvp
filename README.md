# AWS Infrastructure Engineer - Interview Preparation Guide
## Position: AWS Infrastructure Engineer | New York, NY | 12 Years Experience

---

## SECTION 1: OPENING & BEHAVIORAL QUESTIONS

### Q1: Tell me about yourself and your AWS infrastructure experience
**Answer Approach:**
"I'm an AWS Infrastructure Engineer with 12 years of comprehensive experience in cloud infrastructure, with the last 8 years heavily focused on AWS ecosystem. I've designed and managed multi-account AWS environments supporting enterprise applications with strict security and compliance requirements.

In my current role, I manage a landing zone architecture using AWS Control Tower supporting 40+ AWS accounts across dev, staging, and production environments. I've architected hybrid cloud solutions using Direct Connect and Site-to-Site VPNs, implemented infrastructure-as-code using CloudFormation and Terraform, and automated operations using Python boto3 and Lambda.

My expertise spans compute, networking, security, and automation. I've led migrations from on-premise to AWS, optimized cloud costs by 30%, and established CI/CD pipelines for infrastructure deployment. I'm particularly strong in networking—designing complex VPC architectures, implementing Transit Gateway hub-spoke models, and securing environments using Network Firewall and Security Hub."

### Q2: Why are you looking for a new opportunity, and what interests you about this role?
**Answer:**
"I'm seeking a role where I can leverage my AWS expertise in a more challenging, fast-paced environment. This position particularly interests me because it combines infrastructure architecture with automation—exactly where my strengths lie. The hybrid nature also appeals to me as I value both collaborative in-person work and focused remote time.

The tech stack mentioned—Control Tower, Transit Gateway, Infrastructure-as-Code, and Lambda automation—aligns perfectly with what I've been doing. I'm especially excited about the opportunity to work with a team that values infrastructure excellence and security best practices."

---

## SECTION 2: AWS CORE SERVICES - DEEP DIVE

### Q3: Walk me through how you would design a multi-account AWS environment from scratch
**Expert Answer:**
"I'd use AWS Control Tower with Organizations to establish a secure, scalable foundation:

**1. Account Structure:**
- Management Account (root) - only for billing and Control Tower
- Log Archive Account - centralized CloudTrail, Config, and CloudWatch logs
- Security/Audit Account - Security Hub, GuardDuty aggregation
- Network Account - Transit Gateway, Direct Connect, shared VPCs
- Workload Accounts - organized by environment (dev/staging/prod) and business unit

**2. Landing Zone Setup:**
- Deploy Control Tower Landing Zone in us-east-1 (or primary region)
- Configure Account Factory with standardized account templates
- Implement CfCT (Customizations for Control Tower) for custom guardrails
- Set up mandatory guardrails (CloudTrail, Config) and optional guardrails based on compliance needs

**3. Networking Architecture:**
- Create Transit Gateway in Network Account
- Design CIDR strategy with RFC 1918 non-overlapping ranges
- Implement hub-spoke model: Transit Gateway as hub, workload VPCs as spokes
- Configure RAM (Resource Access Manager) to share Transit Gateway across accounts
- Set up Direct Connect Gateway for on-premise connectivity
- Implement VPC Endpoints for AWS services (S3, DynamoDB, etc.) to avoid NAT charges

**4. Security Controls:**
- Implement SCPs (Service Control Policies) at OU level
- Deploy AWS Network Firewall in inspection VPC
- Configure Security Hub with CIS Benchmark standards
- Enable GuardDuty across all accounts with delegated administrator
- Implement IAM Identity Center (SSO) for centralized authentication
- Deploy KMS with customer-managed keys, separate keys per environment

**5. Monitoring & Compliance:**
- Centralized CloudWatch with cross-account log aggregation using Firehose
- CloudTrail organization trail to Log Archive account
- Config rules for compliance monitoring
- SNS topics for critical alerts routed to security team
- EventBridge rules for automated remediation

**6. Automation:**
- Service Catalog portfolios for self-service infrastructure
- Lambda functions for automated tagging, cleanup, and compliance checks
- SSM Parameter Store for configuration management
- CodePipeline for infrastructure deployment

I've implemented this exact pattern for organizations with 30-50+ accounts, and it provides excellent governance while allowing teams autonomy."

### Q4: Explain Transit Gateway vs VPC Peering. When would you use each?
**Expert Answer:**
"I've worked extensively with both, and here's my decision framework:

**Transit Gateway - When to Use:**
- 10+ VPCs need connectivity (VPC Peering doesn't scale—it's n(n-1)/2 connections)
- Hub-spoke network topology preferred
- Need centralized network management
- On-premise connectivity via Direct Connect or VPN for multiple VPCs
- Network segmentation using route tables (isolate dev from prod)
- Inter-region peering required (Transit Gateway Peering)

**Real Example from My Experience:**
At my previous company, we had 35 VPCs across 4 regions. Initially using VPC Peering, we had over 200 peering connections—nightmare to manage. I designed a Transit Gateway solution:
- Regional Transit Gateways (one per region)
- Inter-region TGW peering
- Separate route tables for prod (isolated) and non-prod (full mesh)
- Reduced management overhead by 80%

**VPC Peering - When to Use:**
- Simple, small-scale connectivity (2-5 VPCs)
- Low latency critical (TGW adds ~1-2ms)
- Cost sensitive (no TGW hourly charges, no per-GB charges)
- No need for transitive routing

**Technical Differences:**
- Transit Gateway: Layer 3 routing, supports 5000 VPC attachments, ECMP, centralized routing
- VPC Peering: Direct connection, no single point of failure, lower latency, no transitive routing

**Cost Consideration:**
Transit Gateway costs $0.05/hour per attachment + $0.02/GB data processing. For high-throughput workloads between 2 VPCs, peering is cheaper. For complex topologies, TGW management savings outweigh costs."

### Q5: How do you secure S3 buckets in a multi-account environment?
**Expert Answer:**
"S3 security is multi-layered. Here's my comprehensive approach:

**1. Bucket Policies & Access Control:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedObjectUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    },
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

**2. Organization-Level Controls:**
- SCP to prevent public bucket creation:
```json
{
  "Effect": "Deny",
  "Action": [
    "s3:PutBucketPublicAccessBlock",
    "s3:PutAccountPublicAccessBlock"
  ],
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "s3:x-amz-acl": "private"
    }
  }
}
```

**3. Encryption Strategy:**
- Enable default encryption with KMS CMK (not SSE-S3)
- Separate KMS keys per environment/sensitivity
- Key policies restricting decrypt permissions
- Bucket keys enabled to reduce KMS API calls and costs

**4. Access Logging & Monitoring:**
- S3 server access logging to centralized logging bucket
- CloudTrail data events for API-level auditing
- EventBridge rules for suspicious activities (GetObject from unusual IPs)
- Config rules: s3-bucket-public-read-prohibited, s3-bucket-ssl-requests-only

**5. Network Controls:**
- VPC Endpoints for S3 (Gateway Endpoint) - free and secure
- Bucket policies requiring VPC Endpoint:
```json
"Condition": {
  "StringNotEquals": {
    "aws:sourceVpce": "vpce-1234567"
  }
}
```

**6. Versioning & Lifecycle:**
- Versioning enabled for accidental deletion protection
- MFA Delete for critical buckets
- Lifecycle policies: transition to Glacier after 90 days, delete after 7 years
- Object Lock for compliance (WORM - Write Once Read Many)

**7. Cross-Account Access (Secure Pattern):**
- IAM roles with assume role policies, NOT bucket policies with account principals
- Require ExternalId for third-party access
- Time-bound session tokens

**8. Automated Compliance:**
- Lambda function triggered by Config to automatically remediate public buckets
- Security Hub findings for non-compliant buckets
- Tag policies enforcing cost center and data classification tags

**Real Incident Example:**
We once had a developer accidentally make a bucket public. Within 2 minutes:
1. Config rule detected violation
2. EventBridge triggered Lambda
3. Lambda removed public access and sent SNS alert
4. Security team investigated within 15 minutes
This saved us from potential data exposure."

---

## SECTION 3: NETWORKING DEEP DIVE

### Q6: Design a hybrid cloud network architecture with high availability and security
**Expert Answer:**
"I've designed several hybrid architectures. Here's a production-grade design:

**Architecture Overview:**
```
On-Premise DC ←→ DX/VPN ←→ Transit Gateway ←→ Workload VPCs
                                    ↓
                            Network Firewall
                                    ↓
                            Inspection VPC
```

**1. Connectivity Layer:**
- **Primary:** AWS Direct Connect (1 Gbps or 10 Gbps) via Direct Connect Gateway
  - Two DX connections for redundancy (different facilities)
  - BGP with AS-PATH prepending for active-passive failover
  - VLAN tagging for logical separation (prod, non-prod)
  
- **Backup:** Site-to-Site VPN over internet
  - Two VPN tunnels (AWS creates two automatically for HA)
  - BFD (Bidirectional Forwarding Detection) enabled for fast failover (~10 seconds)
  - ECMP across tunnels if equal cost
  
- **Direct Connect Gateway:**
  - Associates with Virtual Private Gateway or Transit Gateway
  - Supports up to 10 VPC attachments (VGW) or unlimited via TGW
  - Private VIF for VPC connectivity, Public VIF for AWS public services

**2. Transit Gateway Configuration:**
- Hub in centralized Network Account
- Route Tables:
  - **Prod RT:** Only prod VPC attachments, isolated from non-prod
  - **Non-Prod RT:** Dev/staging VPCs, full mesh
  - **Shared Services RT:** AD, DNS, monitoring accessible by all
  - **Inspection RT:** Forces traffic through Network Firewall
  
- BGP Configuration:
  - ASN: Use private ASN (64512-65534)
  - Route propagation from VPN/DX attachments
  - Static routes to on-premise (10.0.0.0/8)

**3. Security Inspection:**
- **Inspection VPC with AWS Network Firewall:**
  - Centralized egress/ingress inspection
  - Suricata-compatible IPS rules
  - Domain filtering (block known malicious domains)
  - TLS inspection for decryption
  
- **Traffic Flow:**
  ```
  Internet → IGW → Network Firewall → TGW → Workload VPC
  Workload VPC → TGW → Network Firewall → NAT Gateway → Internet
  ```

**4. VPC Design (Per Environment):**
- **CIDR Planning:**
  - Prod: 10.10.0.0/16 (65k IPs)
  - Dev: 10.20.0.0/16
  - Staging: 10.30.0.0/16
  - Shared Services: 10.1.0.0/16
  - On-premise: 172.16.0.0/12 (reserved)

- **Subnet Strategy (per AZ, minimum 3 AZs):**
  - Public Subnets: /24 (ALB, NAT Gateway)
  - Private App Subnets: /22 (EC2, ECS)
  - Private Data Subnets: /23 (RDS, ElastiCache)
  - Reserved: /22 for future growth

- **Route Tables:**
  - Public: 0.0.0.0/0 → IGW
  - Private App: 0.0.0.0/0 → NAT Gateway (per AZ)
  - Private Data: No internet route, only internal
  - All: 172.16.0.0/12 → TGW (on-premise), 10.0.0.0/8 → TGW (inter-VPC)

**5. DNS Resolution:**
- **Route 53 Resolver Endpoints:**
  - Inbound Endpoint: On-premise queries AWS resources (db.internal.aws)
  - Outbound Endpoint: AWS queries on-premise DNS (ldap.corp.local)
  - Forwarding rules shared via RAM

- **Private Hosted Zones:**
  - api.prod.internal → ALB
  - Associated with all VPCs needing access

**6. High Availability & Resilience:**
- Multi-AZ deployment (minimum 3 AZs)
- NAT Gateway per AZ (avoid cross-AZ charges and single point of failure)
- Application Load Balancer across AZs
- RDS Multi-AZ with automated failover
- DX + VPN for 99.95%+ connectivity SLA

**7. Security Controls:**
- **NACLs:** Layer 4 stateless filtering (ephemeral ports 1024-65535)
- **Security Groups:** Layer 4 stateful filtering
  - App SG: Allow 443 from ALB SG only
  - DB SG: Allow 3306 from App SG only (principle of least privilege)
  - Bastion/SSM: No inbound SSH, use SSM Session Manager
  
- **VPC Flow Logs:**
  - Enabled on all VPCs, sent to CloudWatch Logs
  - Lambda parsing for anomaly detection (unusual ports, rejected connections)
  - Athena queries for analysis

**8. Cost Optimization:**
- VPC Gateway Endpoints (S3, DynamoDB) - free, no NAT charges
- Interface Endpoints (PrivateLink) for frequently used services (SSM, EC2)
- Single NAT Gateway in dev (cost vs HA tradeoff)
- DX with 10 Gbps cheaper than egress charges for high-throughput workloads

**Real Implementation Metrics:**
In my last implementation:
- Latency: On-premise to AWS: 5-8ms via DX
- Throughput: 8 Gbps sustained with 10 Gbps DX
- Failover: DX to VPN failover in 12 seconds (BFD enabled)
- Availability: 99.97% uptime over 18 months
- Cost: Reduced egress charges by 60% using DX vs internet
"

### Q7: Explain VPC Endpoints and when you'd use Gateway vs Interface endpoints
**Expert Answer:**
"VPC Endpoints are critical for cost and security. I use them extensively:

**Gateway Endpoints (Free!):**
- **Services:** S3 and DynamoDB ONLY
- **How it works:** Updates route table with prefix list
- **Traffic:** Stays within AWS network, never traverses internet/NAT
- **Cost:** FREE (no hourly charge, no per-GB charge)
- **Use case:** Every VPC should have S3 Gateway Endpoint
  
**Example Route Table:**
```
pl-63a5400a (S3 prefix list) → vpce-1234 (Gateway Endpoint)
```

**Real Scenario:**
Before S3 Gateway Endpoint: NAT Gateway costs $0.045/GB + $0.045/hour
After implementation: $0/GB data processing, only S3 storage/request costs
Saved $2,300/month in NAT charges for S3-heavy workload

**Interface Endpoints (PrivateLink - Costs Money):**
- **Services:** 100+ services (EC2, SSM, CloudWatch, etc.)
- **How it works:** ENI in your subnet with private IP
- **Cost:** $0.01/hour per AZ + $0.01/GB data processed
- **DNS:** Private DNS name resolves to endpoint IP

**When to Use Interface Endpoints:**
1. **Security requirement:** No internet access allowed (e.g., private subnets with no NAT)
2. **Compliance:** Data must never traverse public internet
3. **High-frequency API calls:** SSM Session Manager, CloudWatch Logs
4. **Cost analysis:** Data transfer > 100 GB/month per service makes it worthwhile

**My Cost Comparison (Real Data):**
Scenario: 500 GB CloudWatch Logs/month from private subnet

Option 1: NAT Gateway
- NAT cost: 500 GB × $0.045 = $22.50
- NAT hourly: 720 hours × $0.045 = $32.40
- Total: $54.90/month

Option 2: Interface Endpoint (3 AZs)
- Endpoint: 3 AZs × 720 hours × $0.01 = $21.60
- Data processing: 500 GB × $0.01 = $5.00
- Total: $26.60/month
- **Savings: $28.30/month (52%)**

**Services I Always Deploy Interface Endpoints For:**
1. **SSM (Systems Manager):** No bastion hosts, secure shell access
2. **EC2:** Metadata service, instance management
3. **CloudWatch Logs:** High-volume log shipping
4. **Secrets Manager:** Application secrets retrieval
5. **KMS:** Encryption/decryption operations

**Architecture Pattern:**
```
Private Subnet → Interface Endpoint (ENI) → AWS Service
(No NAT Gateway needed!)
```

**Important Configuration:**
- Enable Private DNS: AWS service DNS (e.g., ssm.us-east-1.amazonaws.com) resolves to endpoint
- Security Group: Allow 443 from VPC CIDR
- Endpoint Policy: Restrict actions (e.g., only ssm:StartSession, not ssm:DescribeInstance*)

**Hybrid Strategy I Use:**
- Gateway Endpoints: S3, DynamoDB (always, they're free)
- Interface Endpoints: SSM, EC2, CloudWatch Logs, Secrets Manager (high-frequency)
- NAT Gateway: Remaining AWS APIs and internet egress

This gives best balance of cost, security, and performance."

---

## SECTION 4: IAM & SECURITY

### Q8: Explain the difference between IAM Roles, Policies, and how you implement least privilege
**Expert Answer:**
"IAM is foundational to AWS security. Here's how I architect it:

**IAM Entities:**

**1. IAM Users:**
- **My Practice:** I never create IAM users for application access—only for break-glass scenarios
- **Why:** Users have long-term credentials (access keys) that can leak
- **Alternative:** IAM Identity Center (SSO) for human access, IAM Roles for applications

**2. IAM Roles:**
- **Definition:** An identity with policies, but no credentials
- **How it works:** Temporary security credentials via STS AssumeRole
- **Types I use:**
  - Service Roles: Lambda, EC2, ECS assume these
  - Cross-Account Roles: Account A assumes role in Account B
  - Identity Provider Roles: SAML/OIDC federation (Okta, Azure AD)

**Real Implementation:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**3. IAM Policies:**
- **Identity-based:** Attached to users/groups/roles
- **Resource-based:** Attached to resources (S3 bucket policy, KMS key policy)
- **Boundary:** Maximum permissions (can't exceed even if identity policy allows)
- **SCP:** Organization-level guardrails

**Least Privilege Implementation Strategy:**

**Phase 1: Start with Deny (Reverse Least Privilege):**
Instead of granting broad access and removing, I start with minimum and add:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-app-bucket/uploads/*"
    }
  ]
}
```
Not:
```json
"Action": "s3:*",
"Resource": "*"  ← NEVER do this!
```

**Phase 2: Use IAM Access Analyzer:**
- Analyzes CloudTrail logs to determine actual API usage
- Generates policy based on last 90 days of activity
- Removes unused permissions

**Real Example:**
Developer requested `ec2:*`. I enabled CloudTrail data events, waited 30 days, ran Access Analyzer:
```
Actual usage:
- ec2:DescribeInstances
- ec2:StartInstances
- ec2:StopInstances
```
Final policy grants only these three actions for specific instance IDs (Condition: StringEquals ec2:ResourceTag/Environment: dev)

**Phase 3: Policy Conditions (Critical for Security):**
```json
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "arn:aws:s3:::prod-data/*",
  "Condition": {
    "IpAddress": {
      "aws:SourceIp": "203.0.113.0/24"
    },
    "StringEquals": {
      "aws:RequestedRegion": "us-east-1"
    },
    "DateGreaterThan": {
      "aws:CurrentTime": "2024-01-01T00:00:00Z"
    },
    "DateLessThan": {
      "aws:CurrentTime": "2024-12-31T23:59:59Z"
    }
  }
}
```

**Conditions I use frequently:**
- `aws:SourceVpce`: Require VPC Endpoint
- `aws:SecureTransport`: Require HTTPS
- `aws:RequestedRegion`: Prevent wrong region deployments
- `aws:MultiFactorAuthPresent`: Require MFA for sensitive actions
- `aws:PrincipalOrgID`: Trust only principals from my Organization

**Phase 4: Permissions Boundaries:**
Use case: Allow developers to create roles but limit maximum permissions

```json
{
  "Effect": "Allow",
  "Action": "iam:CreateRole",
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "iam:PermissionsBoundary": "arn:aws:iam::123456789012:policy/DeveloperBoundary"
    }
  }
}
```

Developer Boundary Policy:
```json
{
  "Effect": "Deny",
  "Action": [
    "iam:*",
    "organizations:*",
    "account:*"
  ],
  "Resource": "*"
}
```
Even if developer creates role with AdministratorAccess, the boundary denies IAM actions.

**Phase 5: Service Control Policies (Organization Level):**
```json
{
  "Effect": "Deny",
  "Action": [
    "ec2:RunInstances"
  ],
  "Resource": "arn:aws:ec2:*:*:instance/*",
  "Condition": {
    "StringNotEquals": {
      "ec2:InstanceType": [
        "t3.micro",
        "t3.small",
        "t3.medium"
      ]
    }
  }
}
```
This prevents ANY user in ANY account from launching large instances, even with AdministratorAccess!

**Cross-Account Access Pattern (Secure):**
Account A (Source): Developer needs S3 access in Account B

Account B (Target) Role Trust Policy:
```json
{
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::111111111111:root"
  },
  "Action": "sts:AssumeRole",
  "Condition": {
    "StringEquals": {
      "sts:ExternalId": "unique-random-string-12345"
    }
  }
}
```

**Why ExternalId?** Prevents "confused deputy" problem. Without it, any role in Account A could assume this role.

**IAM Best Practices I Enforce:**
1. **Never use root account** - only for account closure and billing
2. **No long-term credentials** - rotate access keys every 90 days (Config rule)
3. **MFA on all human access** - enforced via SCP
4. **Session duration limits** - max 12 hours for AssumeRole
5. **Tag-based access control** - policies use Condition: aws:RequestTag/ResourceTag
6. **Audit with IAM Access Analyzer** - identify over-privileged roles quarterly
7. **Automated remediation** - Lambda removes unused roles/users after 90 days

**Real Incident - Over-Privileged Role:**
Found an EC2 role with `s3:*` permission. Used CloudTrail Insights:
- Last 90 days: only s3:GetObject and s3:ListBucket used
- Updated policy to only these two actions
- Reduced blast radius by 90%

**Monitoring & Alerts:**
- GuardDuty: Detects compromised credentials (UnauthorizedAccess findings)
- CloudWatch Metric Filter: Alert on root account usage
- Config Rules:
  - iam-user-unused-credentials-check
  - iam-password-policy
  - access-keys-rotated
- Security Hub: Aggregates all security findings

This layered approach has prevented several potential breaches in my experience."

---

## SECTION 5: AUTOMATION & INFRASTRUCTURE AS CODE

### Q9: Walk me through a CloudFormation template for a 3-tier web application with best practices
**Expert Answer:**
"I'll design a production-grade CloudFormation stack with proper separation, modularity, and security:

**Architecture:**
```
ALB (Public) → Auto Scaling Group (Private App) → RDS (Private Data)
```

**Stack Structure (Nested Stacks):**
```
master-stack.yaml
├── network-stack.yaml (VPC, Subnets, Route Tables)
├── security-stack.yaml (Security Groups, NACLs)
├── compute-stack.yaml (ASG, Launch Template)
├── database-stack.yaml (RDS, Secrets Manager)
└── loadbalancer-stack.yaml (ALB, Target Group)
```

**Master Stack (master-stack.yaml):**
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 3-Tier Web Application Master Stack

Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, prod]
    Default: dev
  
  InstanceType:
    Type: String
    Default: t3.medium
    AllowedValues: [t3.small, t3.medium, t3.large]
  
  DBPassword:
    Type: String
    NoEcho: true
    MinLength: 12
    Description: RDS master password (retrieve from Secrets Manager)

Mappings:
  EnvironmentMap:
    dev:
      VpcCidr: 10.20.0.0/16
      MinSize: 1
      MaxSize: 2
      DBInstanceClass: db.t3.small
    prod:
      VpcCidr: 10.10.0.0/16
      MinSize: 2
      MaxSize: 10
      DBInstanceClass: db.r6g.xlarge

Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Sub 'https://s3.amazonaws.com/${TemplateBucket}/network-stack.yaml'
      Parameters:
        VpcCidr: !FindInMap [EnvironmentMap, !Ref Environment, VpcCidr]
        Environment: !Ref Environment
      Tags:
        - Key: Environment
          Value: !Ref Environment
        - Key: ManagedBy
          Value: CloudFormation

  SecurityStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: NetworkStack
    Properties:
      TemplateURL: !Sub 'https://s3.amazonaws.com/${TemplateBucket}/security-stack.yaml'
      Parameters:
        VpcId: !GetAtt NetworkStack.Outputs.VpcId
        Environment: !Ref Environment

  DatabaseStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: SecurityStack
    Properties:
      TemplateURL: !Sub 'https://s3.amazonaws.com/${TemplateBucket}/database-stack.yaml'
      Parameters:
        DBSubnetIds: !GetAtt NetworkStack.Outputs.PrivateDataSubnets
        DBSecurityGroup: !GetAtt SecurityStack.Outputs.DatabaseSecurityGroup
        DBPassword: !Ref DBPassword
        DBInstanceClass: !FindInMap [EnvironmentMap, !Ref Environment, DBInstanceClass]
        Environment: !Ref Environment

  ComputeStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: [DatabaseStack, SecurityStack]
    Properties:
      TemplateURL: !Sub 'https://s3.amazonaws.com/${TemplateBucket}/compute-stack.yaml'
      Parameters:
        SubnetIds: !GetAtt NetworkStack.Outputs.PrivateAppSubnets
        SecurityGroup: !GetAtt SecurityStack.Outputs.AppSecurityGroup
        InstanceType: !Ref InstanceType
        MinSize: !FindInMap [EnvironmentMap, !Ref Environment, MinSize]
        MaxSize: !FindInMap [EnvironmentMap, !Ref Environment, MaxSize]
        DBEndpoint: !GetAtt DatabaseStack.Outputs.DBEndpoint
        Environment: !Ref Environment

  LoadBalancerStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: [ComputeStack, SecurityStack]
    Properties:
      TemplateURL: !Sub 'https://s3.amazonaws.com/${TemplateBucket}/loadbalancer-stack.yaml'
      Parameters:
        VpcId: !GetAtt NetworkStack.Outputs.VpcId
        SubnetIds: !GetAtt NetworkStack.Outputs.PublicSubnets
        SecurityGroup: !GetAtt SecurityStack.Outputs.ALBSecurityGroup
        TargetGroupArn: !GetAtt ComputeStack.Outputs.TargetGroupArn
        Environment: !Ref Environment

Outputs:
  ApplicationURL:
    Description: Application Load Balancer URL
    Value: !GetAtt LoadBalancerStack.Outputs.ALBURL
    Export:
      Name: !Sub '${AWS::StackName}-AppURL'
  
  DBEndpoint:
    Description: RDS Database Endpoint
    Value: !GetAtt DatabaseStack.Outputs.DBEndpoint
    Export:
      Name: !Sub '${AWS::StackName}-DBEndpoint'
```

**Compute Stack (compute-stack.yaml) - Detailed:**
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Auto Scaling Group with Launch Template

Parameters:
  SubnetIds:
    Type: CommaDelimitedList
  SecurityGroup:
    Type: String
  InstanceType:
    Type: String
  MinSize:
    Type: Number
  MaxSize:
    Type: Number
  DBEndpoint:
    Type: String
  Environment:
    Type: String

Resources:
  # IAM Role for EC2 instances
  EC2Role:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
      Policies:
        - PolicyName: S3AccessPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:GetObject
                  - s3:ListBucket
                Resource:
                  - !Sub 'arn:aws:s3:::${ApplicationBucket}'
                  - !Sub 'arn:aws:s3:::${ApplicationBucket}/*'
        - PolicyName: SecretsManagerAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - secretsmanager:GetSecretValue
                Resource: !Sub 'arn:aws:secretsmanager:${AWS::Region}:${AWS::AccountId}:secret:${Environment}/db-password-*'
      Tags:
        - Key: Environment
          Value: !Ref Environment

  EC2InstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref EC2Role

  # Launch Template with best practices
  AppLaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateName: !Sub '${Environment}-app-template'
      LaunchTemplateData:
        ImageId: !Sub '{{resolve:ssm:/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2}}'
        InstanceType: !Ref InstanceType
        IamInstanceProfile:
          Arn: !GetAtt EC2InstanceProfile.Arn
        SecurityGroupIds:
          - !Ref SecurityGroup
        
        # EBS Encryption (Critical!)
        BlockDeviceMappings:
          - DeviceName: /dev/xvda
            Ebs:
              VolumeSize: 20
              VolumeType: gp3
              Encrypted: true
              DeleteOnTermination: true
              Iops: 3000
              Throughput: 125
        
        # Metadata Options (IMDSv2 - Security best practice)
        MetadataOptions:
          HttpTokens: required  # Require IMDSv2
          HttpPutResponseHopLimit: 1
          HttpEndpoint: enabled
        
        # Monitoring
        Monitoring:
          Enabled: true
        
        # User Data for initialization
        UserData:
          Fn::Base64: !Sub |
            #!/bin/bash
            set -e
            
            # Install CloudWatch Agent
            wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
            rpm -U ./amazon-cloudwatch-agent.rpm
            
            # Configure CloudWatch Agent
            cat > /opt/aws/amazon-cloudwatch-agent/etc/config.json <<'EOF'
            {
              "metrics": {
                "namespace": "${Environment}-App",
                "metrics_collected": {
                  "mem": {
                    "measurement": [{"name": "mem_used_percent"}],
                    "metrics_collection_interval": 60
                  },
                  "disk": {
                    "measurement": [{"name": "used_percent"}],
                    "metrics_collection_interval": 60,
                    "resources": ["*"]
                  }
                }
              },
              "logs": {
                "logs_collected": {
                  "files": {
                    "collect_list": [
                      {
                        "file_path": "/var/log/app.log",
                        "log_group_name": "/aws/ec2/${Environment}-app",
                        "log_stream_name": "{instance_id}"
                      }
                    ]
                  }
                }
              }
            }
            EOF
            
            /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
              -a fetch-config \
              -m ec2 \
              -s \
              -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json
            
            # Install application dependencies
            yum update -y
            yum install -y httpd mod_ssl
            
            # Retrieve DB password from Secrets Manager
            DB_PASSWORD=$(aws secretsmanager get-secret-value \
              --secret-id ${Environment}/db-password \
              --query SecretString \
              --output text \
              --region ${AWS::Region})
            
            # Configure application
            cat > /var/www/html/config.php <
            EOF
            
            # Start services
            systemctl start httpd
            systemctl enable httpd
            
            # Signal CloudFormation success
            /opt/aws/bin/cfn-signal \
              --exit-code $? \
              --stack ${AWS::StackName} \
              --resource AutoScalingGroup \
              --region ${AWS::Region}
        
        TagSpecifications:
          - ResourceType: instance
            Tags:
              - Key: Name
                Value: !Sub '${Environment}-app-server'
              - Key: Environment
                Value: !Ref Environment
              - Key: Application
                Value: WebApp
          - ResourceType: volume
            Tags:
              - Key: Name
                Value: !Sub '${Environment}-app-volume'

  # Target Group for ALB
  AppTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: !Sub '${Environment}-app-tg'
      Port: 80
      Protocol: HTTP
      VpcId: !Ref VpcId
      HealthCheckEnabled: true
      HealthCheckPath: /health
      HealthCheckProtocol: HTTP
      HealthCheckIntervalSeconds: 30
      HealthCheckTimeoutSeconds: 5
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 3
      TargetType: instance
      Deregistration DelayTimeoutSeconds: 30
      Tags:
        - Key: Environment
          Value: !Ref Environment

  # Auto Scaling Group
  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      AutoScalingGroupName: !Sub '${Environment}-app-asg'
      LaunchTemplate:
        LaunchTemplateId: !Ref AppLaunchTemplate
        Version: !GetAtt AppLaunchTemplate.LatestVersionNumber
      MinSize: !Ref MinSize
      MaxSize: !Ref MaxSize
      DesiredCapacity: !Ref MinSize
      VPCZoneIdentifier: !Ref SubnetIds
      TargetGroupARNs:
        - !Ref AppTargetGroup
      HealthCheckType: ELB
      HealthCheckGracePeriod: 300
      
      # Termination Policies
      TerminationPolicies:
        - OldestLaunchTemplate
        - OldestInstance
      
      # Tags
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-app-instance'
          PropagateAtLaunch: true
        - Key: Environment
          Value: !Ref Environment
          PropagateAtLaunch: true
    
    CreationPolicy:
      ResourceSignal:
        Count: !Ref MinSize
        Timeout: PT15M
    
    UpdatePolicy:
      AutoScalingRollingUpdate:
        MinInstancesInService: 1
        MaxBatchSize: 2
        PauseTime: PT5M
        WaitOnResourceSignals: true

  # Scaling Policies
  ScaleUpPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AdjustmentType: ChangeInCapacity
      AutoScalingGroupName: !Ref AutoScalingGroup
      Cooldown: 300
      ScalingAdjustment: 1

  ScaleDownPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AdjustmentType: ChangeInCapacity
      AutoScalingGroupName: !Ref AutoScalingGroup
      Cooldown: 300
      ScalingAdjustment: -1

  # CloudWatch Alarms
  HighCPUAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${Environment}-app-high-cpu'
      AlarmDescription: Trigger scale up when CPU > 70%
      MetricName: CPUUtilization
      Namespace: AWS/EC2
      Statistic: Average
      Period: 300
      EvaluationPeriods: 2
      Threshold: 70
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: AutoScalingGroupName
          Value: !Ref AutoScalingGroup
      AlarmActions:
        - !Ref ScaleUpPolicy

  LowCPUAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${Environment}-app-low-cpu'
      AlarmDescription: Trigger scale down when CPU < 30%
      MetricName: CPUUtilization
      Namespace: AWS/EC2
      Statistic: Average
      Period: 300
      EvaluationPeriods: 2
      Threshold: 30
      ComparisonOperator: LessThanThreshold
      Dimensions:
        - Name: AutoScalingGroupName
          Value: !Ref AutoScalingGroup
      AlarmActions:
        - !Ref ScaleDownPolicy

Outputs:
  TargetGroupArn:
    Description: Target Group ARN
    Value: !Ref AppTargetGroup
  
  AutoScalingGroupName:
    Description: Auto Scaling Group Name
    Value: !Ref AutoScalingGroup
```

**Key Best Practices Implemented:**

1. **Security:**
   - EBS encryption enabled
   - IMDSv2 required (prevents SSRF attacks)
   - Secrets Manager for sensitive data (not hardcoded)
   - IAM roles with least privilege
   - No SSH keys (use SSM Session Manager)

2. **High Availability:**
   - Multi-AZ deployment via SubnetIds
   - ELB health checks
   - Rolling updates with min instances in service
   - Termination policies for graceful replacement

3. **Monitoring:**
   - CloudWatch Agent for custom metrics (memory, disk)
   - Application logs to CloudWatch Logs
   - Detailed monitoring enabled
   - Alarms for auto-scaling and notifications

4. **Cost Optimization:**
   - gp3 volumes (cheaper than gp2, better performance)
   - Appropriate instance sizing per environment
   - Deregistration delay 30s (faster scale-in)

5. **Automation:**
   - cfn-signal for creation/update verification
   - UserData for zero-touch provisioning
   - SSM Parameter Store for AMI ID (always latest)

6. **Disaster Recovery:**
   - CreationPolicy ensures instances healthy before complete
   - UpdatePolicy for zero-downtime deployments
   - Health check grace period for app startup time

**Deployment Commands:**
```bash
# Validate template
aws cloudformation validate-template \
  --template-body file://master-stack.yaml

# Deploy with parameters
aws cloudformation create-stack \
  --stack-name prod-webapp \
  --template-body file://master-stack.yaml \
  --parameters \
    ParameterKey=Environment,ParameterValue=prod \
    ParameterKey=InstanceType,ParameterValue=t3.medium \
    ParameterKey=DBPassword,ParameterValue=$(aws secretsmanager get-secret-value --secret-id prod/db-password --query SecretString --output text) \
  --capabilities CAPABILITY_IAM \
  --tags Key=Project,Value=WebApp Key=Owner,Value=Platform-Team

# Monitor deployment
aws cloudformation describe-stack-events \
  --stack-name prod-webapp \
  --query 'StackEvents[*].[ResourceStatus,ResourceType,LogicalResourceId]' \
  --output table

# Get outputs
aws cloudformation describe-stacks \
  --stack-name prod-webapp \
  --query 'Stacks[0].Outputs'
```

**Production Deployment Checklist:**
- [ ] VPC Flow Logs enabled
- [ ] AWS Config rules active
- [ ] Security Hub findings reviewed
- [ ] Backup policies configured (AWS Backup)
- [ ] CloudWatch Dashboard created
- [ ] SNS topics for critical alarms
- [ ] IAM Access Analyzer run
- [ ] Cost allocation tags applied
- [ ] Documentation updated in Confluence

This is the exact pattern I've deployed in 5+ production environments supporting 100k+ requests/day with 99.95% uptime."

---

## SECTION 6: MONITORING, LOGGING & TROUBLESHOOTING

### Q10: How do you implement centralized logging and monitoring across multiple AWS accounts?
**Expert Answer:**
"Centralized observability is critical for multi-account environments. Here's my production architecture:

**Architecture:**
```
Workload Accounts (30+) → Log Archive Account ← Security Account
                              ↓
                    CloudWatch Logs Insights
                         Athena (S3)
                        OpenSearch (optional)
```

**Implementation:**

**1. CloudTrail (Organization Trail):**
```bash
# In Management Account
aws cloudtrail create-trail \
  --name org-audit-trail \
  --s3-bucket-name org-cloudtrail-logs-123456789012 \
  --is-organization-trail \
  --is-multi-region-trail \
  --enable-log-file-validation \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/abcd1234

aws cloudtrail put-event-selectors \
  --trail-name org-audit-trail \
  --event-selectors '[
    {
      "ReadWriteType": "All",
      "IncludeManagementEvents": true,
      "DataResources": [
        {
          "Type": "AWS::S3::Object",
          "Values": ["arn:aws:s3:::*/"]
        },
        {
          "Type": "AWS::Lambda::Function",
          "Values": ["arn:aws:lambda:*:*:function/*"]
        }
      ]
    }
  ]'

aws cloudtrail start-logging --name org-audit-trail
```

**Lifecycle Policy (S3 Bucket):**
```json
{
  "Rules": [
    {
      "Id": "Archive-CloudTrail-Logs",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 90,
          "StorageClass": "INTELLIGENT_TIERING"
        },
        {
          "Days": 365,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 2555
      }
    }
  ]
}
```

**2. CloudWatch Logs Aggregation:**

**Workload Account (Source) - Subscription Filter:**
```python
import boto3

logs_client = boto3.client('logs')

# Create subscription filter to send logs to Kinesis Firehose
logs_client.put_subscription_filter(
    logGroupName='/aws/lambda/my-function',
    filterName='forward-to-central',
    filterPattern='',  # All logs, or '[ERROR]' for errors only
    destinationArn='arn:aws:firehose:us-east-1:999999999999:deliverystream/central-logs',
    roleArn='arn:aws:iam::111111111111:role/CloudWatchLogsToFirehoseRole'
)
```

**Log Archive Account (Destination) - Kinesis Firehose:**
```yaml
Resources:
  CentralLogsFirehose:
    Type: AWS::KinesisFirehose::DeliveryStream
    Properties:
      DeliveryStreamName: central-logs
      DeliveryStreamType: DirectPut
      ExtendedS3DestinationConfiguration:
        BucketARN: !GetAtt CentralLogsBucket.Arn
        RoleARN: !GetAtt FirehoseRole.Arn
        Prefix: 'logs/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/'
        ErrorOutputPrefix: 'errors/'
        CompressionFormat: GZIP
        BufferingHints:
          SizeInMBs: 128
          IntervalInSeconds: 300
        
        # Data transformation (optional)
        ProcessingConfiguration:
          Enabled: true
          Processors:
            - Type: Lambda
              Parameters:
                - ParameterName: LambdaArn
                  ParameterValue: !GetAtt LogTransformFunction.Arn
        
        # CloudWatch Logs for Firehose errors
        CloudWatchLoggingOptions:
          Enabled: true
          LogGroupName: /aws/kinesisfirehose/central-logs
          LogStreamName: S3Delivery

  # Lambda for log transformation/enrichment
  LogTransformFunction:
    Type: AWS::Lambda::Function
    Properties:
      Runtime: python3.11
      Handler: index.lambda_handler
      Code:
        ZipFile: |
          import base64
          import json
          import gzip
          
          def lambda_handler(event, context):
              output = []
              
              for record in event['records']:
                  # Decode and decompress CloudWatch Logs data
                  payload = base64.b64decode(record['data'])
                  decompressed = gzip.decompress(payload)
                  log_data = json.loads(decompressed)
                  
                  # Add metadata
                  log_data['account_id'] = context.invoked_function_arn.split(':')[4]
                  log_data['region'] = context.invoked_function_arn.split(':')[3]
                  log_data['processed_time'] = context.aws_request_id
                  
                  # Re-encode
                  output_data = json.dumps(log_data) + '
'
                  encoded = base64.b64encode(output_data.encode()).decode()
                  
                  output.append({
                      'recordId': record['recordId'],
                      'result': 'Ok',
                      'data': encoded
                  })
              
              return {'records': output}
      Timeout: 60
      MemorySize: 512
```

**3. CloudWatch Cross-Account Dashboard:**

**Log Archive Account - Dashboard:**
```json
{
  "widgets": [
    {
      "type": "log",
      "properties": {
        "query": "SOURCE '/aws/lambda/account-111111111111' | SOURCE '/aws/lambda/account-222222222222' | fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 100",
        "region": "us-east-1",
        "title": "Errors Across All Accounts",
        "queryId": "abcd1234-5678-90ab-cdef-1234567890ab"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/Lambda", "Errors", {"stat": "Sum", "accountId": "111111111111"}],
          ["...", {"stat": "Sum", "accountId": "222222222222"}]
        ],
        "period": 300,
        "stat": "Sum",
        "region": "us-east-1",
        "title": "Lambda Errors by Account"
      }
    }
  ]
}
```

**4. Athena for Log Analysis:**

**Glue Crawler for S3 Logs:**
```yaml
Resources:
  CloudTrailGlueCrawler:
    Type: AWS::Glue::Crawler
    Properties:
      Name: cloudtrail-logs-crawler
      Role: !GetAtt GlueServiceRole.Arn
      DatabaseName: security_logs
      Targets:
        S3Targets:
          - Path: s3://org-cloudtrail-logs-123456789012/AWSLogs/
            Exclusions:
              - '**.json'
      SchemaChangePolicy:
        UpdateBehavior: UPDATE_IN_DATABASE
        DeleteBehavior: LOG
      Schedule:
        ScheduleExpression: 'cron(0 */6 * * ? *)'  # Every 6 hours
```

**Athena Queries:**
```sql
-- Find all S3 bucket policy changes
SELECT 
    useridentity.principalid,
    eventtime,
    eventname,
    awsregion,
    sourceipaddress,
    requestparameters
FROM cloudtrail_logs
WHERE 
    eventsource = 's3.amazonaws.com'
    AND eventname IN ('PutBucketPolicy', 'DeleteBucketPolicy')
    AND eventtime > current_date - interval '7' day
ORDER BY eventtime DESC;

-- Identify unauthorized API calls
SELECT 
    useridentity.arn,
    eventname,
    errorcode,
    errormessage,
    sourceipaddress,
    COUNT(*) as attempt_count
FROM cloudtrail_logs
WHERE 
    errorcode IN ('AccessDenied', 'UnauthorizedOperation')
    AND eventtime > current_date - interval '1' day
GROUP BY 
    useridentity.arn,
    eventname,
    errorcode,
    errormessage,
    sourceipaddress
HAVING COUNT(*) > 10  -- Potential brute force
ORDER BY attempt_count DESC;

-- Track resource deletions
SELECT 
    useridentity.principalid,
    eventtime,
    eventname,
    requestparameters,
    responseelements
FROM cloudtrail_logs
WHERE 
    eventname LIKE '%Delete%'
    AND eventtime > current_date - interval '30' day
ORDER BY eventtime DESC;
```

**5. Real-Time Alerting with EventBridge:**

```yaml
Resources:
  SecurityAlertRule:
    Type: AWS::Events::Rule
    Properties:
      Name: unauthorized-api-calls
      EventPattern:
        source:
          - aws.cloudtrail
        detail-type:
          - AWS API Call via CloudTrail
        detail:
          errorCode:
            - AccessDenied
            - UnauthorizedOperation
      State: ENABLED
      Targets:
        - Arn: !Ref SecurityAlertTopic
          Id: SNSTarget
          InputTransformer:
            InputPathsMap:
              user: $.detail.userIdentity.principalId
              eventName: $.detail.eventName
              sourceIP: $.detail.sourceIPAddress
              time: $.detail.eventTime
            InputTemplate: |
              "Security Alert: Unauthorized API Call"
              "User: "
              "Action: "
              "Source IP: "
              "Time: "

  RootAccountUsageRule:
    Type: AWS::Events::Rule
    Properties:
      Name: root-account-usage
      EventPattern:
        source:
          - aws.cloudtrail
        detail-type:
          - AWS API Call via CloudTrail
        detail:
          userIdentity:
            type:
              - Root
      State: ENABLED
      Targets:
        - Arn: !Ref CriticalAlertTopic
          Id: SNSTarget
```

**6. Custom Metrics with Metric Filters:**

```bash
# Create metric filter for application errors
aws logs put-metric-filter \
  --log-group-name /aws/lambda/my-app \
  --filter-name AppErrorCount \
  --filter-pattern '[ERROR]' \
  --metric-transformations \
    metricName=ApplicationErrors,\
metricNamespace=CustomApp,\
    metricValue=1,\
    defaultValue=0

# Create alarm on custom metric
aws cloudwatch put-metric-alarm \
  --alarm-name high-application-errors \
  --alarm-description "Application error rate > 10/min" \
  --metric-name ApplicationErrors \
  --namespace CustomApp \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-team
```

**7. Performance Monitoring - X-Ray:**

```python
# Lambda function with X-Ray tracing
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all
import boto3

patch_all()  # Auto-instrument AWS SDK calls

@xray_recorder.capture('process_order')
def lambda_handler(event, context):
    
    # Custom subsegment
    subsegment = xray_recorder.begin_subsegment('database_query')
    try:
        # Database operation
        result = query_database()
        subsegment.put_metadata('rows_returned', len(result))
    finally:
        xray_recorder.end_subsegment()
    
    # Annotate for filtering
    xray_recorder.put_annotation('customer_id', event['customerId'])
    xray_recorder.put_annotation('order_total', event['total'])
    
    return {'statusCode': 200, 'body': 'Success'}
```

**8. Cost Monitoring:**

```yaml
Resources:
  CostAnomalyDetector:
    Type: AWS::CE::AnomalyMonitor
    Properties:
      MonitorName: daily-cost-monitor
      MonitorType: DIMENSIONAL
      MonitorDimension: SERVICE

  CostAnomalySubscription:
    Type: AWS::CE::AnomalySubscription
    Properties:
      SubscriptionName: cost-anomaly-alerts
      Frequency: DAILY
      MonitorArnList:
        - !GetAtt CostAnomalyDetector.MonitorArn
      Subscribers:
        - Type: SNS
          Address: !Ref FinanceAlertTopic
      Threshold: 100  # Alert if $100+ anomaly
```

**Real-World Troubleshooting Example:**

**Scenario:** Application experiencing intermittent 500 errors

**My Investigation Process:**

1. **CloudWatch Logs Insights:**
```sql
fields @timestamp, @message, @requestId
| filter @message like /500/
| stats count() by bin(5m) as time_bucket
| sort time_bucket desc
```
Result: Spikes every 15 minutes

2. **X-Ray Service Map:**
- Identified high latency in DynamoDB calls
- 5% of requests timing out

3. **CloudWatch Metrics:**
```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ConsumedReadCapacityUnits \
  --dimensions Name=TableName,Value=Orders \
  --start-time 2024-01-15T00:00:00Z \
  --end-time 2024-01-15T23:59:59Z \
  --period 300 \
  --statistics Maximum
```
Result: Provisioned capacity exceeded during peaks

4. **Solution:** Enabled DynamoDB Auto Scaling
```yaml
ScalingPolicy:
  Type: AWS::ApplicationAutoScaling::ScalingPolicy
  Properties:
    PolicyName: DynamoDBReadAutoScaling
    PolicyType: TargetTrackingScaling
    ScalingTargetId: !Ref ScalingTarget
    TargetTrackingScalingPolicyConfiguration:
      TargetValue: 70.0
      PredefinedMetricSpecification:
        PredefinedMetricType: DynamoDBReadCapacityUtilization
```

**Monitoring Best Practices:**
1. **Log retention:** 30 days hot (CloudWatch), 365 days warm (S3 Standard), 7 years cold (Glacier)
2. **Alert fatigue:** Tune thresholds - start conservative, tighten based on patterns
3. **Dashboards:** Role-specific (Ops, Security, Finance), not one-size-fits-all
4. **Anomaly detection:** AWS-native (CloudWatch Anomaly Detection) better than static thresholds
5. **Cost vs value:** CloudWatch Logs expensive - filter at source, don't log debug in prod"

---

## SECTION 7: DISASTER RECOVERY & HIGH AVAILABILITY

### Q11: Design a disaster recovery strategy with RTO of 1 hour and RPO of 5 minutes
**Expert Answer:**
"For RTO 1 hour / RPO 5 minutes, I'd implement a Warm Standby architecture:

**RPO (Recovery Point Objective) = 5 minutes:**
Maximum acceptable data loss = 5 minutes

**RTO (Recovery Time Objective) = 1 hour:**
Maximum acceptable downtime = 60 minutes

**Architecture:**
```
Primary Region (us-east-1)          DR Region (us-west-2)
├── RDS Multi-AZ (sync)              ├── RDS Read Replica (async, lag <30s)
├── S3 (primary)                     ├── S3 Cross-Region Replication (CRR)
├── DynamoDB (primary)               ├── DynamoDB Global Tables
├── ALB + ASG (active)               ├── ALB + ASG (standby, min capacity)
└── Route 53 (primary)               └── Route 53 (failover)
```

**1. Database Replication:**

**RDS Cross-Region Read Replica:**
```bash
# Create encrypted read replica in DR region
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-db-dr-replica \
  --source-db-instance-identifier arn:aws:rds:us-east-1:123456789012:db:prod-db-primary \
  --db-instance-class db.r6g.xlarge \
  --availability-zone us-west-2a \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-west-2:123456789012:key/abcd1234 \
  --publicly-accessible false \
  --auto-minor-version-upgrade false \
  --monitoring-interval 60 \
  --monitoring-role-arn arn:aws:iam::123456789012:role/rds-monitoring-role

# Monitor replication lag (CRITICAL for RPO)
aws cloudwatch put-metric-alarm \
  --alarm-name rds-replication-lag-high \
  --alarm-description "RDS replica lag > 300 seconds" \
  --metric-name ReplicaLag \
  --namespace AWS/RDS \
  --statistic Average \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 300 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=DBInstanceIdentifier,Value=prod-db-dr-replica \
  --alarm-actions arn:aws:sns:us-west-2:123456789012:critical-alerts
```

**Replica Lag Monitoring:**
- Target: < 30 seconds (well under 5-minute RPO)
- Alert threshold: > 300 seconds (5 minutes)
- Auto-promote if primary region fails

**2. S3 Cross-Region Replication:**

```json
{
  "Role": "arn:aws:iam::123456789012:role/S3-CRR-Role",
  "Rules": [
    {
      "Status": "Enabled",
      "Priority": 1,
      "DeleteMarkerReplication": {
        "Status": "Enabled"
      },
      "Filter": {},
      "Destination": {
        "Bucket": "arn:aws:s3:::prod-data-dr-us-west-2",
        "ReplicationTime": {
          "Status": "Enabled",
          "Time": {
            "Minutes": 15
          }
        },
        "Metrics": {
          "Status": "Enabled",
          "EventThreshold": {
            "Minutes": 15
          }
        },
        "EncryptionConfiguration": {
          "ReplicaKmsKeyID": "arn:aws:kms:us-west-2:123456789012:key/replica-key"
        },
        "StorageClass": "STANDARD_IA",
        "AccessControlTranslation": {
          "Owner": "Destination"
        },
        "Account": "123456789012"
      },
      "SourceSelectionCriteria": {
        "SseKmsEncryptedObjects": {
          "Status": "Enabled"
        },
        "ReplicaModifications": {
          "Status": "Enabled"
        }
      }
    }
  ]
}
```

**S3 Replication Monitoring:**
```bash
# Replication metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/S3 \
  --metric-name ReplicationLatency \
  --dimensions Name=SourceBucket,Value=prod-data-primary \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Maximum
```

**3. DynamoDB Global Tables (Active-Active):**

```python
import boto3

dynamodb = boto3.client('dynamodb')

# Create global table (v2)
response = dynamodb.create-global-table(
    GlobalTableName='Orders',
    ReplicationGroup=[
        {'RegionName': 'us-east-1'},
        {'RegionName': 'us-west-2'}
    ]
)

# Update table settings for both regions
dynamodb.update_table(
    TableName='Orders',
    BillingMode='PAY_PER_REQUEST',  # Auto-scales
    StreamSpecification={
        'StreamEnabled': True,
        'StreamViewType': 'NEW_AND_OLD_IMAGES'
    },
    SSESpecification={
        'Enabled': True,
        'SSEType': 'KMS',
        'KMSMasterKeyId': 'arn:aws:kms:us-east-1:123456789012:key/abcd1234'
    }
)
```

**Replication Lag Monitoring:**
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name dynamodb-replication-lag \
  --metric-name ReplicationLatency \
  --namespace AWS/DynamoDB \
  --statistic Average \
  --period 60 \
  --evaluation-periods 3 \
  --threshold 300000  # 5 minutes in milliseconds \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=TableName,Value=Orders Name=ReceivingRegion,Value=us-west-2
```

**4. Route 53 Failover Configuration:**

```json
{
  "Changes": [
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "Primary-US-East-1",
        "Failover": "PRIMARY",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "prod-alb-us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        },
        "HealthCheckId": "abc123-primary-health-check"
      }
    },
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "Secondary-US-West-2",
        "Failover": "SECONDARY",
        "AliasTarget": {
          "HostedZoneId": "Z3DZXE0Q79N41H",
          "DNSName": "prod-alb-us-west-2.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }
  ]
}
```

**Health Check Configuration:**
```bash
aws route53 create-health-check \
  --type HTTPS \
  --resource-path /health \
  --fully-qualified-domain-name prod-alb-us-east-1.elb.amazonaws.com \
  --request-interval 30 \
  --failure-threshold 3 \
  --health-threshold 2 \
  --measure-latency \
  --alarm-identifier Region=us-east-1,Name=primary-region-unhealthy
```

**5. DR Site (Warm Standby):**

**Infrastructure (Terraform):**
```hcl
# DR Region - us-west-2
module "dr_compute" {
  source = "./modules/compute"
  
  region     = "us-west-2"
  vpc_id     = module.dr_network.vpc_id
  subnet_ids = module.dr_network.private_subnet_ids
  
  # Warm standby: Minimum capacity running
  asg_min_size     = 2   # vs 10 in primary
  asg_max_size     = 20  # vs 50 in primary
  asg_desired_size = 2
  
  instance_type = "t3.medium"  # vs t3.xlarge in primary
  
  # Ready to scale up quickly
  scaling_policies_enabled = true
}
```

**6. Failover Automation (Lambda):**

```python
import boto3
import os

rds = boto3.client('rds', region_name='us-west-2')
asg = boto3.client('autoscaling', region_name='us-west-2')
route53 = boto3.client('route53')

def lambda_handler(event, context):
    """
    Automated DR failover procedure
    Triggered by CloudWatch Alarm or manual invoke
    """
    
    # Step 1: Promote RDS Read Replica to standalone instance
    print("Promoting RDS read replica...")
    try:
        rds.promote_read_replica(
            DBInstanceIdentifier='prod-db-dr-replica',
            BackupRetentionPeriod=7,
            PreferredBackupWindow='03:00-04:00'
        )
        
        # Wait for promotion to complete
        waiter = rds.get_waiter('db_instance_available')
        waiter.wait(
            DBInstanceIdentifier='prod-db-dr-replica',
            WaiterConfig={'Delay': 30, 'MaxAttempts': 40}
        )
        print("RDS promotion complete")
        
    except Exception as e:
        print(f"RDS promotion error: {str(e)}")
        raise
    
    # Step 2: Scale up Auto Scaling Group
    print("Scaling up DR Auto Scaling Group...")
    try:
        asg.update_auto_scaling_group(
            AutoScalingGroupName='dr-app-asg',
            MinSize=10,
            MaxSize=50,
            DesiredCapacity=10
        )
        print("ASG scaled up to production capacity")
        
    except Exception as e:
        print(f"ASG scale-up error: {str(e)}")
        raise
    
    # Step 3: Update Route 53 (force failover)
    print("Updating Route 53 to DR region...")
    try:
        route53.change_resource_record_sets(
            HostedZoneId=os.environ['HOSTED_ZONE_ID'],
            ChangeBatch={
                'Changes': [
                    {
                        'Action': 'UPSERT',
                        'ResourceRecordSet': {
                            'Name': 'app.example.com',
                            'Type': 'A',
                            'SetIdentifier': 'Primary-US-West-2-DR',
                            'Failover': 'PRIMARY',
                            'AliasTarget': {
                                'HostedZoneId': 'Z3DZXE0Q79N41H',
                                'DNSName': 'prod-alb-us-west-2.elb.amazonaws.com',
                                'EvaluateTargetHealth': True
                            }
                        }
                    }
                ]
            }
        )
        print("Route 53 updated successfully")
        
    except Exception as e:
        print(f"Route 53 update error: {str(e)}")
        raise
    
    # Step 4: Notify operations team
    sns = boto3.client('sns')
    sns.publish(
        TopicArn=os.environ['SNS_TOPIC_ARN'],
        Subject='DR Failover Completed - Action Required',
        Message=f'''
        DR failover to us-west-2 completed successfully:
        
        - RDS read replica promoted
        - ASG scaled to production capacity
        - Route 53 updated to DR region
        
        Verify application functionality:
        https://app.example.com
        
        RTO Target: 1 hour
        Actual RTO: {context.get_remaining_time_in_millis() / 60000:.1f} minutes
        '''
    )
    
    return {
        'statusCode': 200,
        'body': 'DR failover initiated successfully'
    }
```

**7. Recovery Time Breakdown:**

| Step | Action | Time | Cumulative |
|------|--------|------|-----------|
| 1 | Detect primary region failure | 2 min | 2 min |
| 2 | Promote RDS read replica | 15 min | 17 min |
| 3 | Scale up DR ASG (2→10 instances) | 5 min | 22 min |
| 4 | DNS propagation (Route 53) | 1 min | 23 min |
| 5 | Application warm-up | 5 min | 28 min |
| 6 | Smoke tests & validation | 10 min | 38 min |
| **Total RTO** | | | **38 minutes** ✓ |

**RTO Buffer: 22 minutes** (target 60 min, actual 38 min)

**8. DR Testing (Quarterly):**

```bash
#!/bin/bash
# DR Test Runbook

echo "===== DR FAILOVER TEST START ====="

# 1. Take snapshot of primary RDS
echo "Creating RDS snapshot..."
aws rds create-db-snapshot \
  --db-instance-identifier prod-db-primary \
  --db-snapshot-identifier dr-test-$(date +%Y%m%d-%H%M)

# 2. Simulate failover (non-destructive)
echo "Invoking DR Lambda in test mode..."
aws lambda invoke \
  --function-name dr-failover-automation \
  --payload '{"test_mode": true}' \
  --region us-west-2 \
  /tmp/dr-test-result.json

# 3. Verify DR site
echo "Testing DR application endpoint..."
curl -f https://dr.app.example.com/health || echo "DR health check failed!"

# 4. Check RDS replication lag
LAG=$(aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name ReplicaLag \
  --dimensions Name=DBInstanceIdentifier,Value=prod-db-dr-replica \
  --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average \
  --query 'Datapoints[0].Average' \
  --output text)

echo "Current replication lag: ${LAG} seconds"
if (( $(echo "$LAG > 300" | bc -l) )); then
  echo "WARNING: Replication lag exceeds 5 minutes!"
fi

# 5. Cleanup (rollback)
echo "Rolling back DR test changes..."
# Reset ASG to warm standby capacity
# Keep RDS replica running

echo "===== DR TEST COMPLETE ====="
```

**9. Cost Optimization:**

**Primary Region Costs (Monthly):**
- Compute: 10 × t3.xlarge × 730h × $0.1664 = $1,215
- RDS: db.r6g.xlarge Multi-AZ × 730h × $0.48 = $350
- Data transfer: 5 TB × $0.09 = $450
- **Total: $2,015/month**

**DR Region Costs (Monthly):**
- Compute: 2 × t3.medium × 730h × $0.0416 = $61
- RDS Replica: db.r6g.xlarge × 730h × $0.24 = $175
- S3 Replication: 1 TB × $0.02 = $20
- **Total: $256/month (13% of primary)**

**Total DR Cost: $256/month for RTO 1hr / RPO 5min**

**10. Disaster Scenarios Handled:**

✓ Region outage (AWS)
✓ AZ failure (Multi-AZ handles)
✓ Database corruption (PITR + snapshots)
✓ Application bug (Blue-green deployment + rollback)
✓ Security incident (Isolated DR environment)
✓ Data center failure (Multi-region)

**Real Experience:**
In 2023, we had a region-wide S3 outage affecting us-east-1. Our DR automation:
- Detected failure in 90 seconds (health check)
- Auto-triggered failover Lambda
- Promoted RDS replica: 12 minutes
- Scaled ASG: 4 minutes
- DNS switched: 60 seconds
- **Total downtime: 18 minutes** (well under 1-hour RTO)
- Zero data loss (RPO < 30 seconds actual)

Post-incident, we reduced health check interval from 2 min to 30 sec, improving detection time."

---

## SECTION 8: SECURITY & COMPLIANCE

### Q12: How do you implement and audit security controls for compliance (HIPAA, PCI-DSS, SOC 2)?
**Expert Answer:**
"I've implemented compliance frameworks for regulated industries. Here's my comprehensive approach:

**Compliance Framework Implementation:**

**1. Data Classification & Tagging:**

```json
// Mandatory Tag Policy (Organization SCP)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": [
        "ec2:RunInstances",
        "rds:CreateDBInstance",
        "s3:CreateBucket"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotLike": {
          "aws:RequestTag/DataClassification": [
            "Public",
            "Internal",
            "Confidential",
            "Restricted"
          ]
        }
      }
    },
    {
      "Effect": "Deny",
      "Action": [
        "ec2:RunInstances",
        "rds:CreateDBInstance",
        "s3:CreateBucket"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotLike": {
          "aws:RequestTag/ComplianceScope": [
            "HIPAA",
            "PCI-DSS",
            "SOC2",
            "None"
          ]
        }
      }
    }
  ]
}
```

**Lambda Auto-Tagger (Remediation):**
```python
import boto3
import json

def lambda_handler(event, context):
    """
    Auto-tag resources based on attributes
    Triggered by CloudTrail CreateResource events
    """
    
    ec2 = boto3.client('ec2')
    resource_id = event['detail']['responseElements']['instancesSet']['items'][0]['instanceId']
    
    # Determine classification from subnet or security group
    subnet_id = event['detail']['requestParameters']['subnetId']
    
    # Check if subnet is in PHI/PCI VPC
    vpc = ec2.describe_subnets(SubnetIds=[subnet_id])
    vpc_tags = vpc['Subnets'][0].get('Tags', [])
    
    compliance_scope = 'None'
    for tag in vpc_tags:
        if tag['Key'] == 'ComplianceScope':
            compliance_scope = tag['Value']
            break
    
    # Apply tags
    ec2.create_tags(
        Resources=[resource_id],
        Tags=[
            {'Key': 'DataClassification', 'Value': 'Restricted'},
            {'Key': 'ComplianceScope', 'Value': compliance_scope},
            {'Key': 'AutoTagged', 'Value': 'true'},
            {'Key': 'TaggedBy', 'Value': 'Lambda-Auto-Tagger'}
        ]
    )
    
    return {'statusCode': 200, 'body': 'Tags applied'}
```

**2. Encryption Everywhere:**

**KMS Key Policy (HIPAA-compliant):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow services to use the key",
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "s3.amazonaws.com",
          "rds.amazonaws.com",
          "logs.amazonaws.com"
        ]
      },
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": [
            "s3.us-east-1.amazonaws.com",
            "rds.us-east-1.amazonaws.com"
          ],
          "aws:PrincipalOrgID": "o-abcd1234"
        }
      }
    },
    {
      "Sid": "Deny unencrypted uploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    }
  ]
}
```

**Config Rules for Encryption Compliance:**
```yaml
Resources:
  EncryptionComplianceRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: hipaa-encryption-at-rest
      Description: Ensures all resources have encryption at rest
      Source:
        Owner: AWS
        SourceIdentifier: ENCRYPTED_VOLUMES
      Scope:
        ComplianceResourceTypes:
          - AWS::EC2::Volume
          - AWS::RDS::DBInstance
          - AWS::S3::Bucket
          - AWS::DynamoDB::Table
          - AWS::EFS::FileSystem
          - AWS::Redshift::Cluster

  EncryptionInTransit:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: pci-encryption-in-transit
      Source:
        Owner: AWS
        SourceIdentifier: ALB_HTTP_TO_HTTPS_REDIRECTION_CHECK
```

**3. Network Isolation (PCI-DSS Cardholder Data Environment):**

**CDE VPC (Isolated):**
```yaml
Resources:
  # PCI-DSS CDE VPC - Completely isolated
  CDEVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.100.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: PCI-CDE-VPC
        - Key: ComplianceScope
          Value: PCI-DSS
        - Key: Environment
          Value: Production

  # NO INTERNET GATEWAY - Internal only
  # NO NAT GATEWAY - No outbound internet
  # NO PEERING to non-compliant VPCs

  # VPC Flow Logs (Required for PCI-DSS 10.x)
  FlowLogRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: vpc-flow-logs.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: CloudWatchLogPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource: '*'

  FlowLog:
    Type: AWS::EC2::FlowLog
    Properties:
      ResourceType: VPC
      ResourceId: !Ref CDEVPC
      TrafficType: ALL
      LogDestinationType: cloud-watch-logs
      LogGroupName: /aws/vpc/pci-cde
      DeliverLogsPermissionArn: !GetAtt FlowLogRole.Arn
      Tags:
        - Key: Compliance
          Value: PCI-DSS-Requirement-10

  # Network ACL (Explicit deny by default)
  PrivateNACL:
    Type: AWS::EC2::NetworkAcl
    Properties:
      VpcId: !Ref CDEVPC
      Tags:
        - Key: Name
          Value: PCI-CDE-Private-NACL

  # Deny all inbound except from application tier
  NACLInboundRule:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref PrivateNACL
      RuleNumber: 100
      Protocol: 6  # TCP
      RuleAction: allow
      CidrBlock: 10.100.1.0/24  # Application tier only
      PortRange:
        From: 3306
        To: 3306

  NACLOutboundRule:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref PrivateNACL
      RuleNumber: 100
      Protocol: 6
      Egress: true
      RuleAction: allow
      CidrBlock: 10.100.1.0/24
      PortRange:
        From: 1024
        To: 65535  # Ephemeral ports
```

**4. Access Control (RBAC + Least Privilege):**

**SSO with MFA Enforcement:**
```yaml
Resources:
  MFAEnforcementPolicy:
    Type: AWS::IAM::ManagedPolicy
    Properties:
      ManagedPolicyName: Require-MFA-Policy
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Sid: DenyAllExceptListedIfNoMFA
            Effect: Deny
            NotAction:
              - iam:CreateVirtualMFADevice
              - iam:EnableMFADevice
              - iam:ListMFADevices
              - iam:ListVirtualMFADevices
              - iam:ResyncMFADevice
              - sts:GetSessionToken
            Resource: "*"
            Condition:
              BoolIfExists:
                aws:MultiFactorAuthPresent: false
```

**Privileged Access Management:**
```yaml
  PrivilegedAccessRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: PrivilegedDatabaseAccess
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
            Action: sts:AssumeRole
            Condition:
              StringEquals:
                sts:ExternalId: !Ref ExternalId
              IpAddress:
                aws:SourceIp:
                  - 203.0.113.0/24  # Corporate VPN only
              Bool:
                aws:MultiFactorAuthPresent: true
              NumericLessThan:
                aws:MultiFactorAuthAge: 3600  # MFA < 1 hour old
      Policies:
        - PolicyName: DatabaseAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - rds:DescribeDBInstances
                  - rds-db:connect
                Resource:
                  - !Sub 'arn:aws:rds:${AWS::Region}:${AWS::AccountId}:db:pci-cardholder-db'
                Condition:
                  DateGreaterThan:
                    aws:CurrentTime: "2024-01-01T09:00:00Z"
                  DateLessThan:
                    aws:CurrentTime: "2024-01-01T17:00:00Z"  # Business hours only
```

**5. Audit Logging (Immutable):**

**CloudTrail with Log File Validation:**
```yaml
Resources:
  ComplianceTrail:
    Type: AWS::CloudTrail::Trail
    Properties:
      TrailName: compliance-audit-trail
      S3BucketName: !Ref AuditLogsBucket
      IncludeGlobalServiceEvents: true
      IsLogging: true
      IsMultiRegionTrail: true
      EnableLogFileValidation: true  # Tamper detection
      KMSKeyId: !GetAtt CloudTrailKMSKey.Arn
      EventSelectors:
        - IncludeManagementEvents: true
          ReadWriteType: All
          DataResources:
            - Type: AWS::S3::Object
              Values:
                - !Sub '${PHIBucket.Arn}/*'  # Log all PHI access
            - Type: AWS::Lambda::Function
              Values:
                - !Sub 'arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function/*'

  # S3 Bucket with Object Lock (WORM - Write Once Read Many)
  AuditLogsBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub 'compliance-audit-logs-${AWS::AccountId}'
      ObjectLockEnabled: true
      ObjectLockConfiguration:
        ObjectLockEnabled: Enabled
        Rule:
          DefaultRetention:
            Mode: GOVERNANCE  # or COMPLIANCE for immutable
            Years: 7  # Retention period
      VersioningConfiguration:
        Status: Enabled
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: aws:kms
              KMSMasterKeyID: !GetAtt AuditLogsKMSKey.Arn
```

**6. Automated Compliance Checks:**

**Config Conformance Packs (Pre-built Compliance):**
```bash
# Deploy HIPAA Conformance Pack
aws configservice put-conformance-pack \
  --conformance-pack-name hipaa-conformance-pack \
  --template-s3-uri s3://aws-configservice-conformancepacks-us-east-1/Operational-Best-Practices-for-HIPAA-Security.yaml \
  --delivery-s3-bucket config-delivery-bucket

# Deploy PCI-DSS Conformance Pack
aws configservice put-conformance-pack \
  --conformance-pack-name pci-dss-conformance-pack \
  --template-s3-uri s3://aws-configservice-conformancepacks-us-east-1/Operational-Best-Practices-for-PCI-DSS-3.2.1.yaml
```

**Custom Config Rules (Lambda-based):**
```python
import boto3
import json

config = boto3.client('config')

def evaluate_compliance(configuration_item):
    """
    Check if RDS instance in PCI scope has:
    - Encryption enabled
    - Backup enabled
    - Multi-AZ enabled
    - Public accessibility disabled
    - Audit logging enabled
    """
    
    compliance_type = 'NON_COMPLIANT'
    annotation = ''
    
    if configuration_item['resourceType'] != 'AWS::RDS::DBInstance':
        return 'NOT_APPLICABLE', ''
    
    # Check if tagged for PCI-DSS
    tags = configuration_item.get('tags', {})
    if tags.get('ComplianceScope') != 'PCI-DSS':
        return 'NOT_APPLICABLE', 'Not in PCI scope'
    
    config_data = configuration_item['configuration']
    
    checks = {
        'Encrypted': config_data.get('storageEncrypted', False),
        'BackupRetention': config_data.get('backupRetentionPeriod', 0) >= 7,
        'MultiAZ': config_data.get('multiAZ', False),
        'PubliclyAccessible': not config_data.get('publiclyAccessible', True),
        'AuditLogging': bool(config_data.get('enabledCloudwatchLogsExports', []))
    }
    
    failed_checks = [k for k, v in checks.items() if not v]
    
    if failed_checks:
        compliance_type = 'NON_COMPLIANT'
        annotation = f"Failed checks: {', '.join(failed_checks)}"
    else:
        compliance_type = 'COMPLIANT'
        annotation = 'All PCI-DSS controls met'
    
    return compliance_type, annotation

def lambda_handler(event, context):
    invoking_event = json.loads(event['invokingEvent'])
    configuration_item = invoking_event['configurationItem']
    
    compliance_type, annotation = evaluate_compliance(configuration_item)
    
    config.put_evaluations(
        Evaluations=[
            {
                'ComplianceResourceType': configuration_item['resourceType'],
                'ComplianceResourceId': configuration_item['resourceId'],
                'ComplianceType': compliance_type,
                'Annotation': annotation,
                'OrderingTimestamp': configuration_item['configurationItemCaptureTime']
            }
        ],
        ResultToken=event['resultToken']
    )
```

**7. Incident Response Automation:**

```yaml
Resources:
  SecurityIncidentRule:
    Type: AWS::Events::Rule
    Properties:
      Name: security-incident-response
      EventPattern:
        source:
          - aws.guardduty
        detail-type:
          - GuardDuty Finding
        detail:
          severity:
            - 7  # High
            - 8  # Critical
            - 9
      Targets:
        - Arn: !GetAtt IncidentResponseLambda.Arn
          Id: IncidentResponse

  IncidentResponseLambda:
    Type: AWS::Lambda::Function
    Properties:
      Runtime: python3.11
      Handler: index.lambda_handler
      Code:
        ZipFile: |
          import boto3
          import json
          from datetime import datetime
          
          ec2 = boto3.client('ec2')
          sns = boto3.client('sns')
          
          def lambda_handler(event, context):
              """
              Automated incident response:
              1. Isolate compromised instance
              2. Take forensic snapshot
              3. Notify security team
              4. Create Security Hub finding
              """
              
              finding = event['detail']
              finding_type = finding['type']
              severity = finding['severity']
              
              # Extract affected resource
              resource = finding['resource']
              instance_id = resource['instanceDetails']['instanceId']
              
              print(f"High-severity finding: {finding_type}")
              print(f"Affected instance: {instance_id}")
              
              # Step 1: Isolate instance (apply isolation security group)
              try:
                  ec2.modify_instance_attribute(
                      InstanceId=instance_id,
                      Groups=['sg-isolation-quarantine']  # Pre-created SG with no rules
                  )
                  print(f"Instance {instance_id} isolated")
              except Exception as e:
                  print(f"Failed to isolate: {str(e)}")
              
              # Step 2: Create forensic snapshot
              volumes = ec2.describe_volumes(
                  Filters=[{'Name': 'attachment.instance-id', 'Values': [instance_id]}]
              )['Volumes']
              
              for volume in volumes:
                  snapshot = ec2.create_snapshot(
                      VolumeId=volume['VolumeId'],
                      Description=f"Forensic snapshot - {finding_type}",
                      TagSpecifications=[{
                          'ResourceType': 'snapshot',
                          'Tags': [
                              {'Key': 'Type', 'Value': 'Forensic'},
                              {'Key': 'IncidentId', 'Value': finding['id']},
                              {'Key': 'Timestamp', 'Value': datetime.utcnow().isoformat()}
                          ]
                      }]
                  )
                  print(f"Snapshot created: {snapshot['SnapshotId']}")
              
              # Step 3: Notify security team
              sns.publish(
                  TopicArn='arn:aws:sns:us-east-1:123456789012:security-incidents',
                  Subject=f'CRITICAL: Security Incident - {finding_type}',
                  Message=json.dumps(finding, indent=2)
              )
              
              return {'statusCode': 200, 'body': 'Incident response executed'}
```

**8. Continuous Compliance Monitoring Dashboard:**

```python
# Generate compliance report
import boto3
from datetime import datetime

config = boto3.client('config')
securityhub = boto3.client('securityhub')

def generate_compliance_report():
    """
    Daily compliance report for auditors
    """
    
    # Config compliance summary
    config_summary = config.describe_compliance_by_config_rule()
    
    compliant = 0
    non_compliant = 0
    
    for rule in config_summary['ComplianceByConfigRules']:
        compliance = rule['Compliance']['ComplianceType']
        if compliance == 'COMPLIANT':
            compliant += 1
        elif compliance == 'NON_COMPLIANT':
            non_compliant += 1
    
    compliance_percentage = (compliant / (compliant + non_compliant)) * 100
    
    # Security Hub findings summary
    findings = securityhub.get_findings(
        Filters={
            'RecordState': [{'Value': 'ACTIVE', 'Comparison': 'EQUALS'}],
            'ComplianceStatus': [{'Value': 'FAILED', 'Comparison': 'EQUALS'}]
        }
    )
    
    critical_findings = len([f for f in findings['Findings'] if f['Severity']['Label'] == 'CRITICAL'])
    high_findings = len([f for f in findings['Findings'] if f['Severity']['Label'] == 'HIGH'])
    
    report = f"""
    COMPLIANCE REPORT - {datetime.now().strftime('%Y-%m-%d')}
    
    CONFIG RULES:
    - Compliant: {compliant}
    - Non-Compliant: {non_compliant}
    - Compliance Rate: {compliance_percentage:.1f}%
    
    SECURITY HUB:
    - Critical Findings: {critical_findings}
    - High Findings: {high_findings}
    
    ACTION REQUIRED:
    {non_compliant} non-compliant resources need remediation
    {critical_findings + high_findings} security findings require immediate attention
    """
    
    return report
```

**Real Compliance Audit Experience:**

During our last SOC 2 Type II audit:
- **Auditor requested:** "Show all access to customer data in last 90 days"
- **My response:** Ran Athena query on CloudTrail logs
```sql
SELECT 
    useridentity.principalid,
    eventtime,
    eventname,
    requestparameters,
    resources[1].arn
FROM cloudtrail_logs
WHERE 
    resources[1].arn LIKE '%customer-data%'
    AND eventtime > current_date - interval '90' day
ORDER BY eventtime DESC;
```
- **Result:** Generated report in 2 minutes, showing zero unauthorized access
- **Auditor feedback:** "Most mature implementation we've seen"

**Key Success Factors:**
1. Automation (Config + Lambda) reduced manual audits by 90%
2. Immutable logs (S3 Object Lock) satisfied all audit requirements
3. Centralized logging made evidence gathering trivial
4. Preventive controls (SCPs) better than detective (Config Rules)"

---

## SECTION 9: COST OPTIMIZATION

### Q13: You notice AWS costs increased 40% this month. Walk me through your investigation and optimization process.
**Expert Answer:**
"40% cost spike requires systematic investigation. Here's my exact process:

**Phase 1: Immediate Triage (First 30 minutes)**

**1. Cost Anomaly Detection:**
```bash
# Check Cost Anomaly Detection alerts
aws ce get-anomalies \
  --start-date $(date -d '30 days ago' +%Y-%m-%d) \
  --max-results 100 \
  --query 'Anomalies[?Impact.TotalImpact>`100`]' \
  --output table
```

**2. Cost Explorer Analysis:**
```python
import boto3
from datetime import datetime, timedelta

ce = boto3.client('ce')

# Compare this month vs last month by service
end_date = datetime.now().strftime('%Y-%m-%d')
start_date = (datetime.now() - timedelta(days=30)).strftime('%Y-%m-%d')
last_month_start = (datetime.now() - timedelta(days=60)).strftime('%Y-%m-%d')

response = ce.get_cost_and_usage(
    TimePeriod={
        'Start': start_date,
        'End': end_date
    },
    Granularity='DAILY',
    Metrics=['UnblendedCost'],
    GroupBy=[
        {'Type': 'DIMENSION', 'Key': 'SERVICE'},
    ]
)

# Identify top cost increases
for group in response['ResultsByTime']:
    for cost_group in group['Groups']:
        service = cost_group['Keys'][0]
        cost = float(cost_group['Metrics']['UnblendedCost']['Amount'])
        print(f"{service}: ${cost:.2f}")
```

**Real Example - Root Cause Found:**

**Investigation Results:**
```
Service            This Month    Last Month    Δ
EC2                $8,500       $6,000       +42%
Data Transfer      $3,200       $800         +300%  ← ANOMALY!
RDS                $2,100       $2,000       +5%
S3                 $1,500       $1,400       +7%
```

**Data Transfer spike - drill down:**
```bash
# Analyze data transfer by resource
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity DAILY \
  --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=USAGE_TYPE \
  --filter file://filter.json

# filter.json
{
  "Dimensions": {
    "Key": "SERVICE",
    "Values": ["Amazon Elastic Compute Cloud - Compute"]
  }
}
```

**Result:** Found NAT Gateway data processing charges exploded

**Phase 2: Root Cause Analysis**

**VPC Flow Logs Analysis:**
```bash
# Query Flow Logs in CloudWatch Insights
aws logs start-query \
  --log-group-name /aws/vpc/flowlogs \
  --start-time $(date -d '7 days ago' +%s) \
  --end-time $(date +%s) \
  --query-string '
    fields @timestamp, srcaddr, dstaddr, bytes
    | filter dstaddr not like /^10\./
    | stats sum(bytes) as total_bytes by srcaddr
    | sort total_bytes desc
    | limit 20
  '
```

**Discovered:** A single EC2 instance (i-0abc123) transferring 5 TB/day to external IP

**Investigation:**
```bash
# Check instance tags
aws ec2 describe-instances --instance-ids i-0abc123 \
  --query 'Reservations[0].Instances[0].Tags'

# Result: dev-data-sync-server

# Check application logs
aws logs tail /aws/ec2/i-0abc123 --follow --since 1h
```

**Root Cause:** Developer wrote a script syncing entire S3 bucket (500 GB) to external service every hour (unnecessary), going through NAT Gateway

**Phase 3: Immediate Cost Reduction**

**1. Stop the bleeding:**
```bash
# Terminate runaway instance
aws ec2 stop-instances --instance-ids i-0abc123

# Estimate savings
# 5 TB/day × 30 days × $0.09/GB = $13,500/month saved
```

**2. Implement S3 VPC Gateway Endpoint (Free!):**
```yaml
Resources:
  S3GatewayEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VPC
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.s3'
      RouteTableIds:
        - !Ref PrivateRouteTable
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal: '*'
            Action:
              - s3:GetObject
              - s3:PutObject
            Resource: '*'
```

**Result:** S3 traffic now bypasses NAT Gateway → **$13,500/month saved**

**Phase 4: Comprehensive Cost Optimization**

**A. Right-Sizing EC2 Instances:**

```bash
# Get Cost Explorer right-sizing recommendations
aws ce get-rightsizing-recommendation \
  --service "Amazon Elastic Compute Cloud - Compute" \
  --page-size 100 \
  --query 'RightsizingRecommendations[?SavingsEstimateMonthly>`100`]'

# Example output:
# Instance: i-0def456
# Current: m5.2xlarge ($280/month)
# Recommended: m5.xlarge ($140/month)
# Savings: $140/month (50%)
# CPU Utilization: 15% average
```

**Compute Optimizer Integration:**
```python
import boto3

compute_optimizer = boto3.client('compute-optimizer')

recommendations = compute_optimizer.get_ec2_instance_recommendations(
    instanceArns=['arn:aws:ec2:us-east-1:123456789012:instance/i-0def456']
)

for rec in recommendations['instanceRecommendations']:
    print(f"Current: {rec['currentInstanceType']}")
    print(f"Finding: {rec['finding']}")
    
    for option in rec['recommendationOptions']:
        print(f"  Recommended: {option['instanceType']}")
        print(f"  Performance Risk: {option['performanceRisk']}")
        print(f"  Savings: ${rec['utilizationMetrics'][0]['value']}%")
```

**Implementation:**
```bash
# Create AMI from current instance
aws ec2 create-image \
  --instance-id i-0def456 \
  --name "rightsizing-snapshot-$(date +%Y%m%d)" \
  --no-reboot

# Launch new smaller instance
aws ec2 run-instances \
  --image-id ami-rightsizing-snapshot \
  --instance-type m5.xlarge \
  --subnet-id subnet-abc123 \
  --security-group-ids sg-xyz789

# Swap EIPs, update DNS
# Terminate old instance
```

**B. RDS Reserved Instances:**

**Current:** 5 × db.r6g.xlarge On-Demand = $2,100/month
**Optimized:** 5 × db.r6g.xlarge 1-year RI (All Upfront) = $1,260/month
**Savings:** $840/month (40%)

```bash
# Purchase Reserved Instance
aws rds purchase-reserved-db-instances-offering \
  --reserved-db-instances-offering-id abcd1234-5678-90ab-cdef \
  --reserved-db-instance-id prod-rds-ri-001 \
  --db-instance-count 5 \
  --offering-type "All Upfront"
```

**C. S3 Intelligent-Tiering:**

```python
import boto3

s3 = boto3.client('s3')

# Enable Intelligent-Tiering on all buckets
buckets = s3.list_buckets()['Buckets']

for bucket in buckets:
    bucket_name = bucket['Name']
    
    try:
        s3.put_bucket_intelligent_tiering_configuration(
            Bucket=bucket_name,
            Id='EntireLogsReduction',
            IntelligentTieringConfiguration={
                'Id': 'EntireLogsReduction',
                'Status': 'Enabled',
                'Tierings': [
                    {
                        'Days': 90,
                        'AccessTier': 'ARCHIVE_ACCESS'
                    },
                    {
                        'Days': 180,
                        'AccessTier': 'DEEP_ARCHIVE_ACCESS'
                    }
                ]
            }
        )
        print(f"Intelligent-Tiering enabled for {bucket_name}")
        
    except Exception as e:
        print(f"Error for {bucket_name}: {str(e)}")
```

**Savings Analysis:**
- **Before:** 10 TB × $0.023/GB (Standard) = $230/month
- **After:** 
  - 2 TB Standard (frequently accessed): $46
  - 8 TB Deep Archive: $8
  - **Total:** $54/month
  - **Savings:** $176/month (76%)

**D. Lambda Memory Optimization:**

```python
import boto3
import json

lambda_client = boto3.client('lambda')
cloudwatch = boto3.client('cloudwatch')

def optimize_lambda_memory(function_name):
    """
    Analyze Lambda memory usage and recommend optimal size
    """
    
    # Get current config
    function = lambda_client.get_function(FunctionName=function_name)
    current_memory = function['Configuration']['MemorySize']
    
    # Get memory utilization metrics
    metrics = cloudwatch.get_metric_statistics(
        Namespace='AWS/Lambda',
        MetricName='MemoryUtilization',
        Dimensions=[{'Name': 'FunctionName', 'Value': function_name}],
        StartTime=datetime.now() - timedelta(days=14),
        EndTime=datetime.now(),
        Period=3600,
        Statistics=['Average', 'Maximum']
    )
    
    avg_memory_used = sum([m['Average'] for m in metrics['Datapoints']]) / len(metrics['Datapoints'])
    
    # Recommend memory size (round up to nearest 64 MB)
    recommended_memory = ((avg_memory_used * 1.2) // 64 + 1) * 64
    
    if recommended_memory < current_memory:
        savings = (current_memory - recommended_memory) / current_memory * 100
        print(f"{function_name}:")
        print(f"  Current: {current_memory} MB")
        print(f"  Average Used: {avg_memory_used:.0f} MB")
        print(f"  Recommended: {recommended_memory:.0f} MB")
        print(f"  Potential Savings: {savings:.0f}%")
        
        # Update function configuration
        lambda_client.update_function_configuration(
            FunctionName=function_name,
            MemorySize=int(recommended_memory)
        )
```

**E. Savings Plans:**

```bash
# Get Savings Plans recommendations
aws ce get_savings_plans_purchase_recommendation \
  --lookback-period-in-days SIXTY_DAYS \
  --term-in-years ONE_YEAR \
  --payment-option ALL_UPFRONT \
  --savings-plans-type COMPUTE_SP

# Result:
# Recommended Commitment: $500/month
# Estimated Savings: $200/month (40% vs On-Demand)
```

**F. Delete Unused Resources:**

```python
import boto3
from datetime import datetime, timedelta

ec2 = boto3.client('ec2')
cloudwatch = boto3.client('cloudwatch')

def find_idle_resources():
    """
    Identify unused resources for cleanup
    """
    
    # Unused EBS volumes (not attached)
    volumes = ec2.describe_volumes(
        Filters=[{'Name': 'status', 'Values': ['available']}]
    )['Volumes']
    
    print(f"\nUnused EBS Volumes: {len(volumes)}")
    total_cost = 0
    for vol in volumes:
        size = vol['Size']
        vol_type = vol['VolumeType']
        cost_per_gb = {'gp3': 0.08, 'gp2': 0.10, 'io1': 0.125}.get(vol_type, 0.10)
        monthly_cost = size * cost_per_gb
        total_cost += monthly_cost
        print(f"  {vol['VolumeId']}: {size} GB {vol_type} (${monthly_cost:.2f}/month)")
    
    print(f"Total EBS waste: ${total_cost:.2f}/month")
    
    # Old EBS snapshots (>1 year)
    snapshots = ec2.describe_snapshots(OwnerIds=['self'])['Snapshots']
    old_snapshots = [s for s in snapshots if (datetime.now() - s['StartTime'].replace(tzinfo=None)).days > 365]
    
    print(f"\nOld Snapshots (>1 year): {len(old_snapshots)}")
    snapshot_cost = sum([s['VolumeSize'] for s in old_snapshots]) * 0.05
    print(f"Estimated cost: ${snapshot_cost:.2f}/month")
    
    # Idle Load Balancers (no traffic in 7 days)
    elbs = boto3.client('elbv2').describe_load_balancers()['LoadBalancers']
    
    idle_elbs = []
    for elb in elbs:
        metrics = cloudwatch.get_metric_statistics(
            Namespace='AWS/ApplicationELB',
            MetricName='RequestCount',
            Dimensions=[{'Name': 'LoadBalancer', 'Value': elb['LoadBalancerArn'].split('/')[-1]}],
            StartTime=datetime.now() - timedelta(days=7),
            EndTime=datetime.now(),
            Period=86400,
            Statistics=['Sum']
        )
        
        total_requests = sum([m['Sum'] for m in metrics['Datapoints']])
        if total_requests == 0:
            idle_elbs.append(elb['LoadBalancerName'])
    
    print(f"\nIdle Load Balancers: {len(idle_elbs)}")
    print(f"Estimated cost: ${len(idle_elbs) * 16.20:.2f}/month ($0.0225/hour × 720 hours)")

# Run cleanup
find_idle_resources()
```

**Phase 5: Ongoing Cost Governance**

**1. Budgets with Alerts:**
```yaml
Resources:
  MonthlyBudget:
    Type: AWS::Budgets::Budget
    Properties:
      Budget:
        BudgetName: monthly-spend-limit
        BudgetLimit:
          Amount: 10000
          Unit: USD
        TimeUnit: MONTHLY
        BudgetType: COST
        CostTypes:
          IncludeTax: true
          IncludeSubscription: true
      NotificationsWithSubscribers:
        - Notification:
            NotificationType: ACTUAL
            ComparisonOperator: GREATER_THAN
            Threshold: 80
          Subscribers:
            - SubscriptionType: EMAIL
              Address: finance@example.com
        - Notification:
            NotificationType: FORECASTED
            ComparisonOperator: GREATER_THAN
            Threshold: 100
          Subscribers:
            - SubscriptionType: EMAIL
              Address: cto@example.com
```

**2. Tag Enforcement (Cost Allocation):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": [
        "ec2:RunInstances",
        "rds:CreateDBInstance"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotLike": {
          "aws:RequestTag/CostCenter": "*",
          "aws:RequestTag/Owner": "*",
          "aws:RequestTag/Project": "*"
        }
      }
    }
  ]
}
```

**3. Scheduled Auto-Scaling (Dev/Test):**
```yaml
# Scale down dev environment outside business hours
Resources:
  ScaleDownSchedule:
    Type: AWS::AutoScaling::ScheduledAction
    Properties:
      AutoScalingGroupName: !Ref DevASG
      DesiredCapacity: 0
      MinSize: 0
      MaxSize: 0
      Recurrence: "0 19 * * MON-FRI"  # 7 PM weekdays

  ScaleUpSchedule:
    Type: AWS::AutoScaling::ScheduledAction
    Properties:
      AutoScalingGroupName: !Ref DevASG
      DesiredCapacity: 2
      MinSize: 2
      MaxSize: 5
      Recurrence: "0 8 * * MON-FRI"  # 8 AM weekdays
```

**Savings:** 
- Dev environment: 10 instances × 11 hours/day × 5 days = 55 hours/week off
- 55/168 = 33% uptime vs 100%
- Savings: $3,000/month × 67% = $2,010/month

**Final Results:**

| Optimization | Monthly Savings |
|-------------|----------------|
| NAT Gateway → S3 Endpoint | $13,500 |
| EC2 Right-sizing | $800 |
| RDS Reserved Instances | $840 |
| S3 Intelligent-Tiering | $176 |
| Lambda Memory Optimization | $120 |
| Delete Unused Resources | $450 |
| Dev Environment Scheduling | $2,010 |
| **Total Savings** | **$17,896/month** |

**Original Spike:** +$5,000/month (40% increase)
**Optimizations:** -$17,896/month
**Net Result:** -$12,896/month (65% reduction from peak!)

**Tools I Use Daily:**
1. AWS Cost Explorer (daily reviews)
2. Cost Anomaly Detection (automated alerts)
3. Compute Optimizer (quarterly right-sizing)
4. Trusted Advisor (weekly checks)
5. CloudWatch Dashboards (real-time monitoring)
6. Custom Lambda scripts (automated cleanup)

This investigation and optimization took 3 weeks total but resulted in permanent 65% cost reduction."

---

## SECTION 10: TROUBLESHOOTING SCENARIOS

### Q14: Production application is experiencing intermittent 504 Gateway Timeouts. How do you troubleshoot?
**Expert Answer:**
"504 errors indicate upstream timeout. Here's my systematic troubleshooting:

**Step 1: Verify the Problem (5 minutes)**

```bash
# Check ALB metrics in CloudWatch
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_Target_5XX_Count \
  --dimensions Name=LoadBalancer,Value=app/prod-alb/abc123 \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum

# Check target response time
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime \
  --dimensions Name=LoadBalancer,Value=app/prod-alb/abc123 \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average,Maximum
```

**Result:** TargetResponseTime spiking to 65 seconds (ALB timeout is 60s)

**Step 2: Identify Affected Layer**

**ALB Access Logs Analysis:**
```bash
# Query ALB logs in S3 using Athena
SELECT 
    request_processing_time,
    target_processing_time,
    response_processing_time,
    elb_status_code,
    target_status_code,
    request_url
FROM alb_logs
WHERE 
    elb_status_code = 504
    AND time > current_timestamp - interval '1' hour
ORDER BY target_processing_time DESC
LIMIT 100;
```

**Result:**
```
target_processing_time: 59.999s (ALB timeout)
target_status_code: - (no response from target)
request_url: /api/reports/generate
```

**Step 3: Dig into Application Layer**

**Check EC2 Target Health:**
```bash
# Get unhealthy targets
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/prod-app/abc123 \
  --query 'TargetHealthDescriptions[?TargetHealth.State!=`healthy`]'
```

**Result:** All targets healthy

**Check Application Logs (CloudWatch Logs Insights):**
```sql
fields @timestamp, @message, @requestId
| filter @message like /\/api\/reports\/generate/
| filter @message like /timeout|error|exception/i
| stats count() by bin(5m) as time_bucket
| sort time_bucket desc
```

**Result:** Timeouts correlate with high database query execution time

**Step 4: Database Investigation**

**RDS Performance Insights:**
```bash
# Get top SQL queries by execution time
aws pi get-resource-metrics \
  --service-type RDS \
  --identifier db-ABC123DEF456GHI789 \
  --metric-queries '[
    {
      "Metric": "db.load.avg",
      "GroupBy": {"Group": "db.sql"}
    }
  ]' \
  --start-time $(date -u -d '1 hour ago' +%s) \
  --end-time $(date -u +%s) \
  --period-in-seconds 300
```

**Found slow query:**
```sql
SELECT * FROM orders o
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE o.created_at >= '2024-01-01'
  AND o.status = 'completed';
-- Execution time: 58 seconds
-- Rows scanned: 50 million
```

**Root Cause:** Missing index on `orders.created_at` + `orders.status`

**Step 5: Immediate Mitigation**

**Option A: Increase ALB Timeout (Quick Fix)**
```bash
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/prod-app/abc123 \
  --attributes Key=deregistration_delay.timeout_seconds,Value=120
```

**Option B: Database Query Optimization**
```sql
-- Add composite index
CREATE INDEX idx_orders_created_status ON orders(created_at, status);

-- Verify index usage
EXPLAIN SELECT * FROM orders 
WHERE created_at >= '2024-01-01' AND status = 'completed';

-- Result: Using index idx_orders_created_status
Done
 
