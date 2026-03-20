---
title: APIを利用して住所から郵便番号を表示するための構築
tags:
  - Python
  - API
private: false
updated_at: '2022-01-16T10:31:49+09:00'
id: 5faa4d8ae599af9dd1a4
organization_url_name: null
slide: false
ignorePublish: false
---
##　はじめに
年があけ年賀状がちらほら届き「『さて郵便番号を書いて返信してやろうか』って、コイツ郵便番号書いてねぇぞ」となりましてググれば一発のところ、『住所を入力して郵便番号を逆引きできればな』と思い立ち、今回のハンズオン構築を行ってみました。

##　参考資料
参考リンクには今回利用したAPIのホームページURLです。『郵便番号から住所の検索するAPI』は日本郵便様始め結構見つかったのですが『住所から郵便番号』はなかなか見当たらなかったのですが大変助かりました。

__[HeartRails Geo API](https://geoapi.heartrails.com/)__
出典:「位置参照情報」(国土交通省)の加工情報・「HeartRails Geo API」(HeartRails Inc.)

##　構成図
![スクリーンショット 2022-01-16 10.18.28.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/7449b5be-91b9-0e76-c018-2000f772bec5.png)

##　ハンズオン
### 1：コードの記述
サンプルコード（コメントアウトで各動作の内容記述あり）

```python:Postcode_Search.py
# ライブラリのインポート
import requests #APIをリクエストのため
import json #json形式で受け取ったデータの処理するため
import os #環境変数のため
import sys #exit()関数でプログラムを終了させるため
from dotenv import load_dotenv #環境変数を利用するため
load_dotenv()

# 環境変数
yahoo_api_key  = "&appid=" + os.getenv('YAHOO')#トークン直打ちでも可能

#yahooジオコーダ変数
geo_api_url = "https://map.yahooapis.jp/geocode/V1/geoCoder?"
y_parm1 = "&output=json" #APIのレスポンスをjson形式に指示す
y_parm2 = "&query=" #調べたい場所の引数のプレフィックス

#緯度経度による町域情報一覧の変数
city_api_url = 'http://geoapi.heartrails.com/api/json?method=searchByGeoLocation'

#緯度経度を抽出する
def find_latitude_and_longitude(area):
  # 変数
  geo = geo_api_url + yahoo_api_key + y_parm1 + y_parm2 + area # Yahoo!ジオコーダによる取得したい場所の情報をURLにする
  geo_info = requests.get(geo) #リクエストする
  geo_obj = json.loads(geo_info.text)#リクエストしたjsonの内容を解析する
  try: #入力した地点の緯度経度が有無で処理が分岐する
    geo_parm = geo_obj["Feature"][0]["Geometry"]["Coordinates"] # オブジェクトから座標情報を取得する
    geo_parm_list = geo_parm.split(",")# 取得した座標をカンマごとにリストで分ける
    longitude = geo_parm_list[0] #経度
    latitude = geo_parm_list[1] #緯度
    return(latitude,longitude) #後続関数に緯度・経度を返す
  except: #緯度経度が取得出来ない場合の処理
    print("申し訳ございませんが、入力した地点の緯度経度が取得出来ませんでした")
    sys.exit() #対話型シェルを修了させるための関数


#緯度経度から町域情報を取得して郵便番号を抽出する
def town_area_information(latitude,longitude):
  # 変数
  t_parm1 = "&x=" + longitude
  t_parm2 = "&y=" + latitude
  cityarea = city_api_url + t_parm1 + t_parm2
  cityarea_info = requests.get(cityarea)#リクエストする
  cityarea_obj = json.loads(cityarea_info.text)#リクエストしたjsonの内容を解析する
  cityarea_parm = cityarea_obj["response"]["location"][0]["postal"]# jsonから郵便番号だけを取得する
  return(cityarea_parm)

if __name__ == "__main__":
  print("郵便番号を知りたい県名から市長村名までを入力ください\n例:埼玉県川口市\n")
  area = input() 
  #area = "埼玉県川口市"
  print(area + "の郵便番号は以下の番号です")
  latitude,longitude = find_latitude_and_longitude(area)
  postcode = town_area_information(latitude,longitude)
  print("〒" + postcode[:3] + "-" + postcode[3:])
```
### 2:部分的な説明
#### 2.1 今回利用するライブラリをインストールする

```python:Postcode_Search.py
# ライブラリのインポート
import requests #APIをリクエストのため
import json #json形式で受け取ったデータの処理するため
import os #環境変数のため
import sys #exit()関数でプログラムを終了させるため
from dotenv import load_dotenv #環境変数を利用するため
load_dotenv()
```
緯度経度が該当しない地名に関しては処理を修了させるために`import sys`を利用

#### 2.2 .envの環境変数を代入する
```python:Postcode_Search.py
# 環境変数
yahoo_api_key  = "&appid=" + os.getenv('YAHOO')#トークン直打ちでも可能

```
.envファイルをから、load_dotenvでファイルの中身を読み取り環境変数として読み込む

```terminal:ディレクトリの構成
.
├── .env
└── Postcode_Search.py
```


#### 2.3 関数で利用するの変数を代入する

```python:Postcode_Search.py
#yahooジオコーダ変数
geo_endpoint = "https://map.yahooapis.jp/geocode/V1/geoCoder?"
y_parm1 = "&output=json" #APIのレスポンスをjson形式に指示す
y_parm2 = "&query=" #調べたい場所の引数のプレフィックス

#緯度経度による町域情報一覧の変数
city_endpoint = 'http://geoapi.heartrails.com/api/json?method=searchByGeoLocation'
```

#### 2.4 find_latitude_and_longitude()関数

```python:Postcode_Search.py
#緯度経度を抽出する
def find_latitude_and_longitude(area):
  # 変数
  geo = geo_endpoint + yahoo_api_key + y_parm1 + y_parm2 + area # Yahoo!ジオコーダによる取得したい場所の情報をURLにする
```
引数`area`を受け取り`geo`に代入する

```python:Postcode_Search.py
  geo_info = requests.get(geo) #リクエストする
  geo_obj = json.loads(geo_info.text)#リクエストしたjsonの内容を解析する
```

__[Yahoo!ジオコーダAPI レスポンスフィールド](https://developer.yahoo.co.jp/webapi/map/openlocalplatform/v1/geocoder.html#response_field)__についての詳細(下記画面は一部抜粋)
![スクリーンショット 2022-01-01 12.38.58.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/58e21954-18f3-0e34-07f1-f2e1a64b8480.png)

__ブラウザでURLを押下した際の画面抜粋__
![スクリーンショット 2022-01-01 13.22.59.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/77ce92b8-72e9-bfde-58fb-555ef57bdc27.png)
※緯度・経度の取得した`順番に注意する`

```python:Postcode_Search.py
  try: #入力した地点の緯度経度が有無で処理が分岐する
    geo_parm = geo_obj["Feature"][0]["Geometry"]["Coordinates"] # オブジェクトから座標情報を取得する
    geo_parm_list = geo_parm.split(",")# 取得した座標をカンマごとにリストで分ける
    longitude = geo_parm_list[0] #経度
    latitude = geo_parm_list[1] #緯度
    return(latitude,longitude) #後続関数に緯度・経度を返す
  except: #緯度経度がない場合の処理
    print("申し訳ございませんが、入力した地点の緯度経度が取得出来ませんでした")
    sys.exit() #対話型シェルを修了させるための関数
```
例外処理として`try,except`を利用しており、下記のような動作とする

```python
try:
  Yahoo!ジオコーダAPIで緯度,経度取得
　　　　後続に引数（緯度,経度）を渡す
except:
  ただし該当なしの場合は「取得出来ませんでしたコメント」及び
　　　　システムの修了
```
#### 2.5 town_area_information()関数

```python:Postcode_Search.py
def town_area_information(latitude,longitude):
  # 変数
  t_parm1 = "&x=" + longitude
  t_parm2 = "&y=" + latitude
  cityarea = city_endpoint + t_parm1 + t_parm2
  cityarea_info = requests.get(cityarea)#リクエストする
```
find_latitude_and_longitude()関数で取得した緯度,経度をAPIにリクエストする

```python:Postcode_Search.py
  cityarea_obj = json.loads(cityarea_info.text)#リクエストしたjsonの内容を解析する
  cityarea_parm = cityarea_obj["response"]["location"][0]["postal"]# jsonから郵便番号だけを取得する
  return(cityarea_parm)
```
今回ご利用させてもらっているAPIのリクエストパラメータ、レスポンスフィールド一覧![スクリーンショット 2022-01-16 9.25.53.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/e850d2a3-175b-c8e8-19b2-2730f3c5dff5.png)

下記形式は`XML形式`で取得した際の見え方の一例
![スクリーンショット 2022-01-16 9.30.32.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/831a1847-fa1c-a5b8-4ab1-c7a6831fdf47.png)
上記の内容を、入力した地域の`json形式`で`postal`部分だけを取得している

#### 2.6 モジュールを直接実行したときだけ実行する動作の指定

```python:Postcode_Search.py
if __name__ == "__main__":
  print("郵便番号を知りたい県名から市長村名までを入力ください\n例:埼玉県川口市\n")
  area = input() 
  #area = "埼玉県川口市"
  print(area + "の郵便番号は以下の番号です")
  latitude,longitude = find_latitude_and_longitude(area)
  postcode = town_area_information(latitude,longitude)
  print("〒" + postcode[:3] + "-" + postcode[3:])
```
出力の見え方として、取得した郵便番号を整形`"〒" + postcode[:3] + "-" + postcode[3:]`

### 3：挙動の確認
該当する緯度・経度がある場合

```terminal:ターミナルで押下
tetutetu214@mbp 0_Qiita_hanson % python Postcode_Search.py
郵便番号を知りたい県名から市長村名までを入力ください
例:埼玉県川口市

埼玉県川口市
埼玉県川口市の郵便番号は以下の番号です
〒332-0016
```

該当する緯度・経度がない場合

```terminal:ターミナルで押下
tetutetu214@mbp 0_Qiita_hanson % python Postcode_Search.py
郵便番号を知りたい県名から市長村名までを入力ください
例:埼玉県川口市

あああああ
あああああの郵便番号は以下の番号です
申し訳ございませんが、入力した地点の緯度経度が取得出来ませんでした
```
__※確認中みつけてしまったこと__

```
tetutetu214@mbp 0_Qiita_hanson % python Postcode_Search.py
郵便番号を知りたい県名から市長村名までを入力ください
例:埼玉県川口市

なぜなの
なぜなのの郵便番号は以下の番号です
〒894-0036
```
`yahooジオコーダAPI`において入力された住所がない場合、上位のレベルで再検索などにより、よっぽど該当しない文言でない限り検索してしまうからか。
完全マッチしか取得出来ないようにパラメータに記述すれば、問題を防げそうだと思っています。が、今回は後回しにします。

##　さいごに
`yahooジオコーダAPI`の出力について問題ありますが、当初の想定であった県名市町村までを入力すれば郵便番号までは取得することはできました。もやもやは残っていますが。
`try,except`などは今回始めて利用してみましたが、挙動としては動くものの記述する場所は正しいのか？など課題を残すような構築となりました。来週もまた再度コードを書きながら理解を深めていきたいと思います。
