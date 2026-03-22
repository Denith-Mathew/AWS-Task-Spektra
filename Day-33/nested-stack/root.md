AWSTemplateFormatVersion: '2010-09-09'
Description: Parent Stack - Deploys Network, Compute, ALB, and Data stacks as nested stacks.
Parameters:
  TemplateBaseUrl:
    Type: String
    Description: S3 URL prefix where child templates are stored (no trailing slash)
  EnvironmentName:
    Type: String
    Default: dev
    Description: Environment name prefix for resource naming
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues:
      - t3.micro
      - t3.small
      - t3.medium
    Description: EC2 instance type for NGINX servers
  MinSize:
    Type: Number
    Default: 1
    Description: Minimum number of EC2 instances in ASG
  MaxSize:
    Type: Number
    Default: 4
    Description: Maximum number of EC2 instances in ASG
  DesiredCapacity:
    Type: Number
    Default: 3
    Description: Desired number of EC2 instances in ASG
    
Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Sub ${TemplateBaseUrl}/network.yaml
      Parameters:
        EnvironmentName: !Ref EnvironmentName
      Tags:
        - Key: Environment
          Value: !Ref EnvironmentName
        - Key: ManagedBy
          Value: CloudFormation

  AlbStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: NetworkStack
    Properties:
      TemplateURL: !Sub ${TemplateBaseUrl}/alb.yaml
      Parameters:
        EnvironmentName: !Ref EnvironmentName
        VpcId: !GetAtt NetworkStack.Outputs.VpcId
        PublicSubnet1Id: !GetAtt NetworkStack.Outputs.PublicSubnet1Id
        PublicSubnet2Id: !GetAtt NetworkStack.Outputs.PublicSubnet2Id
        AlbSecurityGroupId: !GetAtt NetworkStack.Outputs.AlbSecurityGroupId
      Tags:
        - Key: Environment
          Value: !Ref EnvironmentName
        - Key: ManagedBy
          Value: CloudFormation

  ComputeStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: AlbStack
    Properties:
      TemplateURL: !Sub ${TemplateBaseUrl}/compute.yaml
      Parameters:
        EnvironmentName: !Ref EnvironmentName
        VpcId: !GetAtt NetworkStack.Outputs.VpcId
        PrivateSubnet1Id: !GetAtt NetworkStack.Outputs.PrivateSubnet1Id
        PrivateSubnet2Id: !GetAtt NetworkStack.Outputs.PrivateSubnet2Id
        Ec2SecurityGroupId: !GetAtt NetworkStack.Outputs.Ec2SecurityGroupId
        TargetGroupArn: !GetAtt AlbStack.Outputs.TargetGroupArn
        InstanceType: !Ref InstanceType
        MinSize: !Ref MinSize
        MaxSize: !Ref MaxSize
        DesiredCapacity: !Ref DesiredCapacity
      Tags:
        - Key: Environment
          Value: !Ref EnvironmentName
        - Key: ManagedBy
          Value: CloudFormation

  DataStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Sub ${TemplateBaseUrl}/data.yaml
      Parameters:
        EnvironmentName: !Ref EnvironmentName
      Tags:
        - Key: Environment
          Value: !Ref EnvironmentName
        - Key: ManagedBy
          Value: CloudFormation

Outputs:
  AlbDnsName:
    Description: DNS name of the Application Load Balancer (use this to access NGINX)
    Value: !GetAtt AlbStack.Outputs.AlbDnsName
    Export:
      Name: !Sub ${AWS::StackName}-AlbDnsName
  WebsiteUrl:
    Description: S3 static website URL
    Value: !GetAtt DataStack.Outputs.WebsiteUrl
    Export:
      Name: !Sub ${AWS::StackName}-WebsiteUrl
  BucketName:
    Description: Name of the S3 static site bucket
    Value: !GetAtt DataStack.Outputs.BucketName
    Export:
      Name: !Sub ${AWS::StackName}-BucketName
  VpcId:
    Description: VPC ID created by the Network stack
    Value: !GetAtt NetworkStack.Outputs.VpcId
    Export:
      Name: !Sub ${AWS::StackName}-VpcId
  AutoScalingGroupName:
    Description: Name of the Auto Scaling Group
    Value: !GetAtt ComputeStack.Outputs.AutoScalingGroupName
    Export:
      Name: !Sub ${AWS::StackName}-AsgName

<img src="Screenshot 2026-03-19 180443.png">

