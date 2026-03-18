AWSTemplateFormatVersion: '2010-09-09'
Description: 'AWS CloudFormation Template: EC2 Instance'

Parameters:
  AMIId:
    Type: String
    Default: ami-02dfbd4ff395f2a1b
    Description: The AMI ID for the EC2 instance

  InstanceType:
    Type: String
    Default: t3.small
    Description: The instance type for the EC2 instance
  DBUsername:
    # ? Fix 3: no hardcoded username
    Type: String
    Default: admin
    Description: Master username for RDS MySQL

  DBPassword:
    Type: String
    NoEcho: true
    Description: Master password for RDS MySQL

Resources:
  taskVpc:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 192.168.0.0/20
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: taskVpc

  taskPublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref taskVpc
      CidrBlock: 192.168.0.0/24
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: taskPublicSubnet

  taskPrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref taskVpc
      CidrBlock: 192.168.1.0/24
      AvailabilityZone: !Select
        - 0
        - !GetAZs ''
      Tags:
        - Key: Name
          Value: taskPrivateSubnet-1

  taskPrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref taskVpc
      CidrBlock: 192.168.2.0/24
      AvailabilityZone: !Select
        - 1
        - !GetAZs ''
      Tags:
        - Key: Name
          Value: taskPrivateSubnet-2

  taskInternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: taskIGW

  taskVPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref taskVpc
      InternetGatewayId: !Ref taskInternetGateway

  taskPublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref taskVpc
      Tags:
        - Key: Name
          Value: taskPublicRouteTable

  taskPublicRoute:
    Type: AWS::EC2::Route
    DependsOn: taskVPCGatewayAttachment
    Properties:
      RouteTableId: !Ref taskPublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref taskInternetGateway

  taskPublicSubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref taskPublicSubnet
      RouteTableId: !Ref taskPublicRouteTable

  taskPrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref taskVpc
      Tags:
        - Key: Name
          Value: taskPrivateRouteTable

  taskPrivate1SubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref taskPrivateRouteTable
      SubnetId: !Ref taskPrivateSubnet1

  taskPrivate2SubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref taskPrivateRouteTable
      SubnetId: !Ref taskPrivateSubnet2

  taskWebSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow SSH and HTTP access
      VpcId: !Ref taskVpc
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: web-sg

  taskDBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow MySQL access from web security group
      VpcId: !Ref taskVpc
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          SourceSecurityGroupId: !Ref taskWebSecurityGroup
      Tags:
        - Key: Name
          Value: db-sg

  taskEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref AMIId
      InstanceType: !Ref InstanceType
      SubnetId: !Ref taskPublicSubnet
      SecurityGroupIds:
        - !Ref taskWebSecurityGroup
      Tags:
        - Key: Name
          Value: taskEC2Instance

  taskDBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: Subnet group for RDS instance
      SubnetIds:
        - !Ref taskPrivateSubnet1
        - !Ref taskPrivateSubnet2
      Tags:
        - Key: Name
          Value: taskDBSubnetGroup

  taskRDSInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceIdentifier: taskRDSInstance
      DBInstanceClass: db.t3.micro
      AllocatedStorage: 20
      Engine: mysql
      EngineVersion: '8.0'
      MasterUsername: !Ref DBUsername
      MasterUserPassword: !Ref DBPassword
      VPCSecurityGroups:
        - !Ref taskDBSecurityGroup
      DBSubnetGroupName: !Ref taskDBSubnetGroup
      MultiAZ: false
      PubliclyAccessible: false

Outputs:
  EC2PublicIP:
    Description: Public IP address of the EC2 instance
    Value: !GetAtt taskEC2Instance.PublicIp

  RDSEndpointAddress:
    Description: Connection endpoint hostname for the RDS MySQL instance
    Value: !GetAtt taskRDSInstance.Endpoint.Address

  RDSEndpointPort:
    Description: Port for the RDS MySQL instance
    Value: !GetAtt taskRDSInstance.Endpoint.Port