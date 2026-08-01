---
title: "GCP の 403 エラー：原因と解決策"
emoji: "🚫"
type: "tech"
topics: ["gcp", "error"]
published: true
---

:::message
本記事は技術エラー解説サイト [errorlog.jp](https://errorlog.jp/) からの転載です。最新の内容と関連エラーの一覧は元記事を参照してください。
元記事: https://errorlog.jp/posts/gcp_403/
:::

## 冒頭まとめ

GCP の 403 Forbidden は、身元は届いているが、その操作が許されていない状態を示します。認証が通っていない 401 とは段階が違います（[GCP の 401 の記事](https://errorlog.jp/posts/gcp_401/)）。

このエラーの扱いやすさは、応答に含まれる情報の多さにあります。公式文書によれば、コマンド行の道具や窓口が返す文言には、必要な権限の名前、操作しようとした対象、認証に使われた身元、エラーごとの識別子、そして原因を調べるための専用の URL が含まれます。つまり、どの権限が足りないかは推測する必要がありません。応答に書かれています。

原因についても、公式文書が4つに整理しています。必要な権限を持っていない場合、拒否の方針が権限の使用を妨げている場合、主体に対する境界の方針が対象を含んでいない場合、そして対象が存在しない場合です。

4つ目が重要です。対象が存在しない場合も、このエラーになります。実際、文言も「対象に対して権限が拒否されました（あるいは対象が存在しない可能性があります）」という形になっており、両方の可能性を含んだ書き方です。したがって、403 を受け取ったからといって、権限の問題とは限りません。名前の綴りを間違えているだけ、ということがあります。

区分の定義にも、このエラーが要求の妥当性や対象の存在を意味しない、と明記されています。

## エラーの概要

窓口からの応答は、次の形になります。

```json
{
  "error": {
    "code": 403,
    "message": "Permission 'storage.buckets.list' denied on resource (or it may not exist). Remediate access with this Troubleshooter URL or share it with your administrator - https://console.cloud.google.com/iam-admin/troubleshooter;errorId=<識別子> .",
    "details": [
      {
        "@type": "type.googleapis.com/google.rpc.ErrorInfo",
        "reason": "forbidden",
        "domain": "global",
        "metadata": {
          "error_info_id": "<識別子>",
          "permission": "storage.buckets.list"
        }
      }
    ]
  }
}
```

読むべきは `metadata` の中の `permission` です。ここに、不足している権限の名前がそのまま入ります。上の例なら、ファイル置き場の一覧を取得する権限です。

`message` に含まれる URL も見逃せません。これを開くと、なぜ拒否されたかを調べる画面に移動します。自分に調べる権限が無ければ、この URL を管理者に渡せば済みます。

コマンド行の道具からは、同じ内容が別の形で表示されます。

```text
ERROR: (gcloud.storage.buckets.list) PERMISSION_DENIED:
<メールアドレス> does not have storage.buckets.list access to the Google Cloud project.
Permission 'storage.buckets.list' denied on resource (or it may not exist).
This command is authenticated as <メールアドレス> which is the active account
specified by the [core/account] property.
- '@type': type.googleapis.com/google.rpc.ErrorInfo
  domain: storage.googleapis.com
  metadata:
    permission: storage.buckets.list
    error_info_id: <識別子>
  reason: IAM_PERMISSION_DENIED
```

こちらには、認証に使われた身元も明示されます。意図した身元で操作しているかを、その場で確認できます。

## まず最初に：permission と、認証されている身元を読む

第一に、`metadata` の `permission` を読みます。不足している権限の名前がここにあります。役割を当てずっぽうで足す必要はありません。

第二に、認証に使われている身元を確認します。コマンド行の道具なら文言に出ますし、そうでなければ確認できます。意図と違う身元で動いていることが、しばしばあります。

```bash
gcloud auth list
gcloud config list account
```

第三に、対象の名前が正しいかを確認します。前述のとおり、対象が存在しない場合も同じエラーになります。権限の設定を調べ始める前に、綴りとプロジェクトを確かめてください。

第四に、専用の URL を開きます。ここまでで原因が分からなければ、この画面が答えを持っています。

## よくある原因と解決手順

### 原因1：必要な権限を持っていない

最も多い形です。応答に名前が出ているので、その権限を含む役割を付与します。

**Before（役割を推測で足す）：**

```bash
gcloud projects add-iam-policy-binding <プロジェクト> \
  --member=user:<メールアドレス> --role=roles/editor
# → 過剰な権限を与えることになり、しかも直らない場合がある
```

**After（不足している権限を含む役割を選ぶ）：**

```bash
# 応答に出ていた権限を含む役割を探す
gcloud iam roles list --filter="includedPermissions:storage.buckets.list" \
  --format="table(name, title)"

# 見つかった役割を付与する
gcloud projects add-iam-policy-binding <プロジェクト> \
  --member=user:<メールアドレス> --role=roles/storage.objectViewer
```

いま自分がどの権限を持っているかは、対象ごとに確認できます。

```bash
gcloud projects test-iam-permissions <プロジェクト> \
  --permissions=storage.buckets.list,storage.objects.get
```

結果に出た権限だけが、現在与えられているものです。

### 原因2：対象が存在しない

公式文書が挙げている4つ目の原因です。文言自体が「あるいは対象が存在しない可能性があります」と述べているとおり、権限の問題と区別が付かない形で返ります。

これは、対象の存在を秘匿するための設計です。権限の無い者に「存在するが見せない」と答えると、存在自体が漏れてしまいます。

**Before（権限の設定を疑い続ける）：**

```bash
gcloud storage ls gs://my-bucket-typo/
# → 403 が返るので IAM を調べ始める
```

**After（まず名前とプロジェクトを確認する）：**

```bash
gcloud config get-value project
gcloud storage buckets list --format="value(name)" | grep my-bucket
```

一覧に出てこなければ、綴りが違うか、別のプロジェクトにあります。権限の設定を調べるのは、対象の存在を確認したあとです。

### 原因3：拒否の方針や境界の方針に妨げられている

公式文書が挙げている2つ目と3つ目です。役割で権限を与えていても、拒否の方針が上書きしている場合があります。また、主体に対する境界の方針が設定されている場合、その方針が対象を含んでいなければ操作できません。

役割を足しても直らない場合は、この2つを疑います。

```bash
# 拒否の方針を確認する
gcloud iam policies list --attachment-point=<対象> --kind=denypolicies

# 適用されている方針を確認する
gcloud resource-manager org-policies list --project=<プロジェクト>
```

この形は、自分では解決できないことが多い種類です。専用の URL を管理者に渡すのが、最も早い進め方です。公式文書にも、管理者と共有できる形で URL が示されると書かれています。

### 原因4：呼び出し先の窓口が有効になっていない

サービス自体が使える状態になっていない場合です。この場合、権限は正しくても要求が通りません。文言に、その窓口がプロジェクトで使われていない、あるいは無効化されている旨と、有効化のための URL が入ります。識別子は `SERVICE_DISABLED`、古い形式の応答では `accessNotConfigured` です。

```bash
gcloud services list --enabled | grep <サービス>
gcloud services enable <サービス>.googleapis.com
```

有効化した直後は、反映まで数分待つ必要があります。文言にもその旨が書かれています。また、窓口を有効にする操作自体にも権限が要るため、その段階で再び 403 になることがあります。その場合は、プロジェクトの管理者に依頼します。

### 原因5：量の超過を権限の問題と誤解している

区分の定義には、資源を使い切ったことによる拒否にこの区分を使ってはならず、その場合は量の超過の区分を使うこと、と明記されています。原則として、割り当ての上限に達した場合は 403 ではなく 429 です。

ただし、例外が公式に文書化されています。公式のクォータのトラブルシューティングによれば、Compute Engine のクォータ超過は通常 403 で返り、識別子は `QUOTA_EXCEEDED`、レートクォータの場合は `RATE_LIMIT_EXCEEDED` になります。したがって Compute Engine の 403 では、権限を調べる前に識別子を確認してください。`QUOTA_EXCEEDED` や `RATE_LIMIT_EXCEEDED` であれば、権限ではなく量の問題です。

それ以外のサービスでは、`status` の値で判断できます。`PERMISSION_DENIED` なら権限、`RESOURCE_EXHAUSTED` なら量の問題です（[GCP の 429 の記事](https://errorlog.jp/posts/gcp_429/)）。

## 補足：似ているが別のもの

認証が通っていない場合は 401 です。区分の定義に、呼び出し元を特定できない場合に 403 の区分を使ってはならないと明記されています。したがって 403 が返っている時点で、身元は届いています（[GCP の 401 の記事](https://errorlog.jp/posts/gcp_401/)）。

送った内容そのものに問題がある場合は 400 で、区分が3つに分かれます（[GCP の 400 の記事](https://errorlog.jp/posts/gcp_400/)）。対象が見つからない場合は 404 になることもあります。403 と 404 のどちらが返るかは、対象の存在を秘匿する必要があるかどうかで変わります（[GCP の 404 の記事](https://errorlog.jp/posts/gcp_404/)）。

処理が内部で失敗した場合は 500、一時的に処理できない場合は 503 です（[GCP の 500 の記事](https://errorlog.jp/posts/gcp_500/)、[503 の記事](https://errorlog.jp/posts/gcp_503/)）。

## 切り分けの順序

1. `metadata` の `permission` を読む。不足している権限の名前がここにある。
2. 認証に使われている身元を確認する。意図した身元か。
3. 対象の名前とプロジェクトを確認する。存在しない場合も同じエラーになる。
4. その権限を含む役割を探して付与する。役割を推測で足さない。
5. 役割を足しても直らないなら、拒否の方針や境界の方針を疑う。
6. `status` が `RESOURCE_EXHAUSTED` なら量の問題。Compute Engine のクォータ超過は 403 のまま `QUOTA_EXCEEDED` や `RATE_LIMIT_EXCEEDED` で現れる。
7. 自分で解決できない場合は、応答に含まれる専用の URL を管理者に渡す。

## 確認コマンド集

```bash
# 1. 不足している権限と識別子を取り出す
curl -sS -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  "https://<サービス>.googleapis.com/v1/<資源>" | python3 -c "
import json,sys
d=json.load(sys.stdin)['error']
print(d['code'], d['status'])
for x in d.get('details', []):
    if x['@type'].endswith('ErrorInfo'):
        m=x.get('metadata', {})
        print('  permission:', m.get('permission'))
        print('  error_info_id:', m.get('error_info_id'))
        print('  reason:', x.get('reason'), '/ domain:', x.get('domain'))
"

# 2. いまどの身元で操作しているかを確認する
gcloud auth list
gcloud config list account project

# 3. 自分が持っている権限を確認する
gcloud projects test-iam-permissions <プロジェクト> \
  --permissions=<権限1>,<権限2>

# 4. その権限を含む役割を探す
gcloud iam roles list --filter="includedPermissions:<権限>" \
  --format="table(name, title)"

# 5. 対象が実在するかを確認する（存在しない場合も403になる）
gcloud storage buckets list --format="value(name)" | grep <名前>

# 6. 監査の記録から拒否された要求を抽出する
gcloud logging read \
  'protoPayload.status.code=7 AND severity>=ERROR' \
  --limit=20 \
  --format="value(protoPayload.authenticationInfo.principalEmail, protoPayload.methodName, protoPayload.status.message)"
```

## Editor's Note

403 の応答に含まれる識別子と専用の URL は、このエラーの扱いを大きく変えます。

従来、権限の問題を管理者に相談するときは、何をしようとして何が返ってきたかを説明する必要がありました。説明の過程で情報が落ちると、管理者の側でも再現できません。いまは、URL を1つ渡せば、管理者はその画面で、どの主体がどの対象に対してどの権限を求め、どの方針で拒否されたかを確認できます。公式文書にも、自分に管理の権限が無い場合は管理者と共有できる、と書かれています。

文言をそのまま信じると原因を外す、という実例もあります。Cloud SQL の利用者組での相談（[googleapi: Error 403: Access Not Configured](https://groups.google.com/g/google-cloud-sql-discuss/c/yoGQzTRXaOk/m/sBJq8mumCgAJ)）では、別プロジェクトの Compute Engine から Cloud SQL に接続しようとして、Cloud SQL Administration API が使われていない旨の 403（`accessNotConfigured`）が返り続けました。報告者は API を有効化済みで、伝播も待っていました。回答で示された実際の原因は、プログラムが使っていた身元です。Compute Engine の既定のサービスアカウントは、既定では使える API の範囲が絞られており、認証情報ファイルを明示するか、インスタンスの役割とアクセスの範囲を直すことで解決しています。403 の文言が指す先と、実際の原因の層が別だったわけです。本記事の順序——文言を読み、次に身元を確認する——は、この種のずれを拾うためにあります。

もう1つ、公式文書が挙げる4つの原因の並びも示唆的です。権限の不足、拒否の方針、境界の方針、そして対象の不存在。最後の1つだけが、権限とは無関係です。にもかかわらず同じエラーで返るのは、対象の存在を秘匿するためです。

この設計は、調べる側にとっては厄介です。しかし、文言が「あるいは対象が存在しない可能性があります」と併記していることには意味があります。書いてあるとおり、両方を疑うべきだという指示です。権限の設定を30分調べたあとで名前の綴りに気付く、という事態は、この一文を読んでいれば避けられます。

403 は、応答に答えが書かれている珍しいエラーです。`permission` を読む。身元を確認する。名前を確認する。この3つを先に済ませてください。

---

*免責事項：本記事の内容は、執筆時点の公開情報をもとに作成したものです。ソフトウェアの仕様は予告なく変更されることがあります。最新の情報は各ツールの公式サポートページをご確認ください。本記事の情報を利用した結果生じたいかなる損害についても、著者および運営者は責任を負いかねます。*