---
title: EC2(RHEL)でlogrotate設定のハンズオン
tags:
  - EC2
  - logrotate
private: false
updated_at: '2022-12-10T16:30:30+09:00'
id: 9ef7baf655ffbd55e08e
organization_url_name: null
slide: false
ignorePublish: false
---
## はじめに
先日設定したlogrotateの設定を間違えアラートを鳴らした忘備録として、logrotateの設定について記述していきます。

※要件的な部分で`/etc/anacrontab`ではなく、`/etc/logrotate.d/【固有の設定】`を利用しています。

------------


## 問題の原因
下記設定の`/*.log*`としたことで、フォルダ内に出力される毎日のログと、世代管理するために生成された（本来であれば対象外となる）ログの全てがローテーションしてしまい、世代管理で生成されたログの世代管理のログの、、、という収集がつかない程のログが作成されました。

```json
/infra/log/*.log*{
 【個々の設定】
}
```

------------


## 何が起きたか？
logrotateの世代管理はリネームすることでログを管理するため、名前が次第に長くなり、24時間で全てのログのリネームが終了せずゾンビ化してCPUを圧迫していき、アラート通知。

------------


## logrotate　設定ハンズオン
### 0.ログ出力までの準備
#### 0.1.EC2にSSH接続してシェルスクリプト・ログ用のディレクトリ作成
EC2にSSH接続し、管理者権限でシェルスクリプト置き場と、出力ログ置き場となるディレクトリを作成
```json
sudo su -
mkdir -p /infra/{log,script}
```
#### 0.2.EC2にSSH接続してシェルスクリプト・ログ用のディレクトリ作成
検証用シェルスクリプト`logrotate.sh`を作成
```json
vi /infra/scipt/logrotate.sh
【設定後、下記で権限変更して起動できるようにする】
chmod +x logrotate.sh
bash logrotate.sh
```
【検証用シェルスクリプト内容】
```json
#/bin/bash
# reate : 2022/11/12
# desc : logrotate.sh
#############################################
# Local
HOSTNAME=`uname -n | awk -F"." '{print $1}'`
LOG=/infra/log/logrotate.log
#############################################
# Main
# log出力する
echo `LANG=C ; date` : "[INFO]:${HOSTNAME}:logrotate." | sudo tee -a  ${LOG}
exit 0
```
#### 0.3.crontab でシェルスクリプトを10分毎に設定する
シェルスクリプト10分毎に起動するためのcronを設定する
```json
crontab -e
```
【crontab内容】
```json
#おまじない
PATH=/sbin:/bin:/usr/sbin:/user/bin
MAILTO=""
HOME=/

# 分 ... 1分ごとにシェルスクリプトを押下する
# 時
# 日
# 月
# 曜日

*/10 * * * * /infra/script/logrotate.sh
```
#### 0.4.ログ出力されているか確認する
`/infra/log/`の配下に、`logrotate.log`が出力されていることが確認できる。
```json
ll /infra/log
-rw-r--r-- 1 root root 804 Nov 12 01:40 logrotate.log
```
#### 0.5.検証するためにログを用意する
次工程の確認用のためにログを増やします。
今回の検証では`logrotate`を`5`と設定するので、自動的に削除されるように`logrotate.log`の他に`5`つの`logrotate.log-2022*`を作成しておく。

```json
ll /infra/log

total 156
-rw-r--r-- 1 root root 1273 Nov 12 02:50 logrotate.log
-rw-r--r-- 1 root root 1206 Nov 12 02:45 logrotate.log-20221107
-rw-r--r-- 1 root root 1206 Nov 12 02:45 logrotate.log-20221108
-rw-r--r-- 1 root root 1206 Nov 12 02:45 logrotate.log-20221109
-rw-r--r-- 1 root root 1206 Nov 12 02:45 logrotate.log-20221110
-rw-r--r-- 1 root root 1206 Nov 12 02:45 logrotate.log-20221111
```

------------



### 1.logrotateの設定
#### 1.1.logrotateの設定ファイル(logdelete)を作成する
##### ファイル構成
【logdelete作成後、treeコマンドで表示画面（一部抜粋）】
```json
/etc/
├── logrotate.conf(共通設定)
├── logrotate.d
│   ├─【個別の設定ファイル一部省略】
│   └── logdelete(個別設定)
│  
```
`/etc/logrotate.conf`に全ての設定を記載することも可能ですが、`/etc/logrotate.conf`には、`/etc/logrotate.d`配下の個別の設定を呼び出すこともincludeされているため、サービス毎に設定ファイルを作成することが可能です。

##### 設定ファイル(logdelete)の作成
```json
# vi /etc/logrotate.d/logdelete
```
【logdelete内容】

```json
/infra/log/logrotate.log {
		rotate 5
		daily
		ifempty
		missingok
		create 0644 root root
		dateext
		dateformat -%Y%m%d
}
```
【logdelete設定内容の詳細】

|コマンド|内容|
|--|--|
|/infra/log/logrotate.log|対象とするログ|
|rotate|ローテーションする回数の指定|
|daily|ログを毎日ローテーションする|
|ifempty|ログファイルが空でもローテーションする|
|missingok|ログファイルが存在しなくてもエラーを出さずに処理を続行|
|create|（世代管理のため）新しく作成されたログファイルの権限|
|dateext|旧バージョンのログファイルに日付を付加する|
|dateformat|日付フォーマットを指定する|

#### 1.2.logrotateの設定ファイル(/etc/logrotate.d/logdelete(個別設定))に問題ないかでデバッグで検証する
`logrotate -d /etc/logrotate.d/logdelete`を実行します。
-dオプションを付けることで、実際にlogrotateは行われずに、どのように動作するのかをデバッグすることが可能です。

```json
# logrotate -dv /etc/logrotate.d/logdelete
Allocating hash table for state file, size 64 entries
Creating new state
Handling 1 logs

rotating pattern: /infra/log/logrotate.log  after 1 days (5 rotations)
empty log files are rotated, old logs are removed
considering log /infra/log/logrotate.log
Creating new state
  Now: 2022-11-12 02:22
  Last rotated at 2022-11-12 02:00
  log does not need rotating (log has already been rotated)
```

------------


### 2.logが手動で削除されるかの検証
#### 2.1.logrotate.statusの設定変更
logrotaetは初日に対象ログのタイムスタンプを記録するため、初日からログがローテートすることはありません。
ただし実際にローテーションを確認したいので、logrotateが実行完了した際に記録される結果ファイル(/var/lib/logrotate/logrotate.status)の編集していきます。
```json
vi /var/lib/logrotate/logrotate.status 
```
【logrotate.statusの詳細】
当日の日付が記載されている場合、記載よりも24h前の時間を記載（24時間以内だと、1.2.と同様の結果となります）

(例)
```json
logrotate state -- version 2
"/infra/log/logrotate.log" 2022-11-12-2:0:0
```
↓　記録がない場合でも、ログの出力先と時間を記述して保存します
```json
logrotate state -- version 2
"/infra/log/logrotate.log" 2022-11-11-2:0:0
```
#### 2.2.再度1.2.と同様の検証を実行する
再度`logrotate -d /etc/logrotate.d/logdelete`を実行します。

```json
Allocating hash table for state file, size 64 entries
Creating new state
Handling 1 logs
rotating pattern: /infra/log/logrotate.log  after 1 days (5 rotations)
empty log files are rotated, old logs are removed
considering log /infra/log/logrotate.log
  Now: 2022-11-12 03:31
  Last rotated at 2022-11-11 02:00
  log needs rotating
rotating log /infra/log/logrotate.log, log->rotateCount is 5
Converted ' -%Y%m%d' -> '-%Y%m%d'
dateext suffix '-20221112'
glob pattern '-[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'
renaming /infra/log/logrotate.log to /infra/log/logrotate.log-20221112
creating new /infra/log/logrotate.log mode = 0644 uid = 0 gid = 0
removing old log /infra/log/logrotate.log-20221107
```
下から4行目に想定したように`renaming /infra/log/logrotate.log to /infra/log/logrotate.log-20221112`とリネームされていること
最後の行に`/infra/log/logrotate.log-20221107`が削除されていることが確認できました。

【ローテーションの理解】
検証日：2022年11月12日

1. logrotateを実行する
2. logrotate.logをリネームして、新しく`logrotate.log-20221112`を作成する
3. `logrotate.log-2022*`という名前で世代管理している数が、`logrotate`の`5`を超えるため、一番古いネームの`logrotate.log-20221107`が削除される

------------


### 3.cronで実行するための設定
#### 3.1.anacrontabのコメントアウト
要件に従い`anacrontab`をコメントアウトする
```json
vi /etc/anacrontab
```
【anacrontabの設定】
下から3行目の`cron.daily`部分をコメントアウトする
```json
# /etc/anacrontab: configuration file for anacron

# See anacron(8) and anacrontab(5) for details.

SHELL=/bin/sh
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
# the maximal random delay added to the base delay of the jobs
RANDOM_DELAY=45
# the jobs will be started during the following hours only
START_HOURS_RANGE=3-22

#period in days   delay in minutes   job-identifier   command
#1      5       cron.daily              nice run-parts /etc/cron.daily
7      25      cron.weekly             nice run-parts /etc/cron.weekly
@monthly 45    cron.monthly            nice run-parts /etc/cron.monthly
```
#### 3.2.crontabの設定
`crontab`で今回は設定をしていくのでファイルを編集します。
```json
vi /etc/crontab
```
【crontabの設定】
```json
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root

# For details see man 4 crontabs

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name  command to be executed
59 23 * * * root run-parts /etc/cron.daily
```
末尾の部分で、23:59に`/etc/cron.daily`を呼び出すように記述を足しておきます

#### 3.3./etc/cron.daily/logrotateが存在しているか確認する
/etc/cron.daily/配下にlogrotateが存在しているかの確認です、ファイルがない場合は後述のように作成します
```json
cat /etc/cron.daily/logrotate
【logrotateの詳細内容】
chmod 755 logrotate
```
【logrotateの詳細内容】
ファイルを見ると`/usr/sbin/logrotate -s /var/lib/logrotate/logrotate.status /etc/logrotate.conf`の部分から、`/usr/sbin/logrotate　== logrotate`で`-s == logrotateのオプションで、ステータスファイルを参照（/var/lib/logrotate/logrotate.status）`を利用して`/etc/logrotate.conf`を実行していることがわかります。
```json
#!/bin/sh
/usr/sbin/logrotate -s /var/lib/logrotate/logrotate.status /etc/logrotate.conf
EXITVALUE=$?
if [ $EXITVALUE != 0 ]; then
    /usr/bin/logger -t logrotate "ALERT exited abnormally with [$EXITVALUE]"
fi
exit 0
```

------------


### おまけ.　logをcronを利用して削除されるかの検証(※ドライランでなく実際にlogrotate.conf起動させていきます)
#### おまけ.1.crontabを現在の時間に沿って変更する
`crontab`を修正
```json
vi /etc/crontab
```
【crontabの設定】
ちょうど 0分になりそうだったので時間を設定します
```json
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
0 * * * * root run-parts /etc/cron.daily
```

#### おまけ.2.起動したことを確認する
cronの実行ログを確認すると、ちょうど00分に起動を設定した`logrotate`が起動していることが確認できます。
```json
tail /var/log/cron
Nov 12 06:00:01 ip-192-168-20-179 run-parts[12051]: (/etc/cron.daily) starting logrotate
Nov 12 06:00:01 ip-192-168-20-179 run-parts[12051]: (/etc/cron.daily) finished logrotate
Nov 12 06:00:01 ip-192-168-20-179 run-parts[12051]: (/etc/cron.daily) starting update-client-config-packages
Nov 12 06:00:01 ip-192-168-20-179 run-parts[12051]: (/etc/cron.daily) finished update-client-config-packages
```

#### おまけ.3.起動したことを確認する
`logrotate -d /etc/logrotate.conf`で確認していた通りですが、`logrotate.log-20221107`が自動的に削除されていることを確認できました
```json
-rw-r--r-- 1 root root   67 Nov 12 06:00 logrotate.log
-rw-r--r-- 1 root root 1206 Nov 12 02:45 logrotate.log-20221108
-rw-r--r-- 1 root root 1206 Nov 12 02:45 logrotate.log-20221109
-rw-r--r-- 1 root root 1206 Nov 12 02:45 logrotate.log-20221110
-rw-r--r-- 1 root root 1206 Nov 12 02:45 logrotate.log-20221111
-rw-r--r-- 1 root root 2479 Nov 12 05:50 logrotate.log-20221112
```

------------


### 4.まとめ
ここまでの流れを振り返りとして
#### 1.`crontab`により、23：59に`/etc/cron.daily`が起動する
#### 2.`/etc/cron.daily`配下の`logrotate`の記載にあるコマンド`logrotate -s /var/lib/logrotate/logrotate.status /etc/logrotate.conf`を実行する
#### 3.`/etc/logrotate.conf`が設定ファイル`/var/lib/logrotate/logrotate.status`のステータスを参照して実行
#### 4. `/etc/logrotate.conf`にインクルードされている`/etc/logrotate.d/deletelot`（固有の設定）が起動する
という、今回は上記の流れを確認してきました

------------


## おわりに
実際に構築している時は、自分が何処の部分を検証してるのか？など疑問に思ったりなどして最終的な全体図を描けなかったため、諸々のテストに抜け漏れが生じていき、最終的な設定ファイルの記載漏れを防げませんでした。
検証の時間をきちんととり、構築に納得し、求めている値が出力できるまで、粘り切りたいと思える機会になりました。
