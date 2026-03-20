---
title: pymupdfによるPDFデータ抽出ハンズオン
tags:
  - OCR
  - PyMuPDF
private: false
updated_at: '2025-06-29T10:00:28+09:00'
id: dbd03ed5efb3f1a9b31f
organization_url_name: null
slide: false
ignorePublish: false
---
## はじめに
お仕事で、生成AIを用いてPDF（非構造化データ）から構造化データの抽出に取り組んでいます。

従来のAI OCRでは、表組みや複雑なレイアウトを持つPDFからの情報抽出に課題がありました。そこで、Pythonのライブラリ「pymupdf」を導入し、生成AIが参照するヒントを増やすことで、抽出精度を向上させられるのではないかと考えました。

具体的には、「pymupdf」 でPDFからテキストのレイアウト情報を取得し、それを生成AIに提供することで、より文脈に即した回答を得られるのではないかと思ったので、まずは「pymupdf」導入・検証のためのハンズオンをしていきたいと思います。

## pymupdfとは？
PDF、XPS、EPUB、CBZなど、様々なドキュメント形式を利用することが出来るようになるPythonライブラリ。

内部では、C言語で書かれた高速なレンダリングエンジンであるMuPDFを使用しており、高速な処理が特徴。
テキスト抽出だけでなく、画像の抽出、PDFの編集、ページの結合・分割、注釈の追加など、多岐に渡る機能がある。


## ハンズオン

|項番|バージョン番号|
|---|---|
|1|Python 3.11.11|

### 1.環境構築
プロジェクトごとの依存関係を分離するため、仮想環境を作成
- `ProjectName`は、任意の名前を設定してください。


```shell
ProjectName = {任意のプロジェクト名}
mkdir ${ProjectName}
cd ${ProjectName}
python3 -m venv .venv
source .venv/bin/activate
```

### 2.ライブラリインストール
プロジェクトで使用するライブラリのインストール
#### 2.1.requirements.txtの中身
```shell
pymupdf
```

#### 2.2.インストール
ライブラリのインストール
```shell
pip install -r requirements.txt
```

### 3.app.pyの中身
PDFファイル読み込みアプリケーションのコード
- output.txt で、読み込み内容を出力
- `pdf_file = "myresume.pdf"`は、読み込みたいPDFファイルを読者で用意(※今回私は、自身の履歴書を利用)

```shell
import fitz
import json

def extract_text_from_pdf(pdf_path):
    """PDFファイルからテキストを抽出する関数 (dictを使用、詳細レイアウト処理)"""
    try:
        doc = fitz.open(pdf_path)
        all_text = ""

        for page_num, page in enumerate(doc):
            print(f"--- ページ {page_num + 1} の処理 ---")

            text_dict = page.get_text("dict")
            blocks = text_dict["blocks"]

            # ブロックをy座標でソート
            blocks.sort(key=lambda block: block["bbox"][1])

            for block in blocks:
                lines = block["lines"]

                # 行をy座標でソート
                lines.sort(key=lambda line: line["bbox"][1])

                for line in lines:
                    spans = line["spans"]

                    # スパンをx座標でソート
                    spans.sort(key=lambda span: span["bbox"][0])

                    line_text = ""
                    for span in spans:
                        line_text += span["text"]

                    all_text += line_text + "\n"

        return all_text.strip()

    except FileNotFoundError:
        print(f"エラー: PDFファイル '{pdf_path}' が見つかりません。")
        return None
    except fitz.FileDataError as e:
        print(f"エラー: PDFファイルが不正です: {e}")
        return None
    except Exception as e:
        print(f"エラー: PDF処理中に予期せぬエラーが発生しました: {e}")
        return None

if __name__ == "__main__":
    pdf_file = "myresume.pdf"
    extracted_text = extract_text_from_pdf(pdf_file)

    if extracted_text:
        with open("output.txt", "w", encoding="utf-8") as f:
            f.write(extracted_text)
        print("テキストを output.txt に保存しました。")
        # print("--- 抽出されたテキスト ---") # 必要に応じてコンソール出力
        # print(extracted_text)
    else:
        print("テキストを抽出できませんでした。")
```

### 4.出力(一部抜粋)
#### ①PDFデータ
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/172f9ff6-cb77-4fb0-b395-2af384d462c5.png)

#### ①実際に抽出出来た部分
```shell
基本情報
フリガナ
イナムラ　テッペイ
稲村　鉄平
氏名
```
#### ②PDFデータ
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/694365/471c0a04-e964-40ad-a903-715f2acaf32e.png)

#### ②実際に抽出出来た部分
```shell
現在所属するチームでは、DevOpsによるチーム文化醸成のための勉強会を主導していま
す。
その他に、所属チーム以外の会議にも参加して、他チームへの技術的提案の実施。
```

## おわりに
### 得られた知見
- get_text("dict") を使うことで、テキストのレイアウト情報を取得し、ある程度レイアウトを維持した状態でテキストを抽出できることが分かる

### 今後の課題
- 記号(■、●、・)は何処まで制度が出るのかの確認
- 抽出したテキストとレイアウト情報をJSON形式で生成AIに渡してプロンプトに組み込むことで、より文脈に即した回答を得られるか検証する
