AWSTemplateFormatVersion: '2010-09-09'
Description: |
  Data Stack - Creates an S3 bucket configured for static website hosting with a public read bucket policy.
Parameters:
  EnvironmentName:
    Type: String
    Default: dev
    Description: Environment name used as prefix for bucket naming
  IndexDocument:
    Type: String
    Default: index.html
    Description: Default page served when visiting the website root
  ErrorDocument:
    Type: String
    Default: error.html
    Description: Page served when a 404 or error occurs
Resources:
  StaticWebsiteBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    Properties:
      BucketName: !Sub ${EnvironmentName}-static-site-${AWS::AccountId}-${AWS::Region}
      WebsiteConfiguration:
        IndexDocument: !Ref IndexDocument
        ErrorDocument: !Ref ErrorDocument
      PublicAccessBlockConfiguration:
        BlockPublicAcls: false
        BlockPublicPolicy: false
        IgnorePublicAcls: false
        RestrictPublicBuckets: false
      VersioningConfiguration:
        Status: Enabled
      LifecycleConfiguration:
        Rules:
          - Id: DeleteOldVersions
            Status: Enabled
            NoncurrentVersionExpiration:
              NoncurrentDays: 30
      CorsConfiguration:
        CorsRules:
          - AllowedHeaders:
              - '*'
            AllowedMethods:
              - GET
              - HEAD
            AllowedOrigins:
              - '*'
            MaxAge: 3600
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-static-site
        - Key: Environment
          Value: !Ref EnvironmentName
        - Key: Purpose
          Value: StaticWebsite
  StaticWebsiteBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref StaticWebsiteBucket
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Sid: PublicReadGetObject
            Effect: Allow
            Principal: '*'
            Action:
              - s3:GetObject
              
            Resource: !Sub arn:aws:s3:::${StaticWebsiteBucket}/*
Outputs:
  BucketName:
    Description: Name of the S3 static website bucket
    Value: !Ref StaticWebsiteBucket
  BucketArn:
    Description: ARN of the S3 bucket
    Value: !GetAtt StaticWebsiteBucket.Arn
  WebsiteUrl:
    Description: S3 static website URL (HTTP endpoint)
    Value: !GetAtt StaticWebsiteBucket.WebsiteURL
  BucketDomainName:
    Description: S3 bucket domain name (for CloudFront origin if needed)
    Value: !GetAtt StaticWebsiteBucket.RegionalDomainName

  <img src="C:\Users\denit\Desktop\AWS-Task-Spektra\Day-34\screenshot-nested\Screenshot 2026-03-19 192233.png">

  <img src="C:\Users\denit\Desktop\AWS-Task-Spektra\Day-34\screenshot-nested\Screenshot 2026-03-19 192410.png">