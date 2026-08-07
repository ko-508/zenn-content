---
title: "GitHub の Repository not found エラー：原因と解決策"
emoji: "🐙"
type: "tech"
topics: ["github-api", "error"]
published: true
---

:::message
本記事は技術エラー解説サイト [errorlog.jp](https://errorlog.jp/) からの転載です。最新の内容と関連エラーの一覧は元記事を参照してください。
元記事: https://errorlog.jp/posts/github_repository_not_found/
:::

## 冒頭まとめ

git の `Repository not found` は必ず2行で出ますが、その2行は書き手が違います。1行目は GitHub のサーバーが返したレスポンス本文を git がそのまま転記したもので、2行目は git 自身の判断です。書き手を分けて読むと、原因の範囲が一度に絞れます。

決め手は2行目です。git の実装では、`fatal: repository '...' not found` は HTTP 404 を受け取ったときにしか出ません（`remote-curl.c` の分岐と `http.h` の `missing__target`）。一方、2026年8月7日の実測では、GitHub は未認証のリクエストに 404 ではなく 401 を返しました。この2行が並んだ時点で、認証そのものは成功していて、そのトークンや鍵の持ち主にリポジトリが見えていない、という一点に絞られます。

ここで、よくある誤解が2つ崩れます。綴りの見直しは優先順位が下がります。GitHub は所有者名とリポジトリ名を大文字小文字の違いを無視して解決するため、`OCTOCAT/HELLO-WORLD` でも通りました（実測で 200）。改名や移管を疑う必要もほとんどありません。旧パスへの `git clone`・`git fetch`・`git push` は新しい場所への操作として動き続ける、と公式ドキュメントが明記しています。

最初に確定させるべきなのは、いま自分がどのアカウントとして GitHub と通信しているかです。HTTPS なら保存済みの資格情報、SSH なら提示している鍵が、その入口になります。

## エラーの概要

HTTPS で出る場合、ログは次のようになります。GitHub Actions 上の実際の報告では、git の終了コードは 128 でした。

```text
remote: Repository not found.
fatal: repository 'https://github.com/OWNER/REPO.git/' not found
```

URL 末尾のスラッシュは入力の誤りではありません。git は通信前に `end_url_with_slash()` で末尾を揃えており、揃えた後の値を表示しているだけです。ここを直しても何も変わりません。

SSH の場合は文言が変わります。

```text
ERROR: Repository not found.
fatal: Could not read from remote repository.
Please make sure you have the correct access rights
and the repository exists.
```

1行目の `ERROR:` は GitHub の SSH サーバーが出したものです。2行目以降は git の `connect.c` にある `die_initial_contact()` の文言で、相手がプロトコルの応答を1つも返さずに接続を閉じると出ます。SSH に HTTP のステータスコードはないため、判断材料はこの1行だけです。

HTTPS 側は実測で確かめられます。2026年8月7日に探索エンドポイントへ直接送ったところ、存在しないリポジトリへの未認証リクエストに返ってきたのは 401 でした。

```text
HTTP/2 401
www-authenticate: Basic realm="GitHub"
content-type: text/plain; charset=UTF-8
content-length: 21

Repository not found.
```

同じエンドポイントへ無効なトークンを付けると、状態コードは 401 のままで本文だけが変わります。この本文は実在する公開リポジトリに対しても同じで、リポジトリの側については何も語りません。

```text
Invalid username or token. Password authentication is not supported for Git operations.
```

応答本文と2行目は対応しています。本文が `Repository not found.` で2行目が `fatal: repository '...' not found` なら、最終的な応答は 404 です。本文が `Invalid username or token.` で始まり2行目が `fatal: Authentication failed for '...'` なら 401 です。前者は認証が通った上での不可視、後者は認証そのものの失敗で、直す場所が違います。

## まず最初に：いま誰として通信しているかを確定する

原因を推測する前に、通信の主体を1つに固定します。HTTPS なら、git が実際に取り出す資格情報を表示させます。`git credential fill` は、設定と保存先と補助プログラムを経由して、そのエンドポイントに使う値を決める公式の手段です。

```bash
printf 'protocol=https\nhost=github.com\n\n' | git credential fill
```

出力の `username` が想定したアカウントと一致するかを見ます。会社用と個人用を使い分けている環境では、別人の名前が出ることが珍しくありません。

SSH なら、鍵がどのアカウントとして受け付けられるかを確認します。公式ドキュメントが示す成功時の出力は、名前が入った1行です。

```bash
ssh -T git@github.com
# -> Hi USERNAME! You've successfully authenticated, but GitHub does not provide shell access.
```

ここに出た `USERNAME` が想定と違えば、その時点で原因は確定します。想定どおりなら、次はそのアカウントに対象が見えるかどうかの問題へ移ります。

## よくある原因と解決手順

### 原因1：保存済みの資格情報が別のアカウントのもの（HTTPS）

資格情報の保存先に古いトークンや別アカウントのトークンが残っていると、git はそれを黙って使います。値そのものは有効なので認証は成功し、しかしその持ち主には対象が見えないため、GitHub は 404 を返します。公式ドキュメントも、古い資格情報が保存されたままになっていないか確かめるよう促しています。無効なトークンなら 401 になって別の文言が出るので、`Repository not found` が出ているなら値は生きています。疑うべきは値の正しさではなく、持ち主です。

**Before（保存されている値を確かめずに再試行する）：**

```bash
git clone https://github.com/OWNER/REPO.git
# -> remote: Repository not found.
```

**After（使われる資格情報を表示し、違えば消してから入れ直す）：**

```bash
printf 'protocol=https\nhost=github.com\n\n' | git credential fill
printf 'protocol=https\nhost=github.com\n\n' | git credential reject
git clone https://github.com/OWNER/REPO.git
```

`reject` は保存先から該当の値を消します。次の通信で入力を求められるので、そこで正しいアカウントのトークンを渡します。

### 原因2：トークンの適用範囲に、そのリポジトリが入っていない

トークンが有効でも、届く範囲は種類ごとに決まっており、範囲外のリポジトリは権限不足ではなく不在として扱われます。

細かい権限を指定する新しいトークン（fine-grained）は、作成時に対象リポジトリを選びます。公式ドキュメントは、このトークンが常に全公開リポジトリへの読み取りを含むと説明しています。そのため公開リポジトリでは何も起きず、非公開のものだけが見えないという紛らわしい状態になります。

従来型のトークン（classic）はスコープで範囲が決まります。公式ドキュメントは、スコープを1つも与えていないトークンが公開情報にしかアクセスできないと明記しており、非公開リポジトリには `repo` の指定が要ります。

GitHub Actions が自動で用意する `GITHUB_TOKEN` は、さらに範囲が狭くなります。公式ドキュメントは、このトークンの権限がワークフローを含むリポジトリに限られると明記しており、同じ組織の別リポジトリにも届きません。

単一サインオン（SSO）を使う組織では、もう1段あります。従来型のトークンは作成後に組織ごとの承認が必要で、承認前は対象が見えません。

**Before（既定のトークンで別リポジトリを取得しようとする）：**

```yaml
- uses: actions/checkout@v4
  with:
    submodules: recursive
```

**After（対象リポジトリに届く資格情報を明示的に渡す）：**

```yaml
- uses: actions/checkout@v4
  with:
    token: ${{ secrets.CROSS_REPO_TOKEN }}
    submodules: recursive
```

### 原因3：SSH の鍵が、そのリポジトリを見られない相手に結び付いている

SSH で通信の主体を決めるのは秘密鍵です。鍵が別のアカウントに登録されていれば、GitHub はその別人として応答します。鍵が複数あると、意図しないものが先に提示されることもあります。

デプロイ用の鍵には、さらに厳しい制限があります。公式ドキュメントは、この鍵が1つのリポジトリにしか権限を与えず、使い回せないと明記しています。1台のサーバーで複数を扱うなら、作り分けが要ります。SSO を使う組織では、鍵そのものにも承認が必要です。

**Before（どの鍵が使われるか任せている）：**

```bash
git clone git@github.com:OWNER/REPO.git
# -> ERROR: Repository not found.
```

**After（対象ごとに別名を用意し、使う鍵を1つに固定する）：**

```
Host github-work
    Hostname github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_work
    IdentitiesOnly yes
```

```bash
git clone git@github-work:OWNER/REPO.git
```

`IdentitiesOnly yes` は、指定した鍵だけを提示させる設定です。これがないと、エージェントが抱える別の鍵が先に通り、想定と違うアカウントとして扱われることがあります。

### 原因4：トークンを直したのに、通信経路が SSH のまま

トークンを作り直しても状況が変わらないなら、そもそもそれが使われていない可能性があります。公式ドキュメントは、トークンが HTTPS の git 操作でしか使えず、SSH のアドレスを設定している場合は HTTPS への切り替えが要ると明記しています。逆に、鍵を整備したのに URL が HTTPS のままでも同じことが起きます。

**Before（経路を確かめずにトークンだけ入れ替える）：**

```bash
git remote -v
# -> origin  git@github.com:OWNER/REPO.git (fetch)
```

**After（使う経路に合わせてアドレスを揃える）：**

```bash
git remote set-url origin https://github.com/OWNER/REPO.git
git remote -v
```

### 原因5：リポジトリが本当に存在しない

ここまでの確認で主体が正しいと分かった場合に限り、不在を疑います。公式ドキュメントは、存在しないリポジトリへ push しようとした場合にもこの文言が出ると説明しています。

改名や移管そのものは原因になりません。ただし例外が2つ、公式に警告として書かれています。1つは、旧名で新しいリポジトリを作ると転送が失われること。もう1つは GitHub Actions に限った話で、改名されたリポジトリにあるアクションへの呼び出しは転送されず、それを使うワークフローは `repository not found` で失敗します。

綴りについても公式ドキュメントは確認を挙げていますが、大文字小文字は原因になりません。実測では `OCTOCAT/HELLO-WORLD` のように字種を変えたリクエストでも 200 が返っています。疑うべきは字種ではなく、ハイフンとアンダースコアの取り違えや、似た名前の別リポジトリの指定です。

## 補足：似ているが別のもの

`remote: Permission to OWNER/REPO.git denied to USER.` は、別の状況を指します。相手にはリポジトリが見えており、その操作に必要な権限だけが足りていません。公式ドキュメントはこれを、鍵が対象へのアクセスを持たないアカウントに結び付いている場合の文言として説明し、対処は所有者に共同作業者として追加してもらうことだとしています。鍵が別リポジトリのデプロイ用として登録されていると、末尾がアカウント名ではなく `OWNER/OTHER-REPO` になります。

`No anonymous write access.` は、公開リポジトリへ未認証のまま push しようとしたときの応答本文で、実測での状態コードは 401 でした。リポジトリは見えているため、`Repository not found` にはなりません。

`fatal: Authentication failed for '...'` は認証そのものの失敗です。直前の `remote:` 行に `Invalid username or token.` が出ていれば、トークンの値が誤っているか、期限切れか失効です。

`git@github.com: Permission denied (publickey).` は、提示した鍵がどのアカウントとしても受け付けられなかった状態です。GitHub が相手を誰とも認識していないため、リポジトリの判定にまだ進んでいません。

REST API の 404 も考え方は同じですが、応答の読み方が違います。API 側は JSON 本文に `message` と `documentation_url` を返すため、判断材料が増えます。詳しくは [GitHub API の 404 の記事](https://errorlog.jp/posts/github_api_404/)を参照してください。トークンの値そのものが疑わしい場合は [GitHub API の 401 の記事](https://errorlog.jp/posts/github_api_401/)が対応します。

## 切り分けの順序

1. `git remote -v` で、通信経路が HTTPS か SSH かを確定する。以降の確認先がここで分かれる。
2. HTTPS なら `git credential fill` で、実際に使われる資格情報の持ち主を表示する。想定と違えば原因1で確定する。
3. SSH なら `ssh -T git@github.com` で、鍵がどのアカウントとして認識されるかを表示する。想定と違えば原因3で確定する。
4. 確実に見えるはずの公開リポジトリに対して、同じ形のクローンを試す。ここで失敗するなら、原因は対象リポジトリではなく経路や設定の側にある。
5. `curl -i` で探索エンドポイントを直接叩き、状態コードと本文を確認する。
6. 401 なら認証の問題として原因1と原因4を、404 なら不可視の問題として原因2と原因3を見る。
7. トークンや鍵の適用範囲、および SSO の承認状態を確認する。組織の設定画面で承認が必要な場合がある。
8. ここまで問題がなければ、Web 画面で存在そのものを確認する。旧名を再利用していないか、Actions から改名済みリポジトリを呼んでいないかも見る。

## 確認コマンド集

```bash
# 1. 通信経路を確認する（git@ で始まれば SSH、https:// なら HTTPS）
git remote -v

# 2. HTTPS で実際に使われる資格情報の持ち主を表示する
printf 'protocol=https\nhost=github.com\n\n' | git credential fill

# 3. 保存されている資格情報を削除して、入力し直せる状態に戻す
printf 'protocol=https\nhost=github.com\n\n' | git credential reject

# 4. SSH の鍵がどのアカウントとして認識されるかを確認する
ssh -T git@github.com

# 5. 実際に提示された鍵を確認する（offering public key の行を見る）
ssh -v -T git@github.com 2>&1 | grep -i "offering\|Authenticated"

# 6. 探索エンドポイントを直接叩き、状態コードと本文を確認する
curl -i "https://github.com/OWNER/REPO.git/info/refs?service=git-upload-pack"

# 7. トークンを付けた場合の応答を確認する（401 か 404 かで分岐が決まる）
curl -i -u "USERNAME:<your-github-token>" \
  "https://github.com/OWNER/REPO.git/info/refs?service=git-upload-pack"

# 8. 入力待ちを止めて、隠れていた失敗をそのまま表示させる
GIT_TERMINAL_PROMPT=0 git clone https://github.com/OWNER/REPO.git
```

## Editor's Note

この文言がどう人を迷わせるかは、[actions/checkout の Issue #2080](https://github.com/actions/checkout/issues/2080) によく残っています。2025年2月13日に開かれ、現在も開いたままの報告です。

報告者は、非公開リポジトリの中に同じ組織の非公開リポジトリをサブモジュールとして抱えた構成を、GitHub Actions で取得しようとしました。手元の操作でも Web 画面でも扱えているのに、ワークフロー上ではサブモジュールのすべてに `remote: Repository not found.` が返ります。報告者は組織の管理者で、すべてのリポジトリにアクセスできる立場でした。それでもこの文言が出ています。

後から参加した別の利用者が、公式ドキュメントを引いて理由を説明しています。`GITHUB_TOKEN` は GitHub App の設置トークンであり、その権限はワークフローを含むリポジトリに限られる、という一文です。同じ組織が持つリポジトリでも同じだけ信頼されているとは限らないため、これは仕様である、と整理されています。そのうえで、このエラー文言は出来が悪い、とも書き添えられています。

解決の報告も複数あります。従来型のトークンに差し替えた例、`contents` の読み取りと `metadata` を与えた細かい権限のトークンを渡した例、GitHub App の短命なトークンを発行して届かせたいリポジトリを列挙した例です。いずれも直したのはリポジトリの側ではなく、通信の主体でした。存在を疑うより先に誰として通信しているかを確かめる手順が、このエラーには有効に働きます。

---

*免責事項：本記事の内容は、執筆時点の公開情報をもとに作成したものです。ソフトウェアの仕様は予告なく変更されることがあります。最新の情報は各ツールの公式サポートページをご確認ください。本記事の情報を利用した結果生じたいかなる損害についても、著者および運営者は責任を負いかねます。*