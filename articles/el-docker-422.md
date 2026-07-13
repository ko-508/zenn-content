---
title: "Docker の 422 エラー：原因と解決策"
emoji: "🐳"
type: "tech"
topics: ["docker", "error"]
published: true
---

:::message
本記事は技術エラー解説サイト [errorlog.jp](https://errorlog.jp/) からの転載です。最新の内容と関連エラーの一覧は元記事を参照してください。
元記事: https://errorlog.jp/posts/docker_422/
:::

## エラーの概要

Dockerで 422 エラーが発生するのは、Docker APIまたはコンテナレジストリへのリクエストが構文的には正しいものの、含まれるデータが処理要件を満たしていない場合です。Docker Daemon、Docker Compose、レジストリ APIとの通信時にこのエラーが返される典型的なシナリオは、不正なイメージタグ指定、設定値の型違反、あるいは APIスキーマの検証失敗です。

## 実際のエラーメッセージ例

```json
{
  "message": "invalid tag format",
  "code": 422
}
```

```bash
$ docker push myregistry.example.com/app:invalid@tag
Error response from daemon: invalid tag format
```

```yaml
# docker-compose.yml でエラーが発生
ERROR: The Compose file is invalid because:
Service 'web' has invalid value for ports: ports must be an integer or string
```

## よくある原因と解決手順

### 1. イメージタグの形式が不正

Dockerレジストリ APIは RFC 6391 に基づいたタグ形式を要求します。許可されない文字（`@`や大文字の混在）が含まれている場合に 422 が返されます。

**Before（エラーが起きる例）：**
```bash
docker tag myimage:latest myregistry.example.com/app:INVALID@latest
docker push myregistry.example.com/app:INVALID@latest
# Error: invalid tag format
```

**After（修正後）：**
```bash
# タグは小文字のみで、「:」で区切る
docker tag myimage:latest myregistry.example.com/app:v1.0.0
docker push myregistry.example.com/app:v1.0.0
```

### 2. docker-compose.yml の設定値の型違反

`ports`、`mem_limit`、`cpu_shares`など、数値型を期待するフィールドに文字列を指定するとバリデーション失敗で 422 が返されます。

**Before（エラーが起きる例）：**
```yaml
version: '3.8'
services:
  web:
    image: nginx:latest
    ports:
      - "8080"  # 文字列のままだと型エラー
    mem_limit: "512"  # 文字列だと失敗する場合がある
```

**After（修正後）：**
```yaml
version: '3.8'
services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"  # ホストポート:コンテナポートの形式
    mem_limit: 536870912  # バイト単位の数値、または "512m" 形式の文字列
```

### 3. マニフェスト JSON の構造が不正

Dockerイメージをプッシュする際に、レイヤーのダイジェスト値が不正な形式である場合、レジストリが 422 で拒否します。

**Before（エラーが起きる例）：**
```bash
# イメージをビルド中に破損したマニフェスト参照
docker build -t myapp:broken .
docker push myregistry.example.com/myapp:broken
# Error: invalid manifest: sha256: invalid digest
```

**After（修正後）：**
```bash
# イメージを再度ビルドしてプッシュ
docker build -t myregistry.example.com/myapp:v1.0.0 .
docker push myregistry.example.com/myapp:v1.0.0
# 正規のダイジェスト形式で処理されます
```

### 4. API リクエストのボディスキーマ不整合

Docker API（例：`/containers/create`）に POST リクエストを送る際、必須フィールドが欠落しているか、型が異なるとバリデーション失敗で 422 が返されます。

**Before（エラーが起きる例）：**
```bash
curl -X POST http://localhost:2375/containers/create \
  -H "Content-Type: application/json" \
  -d '{"Hostname": "test"}'
# 422: ExposedPorts must be a map
```

**After（修正後）：**
```bash
curl -X POST http://localhost:2375/containers/create \
  -H "Content-Type: application/json" \
  -d '{
    "Image": "ubuntu:latest",
    "Hostname": "test",
    "ExposedPorts": {}
  }'
```

## Docker 固有の注意点

### Docker Compose バージョンと設定値の互換性

`docker-compose.yml`で `version: '3.8'`を指定した場合、古い構文（`1.0`時代の短縮ポート指定）は 422 で拒否されます。バージョンと設定内容の整合性を確認してください。

### レジストリ認証後のプッシュ時エラー

Docker Hub や Private Registry にログイン後、プッシュ時に 422 が出る場合は、イメージ名が登録済みプロジェクトのパス構造に一致しているか確認します。

```bash
# 認証は成功したが 422 エラーが出る場合
docker login -u <username> myregistry.example.com

# プッシュ前にタグ形式をチェック
docker image ls | grep myimage
# タグが完全なレジストリパスになっているか確認
```

### BuildKit キャッシュとダイジェスト値

`DOCKER_BUILDKIT=1`を使用している場合、キャッシュレイヤーのダイジェスト不整合で 422 が発生することがあります。この場合は `--no-cache`フラグを使用してビルドを再実行してください。

```bash
DOCKER_BUILDKIT=1 docker build --no-cache -t myapp:v1 .
```

## それでも解決しない場合

### ログ確認とデバッグコマンド

Docker Daemon のログを確認し、より詳細なエラー情報を取得します。

```bash
# systemd でコンテナを実行している場合
journalctl -u docker -f

# macOS / Windows の Docker Desktop の場合
cat ~/.docker/daemon.json | jq .

# Docker API への直接リクエストをテスト
curl -v --unix-socket /var/run/docker.sock \
  -X GET http://localhost/version
```

### 公式ドキュメント参照

Docker Compose 設定リファレンス（https://docs.docker.com/compose/compose-file/）で、各フィールドの型と制約を確認してください。API スキーマ検証エラーの場合は「Docker Engine API」ドキュメントの `POST /containers/create`セクションを参照します。

### 環境別の確認ポイント

- **Private Registry 使用時**: レジストリの APIバージョンを確認し、サポートされているイメージマニフェスト形式を検証します
- **Kubernetes経由でのデプロイ**: `imagePullPolicy`設定とイメージレジストリの CORS設定を確認します
- **CI/CDパイプライン**: GitHub Actions や GitLab CI のアーティファクトストレージ設定を見直し、イメージダイジェストの計算ロジックをテストします

---

*免責事項：本記事の内容は、執筆時点の公開情報をもとに作成したものです。ソフトウェアの仕様は予告なく変更されることがあります。最新の情報は各ツールの公式サポートページをご確認ください。本記事の情報を利用した結果生じたいかなる損害についても、著者および運営者は責任を負いかねます。*