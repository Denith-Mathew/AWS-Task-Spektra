AWSTemplateFormatVersion: '2010-09-09'
Description: |
  Compute Stack - Creates IAM Role (with SSM), Launch Template for NGINX, and Auto Scaling Group across 2 private subnets.
Parameters:
  EnvironmentName:
    Type: String
    Default: dev
  VpcId:
    Type: AWS::EC2::VPC::Id
    Description: VPC ID from Network stack
  PrivateSubnet1Id:
    Type: AWS::EC2::Subnet::Id
    Description: Private Subnet 1 from Network stack
  PrivateSubnet2Id:
    Type: AWS::EC2::Subnet::Id
    Description: Private Subnet 2 from Network stack
  Ec2SecurityGroupId:
    Type: AWS::EC2::SecurityGroup::Id
    Description: EC2 Security Group ID from Network stack
  TargetGroupArn:
    Type: String
    Description: Target Group ARN from ALB stack
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues:
      - t3.micro
      - t3.small
      - t3.medium
    Description: EC2 instance type
  MinSize:
    Type: Number
    Default: 1
    Description: Minimum ASG instance count
  MaxSize:
    Type: Number
    Default: 4
    Description: Maximum ASG instance count
  DesiredCapacity:
    Type: Number
    Default: 2
    Description: Desired ASG instance count
Resources:
  Ec2IamRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub ${EnvironmentName}-ec2-ssm-role
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
        - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-ec2-role
  Ec2InstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      InstanceProfileName: !Sub ${EnvironmentName}-ec2-instance-profile
      Roles:
        - !Ref Ec2IamRole
  NginxLaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateName: !Sub ${EnvironmentName}-nginx-lt
      LaunchTemplateData:
        ImageId: !Sub '{{resolve:ssm:/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2}}'
        InstanceType: !Ref InstanceType
        IamInstanceProfile:
          Arn: !GetAtt Ec2InstanceProfile.Arn
        SecurityGroupIds:
          - !Ref Ec2SecurityGroupId
        Monitoring:
          Enabled: true
        UserData: !Base64
          Fn::Sub: |
            #!/bin/bash
            set -e

            yum update -y

            # Install NGINX
            amazon-linux-extras install nginx1 -y

            # Create a simple custom index page
            cat > /usr/share/nginx/html/index.html <<EOF
            <!DOCTYPE html>
            <html>
            <head><title>NGINX - ${EnvironmentName}</title></head>
            <body>
              <h1>Hello from NGINX!</h1>
              <p>Environment: ${EnvironmentName}</p>
              <p>Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>
              <p>AZ: $(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)</p>
            </body>
            </html>
            EOF

            # Enable and start NGINX
            systemctl enable nginx
            systemctl start nginx

            # Signal CloudFormation that setup is complete
            /opt/aws/bin/cfn-signal -e $? \
              --stack ${AWS::StackName} \
              --resource NginxAutoScalingGroup \
              --region ${AWS::Region}
        TagSpecifications:
          - ResourceType: instance
            Tags:
              - Key: Name
                Value: !Sub ${EnvironmentName}-nginx-instance
              - Key: Environment
                Value: !Ref EnvironmentName
          - ResourceType: volume
            Tags:
              - Key: Name
                Value: !Sub ${EnvironmentName}-nginx-volume
  NginxAutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    CreationPolicy:
      ResourceSignal:
        Count: !Ref DesiredCapacity
        Timeout: PT10M
    UpdatePolicy:
      AutoScalingRollingUpdate:
        MinInstancesInService: 1
        MaxBatchSize: 1
        PauseTime: PT5M
        WaitOnResourceSignals: true
    Properties:
      AutoScalingGroupName: !Sub ${EnvironmentName}-nginx-asg
      LaunchTemplate:
        LaunchTemplateId: !Ref NginxLaunchTemplate
        Version: !GetAtt NginxLaunchTemplate.LatestVersionNumber
      MinSize: !Ref MinSize
      MaxSize: !Ref MaxSize
      DesiredCapacity: !Ref DesiredCapacity
      VPCZoneIdentifier:
        - !Ref PrivateSubnet1Id
        - !Ref PrivateSubnet2Id
      TargetGroupARNs:
        - !Ref TargetGroupArn
      HealthCheckType: ELB
      HealthCheckGracePeriod: 120
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-nginx-asg
          PropagateAtLaunch: true
        - Key: Environment
          Value: !Ref EnvironmentName
          PropagateAtLaunch: true
  ScaleOutPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AutoScalingGroupName: !Ref NginxAutoScalingGroup
      PolicyType: TargetTrackingScaling
      TargetTrackingConfiguration:
        PredefinedMetricSpecification:
          PredefinedMetricType: ASGAverageCPUUtilization
        TargetValue: 70
Outputs:
  AutoScalingGroupName:
    Description: Name of the Auto Scaling Group
    Value: !Ref NginxAutoScalingGroup
  LaunchTemplateId:
    Description: Launch Template ID
    Value: !Ref NginxLaunchTemplate
  Ec2IamRoleArn:
    Description: IAM Role ARN for EC2 instances
    Value: !GetAtt Ec2IamRole.Arn 

<img src="C:\Users\denit\Desktop\AWS-Task-Spektra\Day-34\screenshot-nested\Screenshot 2026-03-19 181057.png">

<img src="C:\Users\denit\Desktop\AWS-Task-Spektra\Day-34\screenshot-nested\Screenshot 2026-03-19 181431.png">

