# AWS Security Role Invalidation - Quick Demonstration Guide

> **📋 Streamlined walkthrough for executing the credential compromise and response demonstration**

This guide provides the essential commands and steps for running the security demonstration. For detailed evidence and explanations, see [demonstration_evidence.md](demonstration_evidence.md).

## Prerequisites

- AWS CLI configured with appropriate permissions
- CloudFormation stack deployed (see [README.md](../README.md))
- Access to EC2 instances via Session Manager

## Phase 1: Simulate Attack

### 1.1 Access Compromised Instance

```bash
# Connect to instance via Session Manager
aws ssm start-session --target [INSTANCE-ID]
```

### 1.2 Extract Credentials

```bash
# Get role name
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Extract credentials (replace ROLE_NAME with output from above)
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/[ROLE_NAME]
```

📝 **Note the credentials**: AccessKeyId, SecretAccessKey, SessionToken, and Expiration time

### 1.3 Verify On-Instance Access

```bash
# Test AWS access from within instance
aws s3 ls
aws ec2 describe-instances --region us-east-1 --output table
```

## Phase 2: External Attack

### 2.1 Configure Stolen Credentials

On your local machine:

```bash
# Export credentials (use values from Phase 1.2)
export AWS_ACCESS_KEY_ID=[EXTRACTED_ACCESS_KEY]
export AWS_SECRET_ACCESS_KEY=[EXTRACTED_SECRET_KEY]
export AWS_SESSION_TOKEN=[EXTRACTED_SESSION_TOKEN]
```

### 2.2 Verify External Access

```bash
# Confirm attacker access works
aws s3 ls
aws ec2 describe-instances --region us-east-1 --output table
```

✅ **Attack simulation complete** - Credentials confirmed working externally

## Phase 3: Incident Response

### 3.1 Apply Session Revocation

1. Open AWS Console → IAM → Roles
2. Search for `A4L-InstanceRole`
3. Click **"Revoke active sessions"**
4. Confirm the action

⏱️ **Record timestamp** of revocation for reference

### 3.2 Verify Attack Blocked

From attacker's machine:

```bash
# Test if stolen credentials still work
aws s3 ls
# Expected: "Unable to locate credentials" error
```

From EC2 instance:

```bash
# Legitimate access also temporarily blocked
aws s3 ls
# Expected: Access denied error
```

## Phase 4: Service Recovery

### 4.1 Restart Affected Instances

```bash
# Get instance IDs
INSTANCES=$(aws ec2 describe-instances \
  --filters "Name=iam-instance-profile.arn,Values=*A4L-InstanceRole*" \
  --query "Reservations[].Instances[].InstanceId" \
  --output text)

# Stop instances
aws ec2 stop-instances --instance-ids $INSTANCES

# Wait for stopped state
aws ec2 wait instance-stopped --instance-ids $INSTANCES

# Start instances
aws ec2 start-instances --instance-ids $INSTANCES

# Wait for running state
aws ec2 wait instance-running --instance-ids $INSTANCES
```

### 4.2 Verify Service Restoration

```bash
# Reconnect to instance
aws ssm start-session --target [INSTANCE-ID]

# Test restored access
aws s3 ls
# Expected: Successful listing
```

### 4.3 Confirm Attacker Still Blocked

From attacker's machine:

```bash
# Verify stolen credentials remain blocked
aws s3 ls
# Expected: Still getting credential error
```

## Summary Checklist

- [ ] Extracted credentials from compromised instance
- [ ] Confirmed external attack capability
- [ ] Applied session revocation via IAM console
- [ ] Verified attack neutralization
- [ ] Restarted affected instances
- [ ] Confirmed service restoration
- [ ] Validated attacker remains blocked

## Quick Reference

| Action | Time | Impact |
|--------|------|--------|
| Credential extraction | < 30 seconds | Attacker gains access |
| External setup | < 2 minutes | Attack operational |
| Session revocation | < 1 minute | Attack blocked |
| Service recovery | < 6 minutes | Normal operations restored |

## Troubleshooting

### Session Manager Connection Issues
```bash
# Verify instance is running
aws ec2 describe-instance-status --instance-ids [INSTANCE-ID]

# Check SSM agent status
aws ssm describe-instance-information --instance-information-filter-list key=InstanceIds,valueSet=[INSTANCE-ID]
```

### Revocation Policy Not Working
- Ensure you clicked "Revoke active sessions" not just viewed the role
- Check for the inline policy named "RevokeOldSessions"
- Verify the timestamp in the policy is recent

### Service Not Recovering
- Ensure instances fully stopped before starting
- May need to restart applications that cache credentials
- Check CloudWatch logs for errors

---

📚 **For detailed evidence and explanations**, see [demonstration_evidence.md](demonstration_evidence.md)  
🔬 **For security theory**, see [security_theory.md](security_theory.md)  
🏗️ **For infrastructure details**, see [infrastructure_analysis.md](infrastructure_analysis.md)