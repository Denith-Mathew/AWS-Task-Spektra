# AWS CloudFormation — EC2 + RDS MySQL Stack

## Overview

This CloudFormation template provisions a production-style AWS environment featuring:

- A **custom VPC** with public and private subnets
- An **EC2 instance** (web server) in the public subnet
- A **MySQL RDS instance** in private subnets (not internet-accessible)
- Proper **security group segmentation** between the web tier and database tier

This is a classic **two-tier architecture** — the EC2 instance can reach the database, but the database is hidden from the internet.

---

## Architecture

```
Internet
   │
   ▼
Internet Gateway
   │
   ▼
VPC (192.168.0.0/20)
   │
   ├── Public Subnet (192.168.0.0/24)
   │       └── EC2 Instance (web-sg: allows SSH:22, HTTP:80)
   │
   ├── Private Subnet 1 (192.168.1.0/24)  ← AZ 0
   │       └── RDS MySQL (db-sg: allows MySQL:3306 from web-sg only)
   │
   └── Private Subnet 2 (192.168.2.0/24)  ← AZ 1
           └── RDS MySQL (subnet group requirement)
```

> The RDS instance sits in **two private subnets across two AZs** (required by AWS for DB Subnet Groups), but `MultiAZ: false` means only one AZ is actively used.

---

## Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `AMIId` | `String` | `ami-02dfbd4ff395f2a1b` | AMI for the EC2 instance |
| `InstanceType` | `String` | `t3.small` | EC2 instance size |
| `DBUsername` | `String` | `admin` | Master username for RDS MySQL |
| `DBPassword` | `String` (NoEcho) | *(required)* | Master password — hidden in console logs |

> ✅ `NoEcho: true` on `DBPassword` ensures the password is never displayed in the CloudFormation console or CLI output.

---

## Resources Explained

### 1. VPC — `taskVpc`

```yaml
CidrBlock: 192.168.0.0/20
EnableDnsSupport: true
EnableDnsHostnames: true
```

- **CIDR** `192.168.0.0/20` gives 4096 IP addresses across all subnets
- DNS support and hostnames enabled — required for RDS endpoint resolution

---

### 2. Subnets

| Resource | CIDR | Type | AZ |
|----------|------|------|----|
| `taskPublicSubnet` | `192.168.0.0/24` | Public | Auto (any) |
| `taskPrivateSubnet1` | `192.168.1.0/24` | Private | AZ index 0 |
| `taskPrivateSubnet2` | `192.168.2.0/24` | Private | AZ index 1 |

- The **public subnet** has `MapPublicIpOnLaunch: true` so EC2 gets a public IP automatically
- The **two private subnets** are in different AZs — AWS requires this for RDS DB Subnet Groups

---

### 3. Internet Gateway & Routing

- **`taskInternetGateway`** — attached to the VPC via `taskVPCGatewayAttachment`
- **Public Route Table** — routes all traffic (`0.0.0.0/0`) to the IGW; associated with the public subnet only
- **Private Route Table** — has no internet route; associated with both private subnets, keeping the RDS instance isolated

---

### 4. Security Groups

#### Web Security Group — `taskWebSecurityGroup`

| Rule | Protocol | Port | Source |
|------|----------|------|--------|
| SSH | TCP | 22 | `0.0.0.0/0` |
| HTTP | TCP | 80 | `0.0.0.0/0` |

> ⚠️ SSH is open to the internet. For production, restrict port 22 to your specific IP.

#### DB Security Group — `taskDBSecurityGroup`

| Rule | Protocol | Port | Source |
|------|----------|------|--------|
| MySQL | TCP | 3306 | `taskWebSecurityGroup` (SG reference) |

> ✅ This is a best practice — the DB only accepts connections **from the EC2 web security group**, not from any IP. This is called **security group chaining**.

---

### 5. EC2 Instance — `taskEC2Instance`

```yaml
ImageId: !Ref AMIId          # Provided as parameter
InstanceType: !Ref InstanceType  # t3.small by default
SubnetId: !Ref taskPublicSubnet  # Placed in public subnet
SecurityGroupIds:
  - !Ref taskWebSecurityGroup
```

- Launched in the **public subnet** with a public IP
- No `KeyName` defined — add one if you need SSH access
- No `UserData` — no software is auto-installed; you'd need to configure the server manually or add a UserData script

---

### 6. RDS DB Subnet Group — `taskDBSubnetGroup`

```yaml
SubnetIds:
  - !Ref taskPrivateSubnet1
  - !Ref taskPrivateSubnet2
```

AWS requires a **DB Subnet Group** with subnets in at least **two different AZs** before an RDS instance can be created — even for single-AZ deployments.

---

### 7. RDS MySQL Instance — `taskRDSInstance`

```yaml
DBInstanceClass: db.t3.micro
AllocatedStorage: 20          # 20 GB storage
Engine: mysql
EngineVersion: '8.0'
MasterUsername: !Ref DBUsername
MasterUserPassword: !Ref DBPassword
MultiAZ: false                # Single AZ (not highly available)
PubliclyAccessible: false     # No public internet access
```

- Runs **MySQL 8.0** on a `db.t3.micro` instance
- **Not publicly accessible** — only reachable from within the VPC
- **Single-AZ** — lower cost, but no automatic failover

---

## Outputs

| Output | Description |
|--------|-------------|
| `EC2PublicIP` | Public IP of the EC2 instance — use for SSH or HTTP |
| `RDSEndpointAddress` | Hostname to connect to MySQL (e.g., from EC2) |
| `RDSEndpointPort` | Port for MySQL connection (typically `3306`) |

**Example connection from EC2:**
```bash
mysql -h <RDSEndpointAddress> -P 3306 -u admin -p
```

---

## Full Template

```yaml
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
```

---

## Deployment Steps

1. Open the **AWS CloudFormation Console**
2. Click **Create Stack → With new resources**
3. Upload this template
4. Fill in the parameters:
   - `AMIId` — use the correct AMI for your region
   - `InstanceType` — default is `t3.small`
   - `DBUsername` — default is `admin`
   - `DBPassword` — enter a strong password (min 8 characters for MySQL)
5. Click through and **Create Stack**
6. Wait ~5–10 minutes for RDS to provision
7. Go to the **Outputs** tab to retrieve the EC2 IP and RDS endpoint

---

## Potential Improvements

| Issue | Recommendation |
|-------|---------------|
| SSH open to world (`0.0.0.0/0`) | Restrict port 22 to your specific IP |
| No EC2 Key Pair | Add a `KeyName` parameter to enable SSH login |
| No EC2 UserData | Add a startup script to install a web server (Apache/Nginx) |
| Hardcoded AMI ID | Use SSM Parameter Store (`/aws/service/ami-amazon-linux-latest/...`) for always-latest AMI |
| Password in parameters | Use **AWS Secrets Manager** to store and rotate the DB password securely |
| `MultiAZ: false` | Set to `true` for production workloads to enable automatic failover |
| No automated backups specified | Add `BackupRetentionPeriod` to the RDS instance for point-in-time recovery |
