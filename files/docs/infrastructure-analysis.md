# CloudFormation Template - Detailed Component Analysis

> **Repository**: `aws-security-role-invalidation`  
> **This Document Location**: `docs/infrastructure-analysis.md`  
> **CloudFormation Template**: [`infrastructure/A4LHostingInc.yaml`](../infrastructure/A4LHostingInc.yaml)

## Template Overview
**File**: `A4LHostingInc.yaml`  
**Purpose**: AWS Role Invalidation Security Demonstration Infrastructure  
**Original Author**: Adrian Cantrill  
**License**: Apache License Version 2.0

This CloudFormation template creates a complete AWS infrastructure environment designed for security demonstrations, specifically focusing on IAM role invalidation scenarios and security monitoring.

---

## Template Structure

### Metadata Section
```yaml
Metadata:
  LICENSE: Apache License Version 2.0
```
Defines the licensing terms for the template usage.

### Parameters Section

#### `LatestAmiId`
```yaml
Parameters:
  LatestAmiId:
    Description: AMI for Instance (default is latest AmaLinux2)
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: '/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2'
```
- **Purpose**: Dynamically retrieves the latest Amazon Linux 2 AMI ID
- **Type**: SSM Parameter that automatically resolves to current AMI
- **Benefit**: Ensures instances always use the most recent AMI without manual updates

---

## Network Infrastructure Components

### 1. Virtual Private Cloud (VPC)

#### Main VPC Resource
```yaml
VPC:
  Type: AWS::EC2::VPC
  Properties:
    CidrBlock: 10.16.0.0/16
    EnableDnsSupport: true
    EnableDnsHostnames: true
```
- **CIDR Block**: `10.16.0.0/16` provides 65,534 IP addresses
- **DNS Support**: Enables AWS DNS resolution within the VPC
- **DNS Hostnames**: Allows instances to receive public DNS hostnames

#### IPv6 Configuration
```yaml
IPv6CidrBlock:
  Type: AWS::EC2::VPCCidrBlock
  Properties:
    VpcId: !Ref VPC
    AmazonProvidedIpv6CidrBlock: true
```
- **Purpose**: Adds IPv6 support to the VPC
- **Provider**: Uses Amazon's IPv6 address space allocation
- **Benefit**: Enables dual-stack networking (IPv4 + IPv6)

### 2. Internet Gateway
```yaml
InternetGateway:
  Type: 'AWS::EC2::InternetGateway'
InternetGatewayAttachment:
  Type: 'AWS::EC2::VPCGatewayAttachment'
  Properties:
    VpcId: !Ref VPC
    InternetGatewayId: !Ref InternetGateway
```
- **Function**: Provides internet access for public subnets
- **Attachment**: Links the IGW to the VPC for routing

### 3. Routing Configuration

#### Route Table
```yaml
RouteTableWeb: 
  Type: 'AWS::EC2::RouteTable'
  Properties:
    VpcId: !Ref VPC
```

#### Default Routes
```yaml
RouteTableWebDefaultIPv4: 
  Type: 'AWS::EC2::Route'
  Properties:
    RouteTableId: !Ref RouteTableWeb
    DestinationCidrBlock: '0.0.0.0/0'
    GatewayId: !Ref InternetGateway

RouteTableWebDefaultIPv6: 
  Type: 'AWS::EC2::Route'
  Properties:
    RouteTableId: !Ref RouteTableWeb
    DestinationIpv6CidrBlock: '::/0'
    GatewayId: !Ref InternetGateway
```
- **IPv4 Route**: All traffic (`0.0.0.0/0`) directed to Internet Gateway
- **IPv6 Route**: All IPv6 traffic (`::/0`) directed to Internet Gateway
- **Result**: All subnets function as public subnets

### 4. Subnet Configuration

#### Subnet A (AZ-1)
```yaml
SubnetWEBA:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref VPC
    AvailabilityZone: !Select [ 0, !GetAZs '' ]
    CidrBlock: 10.16.48.0/20
    MapPublicIpOnLaunch: true
    Ipv6CidrBlock: 
      Fn::Sub:
        - "${VpcPart}${SubnetPart}"
        - SubnetPart: '03::/64'
          VpcPart: !Select [ 0, !Split [ '00::/56', !Select [ 0, !GetAtt VPC.Ipv6CidrBlocks ]]]
```

#### Subnet B (AZ-2)
```yaml
SubnetWEBB:
  Type: AWS::EC2::Subnet
  Properties:
    AvailabilityZone: !Select [ 1, !GetAZs '' ]
    CidrBlock: 10.16.112.0/20
    Ipv6CidrBlock: SubnetPart: '07::/64'
```

#### Subnet C (AZ-3)
```yaml
SubnetWEBC:
  Type: AWS::EC2::Subnet
  Properties:
    AvailabilityZone: !Select [ 2, !GetAZs '' ]
    CidrBlock: 10.16.176.0/20
    Ipv6CidrBlock: SubnetPart: '0B::/64'
```

**Subnet Analysis:**
- **Multi-AZ Design**: Three subnets across different availability zones
- **CIDR Allocation**: Each subnet gets `/20` (4,094 usable IPs)
- **Public Access**: `MapPublicIpOnLaunch: true` for automatic public IPs
- **IPv6 Support**: Each subnet gets a unique `/64` IPv6 block

---

## IPv6 Workaround Implementation

### Lambda Function for IPv6 Auto-Assignment
```python
import cfnresponse
import boto3

def lambda_handler(event, context):
    if event['RequestType'] is 'Delete':
      cfnresponse.send(event, context, cfnresponse.SUCCESS)
      return

    responseValue = event['ResourceProperties']['SubnetId']
    ec2 = boto3.client('ec2', region_name='${AWS::Region}')
    ec2.modify_subnet_attribute(AssignIpv6AddressOnCreation={
                                    'Value': True
                                  },
                                  SubnetId=responseValue)
```

**Purpose**: CloudFormation limitation workaround
- **Issue**: Cannot set IPv6 auto-assignment directly in subnet resource
- **Solution**: Custom Lambda function modifies subnet after creation
- **Trigger**: Custom resource invokes Lambda for each subnet

### IAM Role for Lambda
```yaml
IPv6WorkaroundRole:
  Type: AWS::IAM::Role
  Properties:
    AssumeRolePolicyDocument:
      Version: '2012-10-17'
      Statement:
      - Effect: Allow
        Principal:
          Service: lambda.amazonaws.com
        Action: sts:AssumeRole
    Policies:
      - PolicyName: ipv6-fix-logs
        PolicyDocument:
          Statement:
          - Effect: Allow
            Action:
            - logs:CreateLogGroup
            - logs:CreateLogStream  
            - logs:PutLogEvents
            Resource: arn:aws:logs:*:*:*
      - PolicyName: ipv6-fix-modify
        PolicyDocument:
          Statement:
          - Effect: Allow
            Action: ec2:ModifySubnetAttribute
            Resource: "*"
```

---

## Security Configuration

### Security Group
```yaml
DefaultInstanceSecurityGroup:
  Type: 'AWS::EC2::SecurityGroup'
  Properties:
    VpcId: !Ref VPC
    GroupDescription: Enable SSH access via port 22 IPv4 & v6
    SecurityGroupIngress: 
      - Description: 'Allow SSH IPv4 IN'
        IpProtocol: tcp
        FromPort: '22'
        ToPort: '22'
        CidrIp: '0.0.0.0/0'
      - Description: 'Allow HTTP IPv4 IN'
        IpProtocol: tcp
        FromPort: '80'
        ToPort: '80'
        CidrIp: '0.0.0.0/0'
      - Description: 'Allow SSH IPv6 IN'
        IpProtocol: tcp
        FromPort: '22'
        ToPort: '22'
        CidrIpv6: ::/0
```

#### Self-Reference Rule
```yaml
DefaultInstanceSecurityGroupSelfReferenceRule:
  Type: "AWS::EC2::SecurityGroupIngress"
  Properties:
    GroupId: !Ref DefaultInstanceSecurityGroup
    IpProtocol: 'tcp'
    FromPort: '0'
    ToPort: '65535'
    SourceSecurityGroupId: !Ref DefaultInstanceSecurityGroup
```

**Security Analysis:**
- **⚠️ SSH Open to World**: Port 22 accessible from `0.0.0.0/0` (security risk)
- **HTTP Access**: Port 80 open for web server access
- **Internal Communication**: Full port range allowed between instances in same security group
- **IPv6 Support**: SSH access enabled for IPv6 addresses

---

## Monitoring and Logging

### CloudWatch Agent Configuration
```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/secure",
            "log_group_name": "/var/log/secure",
            "log_stream_name": "{instance_id}"
          },
          {
            "file_path": "/var/log/httpd/access_log",
            "log_group_name": "/var/log/httpd/access_log",
            "log_stream_name": "{instance_id}"
          },
          {
            "file_path": "/var/log/httpd/error_log", 
            "log_group_name": "/var/log/httpd/error_log",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  },
  "metrics": {
    "append_dimensions": {
      "AutoScalingGroupName": "${aws:AutoScalingGroupName}",
      "ImageId": "${aws:ImageId}",
      "InstanceId": "${aws:InstanceId}",
      "InstanceType": "${aws:InstanceType}"
    },
    "metrics_collected": {
      "cpu": {
        "measurement": ["cpu_usage_idle", "cpu_usage_iowait", "cpu_usage_user", "cpu_usage_system"],
        "metrics_collection_interval": 60,
        "resources": ["*"],
        "totalcpu": false
      },
      "disk": {
        "measurement": ["used_percent", "inodes_free"],
        "metrics_collection_interval": 60,
        "resources": ["*"]
      },
      "diskio": {
        "measurement": ["io_time", "write_bytes", "read_bytes", "writes", "reads"],
        "metrics_collection_interval": 60,
        "resources": ["*"]
      },
      "mem": {
        "measurement": ["mem_used_percent"],
        "metrics_collection_interval": 60
      },
      "netstat": {
        "measurement": ["tcp_established", "tcp_time_wait"],
        "metrics_collection_interval": 60
      },
      "swap": {
        "measurement": ["swap_used_percent"],
        "metrics_collection_interval": 60
      }
    }
  }
}
```

**Monitoring Coverage:**
- **Security Logs**: `/var/log/secure` for authentication events
- **Web Server Logs**: Apache access and error logs
- **System Metrics**: CPU, memory, disk, network statistics
- **Collection Interval**: 60-second intervals for all metrics
- **Identification**: Each log stream tagged with instance ID

---

## IAM Configuration

### Instance Role
```yaml
InstanceRole:
  Type: 'AWS::IAM::Role'
  Properties:
    AssumeRolePolicyDocument:
      Version: 2012-10-17
      Statement:
        - Effect: Allow
          Principal:
            Service: ec2.amazonaws.com
          Action: 'sts:AssumeRole'
    Path: /
    ManagedPolicyArns:
      - "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
      - "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"  
      - "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
      - "arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess"
```

**Permissions Analysis:**
- **CloudWatchAgentServerPolicy**: Send metrics and logs to CloudWatch
- **AmazonSSMManagedInstanceCore**: Systems Manager access for management
- **AmazonS3ReadOnlyAccess**: Read access to all S3 buckets
- **AmazonEC2ReadOnlyAccess**: Describe EC2 resources

### Instance Profile
```yaml
InstanceProfile:
  Type: 'AWS::IAM::InstanceProfile'
  Properties:
    Path: /
    Roles: [!Ref InstanceRole]
```
Links the IAM role to EC2 instances via instance metadata service.

---

## EC2 Instance Configuration

### Instance Properties
```yaml
EC2InstanceA:
  Type: AWS::EC2::Instance
  CreationPolicy:
    ResourceSignal:
      Timeout: PT15M
  Properties:
    InstanceType: "t2.micro"
    ImageId: !Ref LatestAmiId
    IamInstanceProfile: !Ref InstanceProfile
    SubnetId: !Ref SubnetWEBA
    SecurityGroupIds: [!Ref DefaultInstanceSecurityGroup]
```

### UserData Bootstrap Script
```bash
#!/bin/bash -xe
yum -y update                                    # Update system packages
yum -y install httpd wget                       # Install Apache and wget
wget [instance-specific-content] -P /var/www/html  # Download web content
usermod -a -G apache ec2-user                   # Add ec2-user to apache group
chown -R ec2-user:apache /var/www               # Set ownership
chmod 2775 /var/www                             # Set permissions
find /var/www -type d -exec chmod 2775 {} \;    # Directory permissions
find /var/www -type f -exec chmod 0664 {} \;    # File permissions
systemctl enable httpd                          # Enable Apache service
systemctl start httpd                           # Start Apache service

# CloudWatch Agent Installation
rpm -Uvh https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm

# CloudWatch Agent Fix for collectd
mkdir -p /usr/share/collectd/
touch /usr/share/collectd/types.db

# Start CloudWatch Agent with SSM configuration
/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c ssm:${CloudWatchLinuxConfig} -s

# Signal CloudFormation of successful completion
/opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackId} --resource EC2InstanceA --region ${AWS::Region}
```

### Instance-Specific Content
- **Instance A**: Downloads `index.html` and `sophie.jpeg`
- **Instance B**: Downloads `index.html`, `dogs1.jpg`, `dogs2.jpg`, and `bones.png`
- **Instance C**: Similar pattern with different content

**Bootstrap Process:**
1. System updates and package installation
2. Web server configuration and content download
3. User permissions and file ownership setup
4. Service enablement and startup
5. CloudWatch agent installation and configuration
6. CloudFormation success signal

---

## Resource Dependencies

### Dependency Chain
```
VPC → IPv6CidrBlock → Subnets → IPv6Workaround
VPC → InternetGateway → InternetGatewayAttachment → Routes
Subnets → RouteTableAssociations
IAM Role → Instance Profile → EC2 Instances
CloudWatch Config → EC2 Instances (UserData)
```

### Creation Policy
```yaml
CreationPolicy:
  ResourceSignal:
    Timeout: PT15M
```
- CloudFormation waits up to 15 minutes for instance bootstrap completion
- Instance must signal success using `cfn-signal`
- Stack creation fails if signal not received within timeout

---

## Security Considerations

### Intentional Vulnerabilities (For Demonstration)
1. **SSH Access**: Open to entire internet (`0.0.0.0/0`)
2. **Public Subnets**: All instances directly internet-accessible
3. **Broad IAM Permissions**: S3 read access to all buckets
4. **No Encryption**: EBS volumes not explicitly encrypted

### Security Monitoring Capabilities
1. **Authentication Logs**: `/var/log/secure` monitoring
2. **Web Access Logs**: Apache access and error log collection
3. **System Metrics**: Comprehensive performance monitoring
4. **CloudWatch Integration**: Centralized log aggregation

### Recommended Security Improvements
1. Restrict SSH access to specific IP ranges
2. Implement private subnets with NAT gateway
3. Enable EBS encryption
4. Apply principle of least privilege to IAM roles
5. Add VPC Flow Logs
6. Implement AWS Config for compliance monitoring

---

## Cost Optimization

### Cost-Effective Choices
- **Instance Type**: t2.micro (free tier eligible)
- **Storage**: Default GP2 volumes
- **Networking**: No NAT gateways (public subnets)
- **Monitoring**: Standard CloudWatch metrics

### Potential Cost Concerns
- **CloudWatch Logs**: No retention policy set (never expires)
- **Data Transfer**: Internet gateway usage charges
- **Multiple AZs**: Cross-AZ data transfer costs

---

*This analysis covers the complete CloudFormation template structure and implementation details for the AWS Security Role Invalidation Demo project, originally created by Adrian Cantrill.*