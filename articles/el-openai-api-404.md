---
title: "OpenAI API の 404 エラー：原因と解決策"
emoji: "🚫"
type: "tech"
topics: ["openai-api", "error"]
published: true
---

## エラーの概要

OpenAI APIで404エラーが発生する場合、リクエストで指定されたリソース（モデル、アシスタント、ファイルなど）がサーバー上に存在しないことを示しています。このエラーは、存在しないモデル名の指定、削除済みのアシスタントIDの使用、間違ったエンドポイントへのアクセスなど、様々な原因で発生します。OpenAI APIの仕様変更に伴い、廃止されたモデルの使用も404の一般的な原因です。

## 実際のエラーメッセージ例

```json
{
  "error": {
    "message": "The model 'gpt-3.5-turbo-0301' does not exist. You can view a list of available models at https://platform.openai.com/docs/models.",
    "type": "invalid_request_error",
    "param": "model",
    "code": "model_not_found"
  }
}
```

```json
{
  "error": {
    "message": "Could not locate assistant with id 'asst_xxxxxxxxxxxxx'",
    "type": "invalid_request_error",
    "param": "assistant_id",
    "code": "assistant_not_found"
  }
}
```

## よくある原因と解決手順

### 原因1：廃止または存在しないモデル名の指定

OpenAIは定期的にモデルを更新し、古いスナップショット版（例：`gpt-3.5-turbo-0301`）を廃止しています。廃止されたモデル名を指定すると404エラーが返されます。

**Before：**
```python
import openai

response = openai.ChatCompletion.create(
    model="gpt-3.5-turbo-0301",
    messages=[
        {"role": "user", "content": "Hello"}
    ]
)
```

**After：**
```python
import openai
from openai import OpenAI

client = OpenAI()

response = client.chat.completions.create(
    model="gpt-4-turbo",
    messages=[
        {"role": "user", "content": "Hello"}
    ]
)
```

現在のOpenAI APIでは、`gpt-4-turbo`、`gpt-4o`、`gpt-4o-mini` など、明確に指定されたモデル名を使用してください。利用可能なモデルの一覧は[OpenAI Models ページ](https://platform.openai.com/docs/models)で確認できます。

### 原因2：不正なアシスタントID、スレッドID、またはファイルID

Assistants APIを使用している場合、削除済みのアシスタントやスレッド、ファイルのIDを参照すると404が発生します。

**Before：**
```python
client = OpenAI()

# 削除されたアシスタントIDを使用
response = client.beta.threads.create(
    assistant_id="asst_oldidthatnoexists"
)
```

**After：**
```python
client = OpenAI()

# 有効なアシスタントIDを確認して使用
assistants = client.beta.assistants.list()
valid_assistant_id = assistants.data[0].id

response = client.beta.threads.create(
    assistant_id=valid_assistant_id
)
```

アシスタントやスレッドを削除した場合、保存されたIDは無効になります。必ず有効なIDを確認してから使用してください。

### 原因3：誤ったエンドポイントまたはパスの指定

APIキーは正しくても、存在しないエンドポイントにリクエストを送信すると404が返されます。特にカスタム実装やcurlコマンドでエンドポイントを直接指定する場合に発生しやすいです。

**Before：**
```bash
curl https://api.openai.com/v1/chat/complete \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

**After：**
```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

エンドポイントのパスを正確に確認してください。例えば `/chat/complete` ではなく `/chat/completions` です。

## OpenAI API固有の注意点

OpenAI APIでは複数のAPIバージョンと多様なリソースタイプが存在するため、404エラーの原因も多岐にわたります。

**Chat Completions API：** 最新のモデル名は定期的に更新されます。`gpt-3.5-turbo` は常に最新のスナップショット版を指します。特定のバージョンが必要な場合は、公式ドキュメントで現在のスナップショット版番号を確認してください。

**Assistants API：** `v=20240415` などのベータバージョンを指定する場合、古いバージョンではリソースが存在しない可能性があります。RequestHeaderで正しいバージョンを指定してください。

**Organization ID：** 複数のOrganizationに属している場合、`OpenAI(organization="<your-org-id>")` でOrganizationを明示的に指定しないと、アシスタントやファイルが見つからないことがあります。

```python
client = OpenAI(
    api_key="<your-api-key>",
    organization="<your-org-id>"
)
```

**ファイルとベクトルストア：** Files APIでアップロードしたファイルIDは、期限切れや削除で無効になります。必ず最新のファイルIDを確認してから使用してください。

## それでも解決しない場合

**利用可能なリソースを確認する：**
使用しているリソースが実際に存在するか確認してください。

```python
# 利用可能なモデル一覧
models = client.models.list()
for model in models.data:
    print(model.id)

# 作成済みアシスタント一覧
assistants = client.beta.assistants.list()
for assistant in assistants.data:
    print(f"{assistant.id}: {assistant.name}")
```

**APIバージョンとライブラリバージョンを確認する：**
`openai` パッケージを最新版に更新してください。`pip install --upgrade openai` を実行し、ライブラリが最新であることを確認します。

**公式ドキュメントを参照する：**
[OpenAI API Reference](https://platform.openai.com/docs/api-reference) で、使用しているエンドポイント、パラメータ、現在のモデル一覧を確認してください。

**OpenAI Community Forumで報告する：**
他に原因が考えられない場合は、詳細なエラーメッセージ、使用しているコード、APIキーの権限設定（Billing settings）を確認した上で、[OpenAI Community Discussions](https://community.openai.com/) で質問してください。

---

*免責事項：本記事の内容は、執筆時点の公開情報をもとに作成したものです。ソフトウェアの仕様は予告なく変更されることがあります。最新の情報は各ツールの公式サポートページをご確認ください。本記事の情報を利用した結果生じたいかなる損害についても、著者および運営者は責任を負いかねます。*