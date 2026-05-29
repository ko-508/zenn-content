---
title: "Docker の 504 エラー：原因と解決策"
emoji: "🐳"
type: "tech"
topics: ["docker", "error"]
published: true
---

## エラーの概要

504 Gateway Timeout は、Docker デーモンがプロキシ経由で上流サーバーへのリクエストに応答を待つ際に、設定されたタイムアウト時間を超過したことを示すHTTPステータスコードです。Docker Compose を使用している場合やリバースプロキシ（Nginx など）経由でコンテナにアクセスしている場合に頻繁に発生します。このエラーは、サーバー側の処理遅延、ネットワークの問題、またはタイムアウト設定の不適切さが原因となります。

## 実際のエラーメッセージ例

```json
{
  "status": 504,
  "error": "Gateway Timeout",
  "message": "The upstream server failed to respond in time"
}
```

```bash
$ curl -v http://localhost:8080/api/endpoint
< HTTP/1.1 504 Gateway Timeout
< Content-Type: application/json
```

## よくある原因と解決手順

### 原因1：コンテナ内のアプリケーションが処理に時間がかかっている

コンテナ内で実行されるアプリケーションの処理が、プロキシ側で設定されたタイムアウト時間を超過している場合です。これはデータベースクエリの遅延、外部API呼び出しの低速化、または重い計算処理が原因かもしれません。

**Before（タイムアウト設定が短すぎる場合）:**
```yaml
# docker-compose.yml
version: '3.8'
services:
  web:
    image: myapp:latest
    environment:
      REQUEST_TIMEOUT: 5000  # 5秒のタイムアウト
    ports:
      - "8080:3000"
  
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
```

```nginx
# nginx.conf
upstream backend {
    server web:3000;
}

server {
    listen 80;
    proxy_connect_timeout 5s;  # 短すぎるタイムアウト
    proxy_send_timeout 5s;
    proxy_read_timeout 5s;
}
```

**After（タイムアウトを適切に設定）:**
```yaml
version: '3.8'
services:
  web:
    image: myapp:latest
    environment:
      REQUEST_TIMEOUT: 30000  # 30秒に設定
    ports:
      - "8080:3000"
  
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
```

```nginx
# nginx.conf
upstream backend {
    server web:3000;
}

server {
    listen 80;
    proxy_connect_timeout 30s;  # 接続タイムアウト
    proxy_send_timeout 30s;      # データ送信タイムアウト
    proxy_read_timeout 30s;      # レスポンス待機タイムアウト
}
```

### 原因2：Docker Compose のサービス間通信に遅延がある

Docker Compose で複数のサービスを実行している場合、サービス間のネットワーク通信が遅延したり、DNS解決が失敗したりすることがあります。特に CPU やメモリリソースが逼迫している場合に発生しやすいです。

**Before（リソース制限がない設定）:**
```yaml
version: '3.8'
services:
  api:
    image: api-server:latest
    ports:
      - "3000:3000"
  
  database:
    image: postgres:14
    environment:
      POSTGRES_PASSWORD: password
```

**After（リソース制限とヘルスチェックを追加）:**
```yaml
version: '3.8'
services:
  api:
    image: api-server:latest
    ports:
      - "3000:3000"
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
    depends_on:
      database:
        condition: service_healthy
  
  database:
    image: postgres:14
    environment:
      POSTGRES_PASSWORD: password
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 1G
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### 原因3：リバースプロキシの接続バッファ設定が不適切

Nginx などのリバースプロキシが、バックエンドサーバーとの接続バッファを適切に設定していない場合、大容量のレスポンスやスロークライアント向けのデータ送信時に 504 が発生します。

**Before（バッファ設定が最小限の場合）:**
```nginx
upstream backend {
    server web:3000;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend;
        proxy_buffering off;
        proxy_request_buffering off;
    }
}
```

**After（バッファサイズを適切に設定）:**
```nginx
upstream backend {
    server web:3000;
    keepalive 32;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        
        # バッファ設定
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
        proxy_busy_buffers_size 8k;
        
        # タイムアウト設定
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
        
        # Keep-alive設定
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Docker 固有の注意点

**Docker Compose での DNS 遅延：** Docker Compose のサービス名解決が遅延することがあります。`depends_on` の条件に `service_healthy` を指定し、各サービスにヘルスチェックを設定することで、依存関係を明確にしましょう。

**コンテナのログレベルが INFO 以上の場合：** `DOCKER_BUILDKIT=1` や `BUILDKIT_PROGRESS=plain` などのビルド環境変数がタイムアウトに影響することは稀ですが、不要なログ出力でディスクI/Oが圧迫される可能性があります。

**ネットワークドライバーの選択：** デフォルトの bridge ドライバーではなく、`--network` で user-defined network を使用することで、安定した DNS 解決とコンテナ間通信が実現します。Compose では services は自動的にユーザー定義ネットワークに配置されますが、明示的に指定することで予期しない挙動を防げます。

```yaml
version: '3.8'
services:
  web:
    image: myapp:latest
    networks:
      - backend
  
  db:
    image: postgres:14
    networks:
      - backend

networks:
  backend:
    driver: bridge
```

## それでも解決しない場合

**Docker Compose のログを確認する：** `docker-compose logs -f` でサービスのリアルタイムログを確認し、エラーメッセージを詳細に読むことが最初のステップです。特に タイムアウト前のエラーメッセージがあればそれを解析してください。

**プロキシのアクセスログを確認する：** Nginx の場合、`/var/log/nginx/access.log` と `/var/log/nginx/error.log` を確認します。Docker Compose 内では、ボリュームマウントで ホストマシンからログをアクセス可能にしておくと効率的です。

**公式ドキュメント：** Docker Compose の[公式ドキュメント](https://docs.docker.com/compose/)で `timeouts` や `healthcheck` に関するセクションを参照してください。Nginx の詳細設定については [Nginx 公式ドキュメント](https://nginx.org/en/docs/) の proxy module セクションを確認しましょう。

**コミュニティリソース：** GitHub の docker-compose リポジトリ内の Issues で「504」や「timeout」で検索すると、類似事例の解決方法が見つかることがあります。

---

*免責事項：本記事の内容は、執筆時点の公開情報をもとに作成したものです。ソフトウェアの仕様は予告なく変更されることがあります。最新の情報は各ツールの公式サポートページをご確認ください。本記事の情報を利用した結果生じたいかなる損害についても、著者および運営者は責任を負いかねます。*