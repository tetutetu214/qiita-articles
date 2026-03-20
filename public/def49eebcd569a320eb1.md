---
title: Google Cloud Workflows 「default_retry」 に関する検証
tags:
  - GoogleCloud
  - CloudRun
  - workflows
private: false
updated_at: '2026-01-03T21:37:19+09:00'
id: def49eebcd569a320eb1
organization_url_name: null
slide: false
ignorePublish: false
---
## 1.はじめに
### 1.1.背景
WorkflowsからCloudRunを呼び出した際、想定外の `default_retry` によるリトライが発生しました。<br>「なにが原因でリトライしているのか？」と思ったことがきっかけで、今回の検証を行うことにしました。

前回ブログでの[Google Cloud Workflows 変数メモリ上限に関する検証](https://qiita.com/tetutetu214/items/7fe89b86ee0238f284f5)による、変数メモリの上限抵触によるリトライなのか、それとも別の問題でリトライが発生しているのか不明瞭だったため、実際に2種類のエラーを発生させて切り分けて検証を実施していきました。

### 1.2.公式ドキュメントの内容
#### 1.2.1.エラーキャッチするリトライの対象
公式ドキュメントより `default_retry` は以下２種類のエラーをキャッチすることでリトライを実行する。

|種別|対象|
|---|---|
|HTTPステータスコード|429, 502, 503, 504|
|エラータグ|ConnectionError, TimeoutError|

公式ドキュメント：[Workflows http.default_retry](https://docs.cloud.google.com/workflows/docs/reference/stdlib/http/default_retry)

#### 1.2.2.Workflowsの http.get設定値
Workflowsから `http.get` でCloudRunを呼び出す際の設定は以下の通り。
`timeout` を指定しない場合は、タイムアウトにデフォルトの300秒が適用される。

|設定項目|日本語名|設定値|
|---|---|---|
|url|リクエスト先URL|(必須)|
|timeout|タイムアウト秒数|300|
|headers|HTTPヘッダー|なし|
|query|クエリパラメータ|なし|
|auth|認証設定|なし|

公式ドキュメント：[Workflows http.get](https://docs.cloud.google.com/workflows/docs/reference/stdlib/http/get)

## 2.結論
- **原因**: 変数メモリ上限（`MemoryLimitExceededError`）ではなく、CloudRunの処理遅延による `TimeoutError` だった
- **理由**: `http.get` に `timeout` を指定していなかったため、デフォルトの300秒が適用され、それを超過した
- **挙動**: `TimeoutError` は `default_retry` の対象であるため、最大5回のリトライが実行された

## 3.検証内容
`default_retry` を設定した状態で、以下2種類のエラーを意図的に発生させ、リトライされるかどうかを確認する。
### 3.1.Case 1: TimeoutError の検証
Workflowsのタイムアウト時間を10秒に設定し、CloudRunの処理時間を30秒にすることで、わざと`TimeoutError`を発生させる。<br>そして`リトライ`が実行されることを検証する。

### 3.2.Case 2: MemoryLimitExceededError の検証
CloudRunのレスポンスを600KBにし、Workflowsの受け取れる上限にさせることで、わざと` MemoryLimitExceededError`を発生させる。<br>そして`リトライ`が実行されないことを検証する


## 4.検証に利用したコード
### 4.1.CloudRun
- 検証1用 /slow: 指定秒数待機してレスポンス返却（TimeoutError検証用）
- 検証2用 /generate: 指定サイズのJSONを生成して返却（MemoryLimitExceededError検証用）

<details>
<summary>コードの詳細</summary>

```python
import json
import os
import time

from flask import Flask, jsonify, request


app = Flask(__name__)

# 1KB = 1024バイト
BYTES_PER_KB = 1024

@app.route("/health", methods=["GET"])
def health():
    """ヘルスチェック用エンドポイント"""
    return jsonify({"status": "healthy"})


@app.route("/slow", methods=["GET"])
def slow():
    """
    指定秒数待機してからレスポンスを返却する。
    TimeoutError検証用エンドポイント。

    Query Parameters:
        delay: 待機秒数（デフォルト: 30秒）

    Returns:
        JSON: delay, message を含むレスポンス
    """
    #  クエリパラメータから待機秒数を取得（デフォルト: 30秒）
    delay_seconds = request.args.get("delay", default=30, type=int)

    # バリデーション： 0秒以下は不正
    if delay_seconds <= 0:
        return jsonify({"error": "delay must be positive"}), 400

    # バリデーション： 300秒以上は不正
    if delay_seconds > 300:
        return jsonify({"error": "delay must be 300 seconds or less"}), 400

    # 処理開始時刻を記録
    start_time = time.time()

    # 指定秒数待機
    time.sleep(delay_seconds)

    # 実際の経過時間を計算
    elapsed_time = time.time() - start_time

    # Workflowsへ値を返却
    return jsonify({
        "requested_delay_seconds": delay_seconds,
        "actual_elapsed_seconds": round(elapsed_time, 2),
        "message": "Response after delay",
    })


@app.route("/generate", methods=["GET"])
def generate():
    """
    指定サイズのJSONを生成して返却する。
    MemoryLimitExceededError検証用エンドポイント。

    Query Parameters:
        size: 生成するJSONのサイズ(KB)

    Returns:
        JSON: requested_size_kb, actual_size_bytes, padding を含むレスポンス
    """
    # クエリパラメータからサイズを取得    
    size_kb = request.args.get("size", type=int)

    # バリデーション: sizeパラメータは必須    
    if size_kb is None:
        return jsonify({"error": "size parameter is required"}), 400

    # バリデーション: サイズは正の値
    if size_kb <= 0:
        return jsonify({"error": "size must be positive"}), 400

    # バリデーション: サイズ上限（メモリ保護のため）
    if size_kb > 3000:
        return jsonify({"error": "size must be 3000KB or less"}), 400

    # 目標バイト数を計算
    target_bytes = size_kb * BYTES_PER_KB

    # ベースとなるレスポンス構造を作成
    
    # Step1: paddingが空の状態でサイズを計算
    # requested_size_kb：引数で与えられた値(確認用）
    # actual_size_bytes：最終的なJSONのサイズ(確認用、後で上書き）
    # padding：サイズ調整用のダミー文字列(後で"a"を詰める)
    base_response = {
        "requested_size_kb": size_kb,
        "actual_size_bytes": 0,
        "padding": "",
    }

    # Step2: ベースレスポンスのサイズを計算
    # 例: {"requested_size_kb": 400, "actual_size_bytes": 0, "padding": ""} → 約60バイト
    base_size = len(json.dumps(base_response))

    # Step3: paddingに必要なサイズを計算
    # 目標サイズからベースサイズ(Step2の値)を引いた分だけ"a"で埋める
    padding_size = target_bytes - base_size
    if padding_size < 0:
        padding_size = 0

    # Step4: ダミー文字列を生成
    padding = "a" * padding_size

    # Step5: 最終レスポンスを構築
    response = {
        "requested_size_kb": size_kb,
        "actual_size_bytes": target_bytes,
        "padding": padding,
    }

    # Step6: 実際のサイズを再計算して設定
    # Step3でpaddingサイズを計算したが、paddingを詰めると全体サイズが微妙にずれる可能性があるため再計算
    actual_size = len(json.dumps(response))
    response["actual_size_bytes"] = actual_size

    # Step7: Workflowsへ値を返却
    return jsonify(response)


if __name__ == "__main__":
    port = int(os.environ.get("PORT", 8080))
    app.run(host="0.0.0.0", port=port)

```
</details>


### 4.2.Workflows
#### 4.2.1.TimeoutError + default_retry 検証用ワークフロー
TimeoutErrorが発生した場合、default_retryによりリトライされることを確認する。
CloudRunは、実行完了するまで30秒かかる(sleepで待機)ようにし、<br> Workflows側は `http.get`のタイムアウトを10秒にすることでタイムアウトを疑似的に発生させる。

<details>
<summary>コードの詳細</summary>

```yaml
main:
  params: [args]
  steps:
    # 初期化: パラメータとCloudRun URLを設定
    - init:
        assign:
          - delay: ${default(map.get(args, "delay"), 30)}  # デフォルト30秒待機
          - cloudrun_url: ${sys.get_env("CLOUDRUN_URL")}   # 環境変数からURL取得

    # 処理開始ログ出力
    - log_start:
        call: sys.log
        args:
          data: ${"Starting timeout test with delay=" + string(delay) + "s, timeout=10s, retry=default_retry"}
          severity: "INFO"

    # CloudRun呼び出し（TimeoutError発生箇所）
    # timeout: 10秒 < delay: 30秒 のため、必ずTimeoutErrorが発生する
    - call_slow_api:
        try:
          call: http.get
          args:
            url: ${cloudrun_url + "/slow?delay=" + string(delay)}
            timeout: 10   # 10秒でタイムアウト
            auth:
              type: OIDC  # CloudRun認証
          result: api_response
        retry: ${http.default_retry}  # TimeoutErrorはリトライ対象

    # 成功時のレスポンス（TimeoutErrorが発生するため到達しない）
    - return_success:
        return:
          status: "success"
          response: ${api_response.body}
```
</details>

#### 4.2.2.MemoryLimitExceededError + default_retry 検証用ワークフロー
MemoryLimitExceededErrorが発生した場合、default_retryでもリトライされないことを確認する。
CloudRunから600KBのレスポンスを返却し、Workflows側の変数上限（512KB）を超過させることでMemoryLimitExceededErrorを発生させる。

<details>
<summary>コードの詳細</summary>

```yaml
main:
  params: [args]
  steps:
    # 初期化: パラメータとCloudRun URLを設定
    - init:
        assign:
          - size: ${default(map.get(args, "size"), 600)}  # デフォルト600KB（512KB超過）
          - cloudrun_url: ${sys.get_env("CLOUDRUN_URL")}  # 環境変数からURL取得

    # 処理開始ログ出力
    - log_start:
        call: sys.log
        args:
          data: ${"Starting memory test with size=" + string(size) + "KB, retry=default_retry"}
          severity: "INFO"

    # CloudRun呼び出し（MemoryLimitExceededError発生箇所）
    # 600KB > 512KB のため、必ずMemoryLimitExceededErrorが発生する
    - call_large_response_api:
        try:
          call: http.get
          args:
            url: ${cloudrun_url + "/generate?size=" + string(size)}
            auth:
              type: OIDC  # CloudRun認証
          result: api_response
        retry: ${http.default_retry}  # MemoryLimitExceededErrorはリトライ対象外

    # 成功時のレスポンス（MemoryLimitExceededErrorが発生するため到達しない）
    - return_success:
        return:
          status: "success"
          requested_size_kb: ${size}
          response_body_size: ${len(json.encode_to_string(api_response.body))}
```
</details>


## 5.検証結果
`default_retry`は、公式ドキュメントにもあるように`TimeoutError`タグを見てリトライ判定しており、<br>`MemoryLimitExceededError`はリトライ対象外となる。<br>よってリトライが実行された理由として **変数メモリの上限抵触** は関係ないことが判明する。

| Case | エラー | tags | 実行時間 | リトライ回数 | 結果 |
|---|---|---|---|---|---|
| 1 | TimeoutError | ["TimeoutError", "OSError"] | 76秒 | 5回 | リトライ実行 |
| 2 | MemoryLimitExceededError | ["MemoryLimitExceededError", "ResourceLimitError"] | 5秒 | 0回 | リトライなし |

### 5.1.Case 1 の実行時間について
Case 1の実行時間が76秒となっているのは、`default_retry` の指数バックオフが働いているため。
タイムアウト10秒 × 6回（初回＋リトライ5回）に加え、リトライ間のバックオフ待機時間（初期1秒、乗数1.25）が合算されている。

## 6.まとめ
`default_retry` は便利な機能ですが、リトライ対象となるエラーが限定されていることを理解した上で使用する必要があります。

今回の事象では `timeout` を明示的に設定していなかったため、デフォルトの300秒が適用され、CloudRunの処理時間超過により `TimeoutError` が発生した。その結果、想定外のリトライが実行されました。

最初は「変数メモリ上限のせいか」と思い込んでいたところもあったため、思い込みで原因を決めつけず検証で切り分けることの大切さを改めて実感しました。

### 6.1.今後の対策
今回の検証結果を踏まえ、意図しないリトライを防ぐために以下を考慮して設計する。

| 対策 | 内容 |
|---|---|
| タイムアウトの明示化 | `timeout` を処理想定時間に合わせて適切に設定する |
| リトライポリシーのカスタマイズ | `default_retry` に頼らず、必要に応じて `predicate` や `max_retries` を独自に設定する |
| べき等性の担保 | リトライが発生することを前提に、CloudRun側で二重処理が起きないよう設計する |
