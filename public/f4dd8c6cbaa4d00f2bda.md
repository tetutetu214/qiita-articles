---
title: LambdaでS3署名付きURLを発行してSNS送信する構築ハンズオン
tags:
  - S3
  - SNS
  - lambda
  - 署名付きURL
private: false
updated_at: '2023-05-07T11:30:47+09:00'
id: f4dd8c6cbaa4d00f2bda
organization_url_name: null
slide: false
ignorePublish: false
---
## はじめに
以前[CloudFrontで署名付きURLの設定ハンズオン](https://cloud5.jp/presigned_url/)でCLIを利用して署名付きURLを作成しました。
今回はS3にオブジェクトを保存すると、Lambdaが署名付きURLを発行する構築を行なっていきます。

------------

## 構成図

![](https://cloud5.jp/wp-content/uploads/2023/01/スクリーンショット-2023-01-03-9.00.18-640x213.png)

### 挙動について
#### 1.S3 に プレフィックスinfraのオブジェクトを保存する
#### 2.上記の条件を満たしている場合、Lambdaへイベント通知が行われる
#### 3.Lambdaがオブジェクトからから署名付きURLを作成して、SNSへメールを成形して送信する
#### 4.登録したメールアドレスに、Lambdaで成形された内容が通知される
#### 5.メールアドレスのS3署名付きURLをクリックすると、オブジェクトの内容を確認できる

------------

## ハンズオン
### 構築のながれ
#### 1.SNS作成：登録したメールアドレスに、S3に保存したオブジェクト名・バケット名・署名付きURLを送信する
#### 2.Lambda作成：S3署名付きURL等メール内容を生成して、メール送信をする
#### 3.S3作成：オブジェクトを保存する、特定の条件の場合にイベント通知をおこうなう

-----------

### 1.SNS作成：登録したメールアドレスに、S3に保存したオブジェクト名・バケット名・署名付きURLを送信する
##### 1.1 SNSを構築する
署名付きURLを通知する、SNSトピックを構築します。
`TopicPolicy`部分は、検証のため制限していません。

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: SNS Create

# ------------------------------------------------------------#
#  Metadata
# ------------------------------------------------------------#
Metadata:
  "AWS::CloudFormation::Interface":
    ParameterGroups:
      - Label:
          default: "SNS Configuration"
        Parameters:
        - TopicName
        - Endpoint
    
    ParameterLabels:
      TopicName:
        default: "TopicName"
      Endpoint:
        default: "MailAddress"

# ------------------------------------------------------------#
#  InputParameters
# ------------------------------------------------------------#
Parameters:
  TopicName:
    Type: String
    Default: "cfn-sns-topic-inamura"
  Endpoint:
    Type: String
    Default: "XXXXXXXXXX@gmail.com"
  TagsValueUserName:
    Type: String
    Default: "inamura"

# ------------------------------------------------------------#
#  Resources
# ------------------------------------------------------------#
Resources:
  SNSTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: !Ref TopicName
      Subscription:
        - Endpoint: !Ref Endpoint
          Protocol: email
      Tags:
        - Key: "User"
          Value: !Ref TagsValueUserName    

  TopicPolicy:
    Type: AWS::SNS::TopicPolicy
    Properties:
      Topics:
        - !Ref SNSTopic
      PolicyDocument:
        Id: !Ref SNSTopic
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              AWS: "*"
            Action: SNS:Publish
            Resource: !Ref SNSTopic

# ------------------------------------------------------------#
# Output Parameters
# ------------------------------------------------------------#                
Outputs:
  SNSArn:
    Value: !Ref SNSTopic
    Export:
      Name: !Sub "${TopicName}-arn"
  SNSTopicName:
    Value: !Ref TopicName
    Export:
      Name: !Ref TopicName
```

#### 1.2 上記設定後に、SNSから送られてきたメールのサブスクリプションを押下する
①送られてきたメールの赤枠部分をクリックする
![](https://cloud5.jp/wp-content/uploads/2022/11/スクリーンショット-2022-11-26-14.13.42-640x157.png)

②画面が遷移して下記画面が表示されると、サブスクリプションが開始される
![](https://cloud5.jp/wp-content/uploads/2022/11/スクリーンショット-2022-11-26-14.15.17.png)
※上記青枠部分をクリックすると、サブスクリプションが解除される

### 2.Lambda作成：S3署名付きURL等メール内容を生成して、メール送信をする
`Type: "AWS::Lambda::Permission"`部分に`S3`からの通知許可を記載します。
他にLambdaで、S3に配置されたオブジェクトに対して署名付きURLを発行するために、ロールで`S3`への権限をアタッチします（検証のため権限を狭めず `"s3:*"`でアタッチしています）
検証では署名付きURLが発行されてから、3600秒（60分）の間ダウンロード出来るようにしました。

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description:
  Lambda Create
# ------------------------------------------------------------#
#  Metadata
# ------------------------------------------------------------#
Metadata:
  "AWS::CloudFormation::Interface":
    ParameterGroups:
      - Label:
          default: "Lambda Configuration"
        Parameters:
        - FunctionName
        - Description
        - Handler
        - MemorySize
        - Runtime
        - Timeout
        - TagsName

# ------------------------------------------------------------#
#  InputParameters
# ------------------------------------------------------------#
Parameters:
  FunctionName:
    Type: String
    Default: "cfn-lmd-inamura"
  Description:
    Type: String
    Default: "cfn-lmd-inamura"
  Handler:
    Type: String
    Default: "index.lambda_handler"
  MemorySize:
    Type: String
    Default: "128"
  Runtime:
    Type: String
    Default: "python3.9"
  Timeout:
    Type: String
    Default: "10"
  TagsName:
    Type: String
    Description: UserName
    Default: "inamura"
# ------------------------------------------------------------#
#  Resources
# ------------------------------------------------------------#
Resources:
# ------------------------------------------------------------#
#  Lambda
# ------------------------------------------------------------#
  Lambda:
    Type: 'AWS::Lambda::Function'
    Properties:
      Code:
        ZipFile: |
          import boto3
          import os
          import datetime
          import urllib.parse
          import json

          print('Loading function')

          sns_client = boto3.client('sns')
          s3_client = boto3.client('s3')
          SNS = os.environ['SNS']
          Subject = "【件名】SNS通知 "
          Message = "S3にオブジェクトが作成されました"

          def lambda_handler(event, context):
              bucket = urllib.parse.unquote_plus(event['Records'][0]['s3']['bucket']['name'], encoding='utf-8')
              key = urllib.parse.unquote_plus(event['Records'][0]['s3']['object']['key'], encoding='utf-8')
              print("BUCKET NAME:" + bucket)
              print("OBJECT NAME:" + key)
                  

              presigned_url = s3_client.generate_presigned_url(
                ClientMethod = 'get_object',
                Params = {'Bucket' : bucket, 'Key' : key},
                ExpiresIn = 3600,
                HttpMethod = 'GET')
              print (presigned_url)

              date = datetime.datetime.now()
              d = date.strftime('%Y%m%d %H:%M:%S')

              params = {
              'TopicArn': SNS,
              'Subject': Subject + str(d),
              'Message': Message + "\n\n" + "S3バケット名   :" + str(bucket) + "\n" + "オブジェクト名:" + str(key) + "\n" + "署名付きURL :" + str(presigned_url)
              }
          
              sns_client.publish(**params)

      Description: !Ref Description
      FunctionName: !Ref FunctionName
      Handler: !Ref Handler 
      MemorySize: !Ref MemorySize
      Runtime: !Ref Runtime
      Timeout: !Ref Timeout
      Role: !GetAtt LambdaRole.Arn
      Environment:
        Variables:
          SNS: !ImportValue cfn-sns-topic-inamura-arn
          TZ: "Asia/Tokyo"
          
      Tags:
        - Key: "User"
          Value: !Ref TagsName

  LambdaRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub "${FunctionName}-role"
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: "Allow"
            Action: "sts:AssumeRole"
            Principal:
              Service: lambda.amazonaws.com
      Policies:
        - PolicyName: !Sub "${FunctionName}-policy"
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: "Allow"
                Action:
                  - "logs:CreateLogStream"
                  - "logs:PutLogEvents"
                  - "logs:CreateLogGroup"
                Resource: !Sub "arn:${AWS::Partition}:logs:*:*:*"

              - Effect: "Allow"
                Action:
                  - "sns:Publish" 
                Resource: "*"

              - Effect: "Allow"
                Action:
                  - "s3:*" 
                Resource: "*"

  TriggerLambdaPermission:
    Type: "AWS::Lambda::Permission"
    Properties:
      Action: "lambda:InvokeFunction"
      FunctionName: !GetAtt Lambda.Arn
      Principal: "s3.amazonaws.com"
      SourceArn: !Sub "arn:${AWS::Partition}:s3:::*"
# ------------------------------------------------------------#
# Output Parameters
#------------------------------------------------------------#          
Outputs:
  LambdaArn:
    Value: !GetAtt Lambda.Arn
    Export:
      Name: !Sub "${FunctionName}-arn"
  LambdaName:
    Value: !Ref FunctionName
    Export:
      Name: !Sub "${FunctionName}-name"
```

### 3.S3作成：オブジェクトを保存し、特定の条件の場合にイベント通知をおこうなう
`NotificationConfiguration`部分でLambdaに対して通知する条件を記載しています。
今回通知するための条件として`prefix`が`infra`でオブジェクトが配置された場合Lambdaを呼び出します。

S3のバケット名は全世界でユニークのため、`cfn-s3-20230102-inamura`部分は各自修正ください
※現在既にcfn-s3-20220102-inamuraは削除しています
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: CloudFormation to create S3 Bucket
# ------------------------------------------------------------#
#  Metadata
# ------------------------------------------------------------#
Metadata:
  "AWS::CloudFormation::Interface":
    ParameterGroups:
      - Label:
          default: "S3 Configuration"
        Parameters:
        - S3BucketName
        - AccessControl
        - BlockPublicAcls
        - BlockPublicPolicy
        - IgnorePublicAcls
        - RestrictPublicBuckets
        - ExpirationInDays
        - EventBridgeConfiguration
        - Prefix
        - TagsName

# ------------------------------------------------------------#
#  InputParameters
# ------------------------------------------------------------#
Parameters:
  S3BucketName:
    Type: String
    Default: "cfn-s3-20230102-inamura"
    Description: Type of this BacketName.
  VersioningConfiguration:
    Type: String
    Default: "Enabled"
    Description: VersioningConfiguration.
  AccessControl:
    Type: String
    Description: AccessControl.
    Default: "Private"
    AllowedValues: [ "Private", "PublicRead", "PublicReadWrite", "AuthenticatedRead", "LogDeliveryWrite", "BucketOwnerRead", "BucketOwnerFullControl", "AwsExecRead" ]
  BlockPublicAcls: 
    Type: String
    Description: BlockPublicAcls.
    Default: "True"
    AllowedValues: [ "True", "False" ]
  BlockPublicPolicy:
    Type: String
    Description: BlockPublicPolicy.
    Default: "True"
    AllowedValues: [ "True", "False" ]
  IgnorePublicAcls:
    Type: String
    Description: IgnorePublicAcls.
    Default: "True"
    AllowedValues: [ "True", "False" ]
  RestrictPublicBuckets:
    Type: String
    Description: RestrictPublicBuckets.
    Default: "True"
    AllowedValues: [ "True", "False" ]
  ExpirationInDays:
    Type: String
    Description: Lifecycle Days.
    Default: "7"
  Prefix:
    Type: String
    Description: Lambdafunction Trigger Prefix.
    Default: "infra"
  TagsName:
    Type: String
    Description: UserName
    Default: "inamura"
  
# ------------------------------------------------------------#
#  Resources
# ------------------------------------------------------------#
Resources:
# ------------------------------------------------------------#
#  S3
# ------------------------------------------------------------#
  S3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref S3BucketName      
      VersioningConfiguration:
        Status: !Ref VersioningConfiguration
      AccessControl: !Ref AccessControl
      PublicAccessBlockConfiguration:
        BlockPublicAcls: !Ref BlockPublicAcls
        BlockPublicPolicy: !Ref BlockPublicPolicy
        IgnorePublicAcls: !Ref IgnorePublicAcls
        RestrictPublicBuckets: !Ref RestrictPublicBuckets
      LifecycleConfiguration:
        Rules:
          - Id: LifeCycleRule
            Status: Enabled
            ExpirationInDays: !Ref ExpirationInDays
      NotificationConfiguration:
        LambdaConfigurations:
          - Event: "s3:ObjectCreated:*"
            Filter:
              S3Key:
                Rules:
                  - Name: prefix
                    Value: !Ref Prefix
            Function: !ImportValue cfn-lmd-inamura-arn
      Tags:
        - Key: "User"
          Value: !Ref TagsName
# ------------------------------------------------------------#
#  Outputs
# ------------------------------------------------------------#
Outputs:
  S3BucketName:
    Value: !Ref S3Bucket
    Export:
      Name: cfn-s3-BucketName
```

### 挙動の確認
#### ①S3に プレフィックス`infra`　を満たしているオブジェクトを保存する
![](https://cloud5.jp/wp-content/uploads/2023/01/スクリーンショット-2023-01-02-17.35.38-640x162.png)

#### ②`SNS`で設定したメールアドレスにメールが送信される
![](https://cloud5.jp/wp-content/uploads/2023/01/スクリーンショット-2023-01-02-17.36.08-640x94.png)

#### ③メールアドレスにある`署名付きURL`をクリックしてアクセスできる。
![](https://cloud5.jp/wp-content/uploads/2023/01/スクリーンショット-2023-01-02-17.38.11.png)

------------

## さいごに
トリガーとしてAPIGatewayのエンドポイントでLambdaを動かしたり、LambdaのURLを利用してS3署名付きURLを発行したりと、利用できる場面は多いかなと個人的に思いました。
今年も検証を踏まえながら、仕事に活かしていきたいものです。
