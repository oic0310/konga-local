# Kong Gateway + Konga ローカル開発環境

このリポジトリは、Kong Gateway と Konga 管理UIをDocker Composeで構築するローカル開発環境です。

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

- **Kong Gateway**: オープンソースのAPIゲートウェイ
- **Konga**: Kong管理用のWebベースUI
- **PostgreSQL**: Kong/Konga用のデータベース（2つの独立したインスタンス）
- **API認証**: API Key認証
- **レート制限**: トラフィックコントロール
- **レスポンス変換**: URLリライトなど

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
│           Konga UI (port 1337)                  │
│  - Kong設定管理                                  │
│  - Consumer管理                                  │
│  - プラグイン設定                                 │
└────────┬────────────────────────────────────────┘
         ▼
┌──────────────────┐
│ PostgreSQL       │
│ (port 5434)      │
└──────────────────┘
```

### コンポーネント一覧

| コンポーネント | バージョン | ポート | 用途 |
|--------------|----------|--------|------|
| Kong Gateway | 3.5 | 8000, 8001, 8443, 8444 | APIゲートウェイ |
| Konga | latest | 1337 | Kong管理UI |
| PostgreSQL (Kong) | 14 | 5433 | Kong設定DB |
| PostgreSQL (Konga) | 14 | 5434 | Konga設定DB |

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

## クイックスタート

### 1. リポジトリのクローン（または作成）

```bash
# ディレクトリ作成
mkdir kong-local
cd kong-local

# docker-compose.ymlをダウンロードまたは配置
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

起動完了の目安（ログに以下が表示される）:
```
konga              | Konga started on port 1337
kong-gateway       | [Kong] started
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
#   ...
# }
```

### 4. Konga初回セットアップ

1. ブラウザで http://localhost:1337 にアクセス

2. 管理者アカウントを作成:
   - Username: `admin`
   - Email: `admin@example.com`
   - Password: `admin123456` (8文字以上)

3. ログイン後、Kong接続を作成:
   - Connection Name: `Local Kong`
   - Kong Admin URL: `http://kong:8001` ⚠️重要: `localhost`ではなく`kong`
   - 「CREATE CONNECTION」をクリック
   - 「ACTIVATE」をクリック

これで準備完了です！

## アクセス情報

### エンドポイント

| サービス | URL | 用途 |
|---------|-----|------|
| **Kong Proxy** | http://localhost:8000 | APIリクエスト受付 |
| **Kong Proxy (HTTPS)** | https://localhost:8443 | APIリクエスト受付（SSL） |
| **Kong Admin API** | http://localhost:8001 | プログラマティック管理 |
| **Konga Web UI** | http://localhost:1337 | グラフィカル管理画面 |

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

#### Konga用PostgreSQL

```
Host: localhost
Port: 5434
User: konga
Password: kongapass
Database: konga
```

接続例:
```bash
psql -h localhost -p 5434 -U konga -d konga
```

### デフォルト認証情報

| サービス | ユーザー名 | パスワード |
|---------|-----------|-----------|
| Konga管理画面 | admin@example.com | admin123456 |
| Kong PostgreSQL | kong | kongpass |
| Konga PostgreSQL | konga | kongapass |

⚠️ **セキュリティ注意**: 本番環境では必ず強固なパスワードに変更してください。

## 基本操作

### Service作成（Konga UI）

1. Kongaにログイン: http://localhost:1337
2. 左メニュー「SERVICES」→「+ ADD NEW SERVICE」
3. 以下を入力:
   - Name: `test-service`
   - Protocol: `https`
   - Host: `httpbin.org`
   - Port: `443`
4. 「SUBMIT SERVICE」をクリック

### Route作成（Konga UI）

1. Service詳細画面で「Routes」タブをクリック
2. 「+ ADD ROUTE」をクリック
3. 以下を入力:
   - Name: `test-route`
   - Paths: `/test`
   - Methods: GET, POST をチェック
4. 「SUBMIT ROUTE」をクリック

### 動作確認

```bash
# 作成したRouteにアクセス
curl http://localhost:8000/test/get

# 期待される出力: httpbin.orgからのJSONレスポンス
```

### API Key認証の設定

#### 1. プラグイン追加（Konga UI）

1. Service詳細画面で「Plugins」タブをクリック
2. 「+ ADD PLUGIN」→「Security」→「Key Authentication」
3. key_names: `apikey` と入力
4. 「ADD PLUGIN」をクリック

#### 2. Consumer作成（Konga UI）

1. 左メニュー「CONSUMERS」→「+ CREATE CONSUMER」
2. 以下を入力:
   - Username: `test-client`
   - Custom ID: `org-test-client`
3. 「SUBMIT CONSUMER」をクリック

#### 3. API Key発行（Konga UI）

1. Consumer詳細画面で「Credentials」タブをクリック
2. 「API KEYS」セクションの「+」ボタンをクリック
3. Key: `my-secret-api-key-12345` と入力
4. 「SUBMIT」をクリック

#### 4. 認証テスト

```bash
# API Keyなし → 401エラー
curl -i http://localhost:8000/test/get

# API Key付き → 成功
curl -i -H "apikey: my-secret-api-key-12345" http://localhost:8000/test/get
```

### Rate Limiting設定

#### Konga UIで設定

1. Service詳細画面で「Plugins」タブをクリック
2. 「+ ADD PLUGIN」→「Traffic Control」→「Rate Limiting」
3. 以下を入力:
   - minute: `100` (1分あたり100リクエスト)
   - policy: `local`
4. 「ADD PLUGIN」をクリック

#### テスト

```bash
# レート制限のテスト（101回目で429エラーになる）
for i in {1..105}; do
  curl -s -o /dev/null -w "Request $i: %{http_code}\n" \
    -H "apikey: my-secret-api-key-12345" \
    http://localhost:8000/test/get
  sleep 0.1
done
```

### Kong Admin API経由の操作

Konga UIを使わず、直接APIで操作することも可能です。

#### Service作成

```bash
curl -i -X POST http://localhost:8001/services \
  --data name=api-service \
  --data url=https://httpbin.org
```

#### Route作成

```bash
curl -i -X POST http://localhost:8001/services/api-service/routes \
  --data paths[]=/api
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
  --data config.minute=100
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
docker-compose restart
```

### ログ確認

```bash
# すべてのログを表示
docker-compose logs

# 特定のサービスのログ
docker-compose logs kong
docker-compose logs konga

# リアルタイムでログを追跡
docker-compose logs -f

# 最新100行のみ表示
docker-compose logs --tail=100
```

### ステータス確認

```bash
# コンテナの状態確認
docker-compose ps

# Kong Gateway の状態
curl http://localhost:8001/status

# Kongの全設定を確認
curl http://localhost:8001/ | jq
```

### データベース操作

```bash
# Kong DBに接続
docker exec -it kong-database psql -U kong -d kong

# Konga DBに接続
docker exec -it konga-database psql -U konga -d konga

# Kong DBのテーブル一覧
docker exec -it kong-database psql -U kong -d kong -c "\dt"
```

### 設定のバックアップ・リストア

#### decKを使った設定バックアップ（推奨）

```bash
# decKのインストール（macOS）
brew install deck

# Kong設定をエクスポート
deck dump --kong-addr http://localhost:8001 --output-file kong-backup.yaml

# Kong設定をインポート
deck sync --kong-addr http://localhost:8001 --state kong-backup.yaml
```

#### データベースバックアップ

```bash
# Kong DBのバックアップ
docker exec kong-database pg_dump -U kong kong > kong-backup.sql

# Kong DBのリストア
cat kong-backup.sql | docker exec -i kong-database psql -U kong -d kong
```

## トラブルシューティング

### ポート競合エラー

```
Error: Bind for 0.0.0.0:5432 failed: port is already allocated
```

**解決方法**: `docker-compose.yml`のポート番号を変更してください。

```yaml
ports:
  - "5433:5432"  # 左側のポート番号を変更
```

### Kongaが「Failed to connect to Kong」エラー

**原因**: Kong Admin URLの設定が間違っている

**解決方法**: 
- ❌ `http://localhost:8001`
- ✅ `http://kong:8001`

Docker内部のサービス名 `kong` を使用してください。

### コンテナが起動しない

```bash
# ログで原因を確認
docker-compose logs

# 特定のコンテナの詳細ログ
docker logs kong-gateway
docker logs konga

# コンテナの状態確認
docker-compose ps

# ヘルスチェック状態
docker inspect kong-gateway | jq '.[0].State.Health'
```

### データベース接続エラー

```bash
# Kong DBの接続テスト
docker exec -it kong-database psql -U kong -d kong -c "SELECT version();"

# Konga DBの接続テスト
docker exec -it konga-database psql -U konga -d konga -c "SELECT version();"

# ネットワーク確認
docker network inspect kong-local_kong-net
```

### 完全リセット

すべてをクリーンな状態に戻す:

```bash
# コンテナ停止・削除
docker-compose down -v

# 未使用のDockerリソースをクリーンアップ
docker system prune -f

# 再起動
docker-compose up -d
```

### Kong設定が反映されない

```bash
# Kongの再起動
docker-compose restart kong

# 設定を強制的に再読み込み
curl -X POST http://localhost:8001/config?check_hash=1
```

### パフォーマンスが遅い

```bash
# リソース使用状況を確認
docker stats

# 不要なコンテナを削除
docker container prune

# 不要なイメージを削除
docker image prune
```

## 設定ファイル構造

```
kong-local/
├── docker-compose.yml          # メイン設定ファイル
├── kong-backup.yaml            # Kong設定バックアップ（decK形式）
├── README.md                   # このファイル
└── volumes/                    # 永続化データ（自動作成）
    ├── kong_data/              # Kong PostgreSQLデータ
    └── konga_data/             # Konga PostgreSQLデータ
```

## 環境変数のカスタマイズ

`docker-compose.yml`で以下の環境変数を変更できます：

```yaml
environment:
  # Kong設定
  KONG_ADMIN_LISTEN: 0.0.0.0:8001  # Admin APIリッスンアドレス
  KONG_PROXY_ACCESS_LOG: /dev/stdout  # プロキシアクセスログ
  KONG_ADMIN_ACCESS_LOG: /dev/stdout  # 管理アクセスログ
  
  # PostgreSQL設定
  POSTGRES_USER: kong
  POSTGRES_PASSWORD: kongpass
  POSTGRES_DB: kong
```

## 本番環境への移行

このローカル環境から本番環境（AWS等）に移行する際の注意点：

### 1. セキュリティ強化

- [ ] すべてのデフォルトパスワードを変更
- [ ] TLS/SSL証明書の設定
- [ ] 管理API (8001) への外部アクセスを制限
- [ ] PostgreSQLへの外部アクセスを禁止

### 2. データベース

- [ ] RDS Multi-AZ構成
- [ ] 自動バックアップの有効化
- [ ] 暗号化の有効化

### 3. Kong Gateway

- [ ] ECS Fargate または EC2 Auto Scaling
- [ ] ALB/NLBの設定
- [ ] Multi-AZ配置
- [ ] CloudWatch監視の設定

### 4. 設定管理

- [ ] decKによる宣言的設定管理
- [ ] GitOpsによるバージョン管理
- [ ] CI/CDパイプラインの構築

## 参考資料

### 公式ドキュメント

- [Kong Gateway公式ドキュメント](https://docs.konghq.com/)
- [Konga GitHub](https://github.com/pantsel/konga)
- [decK公式ドキュメント](https://docs.konghq.com/deck/)

### プラグインドキュメント

- [Key Authentication](https://docs.konghq.com/hub/kong-inc/key-auth/)
- [Rate Limiting](https://docs.konghq.com/hub/kong-inc/rate-limiting/)
- [Response Transformer](https://docs.konghq.com/hub/kong-inc/response-transformer/)
- [CORS](https://docs.konghq.com/hub/kong-inc/cors/)
- [Request Transformer](https://docs.konghq.com/hub/kong-inc/request-transformer/)

### コミュニティ

- [Kong Discussions](https://github.com/Kong/kong/discussions)
- [Kong Community Forum](https://discuss.konghq.com/)

## ライセンス

- Kong Gateway: Apache 2.0
- Konga: MIT
- PostgreSQL: PostgreSQL License

## サポート

問題が発生した場合：

1. まず[トラブルシューティング](#トラブルシューティング)を確認
2. ログを確認: `docker-compose logs`
3. Kong公式ドキュメントを参照
4. GitHubでIssueを作成

---

**最終更新**: 2025年10月23日
