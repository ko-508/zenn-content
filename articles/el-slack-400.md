---
title: "Slack の 400 エラー：原因と解決策"
emoji: "💬"
type: "tech"
topics: ["slack", "error"]
published: false
---

## エラーの概要

Slackの400エラーは、Slack APIへのリクエストが不正な形式で送信されたか、必須のパラメータが不足していることを示します。クライアントアプリケーション、ボット、Webhookからの連携時に頻出するエラーで、リクエスト自体がサーバーに拒否される状態です。データは破損しないため、設定やリクエスト内容を修正すれば解決できます。

## 実際のエラーメッセージ例

Slack APIから返されるエラーレスポンスの例：

```json
{
  "ok": false,
  "error": "invalid_arg_name",
  "provided": "converstion_id"
}
```

Webhookの場合の典型的なエラー応答：

```json
{
  "ok": false,
  "error": "no_text",
  "needed": "text"
}
```

## よくある原因と解決手順

### 原因1：必須パラメータの欠落

**なぜ発生するか**  
Slack APIのエンドポイントごとに異なる必須パラメータが定められています。これらが欠けているとAPIが400を返します。例えばメッセージ投稿には「channel」と「text」が必須です。

**Before（エラーが起きる場合）**
```python
import requests

token = "<your-slack-bot-token>"
headers = {"Authorization": f"Bearer {token}"}

# textパラメータが欠落している
data = {
    "channel": "C123456789"
}

response = requests.post(
    "https://slack.com/api/chat.postMessage",
    headers=headers,
    json=data
)
print(response.json())  # {"ok": false, "error": "no_text"}
```

**After（修正後）**
```python
import requests

token = "<your-slack-bot-token>"
headers = {"Authorization": f"Bearer {token}"}

# textパラメータを追加
data = {
    "channel": "C123456789",
    "text": "Hello from my bot"
}

response = requests.post(
    "https://slack.com/api/chat.postMessage",
    headers=headers,
    json=data
)
print(response.json())  # {"ok": true, "ts": "1234567890.000001"}
```

### 原因2：Content-Typeヘッダーの誤り

**なぜ発生するか**  
Slack APIはリクエストボディの形式をContent-Typeで判定します。application/jsonを指定すべきところにapplication/x-www-form-urlencodedで送信すると、パラメータ解析に失敗します。

**Before（エラーが起きる場合）**
```bash
curl -X POST https://slack.com/api/chat.postMessage \
  -H "Authorization: Bearer <your-slack-bot-token>" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d '{"channel":"C123456789","text":"test"}'
# 400エラーが返される
```

**After（修正後）**
```bash
curl -X POST https://slack.com/api/chat.postMessage \
  -H "Authorization: Bearer <your-slack-bot-token>" \
  -H "Content-Type: application/json" \
  -d '{"channel":"C123456789","text":"test"}'
# 正常にメッセージが投稿される
```

### 原因3：無効なパラメータ値

**なぜ発生するか**  
チャネルIDやユーザーIDの形式が不正、またはタイムスタンプの値が無効な場合、APIは400を返します。特にチャネルID（Cで始まる）とユーザーID（Uで始まる）を混同することが多いです。

**Before（エラーが起きる場合）**
```javascript
const axios = require('axios');

const config = {
  headers: {
    'Authorization': `Bearer ${process.env.SLACK_TOKEN}`,
    'Content-Type': 'application/json'
  }
};

// チャネルIDが不正な形式
const data = {
  channel: "invalid_channel_name",
  text: "Hello"
};

axios.post('https://slack.com/api/chat.postMessage', data, config)
  .catch(err => console.log(err.response.data));
  // {"ok": false, "error": "channel_not_found"}
```

**After（修正後）**
```javascript
const axios = require('axios');

const config = {
  headers: {
    'Authorization': `Bearer ${process.env.SLACK_TOKEN}`,
    'Content-Type': 'application/json'
  }
};

// 正しいチャネルID形式（Cで始まる11文字）
const data = {
  channel: "C123ABC456",
  text: "Hello"
};

axios.post('https://slack.com/api/chat.postMessage', data, config)
  .then(res => console.log(res.data));
  // {"ok": true, ...}
```

## ツール固有の注意点

### Slack Appのトークン検証

Slack Appから発行されたトークンには有効期限と権限範囲があります。トークンが無効な場合も400が返されることがあります。トークンは環境変数で管理し、定期的に更新してください。

### Incoming Webhookの形式

Incoming Webhookを使用する場合、ペイロードはJSON形式である必要があります。特にレガシーなWebhook統合では、パラメータの大文字小文字の区別が厳密です。

```json
{
  "text": "Message text",
  "channel": "#general",
  "username": "MyBot"
}
```

### リクエスト署名の検証（ボット受信側）

Slackからのリクエストがボット側に到達する際、署名検証が必要です。署名が無効な場合、ボットが400を返すことがあります。リクエストヘッダの「X-Slack-Request-Timestamp」と「X-Slack-Signature」を正確に検証してください。

## それでも解決しない場合

**ログの確認場所**  
ボット実装の場合、アプリケーションログとSlack API呼び出しのレスポンスボディを確認してください。より詳しいエラーメッセージが含まれています。

**デバッグコマンド**  
```bash
# curlでAPIエンドポイントをテスト
curl -X POST https://slack.com/api/chat.postMessage \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"channel":"<channel_id>","text":"test"}' \
  -v
```

**公式ドキュメント参照**  
- [Slack API Documentation](https://api.slack.com/docs) の「API Methods」セクションで該当エンドポイントの必須パラメータを確認
- [Slack API Errors](https://api.slack.com/apis/rate-limits) でエラーコード一覧を確認
- [Incoming Webhooks](https://api.slack.com/messaging/webhooks) でWebhook固有の要件を確認

**コミュニティリソース**  
Slack Developer Community (community.slack.com) で同様の事例を検索するか、GitHub Issues内でSlack SDK（python-slack-sdk等）のトラブルシューティングを参照してください。

---

*免責事項：本記事の内容は、執筆時点の公開情報をもとに作成したものです。ソフトウェアの仕様は予告なく変更されることがあります。最新の情報は各ツールの公式サポートページをご確認ください。本記事の情報を利用した結果生じたいかなる損害についても、著者および運営者は責任を負いかねます。*