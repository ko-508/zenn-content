---
title: "npm の gyp ERR! build error：原因と解決策"
emoji: "🚫"
type: "tech"
topics: ["npm", "error"]
published: true
---

:::message
本記事は技術エラー解説サイト [errorlog.jp](https://errorlog.jp/) からの転載です。最新の内容と関連エラーの一覧は元記事を参照してください。
元記事: https://errorlog.jp/posts/npm_gyp_err_build_error/
:::

## 結論

`gyp ERR! build error` の build は、原因の名前ではありません。失敗したコマンドの名前です。node-gyp の実装は、処理が例外で終わったときに「そのコマンド名」と `error` をつないだ見出しを出します。だから `configure error` や `install error` も同じ書式で現れます。build と出ていれば、設定の段階は通っていて、組み立ての段階で落ちたという意味になります。

もう1つ押さえる点があります。この段階で node-gyp 自身が投げるエラーは、ほぼ1種類しかありません。実装では、呼び出した `make` か `msbuild` が 0 以外で終わったときに、終了コードをそのまま載せたエラーを作ります。つまり `gyp ERR! stack` の行は「外部のプロセスが失敗した」としか言っていません。

本当の原因は、その上にあります。組み立ての出力は node-gyp を素通りして画面に出るため、`gyp ERR!` の並びより前に、実際の失敗が文字として残っています。読む場所はそこです。

## 最初に確認すること

`gyp ERR! stack` の1行目を見ます。

```text
gyp ERR! build error
gyp ERR! stack Error: `make` failed with exit code: 2
```

この形であれば、組み立てそのものが失敗しています。画面をさかのぼり、`error:` で始まる最初の行を探してください。複数並んでいても、最初の1件が起点です。

一方、1行目が次のような文であれば、失敗したのは組み立てではなく準備です。

```text
gyp ERR! stack Error: Could not find *.sln file or Makefile. Did you run "configure"?
```

`gyp ERR!` の末尾に並ぶ System、command、cwd、node -v、node-gyp -v の5行は、失敗のたびに必ず出る環境の記録です。原因は入っていません。ただし node -v の値は、次の原因1の判断に使います。

## 原因別の確認方法と解決策

### 原因1：その Node.js の版に合う完成品が無い {#no-prebuilt-for-node-version}

最も多い入り口です。多くの部品は、よく使われる版に合わせて組み立て済みのものを配っています。該当が無ければ、その場で組み立てが始まります。普段は起きない失敗が、バージョンを上げた直後や新しいコンテナの土台へ移した直後だけ出るのは、このためです。

```text
gyp ERR! node -v v24.0.0
```

この値と、その部品が対応を表明している範囲を照らしてください。対応が追いついていなければ、バージョンを戻すか、対応済みの新しい部品へ上げるかの二択になります。手元で組み立てて直すよりも、この判断のほうが早く済みます。

### 原因2：組み立てに使う道具が無い {#build-tool-not-found}

node-gyp は自分では組み立てません。外部の道具を呼びます。公式の案内では、Unix 系で Python と make と C/C++ の組み立て環境、macOS では Xcode のコマンド行ツールが必要だと示されています。

この不足は多くの場合 `configure error` として先に出ますが、設定が残ったまま道具だけ失われた場合は build の側で出ます。`gyp ERR! stack` に終了コードではなく道具が見つからない旨の文が出ていれば、こちらです。

軽量なコンテナの土台では、この手の道具が最初から入っていません。インストールの手順に組み立て環境を加えてください。

### 原因3：組み立て中に中身が通らない {#compiler-error-in-source}

道具は揃っていて、それでも失敗する場合です。終了コードは 2 になることが多く、画面には `error:` で始まる行が並びます。

```text
../src/binding.cc:42:10: error: no member named 'New' in 'v8::String'
gyp ERR! build error
gyp ERR! stack Error: `make` failed with exit code: 2
```

原因は部品側と土台側のずれです。古い部品が新しい Node.js の内部インターフェースを前提にしていない場合や、逆に新しすぎる組み立て環境が古い書き方を受け付けない場合に起きます。

対処は、その部品の版を対応しているものに合わせることです。読み手がモジュールの作者でない限り、中身を直す選択肢は現実的ではありません。

### 原因4：メモリが尽きて外から止められた {#killed-by-memory-limit}

コンテナや小さな仮想機械で起きます。node-gyp は、呼んだプロセスが信号で終わった場合、終了コードではなく信号の名前を載せます。

```text
gyp ERR! stack Error: `make` got signal: SIGKILL
```

`exit code` ではなく `signal` と出ていれば、組み立てが失敗したのではなく、途中で止められています。メモリの割り当てを増やすか、同時に走らせる本数を減らしてください。本数は `--jobs` か `JOBS` の環境変数で指定できます。

## 近いエラーとの違い

`gyp ERR! configure error` は、組み立ての前の段階です。Python や組み立て環境を探す処理で止まっており、`gyp ERR! find Python` のような別の見出しが一緒に出ます。

`gyp ERR! not ok` は失敗の締めくくりとして必ず出る行で、原因とは関係ありません。

`npm error code 1` は、同じ失敗を npm の側から見た表示です。npm にとっては、インストールの途中で呼んだスクリプトが 0 以外で終わっただけなので、中身の区別はありません。原因を探すなら node-gyp の側の行を読みます。

`Completion callback never invoked!` と `UNCAUGHT EXCEPTION` は、node-gyp の内部で想定外が起きた場合の表示です。どちらも組み立ての失敗ではなく、終了コードも 6 と 7 で分かれています。

## 参考資料

- [node-gyp の README（必要な道具）](https://github.com/nodejs/node-gyp/blob/main/README.md)
- [見出しと環境情報の出力（bin/node-gyp.js）](https://github.com/nodejs/node-gyp/blob/main/bin/node-gyp.js)
- [組み立て段階の実装（lib/build.js）](https://github.com/nodejs/node-gyp/blob/main/lib/build.js)

---

*免責事項：本記事の内容は、執筆時点の公開情報をもとに作成したものです。ソフトウェアの仕様は予告なく変更されることがあります。最新の情報は各ツールの公式サポートページをご確認ください。本記事の情報を利用した結果生じたいかなる損害についても、著者および運営者は責任を負いかねます。*
