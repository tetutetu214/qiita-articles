---
title: API GatewayとLambdaのデータ連携学び直しハンズオン
tags:
  - lambda
  - APIGateway
private: false
updated_at: '2025-07-02T21:28:30+09:00'
id: e8118eecb3bacaecfaba
organization_url_name: null
slide: false
ignorePublish: false
---
## 0.はじめに
API Gateway から Lambda関数への接続について、雰囲気で構築していたことをここに正直に告白致します。
なので改めてデータの渡しかたについて学びなおしたいと思います。

### 0.1.学びなおしたいこと
今回は、以下4点について学びなおしたいと思います。
- パスパラメータ（URLパス内の変数）の取得方法
- クエリパラメータ（URLの？以降）の取得方法
- リクエストボディからのJSONデータの取得方法
- ヘッダー情報の取得方法

## 1.基本的な用語
### 1.1.パス パラメータとは？
- Webサーバの情報にアクセスするために**URLに値を埋込む(URLの一部)**方法
- 個人的理解：特定サーバへアクセスするためのパスのためのパラメータ

```shell
# パスパラメータ 基本の型
https://hogehoge.jp/user/{user_id}

# 実際のURL
https://hogehoge.jp/user/123456
```

### 1.2.クエリ パラメータとは？
- Webサーバーに情報を検索するために**URLに付け加える**方法
- 個人的理解：サーバ内の情報を検索するためのクエリのためのパラメータ
 
```shell
# クエリパラメータ 基本の型
「?(ここからスタート)」+「変数(パラメータの変数)」+「=」+「変数の値(パラメータの値)」

#  例1：「変数：user_id」で「変数の値：12345」
https://hogehoge.jp/?user_id=12345

#  例2：パラメータが複数ある場合は「&」でつなぐ
https://hogehoge.jp/?user_id=12345&name=tetutetu
```

### 1.3.使い分けのポイント

|比較|パス パラメータ|クエリ パラメータ|
|---|---|---|
|役割|リソースの特定・識別|フィルタリング・検索・オプション指定|
|位置|位置URLのパス部分に埋め込む|URLの末尾に`?`以降で追加|
|必須/任意|通常は必須|通常は任意|
|複数指定|階層構造|`&`で区切って複数指定|

### 1.4.リクエストボディとは？
- HTTPリクエストの本文として送信されるデータ
  主にPOST、PUT、PATCHなどのメソッドでデータを送信する際に使用される。
- リクエストボディのデータ構造は、そのAPI作成者が設計して管理する。

##### 例：以下のようなアンケートページの場合
- APIの設計：
  このAPIにはnameとemail、質問の回答としてq1、q2 というフィールドが必要と設計

- フロントエンドの実装：
  ユーザーが入力したデータを集め、決められた形式(name等)でAPI(https://hogehoge.jp/submit-survey)にPOST

```javascript
// フロントエンド側のコード（ユーザーがフォームに入力した後）
const userData = {
  name: "tetutetu",  // ユーザーが入力した名前
  email: "hogehoge@hugahuga.com",  // ユーザーが入力したメール
  q1: "はい",  // 質問1の回答
  q2: "いいえ",  // 質問2の回答
};

// このデータをAPIにPOST
fetch('https://hogehoge.jp/submit-survey', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify(userData)  // ここでJSONに変換してリクエストボディとして送信
});
```

- バックエンドの実装：
  送られてきたリクエストボディを受け取って処理

```python
# Lambda関数側（バックエンド）
def lambda_handler(event, context):
    # リクエストボディを取得してJSONに変換
    body = json.loads(event.get('body', '{}'))
    
    # 各フィールドを取り出す
    name = body.get('name')
    email = body.get('email')
    q1_answer = body.get('q1')
    q2_answer = body.get('q2')
```

### 1.5.ヘッダー情報とは？
- HTTPリクエストの本文**以外**の送信されるデータ
- 基本的なヘッダーはHTTP標準で決まってるが、カスタムヘッダーとして追加も可能で、その部分に関してはAPI作成者が設計・管理する。

- 一般的なヘッダーの例：

|項番|ヘッダー名|詳細|
|---|---|---|
|1|Content-Type|リクエストボディの形式(例：application/json)|
|2|Authorization|認証情報(例：Bearer eyJhbGciOiJ...)|
|3|Accept|クライアントが受け入れられるレスポンス形式|
|4|User-Agent|クライアントの種類(ブラウザ情報など)

- カスタムヘッダーの例：

|項番|ヘッダー名|詳細|
|---|---|---|
|1|X-API-Key|APIキー|
|2|X-Request-ID|リクエスト識別子|
|3|X-Custom-Header|独自の情報|

- バックエンドの実装：
  ヘッダー情報の取得方法

```python
def lambda_handler(event, context):
    # ヘッダー情報を取得
    headers = event.get('headers', {})
    
    # 標準ヘッダー
    content_type = headers.get('Content-Type')
    authorization = headers.get('Authorization')
    
    # カスタムヘッダー
    api_key = headers.get('X-API-Key')
```

## 2.ハンズオン
### 2.0.前提条件
- 以下ハンズオンは、AWS CloudShell環境にて実施する

### 2.1.Lambda用 IAMロール作成
```shell
# 作業フォルダ作成
mkdir 20250429_handson && cd 20250429_handson

# 変数設定
LAMBDA_FUNCTION_NAME="submit-survey"
API_NAME="survey-api"
STAGE_NAME="dev"
REGION="us-east-1"
ROLE_NAME="lambda-submit-survey-role"

# 信頼ポリシードキュメントの作成
cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# IAMロールの作成
aws iam create-role --role-name $ROLE_NAME --assume-role-policy-document file://trust-policy.json

# IAMロールにLambda基本実行ポリシーをアタッチ
aws iam attach-role-policy --role-name $ROLE_NAME --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# IAMロールARNの取得
ROLE_ARN=$(aws iam get-role --role-name $ROLE_NAME --query 'Role.Arn' --output text)
echo "作成したロールARN: $ROLE_ARN"
```

### 2.2.Lambda 作成
#### 2.2.1.Lambda関数作成
```shell
# Lambda関数用のフォルダ作成
mkdir lambda_function

# Lambda関数のPythonコードを作成
cat > lambda_function/lambda_function.py << EOF
import json

def lambda_handler(event, context):
    # 受信イベントの全体をログに出力
    print(f"受信イベント: {json.dumps(event)}")

    # --- パスパラメータの取得 ---
    # pathParameters キーが存在するか確認し、辞書として取得
    # 例: /user/{user_id} の場合、event['pathParameters']['user_id'] で取得
    path_parameters = event.get('pathParameters', {})
    print(f"パスパラメータ: {json.dumps(path_parameters)}")

    # --- クエリパラメータの取得 ---
    # queryStringParameters キーが存在するか確認し、辞書として取得
    # 例: /?user_id=123&name=abc の場合、event['queryStringParameters']['user_id'] で取得
    query_parameters = event.get('queryStringParameters', {})
    print(f"クエリパラメータ: {json.dumps(query_parameters)}")

    # --- ヘッダー情報の取得 ---
    # headers キーが存在するか確認し、辞書として取得
    # ヘッダー名は環境によって大文字小文字が異なる場合があるため、取得時は注意
    headers = event.get('headers', {})
    print(f"ヘッダー情報: {json.dumps(headers)}")
    # 特定のヘッダーを取得する例
    content_type = headers.get('Content-Type') # 一般的
    # または headers.get('content-type') のように小文字で試す場合もある
    authorization = headers.get('Authorization')
    custom_header = headers.get('X-Custom-Header') # カスタムヘッダーの例
    
    print(f"Content-Type: {content_type}")
    print(f"Authorization: {authorization}")
    print(f"X-Custom-Header: {custom_header}")


    # --- リクエストボディの解析 ---
    body = {}
    if event.get('body'):
        try:
            # API Gateway プロキシ統合の場合、bodyは文字列として渡されるためjson.loadsが必要
            body = json.loads(event.get('body'))
            print(f"リクエストボディ: {json.dumps(body)}")

            # 各フィールドを取り出す
            print(f"名前 (ボディ): {body.get('name')}")
            print(f"メール (ボディ): {body.get('email')}")
            print(f"質問1 (ボディ): {body.get('q1')}")
            print(f"質問2 (ボディ): {body.get('q2')}")

        except Exception as e:
            print(f"ボディの解析エラー: {e}")
            # エラーレスポンスを返す例
            return {
                'statusCode': 400,
                'headers': { 'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*' },
                'body': json.dumps({'message': 'リクエストボディの形式が不正です'})
            }

    # 成功レスポンスを返す
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*'  # CORSを有効化
        },
        'body': json.dumps({
            'message': 'データを受信しました',
            'received_data': {
                'pathParameters': path_parameters,
                'queryStringParameters': query_parameters,
                'headers': headers,
                'body': body
            }
        }, ensure_ascii=False)
    }
EOF
```

#### 2.2.2.Lambda関数のzip作成
```shell
cd lambda_function
zip -r ../lambda_function.zip .
cd ..
```
#### 2.2.3.Lambdaのデプロイ
```shell
# Lambdaデプロイコマンド
aws lambda create-function \
  --function-name $LAMBDA_FUNCTION_NAME \
  --runtime python3.9 \
  --handler lambda_function.lambda_handler \
  --role $ROLE_ARN \
  --zip-file fileb://lambda_function.zip

# LambdaのARN取得
LAMBDA_ARN=$(aws lambda get-function --function-name $LAMBDA_FUNCTION_NAME --query 'Configuration.FunctionArn' --output text)
echo "作成したLambda関数ARN: $LAMBDA_ARN"
```
### 2.3.API Gatewayの作成
- APIGatewayの構成

|項番|項目|設定値|説明|
|---|---|---|---|
|1|APIの種類|REST API|構築するAPIのタイプ|
|2|リソース|/submit-survey|クライアントからのURLパス|
|3|メソッド|POST|/submit-survey リソースに対して許可された操作|
|4|統合タイプ|AWS_PROXY|リクエストを Lambda 関数に直接転送する方式（プロキシ統合）|
|5|統合先|$LAMBDA_FUNCTION_NAME|リクエスト処理を実行する Lambda 関数名|
|6|パラメータ・ボディ・ヘッダーの転送|自動|プロキシ統合により`event` に全て含まれる|


```shell
# REST APIの作成
API_ID=$(aws apigateway create-rest-api \
  --name $API_NAME \
  --endpoint-configuration types=REGIONAL \
  --query 'id' --output text)
echo "作成したAPI ID: $API_ID"

# ルートリソースIDの取得
ROOT_RESOURCE_ID=$(aws apigateway get-resources \
  --rest-api-id $API_ID \
  --query 'items[0].id' --output text)
echo "ルートリソースID: $ROOT_RESOURCE_ID"

# /submit-survey リソースの作成
RESOURCE_ID=$(aws apigateway create-resource \
  --rest-api-id $API_ID \
  --parent-id $ROOT_RESOURCE_ID \
  --path-part "submit-survey" \
  --query 'id' --output text)
echo "作成したリソースID: $RESOURCE_ID"

# POSTメソッドの作成
aws apigateway put-method \
  --rest-api-id $API_ID \
  --resource-id $RESOURCE_ID \
  --http-method POST \
  --authorization-type NONE

# Lambda関数との統合
aws apigateway put-integration \
  --rest-api-id $API_ID \
  --resource-id $RESOURCE_ID \
  --http-method POST \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri arn:aws:apigateway:$REGION:lambda:path/2015-03-31/functions/$LAMBDA_ARN/invocations
```

### 2.4.Lambdaにリソースベースポリシーを付与
```shell
# Lambda関数にAPIゲートウェイからの呼び出し権限を付与
aws lambda add-permission \
  --function-name $LAMBDA_FUNCTION_NAME \
  --statement-id apigateway-permission \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:$REGION:$(aws sts get-caller-identity --query 'Account' --output text):$API_ID/*"
```

### 2.5.APIGateway デプロイ
```shell
# APIのデプロイ
aws apigateway create-deployment \
  --rest-api-id $API_ID \
  --stage-name $STAGE_NAME

# API URLの表示
API_URL="https://$API_ID.execute-api.$REGION.amazonaws.com/$STAGE_NAME/submit-survey"
echo "API URL: $API_URL"
```
### 2.6.挙動確認
#### 2.6.1.APIGatewayにコマンド実行
```shell
# 以下コマンドで URLを作成できます
echo "curl -X POST -H \"Content-Type: application/json\" -d '{\"name\":\"てつてつ\",\"email\":\"test@example.com\",\"q1\":\"はい\",\"q2\":\"いいえ\"}' $API_URL"

# レスポンスで出力された curl コマンドを実行
# 以下 例URL
curl -X POST -H "Content-Type: application/json" -d '{"name":"てつてつ","email":"test@example.com","q1":"はい","q2":"いいえ"}' https://${API_ID}.execute-api.us-east-1.amazonaws.com/dev/submit-survey

# 出力レスポンス例(※可視性を重視し、実際の出力から改行しています)
{
  "message": "データを受信しました",
  "received_data": {
    "pathParameters": null,
    "queryStringParameters": null,
    "headers": {
      "accept": "*/*",
      "content-type": "application/json",
      "Host": "{$API_ID}.execute-api.us-east-1.amazonaws.com",
      "User-Agent": "curl/8.5.0",
      "X-Amzn-Trace-Id": "Root=1-68106501-6dd9506f01c1329f40e329b2",
      "X-Forwarded-For": "XX.YY.ZZ.AA",
      "X-Forwarded-Port": "443",
      "X-Forwarded-Proto": "https"
    },
    "body": {
      "name": "てつてつ",
      "email": "test@example.com",
      "q1": "はい",
      "q2": "いいえ"}
    }
}
```

#### 2.6.2.Lambdaのログの確認
- submit-survey関数の CloudWatchLogs画面で以下のログが取得できていることが確認できる
- 受信イベント：`event`すべてのログの取得
- パスパラメータ：今回は存在していないため`null`
- クエリパラメータ：今回は存在していないため`null`
- ヘッダー情報：`一般的なヘッダー内容`
- 特定のヘッダー情報：今回は３つ`content-Type`,`Authorization`,`X-Custom-Header`取得
- リクエストボディ：※Unicodeエスケープシーケンス
- リクエストボディ詳細： `名前`,`メール`,`質問`を取得

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/6822b869-462b-40ca-8368-e6c9c2e0ad19.png)


## 3.おわりに
### 3.1.得られた知見
- API Gatewayプロキシ統合では、リクエストの全てが **event** という一つの辞書オブジェクトとしてLambdaに渡される
- パスパラメータは **event['pathParameters']**、クエリパラメータは **event['queryStringParameters']**、ヘッダー情報は**event['headers']** に含まれる
- **json.dumps() **を使ってログに出力
- リクエストボディは文字列として渡されるため **json.loads()** が必要

### 3.2.今後の課題
- パスパラメータをAPI Gatewayで設定し、それを利用する具体的なユースケースを実装。
- APIキーや認証・認可（Lambdaオーソライザーなど）の仕組みと、その際のヘッダー情報の利用。
