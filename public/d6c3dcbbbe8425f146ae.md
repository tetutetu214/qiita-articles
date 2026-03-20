---
title: EC2インスタンスの時刻設定のハンズオン
tags:
  - AWS
  - EC2
private: false
updated_at: '2022-12-10T15:51:09+09:00'
id: d6c3dcbbbe8425f146ae
organization_url_name: null
slide: false
ignorePublish: false
---
## はじめに
今回はEC2（Windows）インスタンスに、日本時間を設定するためのハンズオンをしていきます。

### 1.NTP（Network Time Protocol：機器の時刻情報を同期するプロトコル）の設定
Windowsインスタンスのシステムが再起動しても、日本時刻が維持されるようにするため（設定しないと再起動のたびにUTC時間が適用される）、AWSが提供しているNTPをEC2（Windows）に設定します
設定方法詳細：[AWSドキュメント：Windows インスタンスの時刻の設定](https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/WindowsGuide/windows-set-time.html)
#### 1.1.コマンドプロンプトで下記コマンドでレジストリキーを追加する
追加された場合"この操作を正しく終了しました。"と表示される
```
reg add "HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /d 1 /t REG_DWORD /f
```
![](https://cloud5.jp/wp-content/uploads/2022/09/スクリーンショット-2022-09-04-11.18.07-640x43.png)

#### 1.2.RealTimeIsUniversal キーが正しく保存されていることを確認する
```
reg query "HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\TimeZoneInformation" /s
```
保存されている場合は上記コマンドのレスポンスとして、一番下の欄に``RealTimeIsUniversal``の表示がされる
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-18-8.46.49-640x161.png)

#### 1.3.``w32tm``コマンドで手動で時刻を同期する
```
w32tm /resync
```
![](https://cloud5.jp/wp-content/uploads/2022/07/スクリーンショット-2022-07-18-8.50.20.png)

### 2.日本時刻に設定する
##### 2.1.Windowsの設定画面から『日付と時刻』を選択
![](https://cloud5.jp/wp-content/uploads/2022/09/スクリーンショット-2022-09-04-11.21.37-640x507.png)

##### 2.2.タイムゾーンを日本時刻に設定する
![](https://cloud5.jp/wp-content/uploads/2022/09/スクリーンショット-2022-09-04-11.23.41-1-549x640.png)

##### 2.3.タイムゾーンが変更されたことを確認する
時刻が日本時間になっているか比較する
![](https://cloud5.jp/wp-content/uploads/2022/09/スクリーンショット-2022-09-04-11.32.46-640x518.png)


## おわりに
インスタンス自体はプライベートサブネットにあるので、時刻を取得できるのか不明だったためハンズオンしてみました。
SGもNACLも穴あけしてないなと思いながら、再度ドキュメントを読み直したら、``インスタンスはインターネットにアクセスする必要はなく、アクセスを許可するためにセキュリティグループルールまたはネットワーク ACL ルールを設定する必要はありません。``とありました（笑）
ちゃんとドキュメントは読むようにします。
