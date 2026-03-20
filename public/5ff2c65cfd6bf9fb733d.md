---
title: Squidインスタンス(EC2)でプロキシ構成のハンズオン
tags:
  - AWS
  - EC2
  - squid
private: false
updated_at: '2023-05-04T14:15:27+09:00'
id: 5ff2c65cfd6bf9fb733d
organization_url_name: null
slide: false
ignorePublish: false
---
## はじめに
検証環境のパブリックサブネットに`NATインスタンス`を構築したものの、プライベートサブネットに構築したインスタンス経由で、どんなインターネットにも接続できてしまうので、`Squidインスタンス`を用いて制限をかけようと思います。


【前回の構築ブログ】
・[NATインスタンス(EC2)をハンズオン](https://cloud5.jp/nat-instance/)

【参考】
・[squid　ドキュメント](http://www.squid-cache.org/)

<br>

 ### ざっくりSquid
Squidはオープンソースのプロキシサーバー及びキャッシュサーバー。
クライアントコンピュータがインターネット等のリクエストを送信する際に、中間でリクエストを処理しアクセス制御等の複数機能を提供する。

【参考】
・[IT用語辞典](https://e-words.jp/w/Squid.html)

------------

## 構成図
[前回構築している構成](https://cloud5.jp/nat-instance/)に、squidを設定したEC2を構築

![](https://cloud5.jp/wp-content/uploads/2023/05/スクリーンショット-2023-05-04-13.03.29-517x640.png)

------------

## ハンズオン
### 構築のながれ
#### 1.EC2(squid)インスタンス構築（マネジメントコンソール）
#### 2.EC2(squid)インスタンスの設定（SSH接続）
#### 3.EC2(クライアント)の設定（リモートデスクトップ）

-----------

### 1. EC2(squid)インスタンス構築（マネジメントコンソール）
#### 1.1.Squidインスタンスをセットアップ
Amazon Web Services (AWS) アカウントにログインし、下記設定値のEC2インスタンスを作成。

|項目|設定値|
|---|---|
|インスタンスタイプ|t2.micro|
|AMI ID|ami-0df2ca8a354185e1e(Amazon Linux)|
|サブネット|パブリックサブネット|
|パブリック IP の自動割り当て|◯|

<br>

#### 1.2. セキュリティグループの設定
インスタンス作成時にセキュリティグループを設定し、プライベートサブネットからのアクセスのみを開放。

【インバウンド】

|タイプ|プロトコル|ポート範囲|ソース|CIDR|
|---|---|---|---|---|
|すべてのトラフィック|すべて|すべて|カスタム|【プライベートサブネットCIDR】|


【アウトバウンド】

|タイプ|プロトコル|ポート範囲|ソース|CIDR|
|---|---|---|---|---|
|すべてのトラフィック|すべて|すべて|カスタム|0.0.0.0/0|


<br>

#### 1.3. IAMロールの設定
SSMよるフリートマネージャよりアクセスも出来るように作成

|許可ポリシー|タイプ|
|---|---|
|AmazonSSMManagedInstanceCore|AWS 管理|

<br>

#### 1.4. インスタンスを起動
上記設定後、『インスタンスを起動』を押下
<br>


#### 1.5.`NATインスタンス`のSG設定
`NATインスタンス`に`Squidインスタンス`からの`3128`のインバウンドを許可する
##### 1.5.1.NATインスタンスのSGへ移動
『対象 NATインスタンス』 →　タブ『セキュリティグループ』　→ 『セキュリティグループ』を押下
![](https://cloud5.jp/wp-content/uploads/2023/05/スクリーンショット-2023-05-04-10.22.46-640x254.png)
<br>

##### 1.5.2.SGの設定画面へ移動
『インバウンドルールの追加』を押下
![](https://cloud5.jp/wp-content/uploads/2023/05/スクリーンショット-2023-05-04-10.23.21-640x132.png)

<br>

##### 1.5.3.`Squidインスタンス`のプライベートIP追加
手順1.4で立ち上がった、`Squidインスタンス`のプライベートIPから`3128`の受け入れを設定
![](https://cloud5.jp/wp-content/uploads/2023/05/スクリーンショット-2023-05-04-10.37.02-640x62.png)
<br>

------------


### 2.EC2(squid)インスタンスの設定（SSH接続）
#### 2.1.squidをインストール
squidをインストールする
```shell
sudo yum install squid
```
<br>

#### 2.2.各サーバごとに設定ファイルを作成
`sudo vi /etc/squid/squid.conf`するのではなく、今後の拡張性より各サーバごとに設定ファイルを編集するため`conf.d`配下に設定ファイルを作成

```shell
sudo mkdir /etc/squid/conf.d
sudo vi /etc/squid/conf.d/client1_restrictions.conf
```
<br>

#### 2.3.設定ファイルを編集する
下記設定を設定ファイルに記載
・1行目：restricted_clientという名前のACLで、「172.16.10.119/32」のクライアントサーバを対象とする
・2行目：yahoo_domainsという名前のACLで、「.yahoo.com」「.yahoo.co.jp」ドメインを対象とする
・3行目：restricted_client は　yahoo_domains　にアクセスが出来ない

```shell
acl restricted_client src 172.16.10.119/32
acl yahoo_domains dstdomain .yahoo.com .yahoo.co.jp
http_access deny restricted_client yahoo_domains
```
<br>

#### 2.4.Squidのメイン設定ファイル（/etc/squid/squid.conf）を編集し、conf.d ディレクトリ内の設定ファイルをインクルードするよう設定
##### 2.4.1.Squidのメイン設定ファイルを編集
```shell
sudo vi /etc/squid/squid.conf
```
<br>
##### 2.4.2.Squidのメイン設定ファイルに、インクルード設定
設定ファイルの最初に以下の行を追加して、conf.d ディレクトリ内のすべての設定ファイルを読み込むように設定します。
```shell
include /etc/squid/conf.d/*.conf
```

<br>

#### 2.5.Squidの設定を反映し、Squidを再起動
```shell
sudo systemctl restart squid
```
<br>

------------

### 3.EC2(クライアント)の設定（リモートデスクトップ）
#### 3.1.フリートマネージャーからリモートデスクトップ(RDP)を選択
『AWS SystemsManager』　→　『フリートマネージャー』 →　『ノードアクション』　→ 『リモートデスクトップ（RDP）との接続』を押下
※個人の環境の影響になるかと思いますが、自分はフリートマネージャーの画面にEC2が反映されるまで10分程度かかりました。
![](https://cloud5.jp/wp-content/uploads/2023/05/スクリーンショット-2023-05-03-16.27.18-640x237.png)
<br>
#### 3.2.フリートマネージャーからリモートデスクトップ(RDP)でアクセス
![](https://cloud5.jp/wp-content/uploads/2023/05/スクリーンショット-2023-05-03-16.29.39-640x637.png)
<br>

#### 3.3.Windowsの[インターネットオプション] > [接続] > [LANの設定]で、プロキシサーバーのアドレスとポート指定
アドレスにはSquidプロキシサーバーのプライベートIPアドレスを、ポートには3128を入力

![](https://cloud5.jp/wp-content/uploads/2023/05/スクリーンショット-2023-05-04-13.16.14.png)

<br>

------------

### 挙動の確認
#### 1.1.リモートデスクトップ(RDP)先から、インターネットへアクセス
![](https://cloud5.jp/wp-content/uploads/2023/05/スクリーンショット-2023-05-03-16.34.05-640x381.png)
<br>

#### 1.2.Squidで制御したyahooへのアクセスが出来ない
![](https://cloud5.jp/wp-content/uploads/2023/05/スクリーンショット-2023-05-04-13.58.16-640x372.png)
<br>
#### 2.EC2(Squid)でログの確認
下記コマンドでアクセスを制御していることを確認
```shell
sudo cat /var/log/squid/access.log | grep TCP_DENIED
```
レスポンスとして下記のようなログが出力される
![](https://cloud5.jp/wp-content/uploads/2023/05/スクリーンショット-2023-05-04-14.00.35-640x69.png)
<br>

------------

## さいごに
やっと”見ちゃダメ”なサイトを制御することができるようになりました。
Squidは他にもキャッシュなどの設定も出来ることなど、新しい発見が多い構築となりました。
