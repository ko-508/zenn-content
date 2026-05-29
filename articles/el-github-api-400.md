---
title: "GitHub API の 400 エラー：原因と解決策"
emoji: "🐙"
type: "tech"
topics: ["github-api", "error"]
published: true
---

## エラーの概要

GitHub APIの400エラーは「Bad Request」を意味し、クライアント側の送信内容に問題があることを示します。リクエストの形式が不正だったり、必須パラメータが不足していたり、パラメータ値が無効だったりするときに発生します。このエラーが発生した場合、サーバー側に問題があるのではなく、APIへの呼び出し方を見直す必要があります。

## 実際のエラーメッセージ例

GitHub APIが返す実際の400エラーレスポンスは以下のようなものです。

```json
{
  "message": "Problems parsing JSON",
  "documentation_url": "https://docs.github.com/rest"
}
```

あるいはバリデーションエラーの場合：

```json
{
  "message": "Validation Failed",
  "errors": [
    {
      "message": "Required field \"title\" is missing",
      "field": "title",
      "code": "missing_field"
    }
  ],
  "documentation_url": "https://docs.github.com/rest/reference/issues#create-an-issue"
}
```

## よくある原因と解決手順

### 原因1：JSONの形式が不正である

**なぜ発生するか**
リクエストボディをJSON形式で送信する際に、ダブルクォートの閉じ忘れやカンマの不足など、JSON形式として正しくない構文を送ってしまった場合に発生します。

**Before（エラーが起きる例）**
```bash
curl -X POST https://api.github.com/repos/<owner>/<repo>/issues \
  -H "Authorization: token <your-token>" \
  -H "Content-Type: application/json" \
  -d '{"title": "New issue" "body": "This is a test"}'
```

上記の例では、`"New issue"`と`"body"`の間にカンマがありません。

**After（修正後）**
```bash
curl -X POST https://api.github.com/repos/<owner>/<repo>/issues \
  -H "Authorization: token <your-token>" \
  -H "Content-Type: application/json" \
  -d '{"title": "New issue", "body": "This is a test"}'
```

### 原因2：必須パラメータが不足している

**なぜ発生するか**
APIエンドポイントが要求する必須パラメータをリクエストに含めていない場合に発生します。例えば、IssueやPull Requestの作成時に`title`が必須なのに省略した場合などです。

**Before（エラーが起きる例）**
```javascript
const octokit = require('@octokit/rest')({
  auth: '<your-token>'
});

await octokit.rest.issues.create({
  owner: '<owner>',
  repo: '<repo>',
  // title フィールドが省略されている
  body: 'Issue description'
});
```

**After（修正後）**
```javascript
const octokit = require('@octokit/rest')({
  auth: '<your-token>'
});

await octokit.rest.issues.create({
  owner: '<owner>',
  repo: '<repo>',
  title: 'Issue Title',  // 必須フィールドを追加
  body: 'Issue description'
});
```

### 原因3：パラメータ値が無効な形式である

**なぜ発生するか**
パラメータ値の型や値そのものが、APIが期待する形式と異なっている場合に発生します。例えば、数値として解釈されるべき値に文字列を渡した場合や、指定可能な値（enum）の範囲外の値を渡した場合などです。

**Before（エラーが起きる例）**
```bash
curl -X GET "https://api.github.com/repos/<owner>/<repo>/issues?state=opened" \
  -H "Authorization: token <your-token>"
```

`state`パラメータの値は「open」「closed」「all」のいずれかであるべきなのに、「opened」という無効な値を指定しています。

**After（修正後）**
```bash
curl -X GET "https://api.github.com/repos/<owner>/<repo>/issues?state=open" \
  -H "Authorization: token <your-token>"
```

### 原因4：Content-Typeヘッダーが不正である

**なぜ発生するか**
リクエストボディを送信する際に、`Content-Type`ヘッダーを指定していなかったり、間違った値を指定していたりすると、サーバーが正しくボディをパースできずエラーになります。

**Before（エラーが起きる例）**
```bash
curl -X POST https://api.github.com/repos/<owner>/<repo>/issues \
  -H "Authorization: token <your-token>" \
  -d '{"title": "New issue", "body": "Test"}'
```

Content-Typeヘッダーを指定していません。

**After（修正後）**
```bash
curl -X POST https://api.github.com/repos/<owner>/<repo>/issues \
  -H "Authorization: token <your-token>" \
  -H "Content-Type: application/json" \
  -d '{"title": "New issue", "body": "Test"}'
```

## GitHub API固有の注意点

GitHub APIには複数のバージョンが存在し、エンドポイントの仕様がバージョンによって異なります。`Accept`ヘッダーでAPI バージョンを指定する場合、不正なバージョン番号を指定すると400エラーが返されます。

公式ドキュメントでは常に最新のREST API仕様が提供されているため、使用しているAPI バージョンと実装コードが一致しているかを確認してください。特にGraphQL APIとREST APIを混同しないことが重要です。

また、レート制限（Rate Limiting）による429エラーと異なり、400エラーはリクエスト形式自体の問題のため、リトライアは効果がありません。むしろリクエスト形式を修正することに注力すべきです。

特定のエンドポイント（例：Pull Request レビューコメントの作成）では、パラメータの組み合わせに対して厳密なバリデーションが行われるため、公式ドキュメントのパラメータ説明を隅々まで読むことが不可欠です。

## それでも解決しない場合

まずは、リクエストの内容をPythonの`json`モジュールやオンラインのJSON検証ツール（JSONlint等）を使って、形式が正しいかを検証してください。

```bash
echo '{"title": "Test", "body": "Body"}' | python3 -m json.tool
```

次に、GitHub APIの公式ドキュメント内の「REST API reference」から、使用しているエンドポイントのページを開き、必須パラメータと各パラメータの型・制約を改めて確認してください。

それでも解決しない場合は、GitHub's Community Forum（discussions.github.com）またはGitHub Support に問い合わせることをお勧めします。リクエストの実例を示すことで、より具体的なアドバイスが得られます。

---

*免責事項：本記事の内容は、執筆時点の公開情報をもとに作成したものです。ソフトウェアの仕様は予告なく変更されることがあります。最新の情報は各ツールの公式サポートページをご確認ください。本記事の情報を利用した結果生じたいかなる損害についても、著者および運営者は責任を負いかねます。*