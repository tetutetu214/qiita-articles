---
title: AWS StepFunctions入門 サーバーレスワークフロー ハンズオン
tags:
  - stepfunctions
private: false
updated_at: '2025-07-05T08:58:07+09:00'
id: 90c7ebc98bac2aaca567
organization_url_name: null
slide: false
ignorePublish: false
---
## 1.はじめに
### 1.1.背景
StepFunctionsの構築をしたことがなかったので、主にStepFunctionsの記述をメインにサーバーレスワークフローの基本をハンズオンして学んでいこうと思います。

今回作成した[GitHub](https://github.com/tetutetu214/20250515_stepfunctions_lambda_handson)へのリンク

## 2.ハンズオン
### 2.1.前提
#### 2.1.1.実行環境
- 本記事のバージョンはあくまで執筆時点のものです。<br>互換性の問題が発生した場合は、適宜読み替えて実行ください。

|項目|設定|
|---|---|
|環境|AWS CloudShell|
|リージョン|バージニア北部(us-east-1)|
|AWS CLI|2.27.8|
|Python|3.9.21|

#### 2.1.2.本ブログのアーキテクチャ
- 今回は簡単なLambda関数2つと、StepFunctionsの組み合わせの構築を実施します。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/11d248d9-d0b5-46c7-b07c-800362ba2cda.png)

#### 2.1.3.ファイル構成
```shell
stepfunctions_lambda_handson/
├── README.md
├── lambda/
│   ├── process_data.py  # データ処理Lambda関数
│   └── notify_result.py # 結果通知Lambda関数
├── statemachine/
│   └── workflow.json    # Step Functions定義
└── template.yaml        # AWS SAMテンプレート
```

### 2.2.データ処理 Lambda関数作成
```shell
# プロジェクトファイル作成
mkdir stepfunctions_lambda_handson && cd stepfunctions_lambda_handson 

# ファイル作成
mkdir -p lambda statemachine

# データ処理を行うLambda関数作成
cat > lambda/process_data.py << EOF
"""
データ処理を行うLambda関数

この関数は入力データを受け取り、簡単な処理を行った後、結果を返します。
"""
import json
import random
import time

def lambda_handler(event, context):
    """
    Lambda関数のエントリーポイント
    
    Parameters:
    event (dict): 入力イベント
    context (LambdaContext): Lambda実行コンテキスト
    
    Returns:
    dict: 処理結果
    """
    print("受信したイベント:", json.dumps(event))
    
    # 入力データを取得
    input_data = event.get('data', {})
    user_id = input_data.get('user_id', 'unknown')
    
    # 処理時間をシミュレート（0.5〜2秒）
    processing_time = random.uniform(0.5, 2.0)
    time.sleep(processing_time)
    
    # 処理結果を生成
    result = {
        'user_id': user_id,
        'processed': True,
        'processing_time': processing_time,
        'timestamp': time.time(),
        'score': random.randint(1, 100)
    }
    
    print(f"処理完了: {json.dumps(result)}")
    
    # 次のステップに渡すデータを返す
    return {
        'status': 'success',
        'result': result
    }
EOF
```
### 2.3.処理結果を通知するLambda関数
```shell

cat > lambda/notify_result.py << EOF
"""
処理結果を通知するLambda関数

この関数は処理結果を受け取り、通知を行ったことをシミュレートします。
実際のプロダクション環境では、SNS、SQS、メール送信などの実装を行います。
"""
import json
import time

def lambda_handler(event, context):
    """
    Lambda関数のエントリーポイント
    
    Parameters:
    event (dict): 入力イベント（前のLambda関数からの出力）
    context (LambdaContext): Lambda実行コンテキスト
    
    Returns:
    dict: 通知結果
    """
    print("受信したイベント:", json.dumps(event))
    
    # 前のステップの結果を取得
    previous_result = event.get('result', {})
    user_id = previous_result.get('user_id', 'unknown')
    score = previous_result.get('score', 0)
    
    # 通知処理をシミュレート
    time.sleep(1)
    
    # スコアに基づいてメッセージを生成
    if score >= 80:
        message = f"おめでとうございます！ユーザー {user_id} のスコアは {score} で優秀です！"
    elif score >= 50:
        message = f"ユーザー {user_id} のスコアは {score} で平均以上です。"
    else:
        message = f"ユーザー {user_id} のスコアは {score} です。改善の余地があります。"
    
    # 通知結果
    notification_result = {
        'user_id': user_id,
        'message': message,
        'notification_sent': True,
        'timestamp': time.time()
    }
    
    print(f"通知完了: {json.dumps(notification_result, ensure_ascii=False)}")
    
    return {
        'status': 'success',
        'notification': notification_result
    }
EOF
```

### 2.4.StepFunctions作成
#### 2.4.1.ステートマシンの基本構造

|項番|項目|説明|
|---|---|---|
|1|Comment|ステートマシンの説明（オプション）|
|2|StatAt|最初に実行するステートの名前|
|3|States|ステートマシンに含まれる全てのステートを定義|

```shell
{
  "Comment": "データ処理と通知のワークフロー",
  "StartAt": "ProcessData",
  "States": {
    // 各ステートの定義
  }
}
```

#### 2.4.2.今回利用されている主なステートの種類

|フィールド|対象ステート|説明|
|---|---|---|
|Type|全般|ステートの種類 (例: Task, Choice, Fail, Succeed)|
|Resource|Task|	実行するリソース (Lambda関数のARNなど)|
|Next|Task, (Choice内)|次に実行するステートの名前|
|Retry|Task|エラー発生時の再試行ポリシー|
|Choices|Choice|評価する条件のリスト|
|Variable|(Choices内)|評価する変数 (JSONパス形式)|
|StringEquals|(Choices内)|文字列比較条件|
|Default|Choice|どのChoicesにも一致しなかった場合の遷移先|
|Cause|Fail|失敗の理由（エラーメッセージ）|
|Error|Fail|エラーコード|


#### 2.4.3.ハンズオンの続き
```shell
cat > statemachine/workflow.json << EOF
{
  "Comment": "データ処理と通知のワークフロー",
  "StartAt": "ProcessData",
  "States": {
    "ProcessData": {
      "Type": "Task",
      "Resource": "${ProcessDataFunctionArn}",
      "Next": "CheckProcessingResult",
      "Retry": [
        {
          "ErrorEquals": ["States.ALL"],
          "IntervalSeconds": 1,
          "MaxAttempts": 2,
          "BackoffRate": 2.0
        }
      ]
    },
    "CheckProcessingResult": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.status",
          "StringEquals": "success",
          "Next": "NotifyResult"
        }
      ],
      "Default": "ProcessingFailed"
    },
    "NotifyResult": {
      "Type": "Task",
      "Resource": "${NotifyResultFunctionArn}",
      "Next": "WorkflowComplete",
      "Retry": [
        {
          "ErrorEquals": ["States.ALL"],
          "IntervalSeconds": 1,
          "MaxAttempts": 2,
          "BackoffRate": 2.0
        }
      ]
    },
    "ProcessingFailed": {
      "Type": "Fail",
      "Cause": "データ処理に失敗しました",
      "Error": "ProcessingError"
    },
    "WorkflowComplete": {
      "Type": "Succeed"
    }
  }
}
EOF
```

### 2.5.AWS SAMテンプレート作成
```shell
cat > template.yaml << EOF
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Step Functions と Lambda を使用したシンプルなワークフロー

Resources:
  # データ処理Lambda関数
  ProcessDataFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: lambda/
      Handler: process_data.lambda_handler
      Runtime: python3.9
      Timeout: 10
      MemorySize: 128
      Policies:
        - AWSLambdaBasicExecutionRole

  # 結果通知Lambda関数
  NotifyResultFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: lambda/
      Handler: notify_result.lambda_handler
      Runtime: python3.9
      Timeout: 10
      MemorySize: 128
      Policies:
        - AWSLambdaBasicExecutionRole

  # Step Functions ステートマシン
  DataProcessingWorkflow:
    Type: AWS::Serverless::StateMachine
    Properties:
      Definition:
        Comment: "データ処理と通知のワークフロー"
        StartAt: "ProcessData"
        States:
          ProcessData:
            Type: "Task"
            Resource: !GetAtt ProcessDataFunction.Arn
            Next: "CheckProcessingResult"
            Retry:
              - ErrorEquals: ["States.ALL"]
                IntervalSeconds: 1
                MaxAttempts: 2
                BackoffRate: 2.0
          CheckProcessingResult:
            Type: "Choice"
            Choices:
              - Variable: "$.status"
                StringEquals: "success"
                Next: "NotifyResult"
            Default: "ProcessingFailed"
          NotifyResult:
            Type: "Task"
            Resource: !GetAtt NotifyResultFunction.Arn
            Next: "WorkflowComplete"
            Retry:
              - ErrorEquals: ["States.ALL"]
                IntervalSeconds: 1
                MaxAttempts: 2
                BackoffRate: 2.0
          ProcessingFailed:
            Type: "Fail"
            Cause: "データ処理に失敗しました"
            Error: "ProcessingError"
          WorkflowComplete:
            Type: "Succeed"
      Policies:
        - LambdaInvokePolicy:
            FunctionName: !Ref ProcessDataFunction
        - LambdaInvokePolicy:
            FunctionName: !Ref NotifyResultFunction

Outputs:
  ProcessDataFunction:
    Description: "データ処理Lambda関数のARN"
    Value: !GetAtt ProcessDataFunction.Arn

  NotifyResultFunction:
    Description: "結果通知Lambda関数のARN"
    Value: !GetAtt NotifyResultFunction.Arn

  StateMachineArn:
    Description: "Step Functions ステートマシンのARN"
    Value: !GetAtt DataProcessingWorkflow.Arn
EOF
```

### 2.6.ローカルでLambda関数をテストするスクリプト

```shell
cat > test_lambda_local.py << EOF
"""
ローカルでLambda関数をテストするためのスクリプト
JSONファイルを引数として受け取ります
"""
import json
import sys
import os

# Lambdaディレクトリをパスに追加
sys.path.append(os.path.join(os.path.dirname(__file__), 'lambda'))

# Lambda関数をインポート
import process_data
import notify_result

def main():
    # コマンドライン引数をチェック
    if len(sys.argv) < 2:
        print("使用方法: python test_lambda_local.py <入力JSONファイル>")
        print("例: python test_lambda_local.py test_input.json")
        sys.exit(1)
    
    # 入力JSONファイルを読み込む
    # 引数の2つ目(test_input.json)を変数格納
    input_file = sys.argv[1]
    try:
        # input_fileを読み込みモード(r)で開く
        with open(input_file, 'r') as f:
            # jsonファイルをPythonのオブジェクトに変換
            test_input = json.load(f)
    except Exception as e:
        print(f"エラー: JSONファイルの読み込みに失敗しました - {e}")
        sys.exit(1)
        
    print("Step Functions + Lambda ローカルテスト\n")
    print(f"入力ファイル: {input_file}")
    
    print("1. ProcessData Lambda関数を実行:")
    print(f"入力: {json.dumps(test_input, indent=2, ensure_ascii=False)}")
    
    # ProcessData Lambda関数を実行
    process_result = process_data.lambda_handler(test_input, None)
    print(f"出力: {json.dumps(process_result, indent=2)}\n")
    
    print("2. NotifyResult Lambda関数を実行:")
    print(f"入力: {json.dumps(process_result, indent=2, ensure_ascii=False)}")
    
    # NotifyResult Lambda関数を実行
    notify_result_output = notify_result.lambda_handler(process_result, None)
    print(f"出力: {json.dumps(notify_result_output, indent=2, ensure_ascii=False)}\n")
    
    print("ワークフロー実行完了!")

if __name__ == "__main__":
    main()
EOF
```
### 2.7.テスト用 JSONファイル
```shell
cat > test_input.json << EOF
{
  "data": {
    "user_id": "user123",
    "timestamp": 1620000000
  }
}
EOF
```

### 2.8.ローカルテストの実行
#### 2.8.1.実行コマンド
- `Lambda`関数の単体テストを実施
- StepFunctionsのステートマシンの動作確認ではありません

```shell
# ローカルテストスクリプトの実行
python test_lambda_local.py test_input.json
```

#### 2.8.2.レスポンス
- Lambdaが実行され、各Lambdaの値が受け渡されていることが確認できる

```shell
Step Functions + Lambda ローカルテスト

(中略)

出力: {
  "status": "success",
  "notification": {
    "user_id": "user123",
    "message": "ユーザー user123 のスコアは 71 で平均以上です。",
    "notification_sent": true,
    "timestamp": 1747313229.2919128
  }
}

ワークフロー実行完了!
```

### 2.9.ビルドとデプロイ
#### 2.9.1.SAMによるビルド
- ビルドが成功すると`Build Succeeded`が表示される

```shell
# samコマンドによるビルド
sam build
```
#### 2.9.2.SAMによるデプロイ
- 以下コマンドを実施するとウィザード形式で質問される
- デプロイコマンド後 `Successfully created/updated stack - stepfunctions-lambda-handson in us-east-1` が表示される。

■ 質問への回答例

|項番|項目|項目(日本語)|回答内容|
|---|---|---|---|
|1|Stack Name [sam-app]|スタック名|stepfunctions-lambda-handson|
|2|AWS Region [us-east-1]|リージョン|us-east-1|
|3|Confirm changes before deploy [y/N]|デプロイを実行する前に、変更内容を確認しますか？|y|
|4|Allow SAM CLI IAM role creation [Y/n]|SAMがIAMロール作成を許可するか？|y|
|5|Disable rollback [y/N]|ロールバックを無効にするか？|n|
|6| ${関数名} Function has no authentication. Is this okay? [y/N]|(※各関数で聞かれる)認証が設定されていませんが、これでよいですか？|y|
|7|Save arguments to configuration file [Y/n]|設定ファイルに引数を保存しますか？|y|
|8|SAM configuration file [samconfig.toml]|SAM設定ファイルの名前|ENTER(デフォルトの「samconfig.toml」)|
|9|SAM configuration environment [default]|ファイル内で異なる環境（開発環境、テスト環境、本番環境など）の設定|ENTER(デフォルトの「default」)

```shell
# samコマンドによるデプロイ
sam deploy --guided
```

#### 2.9.3.デプロイして発生したエラー
- 問題：一度目のデプロイが失敗したため、二度目以降の更新ができない事象が発生しました。
- 原因：スタックが`ROLLBACK_COMPLETE`状態で残存しており、更新が出来ない。
- 解決策：残存しているスタックの削除後、再デプロイ

```shell
# まず失敗したスタックを削除開始
aws cloudformation delete-stack --stack-name stepfunctions-lambda-handson

# スタックが完全に削除されるまで待機(削除が完了まで次のステップに進めない)
aws cloudformation wait stack-delete-complete --stack-name stepfunctions-lambda-handson

# 削除完了したら再デプロイ
sam deploy --guided
```

### 2.10.挙動の確認
- デプロイが完了後、以下コマンドを実施する。

#### 2.10.1.StepFunctionsのARN取得
```sehll
# スタック名を設定
export STACK_NAME=stepfunctions-lambda-handson
export REGION=us-east-1

# StateMachineのARNを取得
export STATE_MACHINE_ARN=$(aws cloudformation describe-stacks --stack-name $STACK_NAME --query "Stacks[0].Outputs[?OutputKey=='StateMachineArn'].OutputValue" --output text --region $REGION)

echo $STATE_MACHINE_ARN
```

#### 2.10.2.CLIでの実行結果の確認
```shell
# Step Functionsの実行を開始
aws stepfunctions start-execution \
    --state-machine-arn $STATE_MACHINE_ARN \
    --input file://test_input.json \
    --region $REGION

# 上記実施後の参考レスポンス
{
    "executionArn": "arn:aws:states:us-east-1:123456789010:execution:DataProcessingWorkflow-XXXXX",
    "startDate": "2025-05-15T13:31:51.145000+00:00"
}
```

#### 2.10.3.AWSマネコンでの実行結果の確認
- AWS マネジメントコンソール > StepFunctions > ステートマシン一覧より「DataProcessingWorkflow-XXXXX」を選択
- 実行画面でステータスが`成功`していることを確認
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/9ceb6258-e505-4800-8689-f1571bdd5bb8.png)

- `名前`部分を押下して、グラフビューでも実行の処理を確認
- `状態の入力`と`状態の出力`で、ローカルで実行した出力がされていることも確認できる

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/6c786675-cd65-4a33-be43-9434283b2c99.png)


<br>

### 2.11.クリーンアップ 
- 以下コマンドで、本ハンズオンで作成したリソースを削除します。<br>不要な課金を避けるために、ハンズオン終了後は実行することを推奨します。
- 削除の際に以下2点が聞かれるが、どちらも`y`と答えることを推奨する。

|項番|項目|項目(日本語)|回答内容|
|---|---|---|---|
|1|Are you sure you want to delete the stack stepfunctions-lambda-handson in the region us-east-1 ? [y/N]|us-east-1リージョンのstepfunctions-lambda-handsonスタックを削除してもよいか?|y|
|2|Are you sure you want to delete the folder stepfunctions-lambda-handson in S3 which contains the artifacts? [y/N]|S3のstepfunctions-lambda-handsonフォルダを削除してもよいか？|y|

```shell
# 削除コマンド
sam delete
```

## 3.おわりに
### 3.1.得られた知見
- StepFunctionsの基本的な使い方を理解する。

### 3.2.今後の課題
- 今回はシンプルな構成でしたが、並列処理(Parallel)や待機(Wait)ステートを含めたより複雑なワークフロー構築に挑戦してみる。
- Bedrockなどの各リソースとの統合した、ワークフローの構築方法を学ぶ。
