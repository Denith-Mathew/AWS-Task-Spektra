AWSTemplateFormatVersion: '2010-09-09'
Description: |
  ALB Stack - Creates an internet-facing Application Load Balancer, Target Group, HTTP Listener, and health checks.
Parameters:
  EnvironmentName:
    Type: String
    Default: dev
  VpcId:
    Type: AWS::EC2::VPC::Id
    Description: VPC ID from Network stack
  PublicSubnet1Id:
    Type: AWS::EC2::Subnet::Id
    Description: Public Subnet 1 ID from Network stack
  PublicSubnet2Id:
    Type: AWS::EC2::Subnet::Id
    Description: Public Subnet 2 ID from Network stack
  AlbSecurityGroupId:
    Type: AWS::EC2::SecurityGroup::Id
    Description: ALB Security Group ID from Network stack
  HealthCheckPath:
    Type: String
    Default: /
    Description: Path ALB uses to health check NGINX instances
  HealthCheckIntervalSeconds:
    Type: Number
    Default: 30
    Description: Seconds between ALB health checks
  HealthyThresholdCount:
    Type: Number
    Default: 2
    Description: Consecutive successful checks before marking instance healthy
  UnhealthyThresholdCount:
    Type: Number
    Default: 3
    Description: Consecutive failed checks before marking instance unhealthy
Resources:
  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Sub ${EnvironmentName}-alb
      Type: application
      Scheme: internet-facing
      Subnets:
        - !Ref PublicSubnet1Id
        - !Ref PublicSubnet2Id
      SecurityGroups:
        - !Ref AlbSecurityGroupId
      LoadBalancerAttributes:
        - Key: idle_timeout.timeout_seconds
          Value: '60'
        - Key: routing.http2.enabled
          Value: 'true'
        - Key: deletion_protection.enabled
          Value: 'false'
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-alb
        - Key: Environment
          Value: !Ref EnvironmentName
  NginxTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: !Sub ${EnvironmentName}-nginx-tg
      VpcId: !Ref VpcId
      Protocol: HTTP
      Port: 80
      TargetType: instance
      HealthCheckEnabled: true
      HealthCheckProtocol: HTTP
      HealthCheckPath: !Ref HealthCheckPath
      HealthCheckPort: traffic-port
      HealthCheckIntervalSeconds: !Ref HealthCheckIntervalSeconds
      HealthyThresholdCount: !Ref HealthyThresholdCount
      UnhealthyThresholdCount: !Ref UnhealthyThresholdCount
      HealthCheckTimeoutSeconds: 5
      Matcher:
        HttpCode: 200-299
      TargetGroupAttributes:
        - Key: deregistration_delay.timeout_seconds
          Value: '30'
        - Key: stickiness.enabled
          Value: 'false'
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-nginx-tg
        - Key: Environment
          Value: !Ref EnvironmentName
  HttpListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ApplicationLoadBalancer
      Protocol: HTTP
      Port: 80
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref NginxTargetGroup
  HttpListenerRule:
    Type: AWS::ElasticLoadBalancingV2::ListenerRule
    Properties:
      ListenerArn: !Ref HttpListener
      Priority: 1
      Conditions:
        - Field: path-pattern
          Values:
            - /*
      Actions:
        - Type: forward
          TargetGroupArn: !Ref NginxTargetGroup
Outputs:
  AlbArn:
    Description: ARN of the Application Load Balancer
    Value: !Ref ApplicationLoadBalancer
  AlbDnsName:
    Description: DNS name to access the NGINX application
    Value: !GetAtt ApplicationLoadBalancer.DNSName
  AlbHostedZoneId:
    Description: Hosted Zone ID of ALB (useful for Route53 alias records)
    Value: !GetAtt ApplicationLoadBalancer.CanonicalHostedZoneID
  TargetGroupArn:
    Description: ARN of the Target Group (passed to Compute stack)
    Value: !Ref NginxTargetGroup
  TargetGroupName:
    Description: Name of the Target Group
    Value: !GetAtt NginxTargetGroup.TargetGroupName
  HttpListenerArn:
    Description: ARN of the HTTP Listener
    Value: !Ref HttpListener

  <img src="C:\Users\denit\Desktop\AWS-Task-Spektra\Day-34\screenshot-nested\Screenshot 2026-03-19 181149.png">

  <img src="C:\Users\denit\Desktop\AWS-Task-Spektra\Day-34\screenshot-nested\Screenshot 2026-03-19 181520.png">

  <img src="C:\Users\denit\Desktop\AWS-Task-Spektra\Day-34\screenshot-nested\Screenshot 2026-03-19 181549.png">

  <img src="C:\Users\denit\Desktop\AWS-Task-Spektra\Day-34\screenshot-nested\Screenshot 2026-03-19 180502.png">