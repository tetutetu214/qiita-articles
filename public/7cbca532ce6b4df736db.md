---
title: SessionManagerで利用するエンドポイントのCFn構築
tags:
  - AWS
  - CloudFormation
  - SessionManager
  - エンドポイント
private: false
updated_at: '2022-12-10T15:44:52+09:00'
id: 7cbca532ce6b4df736db
organization_url_name: null
slide: false
ignorePublish: false
---
## はじめに
最近リソースの消し忘れが多いてつてつです。
消すのは一瞬ですが、再度スクラッチで作ることを考えると手が止まってしまうんですよね。
そんな月曜日の自分に向けてCFnの構築メモを作成していきます。

## サービス概要図(一部抜粋)
今回自分のケースでは、[Session Manager のセットアップ](https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/session-manager-getting-started.html)を利用して、EC2へアクセスを行なっています。
その際にエンドポイントを3つ構築する必要があり、今回はこの3つのエンドポイントをCFnで構築するものです。

![](https://cloud5.jp/wp-content/uploads/2022/08/スクリーンショット-2022-08-20-17.25.17-565x640.png)

## AWS CloudFormationテンプレート
```
AWSTemplateFormatVersion: '2010-09-09'
# ------------------------------------------------------------#
# Input Parameters
# ------------------------------------------------------------#
Parameters:
#VPCID
  VpcId:
    Description : "VPC ID"
    Type: AWS::EC2::VPC::Id

#InterfaceSubnet
  InterfaceSubnetId:
    Description : "Interface Subnet"
    Type : AWS::EC2::Subnet::Id

#Interface Security Group
  InterfaceSecurityGroupId:
    Description : "SecurityGroup"
    Type: AWS::EC2::SecurityGroup::Id

Resources:
# ------------------------------------------------------------#
# Create ssm End Point
# ------------------------------------------------------------#
  ssmEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      ServiceName: !Sub "com.amazonaws.${AWS::Region}.ssm"
      SubnetIds:
        - !Ref InterfaceSubnetId
      VpcId: !Ref 'VpcId'
      VpcEndpointType: Interface
      SecurityGroupIds:
        - !Ref InterfaceSecurityGroupId
      PrivateDnsEnabled: true

  # ------------------------------------------------------------#
  # Create EC2Message End Point
  # ------------------------------------------------------------#
  EC2MessageEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      ServiceName: !Sub "com.amazonaws.${AWS::Region}.ec2messages"
      SubnetIds:
        - !Ref InterfaceSubnetId
      VpcId: !Ref 'VpcId'
      VpcEndpointType: Interface
      SecurityGroupIds:
        - !Ref InterfaceSecurityGroupId
      PrivateDnsEnabled: true

  # ------------------------------------------------------------#
  # Create ssmmessages End Point
  # ------------------------------------------------------------#
  ssmmessagesEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      ServiceName: !Sub "com.amazonaws.${AWS::Region}.ssmmessages"
      SubnetIds:
        - !Ref InterfaceSubnetId
      VpcId: !Ref 'VpcId'
      VpcEndpointType: Interface
      SecurityGroupIds:
        - !Ref InterfaceSecurityGroupId
      PrivateDnsEnabled: true
```

## さいごに
これで少しはスクラッチの憂鬱から逃れられそうです。
サブネットを選択ミスして待てど暮らせどEC2へ繋がらないなど、今週は合計して1時間くらいかかっているかと思ったので、塵つものようなことでも自動化していくことが必要かなと。
もっと身の回りのことを自動化していこうと思います。
