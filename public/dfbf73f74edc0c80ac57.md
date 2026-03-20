---
title: StepFunctionsを利用して別アカウントへオブジェクトをコピーする構築ハンズオン
tags:
  - AWS
  - S3
  - lambda
  - stepfunctions
  - EventBridge
private: false
updated_at: '2022-12-10T16:28:35+09:00'
id: dfbf73f74edc0c80ac57
organization_url_name: null
slide: false
ignorePublish: false
---
## はじめに
要件として、承認プロセスが必要な異なるアカウント間のオブジェウトの移動という要件が与えられました。
承認プロセスだとCodePipelineが最初に思い浮かびましたが、今回のケースだと複数のユーザがS3へオブジェクトを保存するため、同時並行で承認を実現するためにStepFunctionsを選定しました。
その前段として、まずは承認フェイズなしでオブジェクトの移動を実現させる検証のために、こちらの構築を検証しました。

## 今回の構築概要図
1.アカウントAのS3へ、新規オブジェクトを保存する
2.S3にオブジェクトが生成されたことをトリガーにして、EventBridgeが後続のStepFunctionsを起動させる
3.StepFunctionsのワークフローにあるLambdaで、生成されたアカウントA:オブジェクトをアカウントB:S3バケットへコピーする
4.アカウントBのS3に、アカウントAで保存した新規ファイルが移動される
![](https://cloud5.jp/wp-content/uploads/2022/11/スクリーンショット-2022-11-06-9.59.19-640x227.png)
※上記赤枠部分をCFnを利用して構築していきます

## ハンズオン
前提条件：既にアカウント：A 及び　アカウント：B　の　S3バケットは構築されていること

### 1.アカウントA部分の構築
```json
AWSTemplateFormatVersion: 2010-09-09
Description: AWS Step Functions Architecture

Parameters:
    SendBucketName:
      Type: String
      Description: Must be a Send S3 BucketName.
      Default: XXXXXXXXXX

    ReceiveBucketName:
      Type: String
      Description: Must be a Receive S3 BucketName.
      Default: YYYYYYYYYY

##Lambda
Resources:
  Lambda:
    Type: 'AWS::Lambda::Function'
    Properties:
      Description: Lambda function
      FunctionName: object-transfer-lmd
      Handler: index.lambda_handler
      MemorySize: 128
      Timeout: 10
      Role: !GetAtt LambdaRole.Arn
      Runtime: python3.9
      Environment:
        Variables:
          receive: !Ref SendBucketName
          target: !Ref ReceiveBucketName
      Code:
        ZipFile: !Sub |
          import boto3
          import urllib.parse
          import os
          s3 = boto3.client('s3')

          def lambda_handler(event,context):
              from_bucket = os.environ['receive']
              to_bucket = os.environ['target']
              object_key = urllib.parse.unquote_plus(event['detail']['object']['key'], encoding='utf-8')

              s3.copy_object(Bucket=to_bucket, Key=object_key, CopySource={'Bucket': from_bucket, 'Key': object_key}
              )

              return 0

##Lambdaロール
  LambdaRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: object-transfer-lmd-role
      Path: "/"     
      AssumeRolePolicyDocument: 
        Version: "2012-10-17"
        Statement:
        - Effect: "Allow"
          Action: "sts:AssumeRole"
          Principal: 
            Service:
              - "lambda.amazonaws.com"
      ManagedPolicyArns:
      - !Ref LambdaManagedPolicy

  LambdaManagedPolicy:
    Type: AWS::IAM::ManagedPolicy
    Properties:
      ManagedPolicyName: object-transfer-lmd-policy
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Action:
              - "s3:ListBucket"
              - "s3:GetObject"
            Resource: 
              - "arn:aws:s3:::XXXXXXXXXX"
              - "arn:aws:s3:::XXXXXXXXXX/*"
          - Effect: Allow
            Action:
              - "s3:ListBucket"
              - "s3:PutObject"
              - "s3:PutObjectAcl"
            Resource: 
              - "arn:aws:s3:::YYYYYYYYYY"
              - "arn:aws:s3:::YYYYYYYYYY/*"
          - Effect: Allow
            Action:
              - "logs:*"
            Resource: !Sub "arn:${AWS::Partition}:logs:*:*:*"

##StepFunctions
  StepFunctions:
    Type: AWS::StepFunctions::StateMachine
    Properties:
      RoleArn: !GetAtt StepFunctionsRole.Arn
      StateMachineName: object-transfer-stf 
      DefinitionString:
        Fn::Sub: |
          {
              "StartAt": "object-transfer",
              "TimeoutSeconds": 3600,
              "States": {
                    "object-transfer": {
                      "Type": "Task",
                      "Resource": "${Lambda.Arn}",
                      "End": true
                    }
              }
          }

##StepFunctionsロール
  StepFunctionsRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: object-transfer-stf-role
      Path: "/"     
      AssumeRolePolicyDocument: 
        Version: "2012-10-17"
        Statement:
        - Effect: "Allow"
          Action: "sts:AssumeRole"
          Principal: 
            Service:
              - "states.amazonaws.com"
      ManagedPolicyArns:
      - !Ref StepFunctionsManagedPolicy

  StepFunctionsManagedPolicy:
    Type: AWS::IAM::ManagedPolicy
    Properties:
      ManagedPolicyName: object-transfer-stf-policy
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Action:
              - "lambda:InvokeFunction"
            Resource: 
              - !Sub "${Lambda.Arn}:*"

          - Effect: Allow
            Action:
              - "lambda:InvokeFunction"
            Resource: 
              - !Sub "${Lambda.Arn}"

##EventBridge
  EventBridge:
    Type: AWS::Events::Rule
    Properties:
      Name: object-transfer-evb
      State: ENABLED
      EventBusName: default
      EventPattern: 
        source:
          - aws.s3
        detail-type:
          - "Object Created"
        detail:
          bucket:
            name:
              - !Sub "${SendBucketName}"
      Targets:
        - Arn: !GetAtt StepFunctions.Arn
          Id: StepFunctions
          RoleArn: !GetAtt EventBridgeRole.Arn 

  ##EventBridgeロール
  EventBridgeRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: object-transfer-evb-role
      Path: "/"     
      AssumeRolePolicyDocument: 
        Version: "2012-10-17"
        Statement:
        - Effect: "Allow"
          Action: "sts:AssumeRole"
          Principal: 
            Service:
              - "events.amazonaws.com"
      ManagedPolicyArns:
      - !Ref EventBridgePolicy

  EventBridgePolicy:
    Type: AWS::IAM::ManagedPolicy
    Properties:
      ManagedPolicyName: object-transfer-evb-policy
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Action:
              - "states:StartExecution"
            Resource: 
              - !Sub "arn:${AWS::Partition}:states:${AWS::Region}:${AWS::AccountId}:*"
```

### 1.1.CloudFormation抜粋 パラメータ部分
パラメータに`SendBucketName`として、送信元のアカウントAにあたるバケット名を入力します。同様に`ReceiveBucketName`に、送信先のアカウントBにあたるバケット名を入力します。

```json
Parameters:
    SendBucketName:
      Type: String
      Description: Must be a Send S3 BucketName.
      Default: XXXXXXXXXX

    ReceiveBucketName:
      Type: String
      Description: Must be a Receive S3 BucketName.
      Default: YYYYYYYYYY
```

### 1.2. CloudFormation抜粋 Lambda部分
EventBridgeから呼び出されたStepFunctionsの、ワークフローの一部にあたるLambda
送信元はEventBridgeからトリガーの情報からでも値を受け取れますが、送信先のバケット名も必要だったので、どちらも環境変数に入れ込むように設定する。

```json
  Lambda:
    Type: 'AWS::Lambda::Function'
    Properties:
      Description: Lambda function
      FunctionName: object-transfer-lmd
      Handler: index.lambda_handler
      MemorySize: 128
      Timeout: 10
      Role: !GetAtt LambdaRole.Arn
      Runtime: python3.9
      Environment:
        Variables:
          receive: !Ref SendBucketName
          target: !Ref ReceiveBucketName
      Code:
        ZipFile: !Sub |
          import boto3
          import urllib.parse
          import os
          s3 = boto3.client('s3')

          def lambda_handler(event,context):
              from_bucket = os.environ['receive']
              to_bucket = os.environ['target']
              object_key = urllib.parse.unquote_plus(event['detail']['object']['key'], encoding='utf-8')

              s3.copy_object(Bucket=to_bucket, Key=object_key, CopySource={'Bucket': from_bucket, 'Key': object_key}
              )

              return 0
```
### 1.3. CloudFormation抜粋 StepFunctions部分
今回の構築では承認プロセスは無いので、簡素なStepFunctions

【構築後のStepFunctionsの画面】
左側ピンク色の実線部分を構築している、赤点線部分は1.2.で構築したLambdaの部分
![](https://cloud5.jp/wp-content/uploads/2022/11/スクリーンショット-2022-11-06-16.25.41-640x274.png)

```json
  StepFunctions:
    Type: AWS::StepFunctions::StateMachine
    Properties:
      RoleArn: !GetAtt StepFunctionsRole.Arn
      StateMachineName: object-transfer-stf 
      DefinitionString:
        Fn::Sub: |
          {
              "StartAt": "object-transfer",
              "States": {
                    "object-transfer": {
                      "Type": "Task",
                      "Resource": "${Lambda.Arn}",
                      "End": true
                    }
              }
          }
```

### 1.4. CloudFormation抜粋 EventBridge部分
S3にオブジェクトが生成されるとトリガーするEventBridge

```json
  EventBridge:
    Type: AWS::Events::Rule
    Properties:
      Name: object-transfer-evb
      State: ENABLED
      EventBusName: default
      EventPattern: 
        source:
          - aws.s3
        detail-type:
          - "Object Created"
        detail:
          bucket:
            name:
              - !Sub "${SendBucketName}"
      Targets:
        - Arn: !GetAtt StepFunctions.Arn
          Id: StepFunctions
          RoleArn: !GetAtt EventBridgeRole.Arn 
```
### 2.アカウントB バケットポリシーの設定内容
アカウントBのS3 > アクセス許可　>　バケットポリシー部分修正
Z部分にアカウントAの`アカウントID`を入力する
Y部分はアカウントBの`バケット名`を入力する
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::ZZZZZZZZZZZZ:role/object-transfer-lmd-role"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::YYYYYYYYYY/*"
        }
    ]
}
```
### 3.検証テスト
#### 1.アカウントAのS3へ、新規オブジェクトを保存する
![](https://cloud5.jp/wp-content/uploads/2022/11/スクリーンショット-2022-11-06-16.48.37-640x351.png)

#### 2.S3にオブジェクトが生成されたことをトリガーにして、EventBridgeが後続のStepFunctionsを起動させる
#### 3.StepFunctionsのワークフローにあるLambdaで、生成されたアカウントA:オブジェクトをアカウントB:S3バケットへコピーする
![](https://cloud5.jp/wp-content/uploads/2022/11/スクリーンショット-2022-11-06-16.56.53-640x436.png)

#### 4.アカウントBのS3に、アカウントAで保存した新規ファイルが移動される
![](https://cloud5.jp/wp-content/uploads/2022/11/スクリーンショット-2022-11-06-16.57.33-640x142.png)

## おわりに
アカウントまたぎだけなら、もっと小さく構築は出来たと思うのですが、要件や今後の拡張性を調査する環境が欲しかったので、StepFunctionsなどを導入してしまいましたが、それもまた良い学習機会となりました。
最終的にはCFnを利用してサクッと検証環境まで構築出来るようになったので、今後の承認フェイズが発生するパターンにおいても、かなり（自分の）役に立つものを作れたのでは無いかと思ってます。

## 参考URL
参考　AWSデベロッパーガイド:[人間による承諾プロジェクト例をデプロイする](https://docs.aws.amazon.com/ja_jp/step-functions/latest/dg/tutorial-human-approval.html)
参考　AWS公式ドキュメント：[別の AWS アカウントから S3 オブジェクトをコピーするにはどうすればよいですか。](https://aws.amazon.com/jp/premiumsupport/knowledge-center/copy-s3-objects-account/)
