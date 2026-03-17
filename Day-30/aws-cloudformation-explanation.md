# AWS CloudFormation: VPC + EC2 Setup — Full Explanation

---

## Overview

This CloudFormation template provisions a **complete AWS networking and compute stack** including:
- A custom **VPC** (Virtual Private Cloud)
- A **public subnet** with internet access
- An **EC2 instance** (Amazon Linux 2023) with password-based SSH enabled
- An **Apache HTTP server** pre-installed

The screenshot shows a successful **password-based SSH login** to the EC2 instance using the Windows command prompt.

---

## 1. Parameters (User Inputs)

These are configurable values you provide when deploying the stack.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `VpcCidr` | `10.0.0.0/16` | IP address range for the entire VPC |
| `PublicSubnetCidr` | `10.0.1.0/24` | IP range for the public subnet |
| `InstanceType` | `t3.micro` | EC2 instance size |
| `KeyName` | _(required)_ | Name of existing EC2 Key Pair for SSH |
| `AmiId` | `ami-02dfbd4ff395f2a1b` | Amazon Machine Image ID (Amazon Linux 2023) |
| `EnablePublicIP` | `true` | Whether to auto-assign a public IP to the instance |
| `SSHLocation` | `0.0.0.0/0` | IP range allowed to SSH (default: everywhere — **not recommended for production**) |
| `Password` | _(required, hidden)_ | Password for `ec2-user` — hidden using `NoEcho: true` |

> ⚠️ **Security Note:** `SSHLocation: 0.0.0.0/0` allows SSH from any IP. In production, restrict this to your own IP or a corporate CIDR range.

---

## 2. VPC Resources

### 2.1 VPC (`MyVPC`)
```yaml
Type: AWS::EC2::VPC
CidrBlock: 10.0.0.0/16
```
- Creates an isolated network in AWS
- `EnableDnsSupport` and `EnableDnsHostnames` allow AWS DNS to resolve hostnames inside the VPC

---

### 2.2 Internet Gateway (`MyInternetGateway` + `AttachIGW`)
```yaml
Type: AWS::EC2::InternetGateway
```
- An **Internet Gateway (IGW)** is required for any resource inside the VPC to communicate with the public internet
- `AttachIGW` binds the IGW to `MyVPC`

---

### 2.3 Public Subnet (`PublicSubnet`)
```yaml
CidrBlock: 10.0.1.0/24
MapPublicIpOnLaunch: true
AvailabilityZone: !Select [0, !GetAZs ""]
```
- A subnet is a division of the VPC's address space
- `MapPublicIpOnLaunch: true` automatically assigns public IPs to any EC2 launched here
- `!Select [0, !GetAZs ""]` automatically picks the **first Availability Zone** in the region

---

### 2.4 Route Table (`PublicRouteTable` + `DefaultRoute` + `SubnetRouteTableAssociation`)

```yaml
# Default route sends all internet traffic through the Internet Gateway
DestinationCidrBlock: 0.0.0.0/0
GatewayId: !Ref MyInternetGateway
```
- A **Route Table** defines where network traffic is directed
- The default route (`0.0.0.0/0`) sends all external traffic to the IGW
- `SubnetRouteTableAssociation` links the route table to `PublicSubnet`
- `DependsOn: AttachIGW` ensures the IGW is attached before the route is created

---

### 2.5 Security Group (`InstanceSecurityGroup`)

```yaml
SecurityGroupIngress:
  - Port 22 (SSH)  — allowed from SSHLocation (default: anywhere)
  - Port 80 (HTTP) — allowed from anywhere
```
- Acts as a **virtual firewall** for the EC2 instance
- Allows **SSH** (port 22) for remote terminal access
- Allows **HTTP** (port 80) for web traffic to Apache

> 💡 Only **inbound** rules are defined here. All **outbound** traffic is allowed by default in AWS.

---

## 3. EC2 Instance (`MyEC2Instance`)

```yaml
Type: AWS::EC2::Instance
InstanceType: t3.micro
ImageId: ami-02dfbd4ff395f2a1b   # Amazon Linux 2023
SubnetId: !Ref PublicSubnet
SecurityGroupIds:
  - !Ref InstanceSecurityGroup
```

The instance is launched in the public subnet with the security group applied.

---

## 4. User Data Script (Bootstrap on Launch)

The `UserData` block runs automatically when the EC2 instance first boots. It performs several setup tasks:

### Step 1: System Update
```bash
yum update -y
```
Updates all installed packages to their latest versions.

---

### Step 2: Set Password for `ec2-user`
```bash
echo "ec2-user:${Password}" | chpasswd
```
Sets the password for the default `ec2-user` account using the value provided in the `Password` parameter.

> 🔒 The password is passed securely — `NoEcho: true` prevents it from being shown in the console or logs.

---

### Step 3: Enable Password Authentication in SSH
```bash
sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/g' /etc/ssh/sshd_config
```
By default, Amazon Linux **disables** password-based SSH login (only key-pair login is allowed).  
These `sed` commands edit `/etc/ssh/sshd_config` to **enable password authentication**.

---

### Step 4: (Optional) Allow Root Login
```bash
sed -i 's/PermitRootLogin no/PermitRootLogin yes/g' /etc/ssh/sshd_config
```
Enables the `root` user to log in via SSH. **Not recommended for production environments.**

---

### Step 5: Restart SSH Daemon
```bash
systemctl restart sshd
```
Applies the SSH config changes by restarting the service.

---

### Step 6: Install and Start Apache (`httpd`)
```bash
yum install -y httpd
systemctl start httpd
systemctl enable httpd
```
Installs the Apache web server, starts it immediately, and enables it to start automatically on reboot.

---

## 5. Outputs

After the stack is created, CloudFormation exposes these values:

| Output | Description |
|--------|-------------|
| `InstancePublicIP` | The public IP address of the EC2 instance |
| `VPCId` | The ID of the created VPC |
| `SubnetId` | The ID of the created public subnet |
| `SSHCommand` | Ready-to-use SSH command: `ssh ec2-user@<public-ip>` |

---

## 6. Screenshot Explanation

The terminal screenshot shows a **Windows command prompt** (`C:\Users\denit\Downloads>`) successfully connecting to the EC2 instance.

### What happened:
1. **First session:** `ssh ec2-user@44.195.1.90`
   - The host key is permanently added (ED25519 fingerprint)
   - Successfully logged in — Amazon Linux 2023 banner displayed
   - User exited with `exit` command → `Connection to 44.195.1.90 closed`

2. **`clear` command failed:** Windows CMD does not support `clear` — this is a Linux/bash command. On Windows, use `cls` instead.

3. **Second session:** `ssh ec2-user@44.195.1.90`
   - This time, a **password prompt** appeared: `ec2-user@44.195.1.90's password:`
   - Login was successful — the Amazon Linux 2023 banner is shown again
   - Last login timestamp: `Tue Mar 17 10:24:51 2026 from 202.38.182.38`
   - Active prompt: `[ec2-user@ip-10-0-1-22 ~]$` → inside the EC2 instance

> The private IP `ip-10-0-1-22` corresponds to `10.0.1.22`, which falls within the configured subnet `10.0.1.0/24`. ✅

---

## 7. Architecture Diagram (Summary)

```
Internet
    │
    ▼
Internet Gateway (IGW)
    │
    ▼
VPC: 10.0.0.0/16
    │
    └── Public Subnet: 10.0.1.0/24
            │
            ├── Route Table → 0.0.0.0/0 → IGW
            │
            └── EC2 Instance (t3.micro)
                    ├── Public IP: 44.195.1.90
                    ├── Private IP: 10.0.1.22
                    ├── SSH (port 22) — password-based ✅
                    └── HTTP (port 80) — Apache running ✅
```

---

## 8. Key Concepts Summary

| Concept | What It Does |
|---------|--------------|
| **VPC** | Isolated private network in AWS |
| **Internet Gateway** | Connects VPC to the public internet |
| **Subnet** | Sub-division of a VPC's IP space |
| **Route Table** | Defines where traffic goes (like a router) |
| **Security Group** | Firewall rules for EC2 (port-level control) |
| **UserData** | Bootstrap script that runs on first boot |
| **AMI** | Pre-built OS image (Amazon Linux 2023 here) |
| **CloudFormation** | Infrastructure-as-Code — deploy everything from this YAML file |

---

## 9. How to Deploy

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name my-vpc-ec2-stack \
  --parameter-overrides \
    KeyName=your-key-pair \
    AmiId=ami-02dfbd4ff395f2a1b \
    Password=YourSecurePassword123 \
  --capabilities CAPABILITY_IAM
```

Or use the **AWS Console → CloudFormation → Create Stack → Upload Template**.

---

> 📌 **Reminder:** After you're done testing, always **delete the CloudFormation stack** to avoid ongoing charges. Terminating the EC2 alone won't remove the VPC, subnets, or other resources.

<img src="Screenshot 2026-03-17 155832.png">