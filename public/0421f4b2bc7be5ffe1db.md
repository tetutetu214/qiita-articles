---
title: ALBのリスナールールでメンテナンス画面を表示させる構築ハンズオン
tags:
  - AWS
  - CloudFormation
  - lambda
  - ALB
  - メンテナンス画面
private: false
updated_at: '2022-12-10T16:37:14+09:00'
id: 0421f4b2bc7be5ffe1db
organization_url_name: null
slide: false
ignorePublish: false
---
## はじめに
[前回のブログ『ALB をトリガーにして Lambda を実行する構築ハンズオン』](https://cloud5.jp/execute-lambda-by-alb/)同様にCloudFormationを利用して、ALBリスナールールを追加してメンテナンス画面を表示させていきます。

------------

## 構成図
![](https://cloud5.jp/wp-content/uploads/2022/11/スクリーンショット-2022-11-24-20.59.25-640x563.png)

------------

## ハンズオン
### 構築の流れ
#### 1.VPC作成
#### 2.Lambda作成
#### 3.ALB作成
上記の順番で構築を行なっていきます。
最終的に下記画面で優先度を上げることで、画面を切り替えることが出来ます。
①ALBのリスナールールの優先度を変える。
![](https://cloud5.jp/wp-content/uploads/2022/11/スクリーンショット-2022-11-23-23.50.51-640x276.png)
②固定画面が表示される。
![](https://cloud5.jp/wp-content/uploads/2022/11/スクリーンショット-2022-11-23-23.53.18-640x144.png)

### 1.VPC作成
以前記載したブログ[CloudFormationを使ってVPC構築](https://cloud5.jp/cfn_vpc/)に沿って、VPCを構築します。

### 2.Lambda作成
以前記載したブログ[ALB をトリガーにして Lambda を実行する構築ハンズオン](https://cloud5.jp/execute-lambda-by-alb/)
に沿って、Lambdaを構築します。

### 3.ALB作成
前回の構築([ALB をトリガーにして Lambda を実行する構築ハンズオン](https://cloud5.jp/execute-lambda-by-alb/))とは異なり、固定画面へのリスナールールを追加します。

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: 
  ALB Create
# ------------------------------------------------------------#
#  Metadata
# ------------------------------------------------------------#
Metadata:
  "AWS::CloudFormation::Interface":
    ParameterGroups:
      - Label:
          default: "ALB Configuration"
        Parameters:
        - ALBName
        - Type
        - Scheme
        - IpAddressType
      - label:
          default: "ALB TargetGroup"
        Parameters:
        - TGName1
        - TargetType
      - label:
          default: "ALB SecurityGroup"
        Parameters:
        - GroupName
# ------------------------------------------------------------#
#  InputParameters
# ------------------------------------------------------------#
Parameters:
  ALBName:
    Type: String
    Default: "cfn-alb-inamura"
  Type:
    Type: String
    Default: "application"
  Scheme:
    Type: String
    Default: "internet-facing"
  IpAddressType:
    Type: String
    Default: "ipv4"
  TGName1:
    Type: String
    Default: "cfn-tgg1-inamura"
  TargetType:
    Type: String
    Default: "lambda"
  GroupName:
    Type: String
    Default: "cfn-sg-alb-inamura"

# ------------------------------------------------------------#
#  Resources
# ------------------------------------------------------------#
Resources:
# ------------------------------------------------------------#
#  ALB
# ------------------------------------------------------------#
  ALB:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Ref ALBName
      Type: !Ref Type
      Scheme: !Ref Scheme
      IpAddressType: !Ref IpAddressType
      Subnets: 
        - !ImportValue cfn-inamura-public-subneta
        - !ImportValue cfn-inamura-public-subnetc
      SecurityGroups: 
        - !Ref ALBSecurityGroup

  ListenerHTTP:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref TargetGroup1
      LoadBalancerArn: !Ref ALB
      Port: 80
      Protocol: HTTP

  ListenerRule1:
    Type: AWS::ElasticLoadBalancingV2::ListenerRule
    Properties:
      Actions:
        - Type: forward
          TargetGroupArn: !Ref TargetGroup1
      Conditions:
        - Field: path-pattern
          Values: 
            - '*'
      ListenerArn: !Ref ListenerHTTP
      Priority: 1

  ListenerRule2:
    Type: AWS::ElasticLoadBalancingV2::ListenerRule
    Properties:
      Actions:
        - Type: "fixed-response"
          FixedResponseConfig:
            ContentType: 'text/html'
            MessageBody: |
              <!DOCTYPE html>
              <html lang="ja">
              <head>
              <meta charset="UTF-8">
              <title>メンテナンスのお知らせ</title>
              </head>
              <body>
                <h1>ただいまメンテナンス中です</h1>
              </body>
              </html>
            StatusCode: 503
      Conditions:
        - Field: path-pattern
          Values: 
            - '*'
      ListenerArn: !Ref ListenerHTTP
      Priority: 2

  TargetGroup1:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    DependsOn: LambdaInvokePermission
    Properties:
      Name: !Ref TGName1
      TargetType: !Ref TargetType
      Targets:
        - Id: !ImportValue cfn-lmd-inamura-arn

# ------------------------------------------------------------#
#  ALB SG
# ------------------------------------------------------------#
  ALBSecurityGroup:
    Type: "AWS::EC2::SecurityGroup"
    Properties:
        GroupDescription: "ALB SG"
        GroupName: !Ref GroupName
        VpcId: !ImportValue cfn-inamura-vpc
        SecurityGroupIngress:
          - IpProtocol: tcp
            FromPort: 80
            ToPort: 80
            CidrIp: 0.0.0.0/0

# ------------------------------------------------------------#
#  リソースベースポリシー
# ------------------------------------------------------------#
  LambdaInvokePermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !ImportValue cfn-lmd-inamura-arn
      Action: lambda:InvokeFunction
      Principal: elasticloadbalancing.amazonaws.com

# ------------------------------------------------------------#
# Output Parameters
# ------------------------------------------------------------#    
Outputs:
  ALBURL:
    Description: ALB endpoint URL
    Value: !Join
        - ""
        - - http://
          - !GetAtt ALB.DNSName
```
### 挙動の確認
①構築したロードバランサーの`DNS name`を確認する
![](https://cloud5.jp/wp-content/uploads/2022/11/スクリーンショット-2022-11-23-16.31.28-640x153.png)

②インターネットブラウザに`DNS name`を入力して、Lambdaで設定した画面が表示される。
※コールドスタートの場合、体感で1~2分程度表示に時間がかかりました。
![](https://cloud5.jp/wp-content/uploads/2022/11/スクリーンショット-2022-11-23-16.12.15-1-640x103.png)

③AWSマネジメントコンソールからALBのリスナールールの優先度を変える。
![](https://cloud5.jp/wp-content/uploads/2022/11/スクリーンショット-2022-11-23-23.50.51-640x276.png)

④最初に表示されていた②の画面から下記固定画面に表示変更される。
※体感で1分程度未満で切り替わりました。
![](https://cloud5.jp/wp-content/uploads/2022/11/スクリーンショット-2022-11-23-23.53.18-640x144.png)

------------

## さいごに
リスナー部分の文法ミスで何回も書き直しをしましたが、無事に自分の思うような構築が出来ました。
まだ手動での切り替えとなっているので、次は自動的に切り替わるように構築を考えていければと思います。まだまだ検証していくことは多めなので手を動かして知見を増やせればと思います。
