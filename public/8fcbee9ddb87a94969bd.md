---
title: Amazon FSx for Windows File Server構築ハンズオン
tags:
  - AWS
  - FSxForWindows
private: false
updated_at: '2022-12-10T15:41:30+09:00'
id: 8fcbee9ddb87a94969bd
organization_url_name: null
slide: false
ignorePublish: false
---
## はじめに
Amazon FSxの構築ハンズオンをしていきます。

※構築にあたって料金が発生しますので、検証の場合は構築後速やかに削除ください。

## 用語理解
#### Amazon FSx
機能が豊富で高性能なファイルシステムを、クラウド上で起動、実行およびスケーリングすることが可能です。
信頼性、セキュリティ、スケーラビリティ、幅広い機能を備え、さまざまなワークロードをサポートしながらも、コスト効率が高いです。
フルマネージドサービスとして、ハードウェアのプロビジョニング、パッチ適用、バックアップを行うため、お客様はビジネスに専念することができます。

## ハンズオン
###  構築図
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-15-7.57.44-640x300.png)
### 前提条件
・検証のため2つの異なるCIDRをもったVPCを用意します。
※構築が面倒な場合は、[CloudFormationを使ってVPC構築](https://cloud5.jp/cfn_vpc/)
などの記事で構築ください（適宜CIDRなどを変更してご利用下さい）。

### 1.VPC同士のネットワークを構築する
#### 1-1.『VPC』の画面より『ピアリング接続』を選択
画面右上にある『ピアリング接続を作成』を押下する
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-14-19.47.42-640x187.png)
#### 1-2.ピアリング接続の設定
異なるCIDRをもったVPC同士をピアリングする
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-14-20.02.06-640x637.png)
#### 1-3.『アクション』から『リクエストを承諾』を押下する
ピアリング後作成された値をコピーする（``pcx-XXXXXXXXXX``の部分）
※『リクエストを承諾』しないとピアリングの設定が終わらないので注意ください
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-14-20.05.30-640x154.png)
#### 1-4.『ルートテーブル』でのルートを編集
赤枠部分、左ペインの『ルートテーブル』から『ルート』を選択、『ルートを編集』を押下する
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-14-20.23.34-640x236.png)
#### 1-5.ルートを編集
赤枠部分にピアリングしているVPCのCIDRと、1-3.で作成したピアリングの値(``pcx-XXXXXXXXXX``)を入力する
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-14-20.27.54-640x156.png)
もう片方のルートテーブルにも、上記と同じようにピアリングの値を入力する。
自分の環境では、1つのVPCにつきパブリック・プライベート1つずつ、合計2つのルートテーブルがあり、2つのVPCそれぞれに設定が必要なので、合計で4回作業が生じました。

※最初『VPCピアリングだけすればOKだ』と理解していたので、こちらの構築をすっぽかしており30分くらいググりました。
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-14-20.31.44-640x153.png)

### 2.Directory Service(AWS Managed Microsoft AD)を構築する
#### 2-1.『DirectoryService』の画面より『ディレクトリのセットアップ』を押下する
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-14-20.38.04-640x113.png)
#### 2-2.『AWS Managed Microsoft AD』を選択して『次へ』を押下する
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-14-20.38.33-640x333.png)
#### 2-3.ディレクトリ情報を入力する
``ディレクトリのDNS名``と``パスワード``を入力する
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-14-20.39.35-600x640.png)
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-14-20.39.47-640x348.png)
#### 2-4.VPCとサブネットを選択
DirectoryServiceのリソースを配置するVPC、サブネットを選択していきます。
今回はA-VPCのパブリックサブネットに配置します。
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-14-20.41.38-640x398.png)
『次へ』を押下後、20〜45分でリソースが作成されるので気長に待ちます。

### 3.FSxを構築する
#### 3-1.ファイルシステムのタイプを選択する
今回はAmazon FSx for Windowsファイルサーバーを選択します
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-14-20.57.05-640x346.png)

#### 3-2.ファイルシステムを作成

![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-14-20.59.35-640x420.png)
今回BのVPCにFSxを構築していきます（赤枠のSG作成については、下にCFn掲載します）
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-14-21.23.11-1-640x328.png)

##### 3-2-1.SG作成にあたって
・FSxを配置するVPCには次の図にあるような、通信に対してのセキュリティグループが必要です
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-14-21.11.48-640x439.png)
参考URL:[Amazon VPC を使用したファイルシステムアクセスコントロール](https://docs.aws.amazon.com/ja_jp/fsx/latest/WindowsGuide/limit-access-security-groups.html)
##### 3-2-2.SG構築用のCFn
VPC作成後に作成されるVPCIDを各自で入力して、セキュリティグループを作成ください。
```
AWSTemplateFormatVersion: 2010-09-09
Resources: 
  secGroupName:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: SG-fsx
      GroupDescription: SG-fsx
      VpcId: 【VPC作成後 VPC Bに該当する VPC ID を入力ください】
      SecurityGroupEgress:
        - IpProtocol: tcp
          FromPort: 53
          ToPort: 53
          CidrIp: 0.0.0.0/0
        - IpProtocol: udp
          FromPort: 53
          ToPort: 53
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 88
          ToPort: 88
          CidrIp: 0.0.0.0/0
        - IpProtocol: udp
          FromPort: 88
          ToPort: 88
          CidrIp: 0.0.0.0/0
        - IpProtocol: udp
          FromPort: 123
          ToPort: 123
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 135
          ToPort: 135
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 389
          ToPort: 389
          CidrIp: 0.0.0.0/0
        - IpProtocol: udp
          FromPort: 389
          ToPort: 389
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 445
          ToPort: 445
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 464
          ToPort: 464
          CidrIp: 0.0.0.0/0
        - IpProtocol: udp
          FromPort: 464
          ToPort: 464
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 636
          ToPort: 636
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 3268
          ToPort: 3268
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 3269
          ToPort: 3269
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 9389
          ToPort: 9389
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 49152
          ToPort: 65535
          CidrIp: 0.0.0.0/0
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 445
          ToPort: 445
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 5985
          ToPort: 5985
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: "Name"
          Value: "SG-fsx"
```
#### 3-2.ファイルシステムを作成
2.で構築した``AWS Managed Microsoft Active Directory``を選択する(完了していない場合は、選択することが出来ません。)
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-14-21.33.19-640x241.png)

#### 3-3.構築されるまで待機する
『次へ』を押下後、20〜45分でリソースが作成されるので気長に待ちます。
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-14-21.41.51-640x170.png)
リソースが利用可能になると、ステータス部分が『利用可能』と表示されます
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-15-7.59.06-1-640x137.png)

#### 3-4.『アタッチ』から接続方法を確認する
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-15-8.14.38-640x178.png)
※今回はWindowsを参考にしていきます
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-15-8.15.48.png)

### 4.EC2からFSxへの接続
#### 4-1.VPC-AのパブリックサブネットにEC2インスタンス（Win）を立ち上げてDNS設定
設定を押下する
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-15-8.25.22-640x635.png)
赤枠部分をクリックしていきDNS設定の画面を表示する
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-15-8.31.21-640x342.png)

#### 4-2.DNSの値を、2.で構築した『Directory Service』から確認する
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-15-8.34.33-640x393.png)

#### 4-3.確認した値をDNS設定に入力する
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-15-8.41.29.png)

#### 4-4.プロンプトを起動して、3-4の接続コマンドを入力する
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-15-9.00.53.png)

#### 4-5.ディレクトリからFSxがアタッチされていることを確認する
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-15-9.03.21-1-640x366.png)

## さいごに
次回は別のアカウントから紐づけるなどしてFSxの理解を深めていこうと思います。
