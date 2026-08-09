---
title: "npm の ECONNRESET エラー：原因と解決策"
emoji: "🚫"
type: "tech"
topics: ["npm", "error"]
published: true
---

:::message
本記事は技術エラー解説サイト [errorlog.jp](https://errorlog.jp/) からの転載です。最新の内容と関連エラーの一覧は元記事を参照してください。
元記事: https://errorlog.jp/posts/npm_econnreset/
:::

## 結論

`npm error code ECONNRESET` は、接続が相手側から切られたという意味です。npm はこのコードを他のネットワーク系と同じ分岐で扱うため、固有の説明は出ません。表示されるのは「ネットワーク接続に関する問題である」「多くの場合は中継の設定かネットワーク設定に問題がある」という2文と、中継設定の確認を促す1文だけです。

つまり文言からは原因を絞れません。切り分けの材料は3つあります。どの回線で起きるか、失敗する対象が毎回同じか変わるか、そして失敗するまでの時間です。

npm は取得に失敗した場合、既定で2回まで再試行します。待ち時間は10秒から始まり、上限は1分です。これを踏まえると、実行してすぐ落ちる場合と、数十秒かけて落ちる場合では見るべき場所が違います。

## 最初に確認すること

まず、経路の設定を並べて確認します。

```bash
npm config get proxy
npm config get https-proxy
npm config get registry
npm config get maxsockets
```

`proxy` と `https-proxy` が `null` なのに社内の回線から実行している場合、原因1に当たります。環境変数側だけに設定されていることもあるため、そちらも見てください。

```bash
printenv | grep -i proxy
```

次に、失敗する対象が毎回同じかを確かめます。

```bash
npm install --loglevel verbose 2>&1 | tail -40
```

対象が実行ごとに変わるなら、特定のパッケージではなく接続の総量が問題です。同じ対象で止まるなら、その向き先が届いていません。

## 原因別の確認方法と解決策

### 原因1：中継の設定が npm に渡っていない {#proxy-not-configured}

社内の回線からのみ失敗する場合です。npm の説明文も、まずこの可能性を挙げます。

確認方法は設定値の突き合わせです。公式の説明によれば、`HTTP_PROXY` や `http_proxy` の環境変数が設定されていれば、その内容が利用されます。npm 側の設定と環境変数のどちらか一方だけに値が入っていると、経路が定まりません。

対処は、組織から指定されている中継先へ揃えることです。

```bash
npm config set proxy http://proxy.example.com:8080
npm config set https-proxy http://proxy.example.com:8080
```

中継先を通さない宛先がある場合は、除外する一覧も設定してください。

### 原因2：同時に張る接続が多すぎる {#concurrent-connection-limit}

依存の多い環境や継続的インテグレーションで起きます。失敗する対象が実行ごとに変わるのが特徴です。

npm は同じ向き先に対して既定で15本まで接続を張ります。経路上の機器がこれを過剰と判断すると、途中の接続が切られます。

確認方法は現在値の照会と、失敗対象の変化です。

```bash
npm config get maxsockets
```

対処は上限を下げることです。

```bash
npm install --maxsockets 5
```

改善するなら確定です。恒久的に設定する場合は、実行環境ごとの設定ファイルへ書いてください。

### 原因3：再試行が足りていない {#retry-exhausted}

時間をおくと成功する場合です。一時的な切断が、既定の再試行の範囲を超えて続いています。

確認方法は設定値と経過時間の照合です。

```bash
npm config get fetch-retries
npm config get fetch-retry-mintimeout
```

既定では2回まで再試行し、最初の待ち時間は10秒です。実行から十数秒で落ちているなら、再試行は使い切られています。

対処は回数と待ち時間を増やすことです。

```bash
npm install --fetch-retries 5 --fetch-retry-maxtimeout 120000
```

ただし、これは切断そのものを解消しません。原因1や原因2を確認したうえで、それでも断続的に切れる場合の緩和策として使ってください。

### 原因4：経路上の検査装置が接続を切っている {#tls-inspection-device}

組織の回線でのみ失敗し、証明書に関する警告が併せて出る場合です。通信を復号して検査する装置が経路にあると、npm 側の検証が通らず接続が切られます。

確認方法は、同じ経路で他の手段が通るかの比較です。

```bash
npm config get registry
```

対処は、組織が配布している証明書を登録することです。

```bash
npm config set cafile /path/to/corporate-ca.pem
```

`strict-ssl` を無効にすると通ることがありますが、これは検証そのものを止める操作です。公式の設定では既定で有効になっています。無効化は、なりすましを検知できなくなる点を理解したうえで、一時的な切り分けに限って使ってください。

## 近いエラーとの違い

`ENOTFOUND` は名前を引けなかった場合です。npm の実装では `ECONNRESET` と同じ分岐に入るため、表示される説明文は同一になります。区別できるのはコードの行だけです。接続が切られたのではなく、宛先が分からない状態を指します。

`ETIMEDOUT` は応答が返らないまま時間切れになった場合です。こちらも同じ分岐で、説明文は変わりません。切られたのか返ってこないのかで、疑う相手が変わります。

`CERT_HAS_EXPIRED` は証明書の期限切れです。接続は成立しており、検証の段階で止まっています。原因4と経路は近いものの、失敗する場所が違います。

`E401` はレジストリからの認証要求です。通信そのものは成立しています。

## 参考資料

- [npm config（proxy、maxsockets、fetch-retries、strict-ssl）](https://docs.npmjs.com/cli/latest/using-npm/config)
- [npm install](https://docs.npmjs.com/cli/latest/commands/npm-install)
- [エラー文言の生成（error-message.js）](https://github.com/npm/cli/blob/latest/lib/utils/error-message.js)
- [設定項目の定義（definitions.js）](https://github.com/npm/cli/blob/latest/workspaces/config/lib/definitions/definitions.js)

---

*免責事項：本記事の内容は、執筆時点の公開情報をもとに作成したものです。ソフトウェアの仕様は予告なく変更されることがあります。最新の情報は各ツールの公式サポートページをご確認ください。本記事の情報を利用した結果生じたいかなる損害についても、著者および運営者は責任を負いかねます。*