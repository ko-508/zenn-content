---
title: "GitHub の failed to push some refs：原因と解決策"
emoji: "🐙"
type: "tech"
topics: ["github-api", "error"]
published: true
---

:::message
本記事は技術エラー解説サイト [errorlog.jp](https://errorlog.jp/) からの転載です。最新の内容と関連エラーの一覧は元記事を参照してください。
元記事: https://errorlog.jp/posts/github_failed_to_push_some_refs/
:::

## 冒頭まとめ

`error: failed to push some refs to '...'` は、理由を含んでいません。git の実装では、送信の処理が失敗して戻ってきたときに、この1行を無条件で出します。中身が何であれ表示されるため、この行を検索しても自分の状況に合う答えには辿り着きにくくなります。

理由は、この行の1つ上に出ます。`! [rejected]` で始まる行の末尾、括弧の中に入る語がそれです。入りうる語は `non-fast-forward`、`fetch first`、`already exists`、`needs force`、`stale info` の5つと、受け取り側が断ったことを示す `[remote rejected]` です。

その下に続く `hint:` の行にも注意が要ります。git は拒否の理由を集めたうえで、if と else if の並びで最初に当てはまった1件だけを表示します。つまり2つのブランチが別々の理由で拒まれても、助言は片方の分しか出ません。助言のとおりに `git pull` をしても、もう一方は直りません。

助言が1行も出ない場合もあります。`stale info` と `[remote rejected]` は、どちらも助言の対象から外れているためです。何も出ないから情報が無いのではなく、そこが読むべき箇所だと考えてください。

## エラーの概要

出力は次の形になります。

```text
To https://github.com/OWNER/REPO.git
 ! [rejected]        main -> main (fetch first)
error: failed to push some refs to 'https://github.com/OWNER/REPO.git'
hint: Updates were rejected because the remote contains work that you do not
hint: have locally. This is usually caused by another repository pushing to
hint: the same ref. If you want to integrate the remote changes, use
hint: 'git pull' before pushing again.
```

読む順序は下からではありません。`!` で始まる行が先で、`error:` の行は結果の要約です。送ろうとしたブランチが複数あれば、`!` の行も複数並びます。

括弧に入る5語は、git が次の順で判定した結果です。まず送り先がタグで相手に同名があれば `already exists`。次に相手の現在位置が指すオブジェクトを手元が持っていなければ `fetch first`。次に相手の位置か送る位置のどちらかがコミットでなければ `needs force`。最後に、送る位置が相手の位置の子孫でなければ `non-fast-forward` です。`stale info` だけは別経路で、`--force-with-lease` を付けたときの照合に失敗した場合に出ます。

同じ「更新を拒んだ」でも、この5語は互いに排他です。上から順に当てはまった時点で確定するため、たとえば `fetch first` と表示されているときに履歴の前後関係を調べても意味がありません。判定はそこまで進んでいません。

## まず最初に：拒否された行だけを取り出す

出力が長いときは、送信を実際には行わずに結果だけを見ます。

```bash
git push --dry-run origin main
```

`--dry-run` は送信せずに同じ判定を行い、同じ `!` の行を表示します。相手を壊す心配なく何度でも試せます。

次に、相手と手元の位置関係を数えます。

```bash
git fetch origin
git log --oneline main..origin/main
git log --oneline origin/main..main
```

上のコマンドは相手にだけあるコミット、下は手元にだけあるコミットを並べます。上が空でなければ、相手が進んでいます。両方とも空でないなら、履歴が分岐しています。

## よくある原因と解決手順

### 原因1：手元の履歴が相手の続きになっていない（non-fast-forward） {#non-fast-forward}

最も多い形です。相手の現在位置から手元の位置へ辿れないため、git は更新を拒みます。同じブランチを複数人が触っている場合か、手元で `git commit --amend` や `git rebase` を行って履歴を作り直した場合に起きます。

表示はこうなります。

```text
 ! [rejected]        main -> main (non-fast-forward)
hint: Updates were rejected because the tip of your current branch is behind
hint: its remote counterpart. If you want to integrate the remote changes,
hint: use 'git pull' before pushing again.
```

いま作業しているブランチではなく別のブランチが拒まれた場合は、文面が `a pushed branch tip is behind its remote counterpart` に変わります。指し示す対象が違うだけで、判定は同じです。

対処は2つに分かれます。相手の変更を取り込んでよいなら、取り込んでから送り直します。

```bash
git pull --rebase origin main
git push origin main
```

手元で履歴を作り直した結果として相手を上書きしたい場合は、`--force-with-lease` を使います。`--force` は相手の状態を一切見ないため、他人のコミットを消す事故につながります。

### 原因2：相手の現在位置を手元が持っていない（fetch first） {#fetch-first}

`non-fast-forward` と混同されやすい状態です。git は、相手の現在位置が指すオブジェクトが手元のデータベースに無いかどうかを先に調べます。無ければ、続きかどうかを判定する材料自体が無いため `fetch first` になります。

```text
 ! [rejected]        main -> main (fetch first)
```

取得していないだけのこともあれば、取得の範囲を絞っていることが原因のこともあります。自動化の中で `--depth 1` や単一ブランチの取得を使っている場合、手元には最新の1件しか無く、相手の位置が手元に存在しません。

対処は取得です。

```bash
git fetch origin
git push origin main
```

これで解消しない場合は、取得の深さが制限されていないかを確認します。

```bash
git rev-parse --is-shallow-repository
```

`true` が返れば範囲が絞られています。`git fetch --unshallow` で全体を取り直すか、自動化側の取得設定を見直してください。

### 原因3：同じ名前のタグが相手に既にある（already exists） {#already-exists}

タグは特別扱いです。判定の並びで最初に置かれており、相手に同じ名前があれば、中身が何であっても拒まれます。

```text
 ! [rejected]        v1.0.0 -> v1.0.0 (already exists)
hint: Updates were rejected because the tag already exists in the remote.
```

ブランチと違い、位置関係は見ません。タグは特定の時点を指す固定の名前として扱われるためです。

まず相手側を確認します。

```bash
git ls-remote --tags origin
```

対処は、別の名前を付けるか、相手側のタグを削除してから送るかです。公開済みのタグを作り直すと、既に取得した人の手元と食い違うため、名前を変えるほうが安全です。

### 原因4：コミット以外を指す参照を書き換えようとしている（needs force） {#needs-force}

注釈付きタグを軽量タグに置き換える場合などに出ます。git は、書き換え前の位置と書き換え後の位置の両方について、それがコミットとして解決できるかを調べます。どちらか一方でも解決できなければ、この判定に落ちます。

```text
 ! [rejected]        v1.0.0 -> v1.0.0 (needs force)
hint: You cannot update a remote ref that points at a non-commit object,
hint: or update a remote ref to make it point at a non-commit object,
hint: without using the '--force' option.
```

種類を確認します。

```bash
git cat-file -t v1.0.0
```

`tag` なら注釈付き、`commit` なら軽量です。意図した置き換えであれば強制の指定を付け、そうでなければ種類を揃えてください。

### 原因5：--force-with-lease の期待値がずれている（stale info） {#stale-info}

`--force-with-lease` は、相手の現在位置が自分の知っている位置と同じであることを条件に上書きします。食い違えば拒みます。

```text
 ! [rejected]        main -> main (stale info)
```

この場合、助言は出ません。表示されるのは要約行と `!` の行だけです。

相手の位置を直接確認します。

```bash
git ls-remote origin refs/heads/main
```

自分が知っている位置と違っていれば、その間に誰かが送っています。取得して中身を確かめ、上書きしてよいと判断してから送り直してください。

### 原因6：相手側が受け取りを断っている（remote rejected） {#remote-rejected}

`!` の行が `[rejected]` ではなく `[remote rejected]` になっている場合、判定を行ったのはgit ではありません。受け取り側が断っています。

```text
 ! [remote rejected] main -> main (protected branch hook declined)
```

括弧の中には、相手が返した文言がそのまま入ります。GitHub であれば保護されたブランチの設定や ruleset、自動化のために置かれた受け取り側のスクリプトが理由になります。この経路には助言が付きません。読む場所は括弧の中だけです。

規則違反であれば [GitHub の GH013 の記事](https://errorlog.jp/posts/github_gh013_repository_rule_violations/)で扱っています。

## 補足：似ているが別のもの

`Everything up-to-date` は失敗ではありません。送るべき差がないという意味です。コミットを作り忘れているか、送り先の指定が想定と違います。

`fatal: Could not read from remote repository.` や `Repository not found` は、この記事の要約行より手前で止まっています。認証や宛先の段階なので、判定まで進んでいません。前者は [publickey の記事](https://errorlog.jp/posts/github_permission_denied_publickey/)、後者は [Repository not found の記事](https://errorlog.jp/posts/github_repository_not_found/)を参照してください。

`! [remote failure]` は受け取り側が結果を報告しなかった場合です。拒否とは別で、通信が途中で終わったときに出ます。

## 切り分けの順序

1. `!` で始まる行を数える。複数あれば、助言は最優先の1件分しか出ていないと考える
2. `[rejected]` か `[remote rejected]` かを見る。後者なら判定したのは相手側で、括弧の中がすべて
3. `[rejected]` なら括弧の5語を確認する
4. `fetch first` なら `git fetch` を先に行い、取得範囲が絞られていないかを調べる
5. `non-fast-forward` なら相手と手元の差を数え、取り込むか上書きするかを決める
6. `already exists` と `needs force` はタグまわりなので、相手側の同名と種類を確認する
7. `stale info` は助言が出ないため、相手の現在位置を直接見る

## 確認コマンド集

```bash
# 1. 送信せずに判定だけを見る（最初に行う）
git push --dry-run origin main

# 2. 送り先の宛先を確認する
git remote -v

# 3. 相手にだけある変更を数える
git fetch origin
git log --oneline main..origin/main

# 4. 手元にだけある変更を数える
git log --oneline origin/main..main

# 5. 相手の現在位置を直接見る
git ls-remote origin refs/heads/main

# 6. 相手側のタグ一覧を見る
git ls-remote --tags origin

# 7. 取得範囲が絞られていないか調べる
git rev-parse --is-shallow-repository

# 8. 参照が指しているものの種類を調べる
git cat-file -t v1.0.0
```

## Editor's Note

`--force-with-lease` は安全な上書きの手段として広く紹介されています。ところが git 自身は、このこの指定が条件付きでしか安全でないと説明しています。

[公式ドキュメント](https://github.com/git/git/blob/master/Documentation/git-push.adoc)には、背景で `git fetch --all` を走らせる仕組みがあると、この方法は完全に無効化されると書かれています。理由は照合の対象にあります。`--force-with-lease` が期待値として使うのは、手元に保存されている相手の位置の記録です。編集ツールや自動化が裏で取得を行えば、その記録は内容を確認しないまま新しい位置へ進みます。期待値が実際の値に追いついてしまうため、照合は通り、他人のコミットは消えます。

git はこの問題に対して、2020年公開の 2.30 で `--force-if-includes` を追加しました。[リリースノート](https://github.com/git/git/blob/master/Documentation/RelNotes/2.30.0.adoc)には、`--force-with-lease` は自分で `git fetch` をよく管理していない限りコミットを失いやすい、と率直に書かれています。追加された確認は、置き換えようとしている相手の位置を実際に見たうえで手元の内容が作られたかどうかを調べるものです。

実務上の意味はこうです。`stale info` が出たときは、照合が働いたということです。むしろ出なかったときのほうを疑ってください。`--force-with-lease` が静かに通った直後に他人の変更が消えているなら、背景の取得が期待値を進めていた可能性があります。上書きを日常的に行うリポジトリでは、`--force-if-includes` を併せて指定しておくほうが確実です。

---

*免責事項：本記事の内容は、執筆時点の公開情報をもとに作成したものです。ソフトウェアの仕様は予告なく変更されることがあります。最新の情報は各ツールの公式サポートページをご確認ください。本記事の情報を利用した結果生じたいかなる損害についても、著者および運営者は責任を負いかねます。*
