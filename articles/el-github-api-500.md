---
title: "GitHub API の 500 エラー：原因と解決策"
emoji: "🐙"
type: "tech"
topics: ["github-api", "error"]
published: true
---

## エラーの概要

GitHub APIで500エラーが発生する場合、GitHub側のサーバーで予期しない内部エラーが発生していることを示します。クライアント側の設定ミスではなく、サーバー側の障害またはAPIの不具合が原因です。ただし、不正なリクエスト形式やタイムアウト、レート制限を超えた状態でも500が返されることがあり、実際には利用者側で対応可能な原因も含まれます。

## 実際のエラーメッセージ例

```json
{
  "message": "Internal Server Error",
  "documentation_url": "https://docs.github.com/rest"
}
```

curlコマンドでの表示例：

```bash
$ curl -H "Authorization: token <your-token>" https://api.github.com/repos/<owner>/<repo>/issues
HTTP/1.1 500 Internal Server Error
Server: GitHub.com
Content-Type: application/json; charset=utf-8
```

## よくある原因と解決手順

### 原因1：不正なJSONペイロード形式またはエンコーディングエラー

GitHubのAPIサーバーがリクエストボディを解析できない場合、500エラーで応答することがあります。特にJSONの形式が微妙に間違っていたり、文字エンコーディングが指定されていない場合に発生します。

**Before（エラーが起きる例）：**

```python
import requests

payload = {
    "title": "バグ報告",
    "body": "テスト\n改行"  # バイナリデータが含まれる
}
# Content-Type指定なしでPOST
response = requests.post(
    "https://api.github.com/repos/<owner>/<repo>/issues",
    headers={"Authorization": "token <your-token>"},
    data=payload  # json=ではなくdataを使用
)
```

**After（修正後）：**

```python
import requests
import json

payload = {
    "title": "バグ報告",
    "body": "テスト\n改行"
}
response = requests.post(
    "https://api.github.com/repos/<owner>/<repo>/issues",
    headers={
        "Authorization": "token <your-token>",
        "Accept": "application/vnd.github.v3+json",
        "Content-Type": "application/json; charset=utf-8"
    },
    json=payload  # 自動的にJSONエンコード
)
print(response.status_code)
```

### 原因2：APIバージョン指定の不適切またはヘッダーの不足

GitHubはREST APIのバージョンを指定するため、AcceptヘッダーやX-GitHub-Api-Versionヘッダーが必須です。これが正しく指定されないと、古いバージョンのエンドポイントにルーティングされ、予期しない形式のリクエストとして処理される結果500になります。

**Before（エラーが起きる例）：**

```bash
curl -H "Authorization: token <your-token>" \
  https://api.github.com/repos/<owner>/<repo>/pulls
# Acceptヘッダーがない
```

**After（修正後）：**

```bash
curl -H "Authorization: token <your-token>" \
  -H "Accept: application/vnd.github.v3+json" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  https://api.github.com/repos/<owner>/<repo>/pulls
```

### 原因3：レート制限またはレート制限リセット前の連続リクエスト

GitHubのAPIはレート制限（認証済みで1時間5000リクエスト）を設定しており、制限を超えた直後のリクエストが500として返されることがあります。また、バッチ処理で大量のリクエストを短時間で送信した場合も同様です。

**Before（エラーが起きる例）：**

```python
import requests

token = "<your-token>"
headers = {"Authorization": f"token {token}"}

# 1000個のリポジトリに対してループなし待機でアクセス
for i in range(1000):
    response = requests.get(
        f"https://api.github.com/repos/<owner>/repo-{i}",
        headers=headers
    )
    if response.status_code == 500:
        print(f"500 error at iteration {i}")
```

**After（修正後）：**

```python
import requests
import time

token = "<your-token>"
headers = {"Authorization": f"token {token}"}

# レート制限を確認しながら実行
for i in range(1000):
    response = requests.get(
        f"https://api.github.com/repos/<owner>/repo-{i}",
        headers=headers
    )
    
    # Rate-Limit情報を確認
    remaining = int(response.headers.get('X-RateLimit-Remaining', 0))
    reset = int(response.headers.get('X-RateLimit-Reset', 0))
    
    if remaining < 10:
        sleep_time = reset - int(time.time())
        if sleep_time > 0:
            print(f"Rate limit approaching. Waiting {sleep_time} seconds...")
            time.sleep(sleep_time + 1)
    
    if response.status_code == 500:
        # 指数バックオフで再試行
        time.sleep(2 ** i if i < 5 else 30)
        response = requests.get(...)
```

## GitHub API固有の注意点

**GraphQL APIの場合：** REST APIとは異なり、GraphQLのエラーレスポンスは200ステータスコードでbodyに`"errors"`フィールドを含む形式になります。500が返される場合はサーバー側の深刻な障害の可能性が高いです。

```json
{
  "errors": [
    {
      "message": "Something went wrong while executing your query",
      "locations": [{"line": 2, "column": 3}]
    }
  ]
}
```

**Personal Access Token（PAT）の権限不足：** トークンに必要なスコープがない場合、実装によっては500で応答することがあります。`repo`、`read:org`、`gist`など適切なスコープを設定してください。

**Webhook配信の失敗：** GitHubがWebhookペイロードを送信する際、受け取り側のエンドポイントが500を返すと、GitHubは自動的に再試行を実行します。受け取り側のサーバーログを確認し、実装の問題がないか検証してください。

**Rate Limit Headers の活用：** すべてのレスポンスに`X-RateLimit-Limit`、`X-RateLimit-Remaining`、`X-RateLimit-Reset`が含まれます。これらを監視することで、500エラーの多くは事前に防げます。

## それでも解決しない場合

1. **GitHub Status Page確認**：https://www.githubstatus.com/ でGitHubのシステムに障害がないか確認してください。

2. **ネットワークキャプチャ**：tcpdumpやWiresharkで実際のHTTPリクエスト/レスポンスを記録し、ペイロードのバイナリ形式を検証します。

```bash
tcpdump -i any -A 'tcp port 443' | grep -A 20 'POST /repos'
```

3. **公式ドキュメント参照**：https://docs.github.com/en/rest/guides/best-practices-for-using-the-rest-api のベストプラクティスセクションを確認してください。

4. **GitHub Support Contact**：継続的に500エラーが発生する場合、https://support.github.com でサポートチケットを作成し、リクエストIDを含めて報告してください。

5. **コミュニティリソース**：GitHub API関連のIssueはhttps://github.com/github-community/community/discussions で検索すると、既知の問題や回避策が見つかることがあります。

---

*免責事項：本記事の内容は、執筆時点の公開情報をもとに作成したものです。ソフトウェアの仕様は予告なく変更されることがあります。最新の情報は各ツールの公式サポートページをご確認ください。本記事の情報を利用した結果生じたいかなる損害についても、著者および運営者は責任を負いかねます。*