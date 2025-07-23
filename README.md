# AWS Security Role Invalidation Project

> **## Practical demonstration of IAM role session revocation for handling credential leakage incidents**

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![AWS](https://img.shields.io/badge/AWS-CloudFormation-orange.svg)](infrastructure/A4LHostingInc.yaml)
[![Security](https://img.shields.io/badge/Security-IAM%20Credential%20Management-red.svg)](docs/security-theory.md)

## Project Overview

This project demonstrates a critical AWS security incident response technique: **IAM Role Session Revocation**. When EC2 instance credentials are compromised and leaked, this method allows security teams to neutralize the threat without disrupting legitimate services.

### What This Project Demonstrates

- **Real credential extraction** from EC2 instance metadata
- **Local credential usage** by an attacker on external systems  
- **Session revocation technique** using conditional deny policies
- **Service restoration** for legitimate users while keeping attackers locked out

## The Security Scenario

### Background
The Animals4Life organization runs animal blog websites on AWS EC2 instances. Each instance uses IAM roles for secure access to AWS services like S3 and CloudWatch. However, when an instance is compromised, the temporary credentials can be extracted and used from anywhere.

### The Problem
- **Temporary credentials cannot be manually canceled**
- **Changing role permissions affects ALL users of the role**
- **Deleting the role breaks ALL instances using it**
- **Leaked credentials remain valid until natural expiration (hours)**

### The Solution
Use **conditional deny policies** that block access for credentials issued before a specific timestamp, while allowing legitimate users to re-assume the role with fresh credentials.

## Infrastructure Architecture

The CloudFormation template deploys:
- **VPC** with dual-stack networking (IPv4/IPv6)
- **3 public subnets** across multiple availability zones
- **2 EC2 instances** running Animals4Life blog websites
- **IAM role** with S3 and EC2 read permissions
- **CloudWatch monitoring** with comprehensive logging
- **Security groups** with intentional vulnerabilities for demonstration


![Architecture Overview](asset/diagram/architecture-overview.png)
## Quick Start

### Prerequisites
- AWS CLI configured with CloudFormation permissions
- Basic understanding of IAM roles and EC2
- SSH client or Session Manager access

### 1. Deploy Infrastructure
```bash
# Deploy the CloudFormation template
aws cloudformation create-stack \
  --stack-name aws-security-role-invalidation \
  --template-body file://infrastructure/A4LHostingInc.yaml \
  --capabilities CAPABILITY_IAM
```
### 2. Follow the Demonstration
1. **[Evidence Documentation](evidence/commands/demonstration-evidence.md)** - Complete attack and response evidence
2. **[Visual Evidence](files/evidence/screenshots/)** - Screenshots of each phase
3. **[Technical Analysis](/docs/demonstration_guide.md)** - Detailed walkthrough

### 3. Clean Up
```bash
# Delete the CloudFormation stack
aws cloudformation delete-stack --stack-name aws-security-role-invalidation
```

## Documentation

| Document | Description | 
|----------|-------------|
| [**Quick Reference**](docs/quick-reference.md) | **Essential commands for rapid demonstration execution** |
| [Infrastructure Analysis](docs/infrastructure-analysis.md) | Detailed CloudFormation template breakdown |
| [Security Theory](docs/security-theory.md) | IAM role invalidation concepts |
| [Demonstration Guide](docs/demonstration-guide.md) | Complete walkthrough with evidence |
| [Demonstration Evidence](docs/demonstration_evidence.md) | Forensic documentation with screenshots |
| [Vulnerability Analysis](docs/vulnerability_analysis.md) | Security assessment and risk analysis |
| [Lab Guide](docs/lab_guide.md) | Educational exercises and assessments |
| [Attribution](docs/attribution.md) | Credits to Adrian Cantrill |

## Attack Simulation Results

### Credential Extraction (Successful)
```bash
# Extract role name from instance metadata
$ curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

A4L-InstanceRole-TDHLFluqhnhA

# Extract temporary credentials
$ curl http://169.254.169.254/latest/meta-data/iam/security-credentials/A4L-InstanceRole-TDHLFluqhnhA
{
  "AccessKeyId": "ASIA****************G3WA",
  "SecretAccessKey": "bMJW************************************k+je", 
  "Token": "IQoJb3JpZ2luX2VjEMr//////////wEaCXVzLWVhc3QtMSJH...",
  "Expiration": "2025-07-22T07:50:38Z"
}
```

### Local Credential Usage (Successful)

```bash
# Configure stolen credentials on Ubuntu system
export AWS_ACCESS_KEY_ID=ASIAZDHYX3ENS3KXG3WA
export AWS_SECRET_ACCESS_KEY=bMJWZGRd4bmetp1XAJHWyOYl3zeFZ+UAI2ZNk+je  
export AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjEMr//////////wEaCXVzLWVhc3QtMSJH...

# Test unauthorized access
$ aws s3 ls
2025-05-01 21:34:43 cf-templates-nu92qc9cc5d4-us-east-1

$ aws ec2 describe-instances --region us-east-1 --output table
# Successfully lists all EC2 instances
```

## Incident Response Results  

### Session Revocation (Applied)
- **Timestamp**: 2025-07-21 at 9:08 PM CDT  
- **Method**: IAM Console → Role → Revoke Active Sessions
- **Policy Applied**: Conditional deny for tokens issued before timestamp

### Impact Verification (Successful)

```bash
# Attacker credentials blocked
$ aws s3 ls
Unable to locate credentials. You can configure credentials by running "aws configure".

# EC2 instances also lose access (expected)
$ aws s3 ls  # From EC2 instance
AccessDenied
```

### Service Restoration (Successful)  
```bash
# After instance stop/start cycle
$ aws s3 ls  # From EC2 instance  
2025-05-01 21:34:43 cf-templates-nu92qc9cc5d4-us-east-1
# ✅ Legitimate access restored

# Attacker still blocked (permanent)
$ aws s3 ls  # From attacker machine
Unable to locate credentials...
# ❌ Attack neutralized
```

## 🔍 Security Analysis

### Vulnerability Assessment
- **SSH Access**: Open to 0.0.0.0/0 (Intentional for demonstration)
- **Instance Metadata**: Accessible without IMDSv2 enforcement
- **IAM Permissions**: Broad read access to S3 and EC2
- **Network Placement**: Public subnets with direct internet access

### Threat Model  
- **Attack Vector**: Instance compromise → credential extraction → lateral movement
- **Impact**: Unauthorized access to S3 buckets and EC2 metadata
- **Likelihood**: High (if instance is compromised)
- **Detection**: CloudWatch logs show unusual API patterns

### Security Controls
- **Monitoring**: CloudWatch agent collecting security logs
- **Logging**: Authentication attempts in `/var/log/secure`
- **Incident Response**: Session revocation capability
- **Recovery**: Non-disruptive service restoration

## Key Metrics

| Metric | Value | Notes |
|--------|-------|-------|
| **Credential Extraction Time** | < 30 seconds | From compromised instance |
| **Attack Setup Time** | < 2 minutes | Environment variables on Ubuntu |
| **AWS Access Validation** | Immediate | S3 and EC2 API calls successful |
| **Session Revocation Time** | < 1 minute | Via IAM console |
| **Attack Neutralization** | Immediate | Credentials blocked instantly |
| **Service Restoration** | < 5 minutes | Instance stop/start cycle |
| **Credential Lifetime** | ~6.5 hours | Original expiration time |

## Educational Value

### Learning Objectives
- Understand IAM role temporary credential mechanics
- Practice realistic incident response procedures  
- Learn conditional IAM policy techniques
- Experience hands-on security threat simulation

### Skills Demonstrated
- **Cloud Security**: AWS IAM role security architecture
- **Incident Response**: Real-time credential compromise handling
- **Threat Analysis**: Attack vector identification and mitigation
- **Infrastructure Security**: CloudFormation security considerations

### Assessment Questions
1. Why can't temporary credentials be manually invalidated?
2. How does conditional deny policy work with `aws:TokenIssueTime`?
3. What happens to legitimate users during session revocation?
4. How do you restore service without affecting security?

## Real-World Applications

### Enterprise Security Teams
- **Training**: Hands-on credential leakage response
- **Procedures**: Incident response playbook validation  
- **Assessment**: Security control effectiveness testing
- **Compliance**: Audit trail and response capability demonstration

##  Security Notice

**This project intentionally creates vulnerable infrastructure for educational purposes.**

- **Use only in isolated AWS accounts**
- **Never deploy in production environments** 
- **Monitor costs and resource usage**
- **Clean up resources after demonstrations**
- **Follow your organization's security policies**

## Attribution

This project is based on the excellent educational content by **[Adrian Cantrill](https://learn.cantrill.io/)**:

- **Original Course**: AWS Advanced Security Specialty
- **Architecture Design**: Animals4Life hosting scenario  
- **Teaching Methodology**: Hands-on security demonstrations
- **Foundational Concepts**: IAM role session invalidation theory

**Enhancements in this repository**:
- Comprehensive documentation with real evidence
- Detailed infrastructure analysis and security assessment
- Organized demonstration with actual command outputs
- Educational materials for skill development

## Support

### Getting Help
- Review the [demonstration guide](docs/demonstration_evidence.md) for detailed steps
- Check [command outputs](evidence/commands/demonstration_evidence.md) for expected results
- Examine [screenshots](evidence/screenshots/) for visual guidance
- Open an issue for questions or problems

---

**Master AWS credential security through hands-on experience with real attack scenarios and proven response techniques.**