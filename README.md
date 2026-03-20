# qiita-articles

Qiita記事をGitHubで管理し、pushするだけで自動的にQiitaへ投稿・更新するリポジトリです。

## 仕組み

- [qiita-cli](https://github.com/increments/qiita-cli) を使用しています
- `public/` フォルダ内のMarkdownファイルだけがQiitaの記事として扱われます
- **README.mdやそれ以外のファイルはQiitaに投稿されません**
- `main` ブランチにpushすると、GitHub Actionsが自動で `public/` 内の記事をQiitaに投稿・更新します

## 記事の作成方法

### 1. 新しい記事ファイルを作成

```bash
cd ~/qiita-articles
npx qiita new 記事のファイル名
```

`public/記事のファイル名.md` が生成されます。

### 2. 記事を編集

生成されたファイルを開き、フロントマター（先頭の `---` で囲まれた部分）と本文を編集します。

```markdown
---
title: 記事のタイトル
tags:
  - AWS
  - Lambda
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

ここに本文を書く
```

#### フロントマターの説明

| 項目 | 説明 |
|---|---|
| `title` | 記事のタイトル |
| `tags` | タグ（最大5つ） |
| `private` | `true` で限定公開、`false` で全体公開 |
| `id` | Qiita側の記事ID（初回投稿後に自動で入る。手動で変更しない） |
| `ignorePublish` | `true` にするとpushしてもQiitaに投稿されない（下書き用） |
| `slide` | `true` でスライドモード |

### 3. pushして公開

```bash
git add .
git commit -m "記事を追加"
git push origin main
```

これだけでQiitaに自動投稿されます。

## 既存のQiita記事を取り込む

既にQiitaに投稿済みの記事をこのリポジトリに取り込むには：

```bash
npx qiita login
npx qiita pull
```

`public/` に既存記事がダウンロードされます。以降はこのリポジトリで管理できます。

## ローカルプレビュー

投稿前にブラウザでプレビューできます：

```bash
npx qiita preview
```

`http://localhost:8888` でプレビューが表示されます。

## ファイル構成

```
qiita-articles/
├── .github/workflows/publish.yml  # 自動投稿用のGitHub Actions
├── public/                        # Qiita記事（このフォルダだけが投稿対象）
│   └── my-article.md
├── qiita.config.json              # qiita-cliの設定
├── package.json
└── README.md                      # このファイル（Qiitaには投稿されない）
```
