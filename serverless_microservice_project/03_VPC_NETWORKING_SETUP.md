# VPC & Networking Setup - Serverless Order Processing Microservice

## 📋 Overview

This guide covers the complete VPC and networking setup for the serverless microservice. While Lambda functions can run without a VPC, using a VPC provides enhanced security, private communication with AWS services, and better network isolation.

---

## 🎯 VPC Architecture Goals

- **Security**: Isolate resources in private subnets
- **Cost Optimization**: Use VPC endpoints to avoid NAT Gateway costs where possible
- **High Availability**: Multi-AZ deployment
- **Scalability**: Support for future growth
- **Performance**: Low-latency communication between services

---

## 1️⃣ VPC Design Overview

### Network CIDR Planning

```javascript
const vpcDesign = {
  vpc: {
    cidr: '10.0.0.0/16',           // 65,536 IPs
    name: 'order-processing-vpc',
    enableDnsHostnames: true,
    enableDnsSupport: true
  },
  
  availabilityZones: ['us-east-1a', 'us-east-1b', 'us-east-1c'],
  
  subnets: {
    public: [
      { cidr: '10.0.1.0/24', az: 'us-east-1a', name: 'public-subnet-1a' },   // 256 IPs
      { cidr: '10.0.2.0/24', az: 'us-east-1b', name: 'public-subnet-1b' },   // 256 IPs
      { cidr: '10.0.3.0/24', az: 'us-east-1c', name: 'public-subnet-1c' }    // 256 IPs
    ],
    private: [
      { cidr: '10.0.11.0/24', az: 'us-east-1a', name: 'private-subnet-1a' }, // 256 IPs
      { cidr: '10.0.12.0/24', az: 'us-east-1b', name: 'private-subnet-1b' }, // 256 IPs
      { cidr: '10.0.13.0/24', az: 'us-east-1c', name: 'private-subnet-1c' }  // 256 IPs
    ]
  }
};
```

### Visual Diagram

```
                    Internet
                       |
                 Internet Gateway
                       |
    ┌──────────────────┼──────────────────┐
    │                VPC                   │
    │          10.0.0.0/16                 │
    │                                      │
    │  ┌─────────────────────────────┐    │
    │  │    Public Subnets           │    │
    │  │  (For NAT Gateways only)    │    │
    │  │                              │    │
    │  │  10.0.1.0/24 (us-east-1a)   │    │
    │  │  10.0.2.0/24 (us-east-1b)   │    │
    │  │  10.0.3.0/24 (us-east-1c)   │    │
    │  │                              │    │
    │  │  ┌──────┐  ┌──────┐  ┌──────┐   │
    │  │  │ NAT  │  │ NAT  │  │ NAT  │   │
    │  │  │  GW  │  │  GW  │  │  GW  │   │
    │  │  └───┬──┘  └───┬──┘  └───┬──┘   │
    │  └──────┼─────────┼─────────┼──────┘
    │         │         │         │         │
    │  ┌──────▼─────────▼─────────▼──────┐
    │  │    Private Subnets               │
    │  │  (Lambda Functions)              │
    │  │                                   │
    │  │  10.0.11.0/24 (us-east-1a)      │
    │  │  10.0.12.0/24 (us-east-1b)      │
    │  │  10.0.13.0/24 (us-east-1c)      │
    │  │                                   │
    │  │  ┌────────┐  ┌────────┐  ┌─────┐│
    │  │  │Lambda  │  │Lambda  │  │More ││
    │  │  │  API   │  │ Order  │  │Funcs││
    │  │  │Handler │  │Processor│  │     ││
    │  │  └────────┘  └────────┘  └─────┘│
    │  └───────────────────────────────────┘
    │                                      │
    │  VPC Endpoints (Interface):         │
    │  - DynamoDB (Gateway)                │
    │  - S3 (Gateway)                      │
    │  - SQS (Interface)                   │
    │  - SNS (Interface)                   │
    │  - Secrets Manager (Interface)       │
    └──────────────────────────────────────┘
```

---

## 2️⃣ CloudFormation Template - Complete VPC

### Full VPC Stack

Create file: `infrastructure/cloudformation/vpc-stack.yaml`

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'VPC and Networking for Serverless Order Processing Microservice'

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - staging
      - production
    Description: Environment name

  VpcCIDR:
    Type: String
    Default: 10.0.0.0/16
    Description: CIDR block for VPC

Mappings:
  SubnetConfig:
    dev:
      PublicSubnet1CIDR: 10.0.1.0/24
      PublicSubnet2CIDR: 10.0.2.0/24
      PublicSubnet3CIDR: 10.0.3.0/24
      PrivateSubnet1CIDR: 10.0.11.0/24
      PrivateSubnet2CIDR: 10.0.12.0/24
      PrivateSubnet3CIDR: 10.0.13.0/24
    staging:
      PublicSubnet1CIDR: 10.0.1.0/24
      PublicSubnet2CIDR: 10.0.2.0/24
      PublicSubnet3CIDR: 10.0.3.0/24
      PrivateSubnet1CIDR: 10.0.11.0/24
      PrivateSubnet2CIDR: 10.0.12.0/24
      PrivateSubnet3CIDR: 10.0.13.0/24
    production:
      PublicSubnet1CIDR: 10.0.1.0/24
      PublicSubnet2CIDR: 10.0.2.0/24
      PublicSubnet3CIDR: 10.0.3.0/24
      PrivateSubnet1CIDR: 10.0.11.0/24
      PrivateSubnet2CIDR: 10.0.12.0/24
      PrivateSubnet3CIDR: 10.0.13.0/24

Resources:
  # ========================================
  # VPC
  # ========================================
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCIDR
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-order-processing-vpc'
        - Key: Environment
          Value: !Ref Environment

  # ========================================
  # Internet Gateway
  # ========================================
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-igw'
        - Key: Environment
          Value: !Ref Environment

  InternetGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref VPC

  # ========================================
  # Public Subnets
  # ========================================
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: !FindInMap [SubnetConfig, !Ref Environment, PublicSubnet1CIDR]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-public-subnet-1a'
        - Key: Type
          Value: Public

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: !FindInMap [SubnetConfig, !Ref Environment, PublicSubnet2CIDR]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-public-subnet-1b'
        - Key: Type
          Value: Public

  PublicSubnet3:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [2, !GetAZs '']
      CidrBlock: !FindInMap [SubnetConfig, !Ref Environment, PublicSubnet3CIDR]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-public-subnet-1c'
        - Key: Type
          Value: Public

  # ========================================
  # Private Subnets
  # ========================================
  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: !FindInMap [SubnetConfig, !Ref Environment, PrivateSubnet1CIDR]
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-private-subnet-1a'
        - Key: Type
          Value: Private

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: !FindInMap [SubnetConfig, !Ref Environment, PrivateSubnet2CIDR]
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-private-subnet-1b'
        - Key: Type
          Value: Private

  PrivateSubnet3:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [2, !GetAZs '']
      CidrBlock: !FindInMap [SubnetConfig, !Ref Environment, PrivateSubnet3CIDR]
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-private-subnet-1c'
        - Key: Type
          Value: Private

  # ========================================
  # Elastic IPs for NAT Gateways
  # ========================================
  NatGateway1EIP:
    Type: AWS::EC2::EIP
    DependsOn: InternetGatewayAttachment
    Properties:
      Domain: vpc
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-nat-eip-1a'

  NatGateway2EIP:
    Type: AWS::EC2::EIP
    DependsOn: InternetGatewayAttachment
    Properties:
      Domain: vpc
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-nat-eip-1b'

  NatGateway3EIP:
    Type: AWS::EC2::EIP
    DependsOn: InternetGatewayAttachment
    Properties:
      Domain: vpc
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-nat-eip-1c'

  # ========================================
  # NAT Gateways
  # ========================================
  NatGateway1:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGateway1EIP.AllocationId
      SubnetId: !Ref PublicSubnet1
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-nat-1a'

  NatGateway2:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGateway2EIP.AllocationId
      SubnetId: !Ref PublicSubnet2
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-nat-1b'

  NatGateway3:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGateway3EIP.AllocationId
      SubnetId: !Ref PublicSubnet3
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-nat-1c'

  # ========================================
  # Route Tables - Public
  # ========================================
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-public-rt'

  DefaultPublicRoute:
    Type: AWS::EC2::Route
    DependsOn: InternetGatewayAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet1

  PublicSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet2

  PublicSubnet3RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet3

  # ========================================
  # Route Tables - Private
  # ========================================
  PrivateRouteTable1:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-private-rt-1a'

  DefaultPrivateRoute1:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable1
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway1

  PrivateSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable1
      SubnetId: !Ref PrivateSubnet1

  PrivateRouteTable2:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-private-rt-1b'

  DefaultPrivateRoute2:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable2
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway2

  PrivateSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable2
      SubnetId: !Ref PrivateSubnet2

  PrivateRouteTable3:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-private-rt-1c'

  DefaultPrivateRoute3:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable3
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway3

  PrivateSubnet3RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable3
      SubnetId: !Ref PrivateSubnet3

  # ========================================
  # Security Group - Lambda Functions
  # ========================================
  LambdaSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: !Sub '${Environment}-lambda-sg'
      GroupDescription: Security group for Lambda functions
      VpcId: !Ref VPC
      SecurityGroupEgress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
          Description: HTTPS to AWS services
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-lambda-sg'

  # ========================================
  # VPC Endpoints - Gateway
  # ========================================
  DynamoDBEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VPC
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.dynamodb'
      VpcEndpointType: Gateway
      RouteTableIds:
        - !Ref PrivateRouteTable1
        - !Ref PrivateRouteTable2
        - !Ref PrivateRouteTable3

  S3Endpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VPC
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.s3'
      VpcEndpointType: Gateway
      RouteTableIds:
        - !Ref PrivateRouteTable1
        - !Ref PrivateRouteTable2
        - !Ref PrivateRouteTable3

  # ========================================
  # VPC Endpoints - Interface
  # ========================================
  VPCEndpointSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: !Sub '${Environment}-vpc-endpoint-sg'
      GroupDescription: Security group for VPC endpoints
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          SourceSecurityGroupId: !Ref LambdaSecurityGroup
          Description: HTTPS from Lambda
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-vpc-endpoint-sg'

  SQSEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VPC
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.sqs'
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      SubnetIds:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
        - !Ref PrivateSubnet3
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup

  SNSEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VPC
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.sns'
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      SubnetIds:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
        - !Ref PrivateSubnet3
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup

  SecretsManagerEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VPC
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.secretsmanager'
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      SubnetIds:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
        - !Ref PrivateSubnet3
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup

  KMSEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VPC
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.kms'
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      SubnetIds:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
        - !Ref PrivateSubnet3
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup

# ========================================
# Outputs
# ========================================
Outputs:
  VPCId:
    Description: VPC ID
    Value: !Ref VPC
    Export:
      Name: !Sub '${Environment}-VPC-ID'

  PublicSubnet1Id:
    Description: Public Subnet 1 ID
    Value: !Ref PublicSubnet1
    Export:
      Name: !Sub '${Environment}-PublicSubnet1-ID'

  PublicSubnet2Id:
    Description: Public Subnet 2 ID
    Value: !Ref PublicSubnet2
    Export:
      Name: !Sub '${Environment}-PublicSubnet2-ID'

  PublicSubnet3Id:
    Description: Public Subnet 3 ID
    Value: !Ref PublicSubnet3
    Export:
      Name: !Sub '${Environment}-PublicSubnet3-ID'

  PrivateSubnet1Id:
    Description: Private Subnet 1 ID
    Value: !Ref PrivateSubnet1
    Export:
      Name: !Sub '${Environment}-PrivateSubnet1-ID'

  PrivateSubnet2Id:
    Description: Private Subnet 2 ID
    Value: !Ref PrivateSubnet2
    Export:
      Name: !Sub '${Environment}-PrivateSubnet2-ID'

  PrivateSubnet3Id:
    Description: Private Subnet 3 ID
    Value: !Ref PrivateSubnet3
    Export:
      Name: !Sub '${Environment}-PrivateSubnet3-ID'

  LambdaSecurityGroupId:
    Description: Lambda Security Group ID
    Value: !Ref LambdaSecurityGroup
    Export:
      Name: !Sub '${Environment}-Lambda-SG-ID'

  VPCEndpointSecurityGroupId:
    Description: VPC Endpoint Security Group ID
    Value: !Ref VPCEndpointSecurityGroup
    Export:
      Name: !Sub '${Environment}-VPCEndpoint-SG-ID'
```

---

## 3️⃣ Deploy VPC Stack

### Deployment Commands

```powershell
# Navigate to infrastructure directory
cd infrastructure/cloudformation

# Deploy to dev environment
aws cloudformation create-stack `
  --stack-name dev-vpc-stack `
  --template-body file://vpc-stack.yaml `
  --parameters ParameterKey=Environment,ParameterValue=dev `
  --capabilities CAPABILITY_NAMED_IAM `
  --region us-east-1

# Check deployment status
aws cloudformation describe-stacks `
  --stack-name dev-vpc-stack `
  --query 'Stacks[0].StackStatus'

# Wait for completion (takes 5-10 minutes due to NAT Gateways)
aws cloudformation wait stack-create-complete `
  --stack-name dev-vpc-stack
```

### View Stack Outputs

```powershell
# Get all outputs
aws cloudformation describe-stacks `
  --stack-name dev-vpc-stack `
  --query 'Stacks[0].Outputs'

# Get specific output (VPC ID)
aws cloudformation describe-stacks `
  --stack-name dev-vpc-stack `
  --query 'Stacks[0].Outputs[?OutputKey==`VPCId`].OutputValue' `
  --output text
```

---

## 4️⃣ Verification and Testing

### Verify VPC Creation

```javascript
// verify-vpc.js
const AWS = require('aws-sdk');
const ec2 = new AWS.EC2({ region: 'us-east-1' });

async function verifyVPC() {
  console.log('🔍 Verifying VPC setup...\n');

  try {
    // Get VPC details
    const vpcs = await ec2.describeVpcs({
      Filters: [
        { Name: 'tag:Environment', Values: ['dev'] }
      ]
    }).promise();

    if (vpcs.Vpcs.length === 0) {
      console.log('❌ No VPC found with Environment=dev tag');
      return;
    }

    const vpc = vpcs.Vpcs[0];
    console.log(`✅ VPC Found: ${vpc.VpcId}`);
    console.log(`   CIDR: ${vpc.CidrBlock}`);
    console.log(`   DNS Support: ${vpc.EnableDnsSupport}`);
    console.log(`   DNS Hostnames: ${vpc.EnableDnsHostnames}\n`);

    // Get subnets
    const subnets = await ec2.describeSubnets({
      Filters: [
        { Name: 'vpc-id', Values: [vpc.VpcId] }
      ]
    }).promise();

    console.log(`✅ Subnets: ${subnets.Subnets.length}`);
    subnets.Subnets.forEach(subnet => {
      const nameTag = subnet.Tags.find(t => t.Key === 'Name');
      console.log(`   - ${nameTag?.Value || subnet.SubnetId}: ${subnet.CidrBlock} (${subnet.AvailabilityZone})`);
    });

    // Get NAT Gateways
    const natGateways = await ec2.describeNatGateways({
      Filter: [
        { Name: 'vpc-id', Values: [vpc.VpcId] }
      ]
    }).promise();

    console.log(`\n✅ NAT Gateways: ${natGateways.NatGateways.length}`);
    natGateways.NatGateways.forEach(nat => {
      console.log(`   - ${nat.NatGatewayId}: ${nat.State}`);
    });

    // Get VPC Endpoints
    const endpoints = await ec2.describeVpcEndpoints({
      Filters: [
        { Name: 'vpc-id', Values: [vpc.VpcId] }
      ]
    }).promise();

    console.log(`\n✅ VPC Endpoints: ${endpoints.VpcEndpoints.length}`);
    endpoints.VpcEndpoints.forEach(ep => {
      console.log(`   - ${ep.ServiceName}: ${ep.VpcEndpointType} (${ep.State})`);
    });

    console.log('\n✨ VPC verification complete!');
  } catch (error) {
    console.error('❌ Error verifying VPC:', error.message);
  }
}

verifyVPC();
```

```powershell
# Run verification
node verify-vpc.js
```

---

## 5️⃣ Cost Optimization Options

### Option 1: Single NAT Gateway (Dev Only)

```yaml
# Cost: ~$32/month (1 NAT Gateway)
# For dev-stack.yaml only - modify to use single NAT Gateway

# Remove NatGateway2 and NatGateway3
# Update PrivateRouteTable2 and PrivateRouteTable3 to use NatGateway1
DefaultPrivateRoute2:
  Type: AWS::EC2::Route
  Properties:
    RouteTableId: !Ref PrivateRouteTable2
    DestinationCidrBlock: 0.0.0.0/0
    NatGatewayId: !Ref NatGateway1  # Changed from NatGateway2

DefaultPrivateRoute3:
  Type: AWS::EC2::Route
  Properties:
    RouteTableId: !Ref PrivateRouteTable3
    DestinationCidrBlock: 0.0.0.0/0
    NatGatewayId: !Ref NatGateway1  # Changed from NatGateway3
```

### Option 2: VPC Endpoints Only (No NAT)

```yaml
# Cost: ~$7-10/month (Interface endpoints only)
# Remove all NAT Gateways
# Lambda can only access AWS services via VPC endpoints
# No internet access from Lambda (fine if you don't need external APIs)
```

### Option 3: No VPC (Simplest)

```javascript
// Cost: $0
// Deploy Lambda functions outside VPC
// They can access AWS services and internet by default
// Trade-off: Less network isolation

const lambdaConfig = {
  vpc: undefined  // Remove VPC config
};
```

---

## 6️⃣ Network Access Control Lists (NACLs)

### Custom NACL for Private Subnets

```yaml
# Add to vpc-stack.yaml (optional - for enhanced security)

PrivateNetworkAcl:
  Type: AWS::EC2::NetworkAcl
  Properties:
    VpcId: !Ref VPC
    Tags:
      - Key: Name
        Value: !Sub '${Environment}-private-nacl'

# Inbound Rules
PrivateNetworkAclInboundHTTPS:
  Type: AWS::EC2::NetworkAclEntry
  Properties:
    NetworkAclId: !Ref PrivateNetworkAcl
    RuleNumber: 100
    Protocol: 6  # TCP
    RuleAction: allow
    CidrBlock: 0.0.0.0/0
    PortRange:
      From: 443
      To: 443

PrivateNetworkAclInboundEphemeral:
  Type: AWS::EC2::NetworkAclEntry
  Properties:
    NetworkAclId: !Ref PrivateNetworkAcl
    RuleNumber: 200
    Protocol: 6  # TCP
    RuleAction: allow
    CidrBlock: 0.0.0.0/0
    PortRange:
      From: 1024
      To: 65535

# Outbound Rules
PrivateNetworkAclOutboundHTTPS:
  Type: AWS::EC2::NetworkAclEntry
  Properties:
    NetworkAclId: !Ref PrivateNetworkAcl
    RuleNumber: 100
    Protocol: 6  # TCP
    Egress: true
    RuleAction: allow
    CidrBlock: 0.0.0.0/0
    PortRange:
      From: 443
      To: 443

PrivateNetworkAclOutboundEphemeral:
  Type: AWS::EC2::NetworkAclEntry
  Properties:
    NetworkAclId: !Ref PrivateNetworkAcl
    RuleNumber: 200
    Protocol: 6  # TCP
    Egress: true
    RuleAction: allow
    CidrBlock: 0.0.0.0/0
    PortRange:
      From: 1024
      To: 65535

# Associate with private subnets
PrivateSubnet1NetworkAclAssociation:
  Type: AWS::EC2::SubnetNetworkAclAssociation
  Properties:
    SubnetId: !Ref PrivateSubnet1
    NetworkAclId: !Ref PrivateNetworkAcl
```

---

## 7️⃣ Monitoring VPC

### CloudWatch Metrics

```javascript
// Enable VPC Flow Logs
const flowLogConfig = {
  resourceType: 'VPC',
  resourceId: 'vpc-xxxxxx',
  trafficType: 'ALL',  // ALL, ACCEPT, or REJECT
  logDestinationType: 'cloud-watch-logs',
  logGroupName: '/aws/vpc/flowlogs',
  deliverLogsPermissionArn: 'arn:aws:iam::123456789012:role/flowlogsRole'
};
```

### CloudFormation Addition

```yaml
# Add to vpc-stack.yaml

VPCFlowLogsRole:
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
                - logs:DescribeLogGroups
                - logs:DescribeLogStreams
              Resource: '*'

VPCFlowLogsLogGroup:
  Type: AWS::Logs::LogGroup
  Properties:
    LogGroupName: !Sub '/aws/vpc/${Environment}-flowlogs'
    RetentionInDays: 7

VPCFlowLog:
  Type: AWS::EC2::FlowLog
  Properties:
    ResourceType: VPC
    ResourceId: !Ref VPC
    TrafficType: ALL
    LogDestinationType: cloud-watch-logs
    LogGroupName: !Ref VPCFlowLogsLogGroup
    DeliverLogsPermissionArn: !GetAtt VPCFlowLogsRole.Arn
    Tags:
      - Key: Name
        Value: !Sub '${Environment}-vpc-flowlog'
```

---

## 8️⃣ Multi-Stage VPC Strategy

### Strategy 1: Separate VPCs per Stage

```javascript
const multiStageStrategy = {
  dev: {
    vpc: 'dev-vpc (10.0.0.0/16)',
    natGateways: 1,  // Cost optimization
    endpoints: ['DynamoDB', 'S3'],  // Minimal
    cost: '$32/month'
  },
  staging: {
    vpc: 'staging-vpc (10.1.0.0/16)',
    natGateways: 2,  // Some redundancy
    endpoints: ['DynamoDB', 'S3', 'SQS', 'SNS'],
    cost: '$64/month'
  },
  production: {
    vpc: 'prod-vpc (10.2.0.0/16)',
    natGateways: 3,  // Full redundancy
    endpoints: ['All services'],
    cost: '$96/month'
  }
};
```

### Strategy 2: Shared VPC with Separate Subnets

```javascript
const sharedVPCStrategy = {
  vpc: '10.0.0.0/16',
  subnets: {
    dev: '10.0.0.0/18',      // 16,384 IPs
    staging: '10.0.64.0/18',  // 16,384 IPs
    production: '10.0.128.0/18' // 16,384 IPs
  },
  isolation: 'Security Groups + IAM',
  cost: 'Lower (shared NAT Gateways)',
  risk: 'Requires strict security controls'
};
```

---

## 🎯 Best Practices

### DO ✅

1. **Use VPC Endpoints**: Save costs and improve security
2. **Enable DNS hostnames**: Required for VPC endpoints
3. **Tag everything**: Environment, Project, CostCenter
4. **Multi-AZ**: Deploy across multiple availability zones
5. **Flow Logs**: Enable for security auditing
6. **Private subnets**: Place Lambda in private subnets
7. **Security groups**: Use security groups over NACLs (stateful)

### DON'T ❌

1. **Don't use default VPC**: Create dedicated VPCs
2. **Don't over-provision**: Start small, scale as needed
3. **Don't hardcode IPs**: Use CloudFormation outputs
4. **Don't skip monitoring**: Always enable Flow Logs
5. **Don't ignore costs**: NAT Gateways are expensive
6. **Don't use overly permissive NACLs**: Be specific
7. **Don't forget cleanup**: Delete unused resources

---

## 🎯 Next Steps

VPC and networking setup complete! You now have:

- ✅ VPC with public and private subnets
- ✅ NAT Gateways for internet access
- ✅ VPC endpoints for AWS services
- ✅ Security groups configured
- ✅ Multi-AZ deployment ready
- ✅ Cost-optimized for dev environment

**Ready to proceed?**

👉 [IAM Security Setup](./04_IAM_SECURITY_SETUP.md)

---

**Last Updated**: November 12, 2025  
**Version**: 1.0.0
