AWSTemplateFormatVersion: '2010-09-09'
Description: |
  Network Stack - Creates VPC, public/private subnets across 2 AZs, Internet Gateway, NAT Gateway, Route Tables, and Security Groups.
Parameters:
  EnvironmentName:
    Type: String
    Default: dev
    Description: Prefix used for naming all resources
Mappings:
  SubnetConfig:
    VPC:
      CIDR: 10.0.0.0/16
    PublicSubnet1:
      CIDR: 10.0.1.0/24
    PublicSubnet2:
      CIDR: 10.0.2.0/24
    PrivateSubnet1:
      CIDR: 10.0.3.0/24
    PrivateSubnet2:
      CIDR: 10.0.4.0/24
Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !FindInMap
        - SubnetConfig
        - VPC
        - CIDR
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-vpc
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-igw
  InternetGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !FindInMap
        - SubnetConfig
        - PublicSubnet1
        - CIDR
      AvailabilityZone: !Select
        - 0
        - !GetAZs ''
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-public-subnet-1
        - Key: Type
          Value: Public
  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !FindInMap
        - SubnetConfig
        - PublicSubnet2
        - CIDR
      AvailabilityZone: !Select
        - 1
        - !GetAZs ''
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-public-subnet-2
        - Key: Type
          Value: Public
  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !FindInMap
        - SubnetConfig
        - PrivateSubnet1
        - CIDR
      AvailabilityZone: !Select
        - 0
        - !GetAZs ''
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-private-subnet-1
        - Key: Type
          Value: Private
  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !FindInMap
        - SubnetConfig
        - PrivateSubnet2
        - CIDR
      AvailabilityZone: !Select
        - 1
        - !GetAZs ''
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-private-subnet-2
        - Key: Type
          Value: Private
  NatGatewayEIP:
    Type: AWS::EC2::EIP
    DependsOn: InternetGatewayAttachment
    Properties:
      Domain: vpc
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-nat-eip
  NatGateway:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGatewayEIP.AllocationId
      SubnetId: !Ref PublicSubnet1
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-nat-gw
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-public-rt
  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: InternetGatewayAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
  PublicSubnet1RouteTableAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet1
      RouteTableId: !Ref PublicRouteTable
  PublicSubnet2RouteTableAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet2
      RouteTableId: !Ref PublicRouteTable
  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-private-rt
  PrivateRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway
  PrivateSubnet1RouteTableAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet1
      RouteTableId: !Ref PrivateRouteTable
  PrivateSubnet2RouteTableAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet2
      RouteTableId: !Ref PrivateRouteTable
  AlbSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for the Application Load Balancer
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
          Description: Allow HTTP from internet
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
          Description: Allow HTTPS from internet
      SecurityGroupEgress:
        - IpProtocol: -1
          CidrIp: 0.0.0.0/0
          Description: Allow all outbound
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-alb-sg
  Ec2SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for NGINX EC2 instances - only allows traffic from ALB
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !Ref AlbSecurityGroup
          Description: Allow HTTP from ALB only
      SecurityGroupEgress:
        - IpProtocol: -1
          CidrIp: 0.0.0.0/0
          Description: Allow all outbound (for yum updates, SSM)
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-ec2-sg
Outputs:
  VpcId:
    Description: VPC ID
    Value: !Ref VPC
  PublicSubnet1Id:
    Description: Public Subnet 1 ID (AZ-1)
    Value: !Ref PublicSubnet1
  PublicSubnet2Id:
    Description: Public Subnet 2 ID (AZ-2)
    Value: !Ref PublicSubnet2
  PrivateSubnet1Id:
    Description: Private Subnet 1 ID (AZ-1)
    Value: !Ref PrivateSubnet1
  PrivateSubnet2Id:
    Description: Private Subnet 2 ID (AZ-2)
    Value: !Ref PrivateSubnet2
  AlbSecurityGroupId:
    Description: Security Group ID for the ALB
    Value: !Ref AlbSecurityGroup
  Ec2SecurityGroupId:
    Description: Security Group ID for EC2 instances
    Value: !Ref Ec2SecurityGroup
  NatGatewayId:
    Description: NAT Gateway ID
    Value: !Ref NatGateway

  <img src="Screenshot 2026-03-19 181228.png">

  <img src="screenshot-nested\Screenshot 2026-03-19 181331.png">