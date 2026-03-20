---
title: '天気予報情報をスクレイピング（Python）で、LINE NotifyによりLINE通知の構築 '
tags:
  - Python
  - スクレイピング
  - LineNotify
private: false
updated_at: '2021-12-09T11:51:36+09:00'
id: 14a2bda4cb6fcca5b0c4
organization_url_name: null
slide: false
ignorePublish: false
---
#　はじめに
今回は『スクレイピングって、どんな動きをしているの？』と中の動きがサッパリだったので、構築を交えながらスクレイピングについて学習していきたいと思いたち、下記参考資料を読みながら手を動かして構築してみました

#　参考資料
参考リンクにはcronで自動化や、お天気の地方コードなどの記述もあり、どちらもハンズオンを進める中で大変理解が捗りました

[__LINE Notify + Pythonで天気情報を取得する方法__](https://qiita.com/S_eki/items/206ddb321768ad4e7544)
[__【Python】Yahoo天気予報をスクレイピングしてデータを入手する__](https://www.ishilog.com/python-scraping-yahoo-weather/)
[__データ収集を大幅に効率化する「スクレイピング」とは？
手法やルール・注意点を解説！__](https://data.wingarc.com/scraping-27053)

#　構成図
![スクリーンショット 2021-12-04 10.06.09.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/f2d9a2bf-1698-3c62-5707-99cfda0fef85.png)


#　ハンズオン
## 1：[LINE Notify](https://notify-bot.line.me/ja/)でアクセストークンを発行する
### 1：ログインをする
![スクリーンショット 2021-12-04 10.13.09.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/abfc2cb4-42c0-bc27-b225-87c974d2707b.png)

### 2:アクセストークンの発行を押下
![スクリーンショット 2021-12-04 10.15.09.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/b411e7b9-3352-d563-ace8-68679bb5854d.png)

### 3：トークンを発行する
LINE　Notifyから送られてくるメッセージのタイトルみたいな形で毎回表示されます。
送信先（今回は製作者宛に選択しています）
![スクリーンショット 2021-12-04 9.28.44.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/5339126c-900c-43f4-877f-3a23bc61a6b0.png)

### 4：トークンが発行されたのを確認する（要保存）
要保存と書きましたが、忘れたら気軽に再発行すれば構いません。
![スクリーンショット 2021-12-04 9.09.26.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/63324f5e-4c16-5fb4-6faa-8f43bd67cbf1.png)

### 5：画面から連携されていることを確認する
![スクリーンショット 2021-12-04 9.29.53.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/ff9affb2-8289-9187-6818-1bc2586f623b.png)

以上で__LINE Notify__の設定は終了です

## 2：コードの記述
### 1：サンプルコード

```python:weather.py
import requests
from bs4 import BeautifulSoup

#LINE Notifyと連携するためのtoken
line_notify_token = 'XXXXXXXXX'
line_notify_api = 'https://notify-api.line.me/api/notify'

#requests.getでHTMLを取得
r = requests.get('https://weather.yahoo.co.jp/weather/jp/11/4310.html')
#BeautifulSoupを使用してパース
soup = BeautifulSoup(r.content, "html.parser")

wc = soup.find(class_="forecastCity")
#print(wc)

#.strip():前後の空白文字の削除・.splitlines():改行コードで分割
ws = [i.strip() for i in wc.text.splitlines()]
#print(ws)

#リスト内包表記で""でないものをリスト化
wl = [i for i in ws if i != ""]
#print(wl)

message = ("\n" + "埼玉さいたま市:" + wl[0] + "\n" + wl[1] + "\n"  + "最高気温:" + wl[2] + "\n"+ "最低気温:" + wl[3] + "\n" + "\n" + wl[18] + "\n" + wl[19] + "\n" + "最高気温:" + wl[20] + "\n" + "最低気温:" + wl[21] )

#LINENotifyへ通知の記述
payload = {'message': message}
headers = {'Authorization': 'Bearer ' + line_notify_token} 
line_notify = requests.post(line_notify_api, data=payload, headers=headers)
```

## 3：コードの解説
### 1:事前準備としてrequestsとBeauifulSoupをインストールする
フォルダの作成

```terminal:ローカル環境ディレクトリ作成 
tetutetu214@mbp mkdir 20211204_weather      
tetutetu214@mbp cd 20211204_weather
tetutetu214@mbp 20211204_weather touch weather.py
pip install requests 
pip install beautifulsoup4
```

```python:コード記述部分
import requests
from bs4 import BeautifulSoup
```

`requests`はHPを開くことが出来て、`requests.get`とすることで`GETメソッド`に相当します。
`BeautifuSoup`は、HTMLファイル・XMLファイルからデータを抽出するためのライブラリで、これらを組み合わせて今回は情報を取得します

### 2:Yahoo天気予報ページのURLからHTML取得・パース
今回は埼玉県の天気予報を取得していきます→https://weather.yahoo.co.jp/weather/jp/11/4310.html

![スクリーンショット 2021-12-04 10.36.30.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/c3501768-a0b9-eafc-d66c-4a4a9a0d2481.png)

```python:コード記述部分
#requests.getでHTMLを取得
r = requests.get('https://weather.yahoo.co.jp/weather/jp/11/4310.html')
#BeautifulSoupを使用してパース
soup = BeautifulSoup(r.content, "html.parser")
```
`requests.get`を利用してURLにアクセスしてHTML情報を取得して、`BeautifulSoup`で取得した情報から`パース`の作成をしています。

__パース（parser）とは？__
WebブラウザがHTMLファイルを表示するためにタグなどの構造・属性を解析して、表示可能なデータ構造に変換させるプログラムのこと。
requestsでHTML情報を取得しても解析しないと表示ができないため、BeautifulSoupで表示可能な情報に変換させている。

### 3:データの抽出

```python:コード記述部分
wc = soup.find(class_="forecastCity")
```
上記のコード部分は、HPの「forecastCity」というクラスの該当部分をリストで格納している動きとなります。
![スクリーンショット 2021-12-04 11.37.42.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/43331e8e-298c-41a7-a736-0fdfaf9fd6c8.png)

```terminal:この時点でのprint(wc)のターミナルの表示
<div class="forecastCity">
<table>
<tr>
<td>
<div>
<p class="date">
<span>12月4日</span>(<span class="daySat">土</span>)

        </p>
________________________________________________________
省略
________________________________________________________
</dl>
</div>
</td>
</tr>
</table>
</div>

```

### 4:リストの加工

取得したwcのテキストデータの改行コード毎に分割をして、分割した文字の前後に空白文字であれば削除として加工をしていきます

```python:コード記述部分
#.strip():前後の空白文字の削除・.splitlines():改行コードで分割
ws = [i.strip() for i in wc.text.splitlines()]
```

```terminal:この時点でのprint(ws)のターミナルの表示
['', '', '', '', '', '', '12月4日(土)', '', '', '', '晴れ', '', '', '15℃[0]', '3℃[+1]', '', '', '', '時間', '0-6', '6-12', '12-18', '18-24', '', '', '降水', '---', '---', '10％', '0％', '', '', '', '風：', '北の風', '波：', '---', '', '', '', '', '', '', '12月5日(日)', '', '', '', '晴れ', '', '', '11℃[-4]', '3℃[0]', '', '', '', '時間', '0-6', '6-12', '12-18', '18-24', '', '', '降水', '0％', '0％', '0％', '0％', '', '', '', '風：', '北の風後東の風', '波：', '---', '', '', '', '', '']
```
上記コードの空白要素を消していく加工をします
if文で""でない部分を表示します

```python:コード記述部分
#リスト内包表記
wl = [i for i in ws if i != ""]
```

```terminal:この時点でのprint(wl)のターミナルの表示
['12月4日(土)', '晴れ', '15℃[0]', '3℃[+1]', '時間', '0-6', '6-12', '12-18', '18-24', '降水', '---', '---', '10％', '0％', '風：', '北の風', '波：', '---', '12月5日(日)', '晴れ', '11℃[-4]', '3℃[0]', '時間', '0-6', '6-12', '12-18', '18-24', '降水', '0％', '0％', '0％', '0％', '風：', '北の風後東の風', '波：', '---']
```
__【リスト対応表】__
![スクリーンショット 2021-12-04 12.16.02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/2b2b775d-7c3d-aece-b6a7-8bedbfed98eb.png)

### 5:リストからデータの出力

```python:コード記述部分
message = ("\n" + "埼玉さいたま市:" + wl[0] + "\n" + wl[1] + "\n"  + "最高気温:" + wl[2] + "\n"+ "最低気温:" + wl[3] + "\n" + "\n" + wl[18] + "\n" + wl[19] + "\n" + "最高気温:" + wl[20] + "\n" + "最低気温:" + wl[21] )
```

```terminal:この時点でのprint(message)のターミナルの表示
埼玉さいたま市:12月4日(土)
晴れ
最高気温:15℃[0]
最低気温:3℃[+1]

12月5日(日)
晴れ
最高気温:11℃[-4]
最低気温:3℃[0]
```

### 6:出力をLINE Notifyで出力をする

```python:コード記述部分
#LINE Notifyと連携するためのtoken
line_notify_token = 'XXXXXXXXXXX'
line_notify_api = 'https://notify-api.line.me/api/notify'

_______________________________________
省略
_______________________________________

#LINENotifyへ通知の記述
payload = {'message': message}
headers = {'Authorization': 'Bearer ' + line_notify_token} 
line_notify = requests.post(line_notify_api, data=payload, headers=headers)
```
[LINE Notifyドキュメント](https://notify-bot.line.me/doc/ja/)より通知系のドキュメントを確認してAPIの形はPOSTと確認出来、`requests.post`を利用することでPOST送信の形をとる。

![スクリーンショット 2021-12-04 9.12.47.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/8bdcb7d9-7e78-e167-1aa6-6f53c6d06af4.png)

ここまででコードの解説は終了です

## 4：挙動の確認
```terminal:ターミナルでコードを押下する
tetutetu214@mbp 20211204_weather python weather.py
```

上記打刻後LINEに下記通知がされました。
cronを利用すれば決まった時間に通知を送ることも可能です。
![スクリーンショット 2021-12-04 10.45.34.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/32acabb9-4f04-feba-f967-65432284abc3.png)

#　さいごに
スクレイピングの知識も乏しく、なんとなく「情報をとってこれるんだよね！便利だが知らん」からの知識から、ある程度の書き方を理解して書けるようになりました。　また別のサイトからの情報を取得しながら理解を深めていきたいと思うのでした。
地味にLINE Notifyの通知も応用が効きやすい技術で、いろいろ掛け合わせながら師走も学習をしていきます。
