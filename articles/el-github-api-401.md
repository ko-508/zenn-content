---
title: "GitHub API の 401 エラー：原因と解決策"
emoji: "🐙"
type: "tech"
topics: ["github-api", "error"]
published: true
---

## エラーの概要

GitHub APIで 401 Unauthorizedエラーが発生するのは、リクエストに対する認証が失敗した場合です。このエラーは、認証情報が完全に欠落している、形式が正しくない、または無効な状態を示しています。GitHub APIを呼び出すときに最も頻繁に遭遇するエラーの一つであり、適切な認証情報を提供することで解決できます。

## 実際のエラーメッセージ例

curlコマンドで認証情報なしでGitHub APIにアクセスした場合：

```bash
$ curl https://api.github.com/user
{
  "message": "Requires authentication",
  "documentation_url": "https://docs.github.com/rest/reference/users#get-the-authenticated-user"
}
```

PythonのrequestsライブラリでPersonal Access Token（PAT）が無効な場合：

```json
{
  "message": "Bad credentials",
  "documentation_url": "https://docs.github.com/rest"
}
```

## よくある原因と解決手順

### 原因1：Personal Access Token（PAT）が無効または期限切れ

GitHub APIの認証に使用するPATが期限切れになったり、削除されたりすると401エラーが発生します。PATには最大100年の有効期限を設定できますが、明示的に有効期限を設定している場合は有効期限の管理が必要です。

**Before（エラーが起きるコード）:**

```bash
# 3年前に作成した期限切れのPATを使用
$ curl -H "Authorization: token ghp_xxxxxxxxxxxxxxxxxxxxxxxxxx" \
  https://api.github.com/user
# 401 Unauthorized
```

**After（修正後）:**

新しいPATを生成します。GitHubの設定画面で「Settings > Developer settings > Personal access tokens」に移動し、「Generate new token」をクリックします。必要なscopeを選択（通常は`repo`と`user`）し、新しいトークンを生成してください。

```bash
# 新しく生成したPATを使用
$ curl -H "Authorization: token ghp_yyyyyyyyyyyyyyyyyyyyyyyyyyyy" \
  https://api.github.com/user
```

### 原因2：Authorizationヘッダーの形式が正しくない

GitHub APIは特定の形式でAuthorizationヘッダーを受け取ります。`Bearer`ではなく`token`キーワードを使用する必要があります。また、ヘッダー名や値のスペース配置のミスも401エラーの原因になります。

**Before（エラーが起きるコード）:**

```bash
# 間違った形式1：Bearerを使用
curl -H "Authorization: Bearer ghp_xxxxxxxxxxxxxxxxxxxxxxxxxx" \
  https://api.github.com/user

# 間違った形式2：スペースが足りない
curl -H "Authorization:token ghp_xxxxxxxxxxxxxxxxxxxxxxxxxx" \
  https://api.github.com/user

# 間違った形式3：トークンをクォートで囲んでいる
curl -H 'Authorization: token "ghp_xxxxxxxxxxxxxxxxxxxxxxxxxx"' \
  https://api.github.com/user
```

**After（修正後）:**

```bash
# 正しい形式：tokenキーワード＋スペース＋トークン
curl -H "Authorization: token ghp_xxxxxxxxxxxxxxxxxxxxxxxxxx" \
  https://api.github.com/user

# Pythonでの正しい例
import requests
headers = {
    "Authorization": "token ghp_xxxxxxxxxxxxxxxxxxxxxxxxxx",
    "Accept": "application/vnd.github.v3+json"
}
response = requests.get("https://api.github.com/user", headers=headers)
```

### 原因3：トークンのscope不足

生成したPATのscopeが制限されていると、特定のエンドポイントにアクセスするときに401エラーが発生します。例えば、`repo`スコープなしではプライベートリポジトリへのアクセスができません。

**Before（エラーが起きる設定）:**

```bash
# `repo`スコープなしのPATで、プライベートリポジトリにアクセス
curl -H "Authorization: token ghp_xxxx_publicscope_only" \
  https://api.github.com/repos/<owner>/<private-repo>
# 401 Unauthorized
```

**After（修正後）:**

GitHubの「Settings > Developer settings > Personal access tokens」で既存トークンを選択し、必要なscopeを追加します。または新しいトークンを生成する際に適切なscopeを選択してください。

```bash
# 適切なスcopeを持つPATで同じリクエスト
curl -H "Authorization: token ghp_xxxx_with_repo_scope" \
  https://api.github.com/repos/<owner>/<private-repo>
```

### 原因4：環境変数またはコンフィグの認証情報がセットされていない

GitHub CLIやGit自体を使用している場合、環境変数や`~/.gitconfig`ファイルに認証情報が設定されていないと401エラーが発生します。

**Before（エラーが起きる状態）:**

```bash
# GitHub CLIが認証されていない状態
$ gh api user
# Error: HTTP 401: Requires authentication
```

**After（修正後）:**

```bash
# GitHub CLIで認証
$ gh auth login
# プロンプトに従い、GitHub.comを選択、
# HTTPS protocolを選択、PATを入力してログイン

# その後、APIコマンドが使用可能に
$ gh api user
```

## ツール固有の注意点

### GitHub APIのバージョン指定

GitHubは複数のAPIバージョンをサポートしており、古いバージョンへのリクエストは認証要件が異なる場合があります。REST API v3を使用する際は、`Accept`ヘッダーで明示的にバージョンを指定することが推奨されます。

```bash
curl -H "Authorization: token <PAT>" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/user
```

### GraphQL APIの認証

GitHub GraphQL APIは、Authorizationヘッダーの形式がREST APIと同じですが、エンドポイントが異なります。GraphQL API使用時も同じPATが有効ですが、リクエスト形式に注意が必要です。

```bash
curl -X POST \
  -H "Authorization: token <PAT>" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ viewer { login } }"}' \
  https://api.github.com/graphql
```

### Organization内でのAPI利用

OrganizationのプライベートリポジトリやOrganizationメンバーとしてのアクセスが必要な場合、PAT生成時に`read:org`スコープを追加する必要があります。

## それでも解決しない場合

### 確認すべき手順

1. **トークンの有効性確認**：以下のコマンドで現在のトークンが有効か確認してください。

```bash
curl -H "Authorization: token <PAT>" https://api.github.com/user
```

2. **トークンのscope確認**：トークン生成時に付与されたscopeを確認してください。

```bash
curl -H "Authorization: token <PAT>" https://api.github.com/ \
  | grep -i "x-oauth-scopes"
```

3. **ネットワークの確認**：プロキシやファイアウォール経由でGitHubに接続している場合、リクエストが正しく転送されているか確認してください。

### 公式ドキュメント参照

- [GitHub REST API authentication](https://docs.github.com/rest/authentication)：認証方法の詳細
- [Creating a personal access token](https://docs.github.com/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)：PAT作成ガイド
- [GitHub API Rate Limits](https://docs.github.com/rest/rate-limit)：レート制限関連情報

### デバッグのコツ

詳細なレスポンスを確認するには、verbose モードでcurlを実行してください。

```bash
curl -v -H "Authorization: token <PAT>" https://api.github.com/user
```

GitHub APIの応答ヘッダーに含まれる`X-RateLimit-*`や`X-GitHub-Request-Id`といった情報は、GitHub Supportへの問い合わせ時に役立ちます。これらの情報を記録しておくと、問題解決が効率的になります。

---

*免責事項：本記事の内容は、執筆時点の公開情報をもとに作成したものです。ソフトウェアの仕様は予告なく変更されることがあります。最新の情報は各ツールの公式サポートページをご確認ください。本記事の情報を利用した結果生じたいかなる損害についても、著者および運営者は責任を負いかねます。*