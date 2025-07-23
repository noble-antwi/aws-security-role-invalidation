# AWS Security Role Invalidation - Educational Lab Guide

> **🎓 Step-by-step educational laboratory for learning IAM credential security and incident response**

## 🎯 Lab Overview

### Learning Objectives
By completing this lab, participants will:
- ✅ Understand IAM role temporary credential mechanisms
- ✅ Practice realistic credential compromise scenarios  
- ✅ Master session revocation incident response techniques
- ✅ Implement security monitoring and detection capabilities
- ✅ Evaluate security controls and recommend improvements

### Prerequisites
- **AWS Account**: With CloudFormation deployment permissions
- **AWS CLI**: Configured with appropriate credentials
- **Basic Knowledge**: IAM roles, EC2 instances, CloudFormation
- **Lab Environment**: Isolated AWS account (recommended)
- **Time Required**: 2-3 hours

### Lab Environment
This lab uses the Animals4Life hosting scenario with intentionally vulnerable infrastructure to demonstrate real-world security incidents and response procedures.

## 📋 Lab Setup

### Step 1: Environment Preparation

**1.1 Account Verification**
```bash
# Verify AWS CLI access
aws sts get-caller-identity

# Check region configuration  
aws configure get region
# Should be: us-east-1
```

**1.2 Download Lab Materials**
```bash
# Clone the repository
git clone https://github.com/yourusername/aws-security-role-invalidation.git
cd aws-security-role-invalidation
```

**1.3 Review Infrastructure**
```bash
# Examine the CloudFormation template
cat infrastructure/A4LHostingInc.yaml

# Validate template syntax
aws cloudformation validate-template \
  --template-body file://infrastructure/A4LHostingInc.yaml
```

### Step 2: Infrastructure Deployment

**2.1 Deploy CloudFormation Stack**
```bash
aws cloudformation create-stack \
  --stack-name aws-security-lab \
  --template-body file://infrastructure/A4LHostingInc.yaml \
  --capabilities CAPABILITY_IAM
```

**2.2 Monitor Deployment Progress**
```bash
# Check stack status
aws cloudformation describe-stacks \
  --stack-name aws-security-lab \
  --query 'Stacks[0].StackStatus'

# Wait for CREATE_COMPLETE status (approximately 10-15 minutes)
```

**2.3 Retrieve Stack Outputs**
```bash
# Get instance IDs and other outputs
aws cloudformation describe-stacks \
  --stack-name aws-security-lab \
  --query 'Stacks[0].Outputs'
```

## 🔴 Lab Exercise 1: Attack Simulation

### Objective
Simulate a realistic credential compromise scenario by extracting temporary credentials from a compromised EC2 instance.

### 1.1 Instance Access

**Connect to Target Instance**:
```bash
# Get instance ID from stack outputs
INSTANCE_ID=$(aws cloudformation describe-stacks \
  --stack-name aws-security-lab \
  --query 'Stacks[0].Outputs[?OutputKey==`InstanceAId`].OutputValue' \
  --output text)

# Connect via Session Manager
aws ssm start-session --target $INSTANCE_ID
```

**Verify Connection**:
```bash
# Inside the instance
whoami
# Expected: ssm-user

pwd  
# Expected: /usr/bin or /home/ssm-user
```

### 1.2 Role Discovery

**Extract Role Information**:
```bash
# Discover attached IAM role
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

**Expected Output**:
```
A4L-InstanceRole-[RANDOM_STRING]
```

**📝 Lab Question 1**: What is the full name of the IAM role attached to this instance?

### 1.3 Credential Extraction

**Extract Temporary Credentials**:
```bash
# Replace [ROLE_NAME] with actual role name from previous step
ROLE_NAME="A4L-InstanceRole-[RANDOM_STRING]"
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/$ROLE_NAME
```

**Expected Output Structure**:
```json
{
  "Code": "Success",
  "LastUpdated": "2025-07-22T01:15:24Z", 
  "Type": "AWS-HMAC",
  "AccessKeyId": "ASIA...",
  "SecretAccessKey": "...",
  "Token": "...",
  "Expiration": "2025-07-22T07:50:38Z"
}
```

**📝 Lab Question 2**: How many hours of validity do these credentials have remaining?

### 1.4 On-Instance Testing

**Test AWS Access**:
```bash
# List S3 buckets
aws s3 ls

# Describe EC2 instances  
aws ec2 describe-instances --region us-east-1 --output table
```

**📝 Lab Question 3**: What AWS resources can you access with these credentials?

## 🌐 Lab Exercise 2: External Attack Execution

### Objective
Demonstrate how stolen credentials can be used from external systems, simulating real-world attack scenarios.

### 2.1 External Environment Setup

**Local Machine Preparation**:
```bash
# Verify no existing AWS credentials
aws s3 ls
# Expected: Error message about missing credentials
```

### 2.2 Credential Configuration

**Set Environment Variables**:
```bash
# Export the stolen credentials (replace with actual values from Exercise 1)
export AWS_ACCESS_KEY_ID="ASIA..."
export AWS_SECRET_ACCESS_KEY="..."  
export AWS_SESSION_TOKEN="..."
```

**Verify Configuration**:
```bash
# Test access from external system
aws s3 ls
aws ec2 describe-instances --region us-east-1 --output table
```

**📝 Lab Question 4**: What is the security implication of credentials working from external systems?

### 2.3 Attack Impact Assessment

**Enumerate Accessible Resources**:
```bash
# List all S3 buckets
aws s3 ls

# Get S3 bucket details
aws s3api list-buckets --query 'Buckets[].Name'

# Enumerate EC2 instances
aws ec2 describe-instances --query 'Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,State:State.Name}'

# Check IAM permissions
aws iam get-account-summary
```

**📝 Lab Question 5**: Document all resources and data you can access with the stolen credentials.

## 🛡️ Lab Exercise 3: Incident Response

### Objective
Practice the session revocation technique to neutralize compromised credentials while maintaining legitimate service access.

### 3.1 Threat Assessment

**Identify Compromised Role**:
```bash
# From the AWS Console, navigate to:
# IAM → Roles → Search for "A4L-InstanceRole"
```

**Current Permissions Analysis**:
- Review attached managed policies
- Document current access scope
- Assess potential impact

**📝 Lab Question 6**: What are the current permissions of the compromised role?

### 3.2 Session Revocation Process

**Apply Session Revocation**:
1. **AWS Console**: Navigate to IAM → Roles → [Role Name]
2. **Click**: "Revoke active sessions"  
3. **Acknowledge**: Impact warning
4. **Confirm**: Revocation action
5. **Record**: Timestamp of revocation

**Document Revocation Policy**:
```bash
# After revocation, examine the new inline policy
aws iam get-role-policy \
  --role-name [ROLE_NAME] \
  --policy-name [REVOCATION_POLICY_NAME]
```

**📝 Lab Question 7**: What is the exact timestamp used in the revocation policy condition?

### 3.3 Impact Verification

**Test Attacker Credentials**:
```bash
# From your external system with stolen credentials
aws s3 ls