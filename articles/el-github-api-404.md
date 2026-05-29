---
title: "GitHub API の 404 エラー：原因と解決策"
emoji: "🐙"
type: "tech"
topics: ["github-api", "error"]
published: true
---

## エラーの概要

GitHub APIで404エラーが返される場合、リクエストで指定したリソース（リポジトリ、ユーザー、プルリクエストなど）がサーバー上に存在しないことを示します。このエラーはGitHub APIの認証が成功している場合でも発生し、エンドポイントのURLやパラメータの誤りが主な原因となります。

## 実際のエラーメッセージ例

```json
{
  "message": "Not Found",
  "documentation_url": "https://docs.github.com/rest/reference/repos#get-a-repository",
  "status": 404
}
```

```json
{
  "message": "Validation Failed",
  "errors": [
    {
      "message": "The listed users and repositories cannot be searched either because the resources do not exist or you do not have permission to view them.",
      "resource": "Search",
      "field": "q",
      "code": "invalid"
    }
  ],
  "documentation_url": "https://docs.github.com/rest/reference/search"
}
```

## よくある原因と解決手順

### 原因1：リポジトリ名またはオーナー名のスペルミス

APIエンドポイントで指定したリポジトリ名やオーナー名に誤りがあると、サーバーがそのリソースを検索できず404が返されます。

**Before（エラーが起きるコード）:**
```bash
curl -H "Authorization: token <your-github-token>" \
  https://api.github.com/repos/microsoft/vscode/contents/LICENCE.md
```

この例は、実際の`LICENSE.md`ファイルを`LICENCE.md`（英国式スペル）と誤って指定しているため、404が返されます。

**After（修正後）:**
```bash
curl -H "Authorization: token <your-github-token>" \
  https://api.github.com/repos/microsoft/vscode/contents/LICENSE.md
```

### 原因2：プライベートリポジトリへのアクセス権限不足

プライベートリポジトリにアクセスする際、認証トークンが不足していたり、有効期限が切れていたり、そのリポジトリへのアクセス権限がないと404が返されます。GitHub APIは権限がない場合、存在しないふりをする設計になっています。

**Before（認証情報なし）:**
```python
import requests

response = requests.get(
    "https://api.github.com/repos/myorg/private-repo"
)
print(response.status_code)  # 404
```

**After（認証情報付き）:**
```python
import requests

headers = {
    "Authorization": "token <your-github-token>",
    "Accept": "application/vnd.github.v3+json"
}

response = requests.get(
    "https://api.github.com/repos/myorg/private-repo",
    headers=headers
)
print(response.status_code)  # 200
```

### 原因3：APIのバージョンやエンドポイント形式の変更

GitHub APIのバージョン更新により、エンドポイント形式が変わった場合や、非推奨となったエンドポイントを使用している場合に404が返されます。

**Before（古いエンドポイント形式）:**
```bash
curl -H "Authorization: token <your-github-token>" \
  https://api.github.com/repos/<owner>/<repo>/pulls/<number>/reviews
```

**After（現在推奨されるエンドポイント）:**
```bash
curl -H "Authorization: token <your-github-token>" \
  "https://api.github.com/repos/<owner>/<repo>/pulls/<number>/reviews" \
  -H "Accept: application/vnd.github.v3+json"
```

### 原因4：パスパラメータの欠落またはフォーマット誤り

ブランチ名、タグ名、ファイルパスなどを含むエンドポイントでは、URLエンコードが必要な場合があります。スペースや特殊文字が含まれる場合、正しくエンコードしないと404が返されます。

**Before（エンコード不足）:**
```bash
curl -H "Authorization: token <your-github-token>" \
  "https://api.github.com/repos/myorg/myrepo/contents/path/my file.txt"
```

**After（URLエンコード済み）:**
```bash
curl -H "Authorization: token <your-github-token>" \
  "https://api.github.com/repos/myorg/myrepo/contents/path/my%20file.txt"
```

## ツール固有の注意点

### GitHub REST APIとGraphQL APIの違い

GitHub APIには2つのタイプがあり、404の原因や対応方法が異なります。

REST APIでは、エンドポイントのパス形式が厳密です。例えば、以下は異なるリソースを指しており、どちらかが存在しなければ404になります。

```bash
# プルリクエスト取得
https://api.github.com/repos/<owner>/<repo>/pulls/<number>

# プルリクエストレビュー取得
https://api.github.com/repos/<owner>/<repo>/pulls/<number>/reviews
```

GraphQL APIを使う場合は、クエリの構造が異なり、404ではなく異なる形式のエラーレスポンスが返される可能性があります。

### 組織とチームの権限確認

公開リポジトリであっても、組織の設定により特定のエンドポイント（例：`/orgs/<org>/members`）が非公開の場合、権限がないユーザーには404が返されます。適切なスコープを持つPersonal Access Tokenを使用してください。

### リリース・タグ・ブランチの存在確認

タグやブランチ名を指定するエンドポイント（例：`/repos/<owner>/<repo>/contents/<path>?ref=<branch>`）では、指定した参照が存在しなければ404が返されます。以下のコマンドで先に存在確認を行いましょう。

```bash
# ブランチ一覧確認
curl -H "Authorization: token <your-github-token>" \
  "https://api.github.com/repos/<owner>/<repo>/branches"

# タグ一覧確認
curl -H "Authorization: token <your-github-token>" \
  "https://api.github.com/repos/<owner>/<repo>/tags"
```

## それでも解決しない場合

### 確認すべきポイントとデバッグ手順

1. **トークンの有効性確認**：以下のコマンドでトークンが有効かつ正しいスコープを持つか確認します。

```bash
curl -H "Authorization: token <your-github-token>" \
  https://api.github.com/user
```

2. **リソースの実在確認**：ブラウザでGitHub.comにログインし、対象のリポジトリ・ユーザー・ファイルが本当に存在するか目視確認してください。

3. **APIレスポンスヘッダーの確認**：以下のコマンドで詳細情報を取得します。

```bash
curl -i -H "Authorization: token <your-github-token>" \
  "https://api.github.com/repos/<owner>/<repo>"
```

### 公式ドキュメント参照

- **GitHub REST API ドキュメント**：https://docs.github.com/rest
- **トラブルシューティングガイド**：https://docs.github.com/rest/guides/troubleshooting
- **認証ガイド**：https://docs.github.com/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens

### コミュニティリソース

問題が解決しない場合は、以下を参照してください：

- **GitHub Community Forum**：https://github.com/orgs/community/discussions
- **GitHub API関連のStack Overflow**：[github-api タグ付き質問](https://stackoverflow.com/questions/tagged/github-api)
- **GitHub Status Page**：https://www.githubstatus.com （APIの障害有無確認）

---

*免責事項：本記事の内容は、執筆時点の公開情報をもとに作成したものです。ソフトウェアの仕様は予告なく変更されることがあります。最新の情報は各ツールの公式サポートページをご確認ください。本記事の情報を利用した結果生じたいかなる損害についても、著者および運営者は責任を負いかねます。*