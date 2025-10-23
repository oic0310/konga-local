# Kong Gateway OSS + Kong Manager ローカル開発環境

このリポジトリは、Kong Gateway OSS と Kong Manager（公式GUI）をDocker Composeで構築するローカル開発環境です。

## 📋 目次

- [概要](#概要)
- [システム構成](#システム構成)
- [前提条件](#前提条件)
- [クイックスタート](#クイックスタート)
- [アクセス情報](#アクセス情報)
- [基本操作](#基本操作)
- [トラブルシューティング](#トラブルシューティング)
- [参考資料](#参考資料)

## 概要

このプロジェクトは、以下の機能を提供します：

- **Kong Gateway OSS**: オープンソースのAPIゲートウェイ（Apache 2.0ライセンス）
- **Kong Manager OSS**: Kong公式の管理GUI（無償版）
- **PostgreSQL**: Kong用のデータベース
- **API認証**: API Key認証
- **レート制限**: トラフィックコントロール
- **レスポンス変換**: URLリライトなど

### 💰 すべて無償・オープンソース

- Kong Gateway OSS: Apache 2.0ライセンス
- Kong Manager: Kong Gateway OSSに含まれる（無償）
- PostgreSQL: PostgreSQL License
- 商用利用可能

## システム構成

```
┌─────────────────────────────────────────────────┐
│                  クライアント                      │
└─────────────────┬───────────────────────────────┘
                  │ HTTP/HTTPS
                  ▼
┌─────────────────────────────────────────────────┐
│          Kong Gateway (port 8000)               │
│  - API Key認証                                   │
│  - レート制限                                     │
│  - レスポンス変換                                  │
│  - プラグイン管理                                  │
└────────┬────────────────────┬───────────────────┘
         │                    │
         │                    ▼
         │          ┌──────────────────┐
         │          │  上流API         │
         │          │  (AppSync等)     │
         │          └──────────────────┘
         ▼
┌──────────────────┐
│ PostgreSQL       │
│ (port 5433)      │
└──────────────────┘

┌─────────────────────────────────────────────────┐
│      Kong Manager (port 8002)                   │
│  - Services管理                                  │
│  - Routes管理                                    │
│  - Consumers管理                                 │
│  - Plugins管理                                   │
│  - ダッシュボード                                 │
└────────┬────────────────────────────────────────┘
         │
         ▼
    Kong Admin API
    (port 8001)
```

### コンポーネント一覧

| コンポーネント | バージョン | ポート | 用途 | ライセンス |
|--------------|----------|--------|------|-----------|
| Kong Gateway | 3.5 | 8000, 8001, 8443, 8444, 8002 | APIゲートウェイ | Apache 2.0 |
| Kong Manager | 3.5 | 8002 | Kong管理GUI | Apache 2.0 |
| PostgreSQL | 14 | 5433 | Kong設定DB | PostgreSQL |

## 前提条件

以下のソフトウェアがインストールされている必要があります：

- **Docker**: 20.10.0 以上
- **Docker Compose**: 2.0.0 以上

### インストール確認

```bash
# Dockerのバージョン確認
docker --version

# Docker Composeのバージョン確認
docker-compose --version
```

### Apple Silicon (M1/M2/M3) Mac の場合

このdocker-compose.ymlはApple Silicon（ARM64）にネイティブ対応しています。
エミュレーション不要で高速に起動します（2-3分程度）。

## docker-compose.yml

以下の内容で`docker-compose.yml`を作成してください：

```yaml
version: '3.8'

services:
  # Kong用のPostgreSQLデータベース
  kong-database:
    image: postgres:14
    container_name: kong-database
    environment:
      POSTGRES_USER: kong
      POSTGRES_DB: kong
      POSTGRES_PASSWORD: kongpass
    ports:
      - "5433:5432"
    volumes:
      - kong_data:/var/lib/postgresql/data
    networks:
      - kong-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U kong"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Kongのデータベースマイグレーション
  kong-migration:
    image: kong:3.5
    container_name: kong-migration
    command: kong migrations bootstrap
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-database
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: kongpass
      KONG_PG_DATABASE: kong
    networks:
      - kong-net
    depends_on:
      kong-database:
        condition: service_healthy
    restart: on-failure

  # Kong Gateway本体 + Kong Manager
  kong:
    image: kong:3.5
    container_name: kong-gateway
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-database
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: kongpass
      KONG_PG_DATABASE: kong
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: 0.0.0.0:8001
      # Kong Manager OSS を有効化
      KONG_ADMIN_GUI_LISTEN: 0.0.0.0:8002
      KONG_ADMIN_GUI_URL: http://localhost:8002
    ports:
      - "8000:8000"  # プロキシポート(HTTPリクエスト受付)
      - "8443:8443"  # プロキシポート(HTTPS)
      - "8001:8001"  # Admin API
      - "8444:8444"  # Admin API(HTTPS)
      - "8002:8002"  # Kong Manager GUI
    networks:
      - kong-net
    depends_on:
      kong-database:
        condition: service_healthy
      kong-migration:
        condition: service_completed_successfully
    healthcheck:
      test: ["CMD", "kong", "health"]
      interval: 10s
      timeout: 10s
      retries: 10
    restart: always

networks:
  kong-net:
    driver: bridge

volumes:
  kong_data:
```

## クイックスタート

### 1. セットアップ

```bash
# ディレクトリ作成
mkdir kong-local
cd kong-local

# docker-compose.ymlを作成（上記内容をコピー）
nano docker-compose.yml
```

### 2. 環境の起動

```bash
# バックグラウンドで起動
docker-compose up -d

# 起動状態を確認
docker-compose ps

# ログを確認（Ctrl+Cで終了）
docker-compose logs -f
```

起動完了の目安（約2-3分）:
```
kong-gateway       | [Kong] started
kong-database      | database system is ready to accept connections
```

### 3. Kong動作確認

```bash
# Kong Admin APIの疎通確認
curl http://localhost:8001/status

# 期待される出力:
# {
#   "database": {
#     "reachable": true
#   },
#   "server": {
#     "connections_active": 1,
#     ...
#   }
# }
```

### 4. Kong Managerにアクセス

ブラウザで以下にアクセス:
```
http://localhost:8002
```

Kong Managerのダッシュボードが表示されます。

**認証不要**: Kong Manager OSSは認証なしで使用できます（Enterprise版では認証機能があります）。

これで準備完了です！

## アクセス情報

### エンドポイント

| サービス | URL | 用途 |
|---------|-----|------|
| **Kong Proxy** | http://localhost:8000 | APIリクエスト受付 |
| **Kong Proxy (HTTPS)** | https://localhost:8443 | APIリクエスト受付（SSL） |
| **Kong Admin API** | http://localhost:8001 | プログラマティック管理 |
| **Kong Manager** | http://localhost:8002 | グラフィカル管理画面 |

### データベース接続情報

#### Kong用PostgreSQL

```
Host: localhost
Port: 5433
User: kong
Password: kongpass
Database: kong
```

接続例:
```bash
psql -h localhost -p 5433 -U kong -d kong
```

または:
```bash
docker exec -it kong-database psql -U kong -d kong
```

### デフォルト認証情報

| サービス | 認証 |
|---------|------|
| Kong Manager | 認証なし（OSS版は認証機能なし） |
| Kong PostgreSQL | User: `kong` / Password: `kongpass` |

⚠️ **セキュリティ注意**: 本番環境では必ず強固なパスワードに変更してください。

## 基本操作

### Kong Managerでの操作

#### 1. Serviceの作成

1. Kong Managerにアクセス: http://localhost:8002
2. 左メニュー「**Services**」をクリック
3. 「**New Service**」ボタンをクリック
4. 以下を入力:
   - **Name**: `test-service`
   - **Protocol**: `https`
   - **Host**: `httpbin.org`
   - **Port**: `443`
   - **Path**: `/` （オプション）
5. 「**Create**」ボタンをクリック

#### 2. Routeの作成

1. 作成したService（test-service）をクリック
2. 「**Routes**」タブをクリック
3. 「**New Route**」ボタンをクリック
4. 以下を入力:
   - **Name**: `test-route`
   - **Protocols**: `http`, `https` を選択
   - **Paths**: `/test` と入力
   - **Methods**: `GET`, `POST` を選択
5. 「**Create**」ボタンをクリック

#### 3. 動作確認

```bash
# 作成したRouteにアクセス
curl http://localhost:8000/test/get

# 期待される出力: httpbin.orgからのJSONレスポンス
# {
#   "args": {},
#   "headers": {
#     "Host": "httpbin.org",
#     ...
#   },
#   "url": "https://httpbin.org/get"
# }
```

### API Key認証の設定

#### 1. Key Authプラグインの追加

1. Service詳細画面で「**Plugins**」タブをクリック
2. 「**New Plugin**」ボタンをクリック
3. 「**Authentication**」カテゴリから「**Key Auth**」を選択
4. 設定:
   - **Key Names**: `apikey` （デフォルトのまま、またはカスタマイズ）
   - **Hide Credentials**: チェックON（推奨）
5. 「**Create**」ボタンをクリック

#### 2. Consumerの作成

1. 左メニュー「**Consumers**」をクリック
2. 「**New Consumer**」ボタンをクリック
3. 以下を入力:
   - **Username**: `test-client`
   - **Custom ID**: `org-test-client` （オプション）
4. 「**Create**」ボタンをクリック

#### 3. API Keyの発行

1. 作成したConsumer（test-client）をクリック
2. 「**Credentials**」タブをクリック
3. 「**New Key Auth Credential**」ボタンをクリック
4. 設定:
   - **Key**: `my-secret-api-key-12345` （任意の文字列、空欄の場合は自動生成）
5. 「**Create**」ボタンをクリック

#### 4. 認証テスト

```bash
# API Keyなし → 401エラー
curl -i http://localhost:8000/test/get
# HTTP/1.1 401 Unauthorized
# {"message":"No API key found in request"}

# API Key付き → 成功
curl -i -H "apikey: my-secret-api-key-12345" http://localhost:8000/test/get
# HTTP/1.1 200 OK

# 間違ったAPI Key → 401エラー
curl -i -H "apikey: wrong-key" http://localhost:8000/test/get
# HTTP/1.1 401 Unauthorized
```

### Rate Limitingの設定

#### 1. Rate Limitingプラグインの追加

1. Service詳細画面で「**Plugins**」タブをクリック
2. 「**New Plugin**」ボタンをクリック
3. 「**Traffic Control**」カテゴリから「**Rate Limiting**」を選択
4. 設定:
   - **Config.Minute**: `100` （1分あたり100リクエスト）
   - **Config.Policy**: `local`
5. 「**Create**」ボタンをクリック

#### 2. テスト

```bash
# レート制限のテスト
for i in {1..105}; do
  echo "Request $i:"
  curl -s -o /dev/null -w "%{http_code}\n" \
    -H "apikey: my-secret-api-key-12345" \
    http://localhost:8000/test/get
  sleep 0.1
done

# 結果:
# Request 1-100: 200 OK
# Request 101-105: 429 Too Many Requests
```

レスポンスヘッダーで残り回数を確認:
```bash
curl -i -H "apikey: my-secret-api-key-12345" http://localhost:8000/test/get

# ヘッダーに以下が含まれる:
# X-RateLimit-Limit-Minute: 100
# X-RateLimit-Remaining-Minute: 99
# RateLimit-Limit: 100
# RateLimit-Remaining: 99
# RateLimit-Reset: 45
```

### Kong Admin API経由の操作

Kong ManagerのGUIを使わず、Admin APIで直接操作することも可能です。

#### Service作成

```bash
curl -i -X POST http://localhost:8001/services \
  --data name=api-service \
  --data url=https://httpbin.org
```

#### Route作成

```bash
curl -i -X POST http://localhost:8001/services/api-service/routes \
  --data paths[]=/api \
  --data name=api-route
```

#### Consumer作成

```bash
curl -i -X POST http://localhost:8001/consumers \
  --data username=client-a \
  --data custom_id=org-client-a
```

#### API Key発行

```bash
curl -i -X POST http://localhost:8001/consumers/client-a/key-auth \
  --data key=client-a-secret-key
```

#### プラグイン追加

```bash
# Key Authentication
curl -i -X POST http://localhost:8001/services/api-service/plugins \
  --data name=key-auth

# Rate Limiting
curl -i -X POST http://localhost:8001/services/api-service/plugins \
  --data name=rate-limiting \
  --data config.minute=100 \
  --data config.policy=local
```

#### 全設定の確認

```bash
# すべてのServicesを確認
curl http://localhost:8001/services | jq

# すべてのRoutesを確認
curl http://localhost:8001/routes | jq

# すべてのConsumersを確認
curl http://localhost:8001/consumers | jq

# すべてのPluginsを確認
curl http://localhost:8001/plugins | jq
```

## 運用コマンド

### 起動・停止

```bash
# 起動
docker-compose up -d

# 停止
docker-compose down

# 停止 + ボリューム削除（データも削除）
docker-compose down -v

# 再起動
docker-compose restart kong

# 特定サービスのみ再起動
docker-compose restart kong
```

### ログ確認

```bash
# すべてのログを表示
docker-compose logs

# Kong Gatewayのログ
docker-compose logs kong

# リアルタイムでログを追跡
docker-compose logs -f kong

# 最新100行のみ表示
docker-compose logs --tail=100 kong
```

### ステータス確認

```bash
# コンテナの状態確認
docker-compose ps

# Kong Gateway の状態
curl http://localhost:8001/status

# Kongの全設定を確認
curl http://localhost:8001/ | jq

# Kongのバージョン確認
curl http://localhost:8001/ | jq '.version'
```

### データベース操作

```bash
# Kong DBに接続
docker exec -it kong-database psql -U kong -d kong

# テーブル一覧
docker exec -it kong-database psql -U kong -d kong -c "\dt"

# Services一覧をSQLで確認
docker exec -it kong-database psql -U kong -d kong -c "SELECT * FROM services;"

# Consumers一覧をSQLで確認
docker exec -it kong-database psql -U kong -d kong -c "SELECT * FROM consumers;"
```

### 設定のバックアップ・リストア

#### decKを使った設定管理（推奨）

```bash
# decKのインストール（macOS）
brew install deck

# Kong設定をYAMLファイルにエクスポート
deck dump --kong-addr http://localhost:8001 --output-file kong-backup.yaml

# エクスポートしたYAMLの内容確認
cat kong-backup.yaml

# Kong設定をYAMLファイルからインポート
deck sync --kong-addr http://localhost:8001 --state kong-backup.yaml

# 差分確認（実際に適用せずに確認のみ）
deck diff --kong-addr http://localhost:8001 --state kong-backup.yaml
```

#### データベース直接バックアップ

```bash
# Kong DBのバックアップ
docker exec kong-database pg_dump -U kong kong > kong-backup-$(date +%Y%m%d).sql

# Kong DBのリストア
cat kong-backup-20250101.sql | docker exec -i kong-database psql -U kong -d kong

# バックアップファイルの確認
ls -lh kong-backup-*.sql
```

## decK設定管理の例

### kong.yaml サンプル

```yaml
_format_version: "3.0"

services:
  - name: test-service
    url: https://httpbin.org
    protocol: https
    host: httpbin.org
    port: 443
    path: /
    
    routes:
      - name: test-route
        paths:
          - /test
        methods:
          - GET
          - POST
        protocols:
          - http
          - https
    
    plugins:
      - name: key-auth
        config:
          key_names:
            - apikey
          hide_credentials: true
      
      - name: rate-limiting
        config:
          minute: 100
          policy: local

consumers:
  - username: test-client
    custom_id: org-test-client
    keyauth_credentials:
      - key: my-secret-api-key-12345
```

### decKでの運用フロー

```bash
# 1. 現在の設定をエクスポート
deck dump --kong-addr http://localhost:8001 --output-file kong.yaml

# 2. kong.yamlを編集（VSCodeなど）
code kong.yaml

# 3. 差分を確認
deck diff --kong-addr http://localhost:8001 --state kong.yaml

# 4. 設定を適用
deck sync --kong-addr http://localhost:8001 --state kong.yaml

# 5. Gitで管理
git add kong.yaml
git commit -m "Add new service and route"
git push
```

## トラブルシューティング

### ポート競合エラー

```
Error: Bind for 0.0.0.0:5432 failed: port is already allocated
```

**解決方法**: `docker-compose.yml`のポート番号を変更してください。

```yaml
ports:
  - "5433:5432"  # 左側のポート番号を変更（5432→5433）
```

### Kong Managerにアクセスできない

**原因1**: Kongがまだ起動していない

```bash
# 起動状態を確認
docker-compose ps

# ログでエラーを確認
docker-compose logs kong
```

**原因2**: ブラウザのキャッシュ

```bash
# ブラウザでハードリロード
# Chrome/Edge: Cmd+Shift+R (Mac) / Ctrl+Shift+R (Windows)
# Safari: Cmd+Option+R
```

### コンテナが起動しない

```bash
# ログで原因を確認
docker-compose logs

# Kong Gatewayの詳細ログ
docker logs kong-gateway

# コンテナの状態確認
docker-compose ps

# ヘルスチェック状態
docker inspect kong-gateway | jq '.[0].State.Health'
```

### データベース接続エラー

```bash
# Kong DBの接続テスト
docker exec -it kong-database psql -U kong -d kong -c "SELECT version();"

# PostgreSQLのログ確認
docker-compose logs kong-database

# ネットワーク確認
docker network inspect kong-local_kong-net
```

### Kong Admin APIが応答しない

```bash
# Kongのヘルスチェック
curl http://localhost:8001/status

# Kongの起動を確認
docker exec kong-gateway kong health

# Kongの設定確認
docker exec kong-gateway kong config -c /etc/kong/kong.conf parse
```

### 設定が反映されない

```bash
# Kongの再起動
docker-compose restart kong

# Kong設定のリロード（ダウンタイムなし）
docker exec kong-gateway kong reload

# Admin APIから確認
curl http://localhost:8001/services
curl http://localhost:8001/routes
curl http://localhost:8001/plugins
```

### 完全リセット

すべてをクリーンな状態に戻す:

```bash
# コンテナ停止・削除（ボリュームも削除）
docker-compose down -v

# 未使用のDockerリソースをクリーンアップ
docker system prune -f

# Kongイメージを最新に更新
docker pull kong:3.5

# 再起動
docker-compose up -d
```

### パフォーマンスが遅い

```bash
# リソース使用状況を確認
docker stats

# 不要なコンテナを削除
docker container prune -f

# 不要なイメージを削除
docker image prune -f

# Kongのワーカー数を増やす（docker-compose.ymlに追加）
environment:
  KONG_NGINX_WORKER_PROCESSES: "4"
```

### Apple Silicon (M1/M2/M3) での問題

このdocker-compose.ymlはARM64ネイティブ対応なので、通常は問題ありません。

もしプラットフォーム警告が出る場合:

```yaml
kong:
  image: kong:3.5
  platform: linux/arm64  # 明示的に指定
```

## Kong Manager OSSの制限事項

Kong Manager OSSは無償版のため、以下の機能は含まれません：

| 機能 | OSS版 | Enterprise版 |
|------|-------|-------------|
| 基本的な管理機能 | ✅ | ✅ |
| ユーザー認証・RBAC | ❌ | ✅ |
| ワークスペース | ❌ | ✅ |
| 高度な監視・分析 | ❌ | ✅ |
| Dev Portal | ❌ | ✅ |
| エンタープライズプラグイン | ❌ | ✅ |

OSS版で十分な場合がほとんどです。必要に応じてEnterprise版の評価版を試すこともできます。

## 設定ファイル構造

```
kong-local/
├── docker-compose.yml          # メイン設定ファイル
├── kong.yaml                   # decK設定ファイル（オプション）
├── kong-backup.yaml            # Kong設定バックアップ
├── kong-backup-20250101.sql    # DBバックアップ
├── README.md                   # このファイル
└── volumes/                    # 永続化データ（自動作成）
    └── kong_data/              # Kong PostgreSQLデータ
```

## 環境変数のカスタマイズ

`docker-compose.yml`で以下の環境変数を変更できます：

```yaml
environment:
  # 基本設定
  KONG_DATABASE: postgres
  KONG_PG_HOST: kong-database
  KONG_PG_USER: kong
  KONG_PG_PASSWORD: kongpass
  KONG_PG_DATABASE: kong
  
  # Admin API設定
  KONG_ADMIN_LISTEN: 0.0.0.0:8001
  
  # Kong Manager設定
  KONG_ADMIN_GUI_LISTEN: 0.0.0.0:8002
  KONG_ADMIN_GUI_URL: http://localhost:8002
  
  # ログ設定
  KONG_PROXY_ACCESS_LOG: /dev/stdout
  KONG_ADMIN_ACCESS_LOG: /dev/stdout
  KONG_PROXY_ERROR_LOG: /dev/stderr
  KONG_ADMIN_ERROR_LOG: /dev/stderr
  
  # パフォーマンス設定
  KONG_NGINX_WORKER_PROCESSES: "auto"  # CPUコア数に応じて自動
  KONG_UPSTREAM_KEEPALIVE_MAX_REQUESTS: 100
  KONG_UPSTREAM_KEEPALIVE_POOL_SIZE: 60
```

## 本番環境への移行

このローカル環境から本番環境（AWS等）に移行する際の注意点：

### 1. セキュリティ強化

- [ ] すべてのデフォルトパスワードを変更
- [ ] TLS/SSL証明書の設定
- [ ] Admin API (8001) への外部アクセスを制限
- [ ] Kong Manager (8002) への外部アクセスを制限またはEnterprise版のRBAC導入
- [ ] PostgreSQLへの外部アクセスを禁止

### 2. データベース

- [ ] RDS Multi-AZ構成
- [ ] 自動バックアップの有効化
- [ ] 暗号化の有効化
- [ ] パフォーマンスモニタリング

### 3. Kong Gateway

- [ ] ECS Fargate または EKS
- [ ] ALB/NLBの設定
- [ ] Multi-AZ配置
- [ ] Auto Scaling設定
- [ ] CloudWatch監視の設定
- [ ] X-Ray分散トレーシング

### 4. 設定管理

- [ ] decKによる宣言的設定管理
- [ ] GitOpsによるバージョン管理
- [ ] CI/CDパイプラインの構築
- [ ] Terraform/CloudFormationでのIaC化

### 5. モニタリング

- [ ] Prometheus + Grafana
- [ ] CloudWatch Logs/Metrics
- [ ] アラート設定
- [ ] ログ集約（CloudWatch Logs Insights等）

## よくある質問 (FAQ)

### Q1: Kong Manager OSSは商用利用できますか？

**A**: はい、Apache 2.0ライセンスで商用利用可能です。

### Q2: Kong Manager OSSとEnterprise版の違いは？

**A**: OSS版は基本的な管理機能のみ。Enterprise版はRBAC、ワークスペース、高度な監視機能などが追加されます。

### Q3: Kongaとの違いは？

**A**: 
- Kong Manager: Kong公式、高速、ARM64ネイティブ対応
- Konga: コミュニティ製、機能豊富、ARM64非対応（遅い）

### Q4: decKとは？

**A**: Kongの設定をYAMLファイルで宣言的に管理するCLIツール。GitOpsに最適。

### Q5: データベースなしで動きますか？

**A**: はい、DB-lessモードで動作可能ですが、動的な設定変更ができません。

```yaml
kong:
  environment:
    KONG_DATABASE: "off"
    KONG_DECLARATIVE_CONFIG: /kong/declarative/kong.yml
```

### Q6: 複数のKongインスタンスを起動できますか？

**A**: はい、同じPostgreSQLを共有して複数のKongインスタンスを起動できます（スケールアウト）。

### Q7: プラグインを追加できますか？

**A**: はい、Kong Hubから様々なプラグインを利用できます。カスタムプラグインも作成可能です。

## 参考資料

### 公式ドキュメント

- [Kong Gateway公式ドキュメント](https://docs.konghq.com/)
- [Kong Manager OSS](https://docs.konghq.com/gateway/latest/kong-manager-oss/)
- [decK公式ドキュメント](https://docs.konghq.com/deck/)
- [Kong Admin API](https://docs.konghq.com/gateway/latest/admin-api/)

### プラグインドキュメント

- [Key Authentication](https://docs.konghq.com/hub/kong-inc/key-auth/)
- [Rate Limiting](https://docs.konghq.com/hub/kong-inc/rate-limiting/)
- [Response Transformer](https://docs.konghq.com/hub/kong-inc/response-transformer/)
- [Request Transformer](https://docs.konghq.com/hub/kong-inc/request-transformer/)
- [CORS](https://docs.konghq.com/hub/kong-inc/cors/)
- [JWT](https://docs.konghq.com/hub/kong-inc/jwt/)
- [OAuth 2.0](https://docs.konghq.com/hub/kong-inc/oauth2/)

### Kong Hubでプラグイン検索

- [Kong Plugin Hub](https://docs.konghq.com/hub/)

### コミュニティ

- [Kong Discussions](https://github.com/Kong/kong/discussions)
- [Kong Community Forum](https://discuss.konghq.com/)
- [Kong GitHub](https://github.com/Kong/kong)

### チュートリアル

- [Kong Getting Started](https://docs.konghq.com/gateway/latest/get-started/)
- [Kong Plugin Development](https://docs.konghq.com/gateway/latest/plugin-development/)

## ライセンス

- Kong Gateway OSS: Apache 2.0
- Kong Manager OSS: Apache 2.0（Kong Gateway OSSに含まれる）
- PostgreSQL: PostgreSQL License

## サポート

問題が発生した場合：

1. まず[トラブルシューティング](#トラブルシューティング)を確認
2. ログを確認: `docker-compose logs kong`
3. [Kong公式ドキュメント](https://docs.konghq.com/)を参照
4. [Kong Discussions](https://github.com/Kong/kong/discussions)で質問
5. GitHubでIssueを作成

---

**最終更新**: 2025年10月24日  
**Kong Gateway バージョン**: 3.5  
**対応プラットフォーム**: macOS (Intel/Apple Silicon), Linux, Windows
