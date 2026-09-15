---
title: "PostgreSQL の relation does not exist：原因と解決策"
emoji: "🐘"
type: "tech"
topics: ["postgresql", "error"]
published: true
---

:::message
本記事は技術エラー解説サイト [errorlog.jp](https://errorlog.jp/) からの転載です。最新の内容と関連エラーの一覧は元記事を参照してください。
元記事: https://errorlog.jp/posts/postgresql_relation_does_not_exist/
:::

## 結論

`relation "users" does not exist` は、対象がこの世に無いという意味ではありません。指定した名前を、今の接続から解決できなかったという意味です。SQL の状態コードは 42P01、名称は undefined_table です。

relation はテーブルだけを指しません。公式ドキュメントは、pg_class がインデックス、シーケンス、ビュー、実体化ビューなども扱い、これらをまとめて relation と呼ぶと説明しています。

読み分けの起点は、文言の中に点があるかどうかです。スキーマ名を自分で書いた場合は `relation "public.users" does not exist` と点付きになり、書かなかった場合は `relation "users" does not exist` と名前だけになります。前者は指定したスキーマの中で見つからず、後者は検索経路をたどって見つからなかったという意味です。

そして、この文言は「無い」と「見えない」を区別しません。権限が足りずに検索経路から外れたスキーマも、存在しないスキーマも同じ結果です。

## 最初に確認すること

今の接続がどこを見ているかを確認します。

```sql
SELECT current_database(), current_user;
SHOW search_path;
```

初期状態の検索経路は `"$user", public` です。先頭は利用者名と同じスキーマを指します。

次に、対象の所在を調べます。

```sql
SELECT n.nspname, c.relname, c.relkind
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relname = 'users';
```

行が返らなければ、このデータベースには登録されていません。返るのに参照できないなら、スキーマか綴りの問題です。

## 原因別の確認方法と解決策

### 原因1：対象が検索経路に入っていないスキーマにある {#search-path-not-including-schema}

上の照会で `nspname` が `public` 以外になっている場合です。修飾しない参照は検索経路を順にたどり、最初に一致したものを使います。経路に入っていないスキーマの中身は参照できません。

対処は、呼び出し側で修飾するか経路へ加えるかです。

```sql
SELECT * FROM app.users;
SET search_path TO app, public;
```

`SET` はそのセッションの間だけ有効です。毎回同じ状態にしたい場合は、ロールやデータベースへ既定値を設定します。

### 原因2：スキーマへの USAGE 権限が無い {#schema-usage-not-granted}

同じクエリが、利用者を変えると成功する場合です。公式ドキュメントは、自分が所有していないスキーマの中身へ既定では触れられず、所有者が USAGE 権限を与える必要があると説明しています。

問題は見え方です。実装は検索経路を組み立てる段階で、名前を認識できないスキーマと読み取り権限が無いスキーマを一覧から外します。注記によれば、検索経路の設定自体はすでに受理されているため、ここではエラーにできないからです。

確認方法は2つあります。

```sql
SELECT has_schema_privilege(current_user, 'app', 'USAGE');
SELECT * FROM app.users;
```

下のようにスキーマ名を付けて実行すると、権限が原因なら文言が `permission denied for schema app` へ変わります。修飾したときだけ理由が表に出ます。

対処は権限の付与です。

```sql
GRANT USAGE ON SCHEMA app TO app_user;
```

### 原因3：作成した綴りと参照した綴りが違う {#quoted-identifier-case}

二重引用符で囲んで作成した名前を、囲まずに参照している場合です。公式ドキュメントは、囲まない名前は常に小文字に畳まれると説明しています。`FOO`、`foo`、`"foo"` は同じものですが、`"Foo"` と `"FOO"` はそれらとも互いとも別です。

確認方法は、登録されている綴りを見ることです。

```sql
SELECT relname FROM pg_class WHERE relname ILIKE 'users';
```

`Users` のように大文字が含まれていれば確定です。

対処は、参照する側も囲むか、名前を小文字へ変えるかです。

```sql
SELECT * FROM "Users";
ALTER TABLE "Users" RENAME TO users;
```

後者を選ぶ場合は、その名前を使うクライアント側もあわせて直します。

### 原因4：接続先が違う、または作られていない {#different-database-or-not-created}

`current_database()` の値が想定と違う場合です。接続文字列や環境変数が別のデータベースを指していると、サーバーとポートは合っているため接続だけ成功し、作成の手順はもう一方へ適用されています。

```sql
SELECT current_database();
```

接続先が合っているのに行が返らない場合は、まだ作られていません。作成の手順が失敗していないかログを確認してください。

## 近いエラーとの違い

`column "..." does not exist`（42703）は、対象そのものは見つかったうえで、その中のカラム名で失敗しています。

`permission denied for table ...`（42501）は、名前の解決が済んだあとの権限検査で止まっています。テーブル自体への権限が足りないだけなら、この記事の文言にはなりません。スキーマ側の USAGE が足りない場合だけ、修飾の有無で文言が入れ替わります。

`database "..." does not exist`（3D000）は、接続の段階で失敗しています。

`relation "..." does not exist, skipping` はエラーではなく通知です。`IF EXISTS` を付けた削除や変更で対象が無かったときに出ます。

同じ文言に `There is a WITH item named ...` という詳細が付く場合は別の状況です。`WITH` で定義した名前を、まだ参照できない位置から呼んでいます。`WITH RECURSIVE` を使うか並び順を変えるようにという助言が一緒に出ます。

## 参考資料

- [スキーマと検索パス（PostgreSQL 公式）](https://www.postgresql.org/docs/current/ddl-schemas.html)
- [識別子と大文字小文字の扱い（PostgreSQL 公式）](https://www.postgresql.org/docs/current/sql-syntax-lexical.html)
- [pg_class（PostgreSQL 公式）](https://www.postgresql.org/docs/current/catalog-pg-class.html)
- [エラーコード一覧（PostgreSQL 公式）](https://www.postgresql.org/docs/current/errcodes-appendix.html)
- [名前解決の実装（namespace.c）](https://github.com/postgres/postgres/blob/REL_18_STABLE/src/backend/catalog/namespace.c)
- [文言の生成箇所（parse_relation.c）](https://github.com/postgres/postgres/blob/REL_18_STABLE/src/backend/parser/parse_relation.c)

---

*免責事項：本記事の内容は、執筆時点の公開情報をもとに作成したものです。ソフトウェアの仕様は予告なく変更されることがあります。最新の情報は各ツールの公式サポートページをご確認ください。本記事の情報を利用した結果生じたいかなる損害についても、著者および運営者は責任を負いかねます。*
