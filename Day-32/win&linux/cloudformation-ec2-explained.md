# AWS CloudFormation — EC2 Linux/Windows Web Server Stack

## Overview

This CloudFormation template provisions a complete, self-contained AWS environment that launches one or both of the following:

- A **Linux EC2 instance** running **Apache HTTP Server**
- A **Windows EC2 instance** running **IIS (Internet Information Services)**

The stack includes its own VPC, public subnet, internet gateway, route table, and security group — making it fully standalone and repeatable.

---

## Architecture

```
Internet
   │
   ▼
Internet Gateway
   │
   ▼
VPC (10.0.0.0/16)
   │
   ▼
Public Subnet (10.0.1.0/24)
   │
   ├── Linux EC2 (Apache)   ← conditional on OSChoice
   └── Windows EC2 (IIS)    ← conditional on OSChoice
```

---

## Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `KeyName` | `AWS::EC2::KeyPair::KeyName` | *(required)* | EC2 Key Pair for SSH/RDP access |
| `InstanceType` | `String` | `t3.micro` | EC2 instance size |
| `OSChoice` | `String` | `Linux` | Which OS(es) to launch: `Linux`, `Windows`, or `Both` |
| `LinuxAMI` | SSM Parameter | Amazon Linux 2023 (latest) | Auto-resolved from SSM Parameter Store |
| `WindowsAMI` | SSM Parameter | Windows Server 2022 (latest) | Auto-resolved from SSM Parameter Store |

> **Tip:** Using SSM Parameter Store for AMI IDs ensures the stack always uses the latest Amazon-maintained images without manual updates.

---

## Conditions

The template uses CloudFormation conditions to decide which EC2 instances to create:

| Condition | Triggers when |
|-----------|--------------|
| `LaunchLinux` | `OSChoice` is `Linux` or `Both` |
| `LaunchWindows` | `OSChoice` is `Windows` or `Both` |

This avoids creating unused resources while keeping a single unified template.

---

## Resources Created

### 1. Networking

- **VPC** — CIDR `10.0.0.0/16`
- **Public Subnet** — CIDR `10.0.1.0/24`, auto-assigns public IPs, placed in the first available AZ
- **Internet Gateway** — Attached to the VPC for outbound/inbound internet traffic
- **Route Table + Route** — Default route (`0.0.0.0/0`) pointing to the Internet Gateway
- **Subnet-Route Table Association** — Links the public subnet to the route table

### 2. Security Group

| Rule | Protocol | Port | Source | Purpose |
|------|----------|------|--------|---------|
| SSH | TCP | 22 | `0.0.0.0/0` | Linux access (open to all — consider restricting) |
| RDP | TCP | 3389 | `14.99.81.62/32` | Windows access (restricted to a specific IP) |
| HTTP | TCP | 80 | `0.0.0.0/0` | Public web traffic for both servers |

> ⚠️ **Security Note:** SSH is currently open to the world (`0.0.0.0/0`). It's recommended to restrict this to your own IP, similar to how RDP is already restricted.

### 3. Linux EC2 Instance *(conditional)*

- Launched only when `LaunchLinux` condition is true
- Uses **Amazon Linux 2023** AMI (from SSM)
- **UserData** script runs at launch:
  - Updates all packages (`dnf update`)
  - Installs Apache (`httpd`)
  - Starts and enables the Apache service
  - Writes a simple HTML page to `/var/www/html/index.html`

### 4. Windows EC2 Instance *(conditional)*

- Launched only when `LaunchWindows` condition is true
- Uses **Windows Server 2022** AMI (from SSM)
- **UserData** PowerShell script runs at launch:
  - Installs IIS with management tools (`Install-WindowsFeature`)
  - Writes a simple HTML page to `C:\inetpub\wwwroot\index.html`

---

## Outputs

| Output | Condition | Description |
|--------|-----------|-------------|
| `LinuxPublicIP` | `LaunchLinux` | Public IP of the Linux instance |
| `WindowsPublicIP` | `LaunchWindows` | Public IP of the Windows instance |

After stack creation, use these IPs to:
- Open `http://<IP>` in a browser to see the web server
- SSH into Linux: `ssh -i <KeyName>.pem ec2-user@<LinuxPublicIP>`
- RDP into Windows using `<WindowsPublicIP>` from the whitelisted IP

---

## Full Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: EC2 Linux/Windows/Both with Apache/IIS and fixed MyIP restriction

# ---------------- PARAMETERS ----------------
Parameters:

  KeyName:
    Type: AWS::EC2::KeyPair::KeyName

  InstanceType:
    Type: String
    Default: t3.micro

  OSChoice:
    Type: String
    AllowedValues:
      - Linux
      - Windows
      - Both
    Default: Linux

  LinuxAMI:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64

  WindowsAMI:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-windows-latest/Windows_Server-2022-English-Full-Base

# ---------------- CONDITIONS ----------------
Conditions:
  LaunchLinux: !Or
    - !Equals
      - !Ref OSChoice
      - Linux
    - !Equals
      - !Ref OSChoice
      - Both
  LaunchWindows: !Or
    - !Equals
      - !Ref OSChoice
      - Windows
    - !Equals
      - !Ref OSChoice
      - Both

# ---------------- RESOURCES ----------------
Resources:

  # VPC
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16

  # Subnet
  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      MapPublicIpOnLaunch: true
      AvailabilityZone: !Select
        - 0
        - !GetAZs ''

  # Internet Gateway
  InternetGateway:
    Type: AWS::EC2::InternetGateway

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  # Route Table
  RouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref RouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  SubnetRouteAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet
      RouteTableId: !Ref RouteTable

  # Security Group (MyIP HARDCODED HERE)
  InstanceSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow MyIP SSH/RDP and HTTP public
      VpcId: !Ref VPC
      SecurityGroupIngress:

        # SSH (Linux)
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0

        # RDP (Windows)
        - IpProtocol: tcp
          FromPort: 3389
          ToPort: 3389
          CidrIp: 14.99.81.62/32

        # HTTP (Public Website)
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  # ---------------- LINUX EC2 ----------------
  LinuxInstance:
    Condition: LaunchLinux
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      SubnetId: !Ref PublicSubnet
      SecurityGroupIds:
        - !Ref InstanceSecurityGroup
      ImageId: !Ref LinuxAMI
      UserData: !Base64 |
        #!/bin/bash
        dnf update -y
        dnf install httpd -y
        systemctl start httpd
        systemctl enable httpd
        echo "<h1>Linux Apache Server</h1>" > /var/www/html/index.html

  # ---------------- WINDOWS EC2 ----------------
  WindowsInstance:
    Condition: LaunchWindows
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      SubnetId: !Ref PublicSubnet
      SecurityGroupIds:
        - !Ref InstanceSecurityGroup
      ImageId: !Ref WindowsAMI
      UserData: !Base64 |
        Fn::Sub: |
          <powershell>
          Install-WindowsFeature -name Web-Server -IncludeManagementTools
          Set-Content -Path "C:\inetpub\wwwroot\index.html" -Value "<h1>Windows IIS Server</h1>"
          </powershell>

# ---------------- OUTPUTS ----------------
Outputs:

  LinuxPublicIP:
    Condition: LaunchLinux
    Description: Linux Server Public IP
    Value: !GetAtt LinuxInstance.PublicIp

  WindowsPublicIP:
    Condition: LaunchWindows
    Description: Windows Server Public IP
    Value: !GetAtt WindowsInstance.PublicIp
```

---

## Deployment Steps

1. Open the **AWS CloudFormation Console**
2. Click **Create Stack → With new resources**
3. Upload this template file
4. Fill in the parameters:
   - Select your **KeyName**
   - Choose **OSChoice** (`Linux`, `Windows`, or `Both`)
   - Optionally change the **InstanceType**
5. Click through and **Create Stack**
6. Once the stack status is `CREATE_COMPLETE`, go to the **Outputs** tab to get the public IP(s)

---

## Potential Improvements

- **Restrict SSH** to your own IP (same as RDP) instead of `0.0.0.0/0`
- **Add an Elastic IP** to keep the public IP stable across instance reboots
- **Use a Parameter for MyIP** instead of hardcoding `14.99.81.62/32` in the template
- **Add HTTPS (port 443)** with an SSL certificate for production use
- **Add CloudWatch Logs** to capture UserData script output for troubleshooting
