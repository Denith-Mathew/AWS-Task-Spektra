# 📦 EC2 to S3 Access using CloudFormation (Least Privilege)

## 🎯 Objective

Create:

* EC2 instance
* S3 bucket
* IAM Role with **least privilege**
* Allow EC2 to:

  * List objects in S3
  * Read (display) objects from S3

---

## 🏗️ CloudFormation Template (YAML)

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: EC2 instance with least privilege S3 access

Parameters:
  InstanceType:
    Type: String
    Default: t3.micro

  KeyName:
    Type: AWS::EC2::KeyPair::KeyName

Resources:

  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16

  InternetGateway:
    Type: AWS::EC2::InternetGateway

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref MyVPC
      InternetGatewayId: !Ref InternetGateway

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 10.0.1.0/24
      MapPublicIpOnLaunch: true

  RouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MyVPC

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

  InstanceSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow SSH
      VpcId: !Ref MyVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0

  MyS3Bucket:
    Type: AWS::S3::Bucket

  EC2Role:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: S3LeastPrivilege
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action:
                  - s3:ListBucket
                Resource: !GetAtt MyS3Bucket.Arn

              - Effect: Allow
                Action:
                  - s3:GetObject
                Resource: !Sub "${MyS3Bucket.Arn}/*"

  InstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref EC2Role

  EC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      KeyName: !Ref KeyName
      SubnetId: !Ref PublicSubnet
      SecurityGroupIds:
        - !Ref InstanceSecurityGroup
      IamInstanceProfile: !Ref InstanceProfile
      ImageId: ami-0c02fb55956c7d316  # Change based on region

Outputs:
  BucketName:
    Value: !Ref MyS3Bucket

  InstanceId:
    Value: !Ref EC2Instance
```

---

## 🚀 Steps to Deploy

1. Go to AWS Console → CloudFormation
2. Click **Create Stack**
3. Upload this YAML file
4. Provide:

   * KeyPair name
5. Create stack

---

## 🖥️ Connect to EC2

```bash
ssh -i your-key.pem ec2-user@
```

---

## 📂 List Buckets

```bash
aws s3 ls
```

---

## 📁 List Objects in Bucket

```bash
aws s3 ls s3://
```

---

## 📄 Display Object Content

### Method 1: Download + View

```bash
aws s3 cp s3:/// .
cat 
```

---

### Method 2: Direct Output

```bash
aws s3 cp s3:/// -
```

---

## ❌ Expected Restrictions (Security Check)

These commands should fail:

```bash
aws s3 rm s3://bucket/file.txt
aws s3 cp file.txt s3://bucket/
```

---

## 🧠 Summary

| Feature       | Status     |
| ------------- | ---------- |
| List bucket   | ✅ Allowed  |
| List objects  | ✅ Allowed  |
| Read object   | ✅ Allowed  |
| Upload/Delete | ❌ Denied   |

---

## 🔥 Result

You successfully implemented:

* EC2 + S3 using CloudFormation
* IAM Least Privilege
* Secure object access from EC2