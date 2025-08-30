
## License

This project is based on [Dify](https://github.com/langgenius/dify) by [LangGenius, Inc.](https://github.com/langgenius), which is licensed under a modified version of the Apache License 2.0.

**Important Notes:**
- This is a fork of the original Dify project with Podman support
- Original copyright and license terms apply
- For commercial multi-tenant use, please refer to the original license terms
- Original project: https://github.com/langgenius/dify

### License Details
- **Original License**: Modified Apache License 2.0
- **Original Copyright**: © 2025 LangGenius, Inc.
- **Additional Conditions**: See [LICENSE](LICENSE) file for full details



# WSL用Difyセットアップ手順（Podman対応版）

## 前提条件
- Windows
- WSL2
- Git

## セットアップ手順

まずはPodmanのインストールを行います。

### 1. Podmanのインストールと設定
```bash
# システムの更新
sudo apt update

# Podmanのインストール
sudo apt install -y podman

# Python pip とpipxのインストール（podman-compose用）
sudo apt install -y python3-pip pipx

# podman-composeのインストール
pipx install podman-compose

# Podmanの設定確認
podman info | grep -i network
```

### 2. Difyリポジトリのクローン
```bash
git clone https://github.com/langgenius/dify.git
cd dify/docker
```

### 3. 環境変数ファイルの作成
```bash
cp .env.example .env

# Rootless対応のポート設定（1024未満のポートは使用不可）
sed -i 's/EXPOSE_NGINX_PORT=80/EXPOSE_NGINX_PORT=8080/' .env
sed -i 's/EXPOSE_NGINX_SSL_PORT=443/EXPOSE_NGINX_SSL_PORT=8443/' .env

# 設定の確認
grep "EXPOSE_NGINX" .env
```

### 4. 短縮名解決の設定
Podmanで短縮名（例：`postgres:15-alpine`）が解決されない問題を解決します。

```bash
# registries.confファイルを編集
sudo tee /etc/containers/registries.conf <<EOF
[registries.search]
registries = ['docker.io']
EOF

# 設定を確認
cat /etc/containers/registries.conf
```

### 5. ネットワークの作成
```bash
# メインネットワークの作成
podman network create dify_main_network

# ネットワーク一覧の確認
podman network ls
```

### 6. 設定の確認
```bash
# docker-compose.yamlの設定を確認
podman-compose config

# ネットワーク接続確認
podman ps --format "table {{.Names}}\t{{.Networks}}" 2>/dev/null || echo "まだコンテナが起動していません"
```

### 7. Difyサービスの起動
```bash
# バックグラウンドで全サービスを起動
podman-compose up -d

# 起動状況の確認
podman-compose ps
```

### 8. ログの確認
```bash
# 全サービスのログを確認
podman-compose logs -f

# 特定のサービスのログを確認
podman-compose logs -f api
podman-compose logs -f web
podman-compose logs -f db
podman-compose logs -f redis
```

### 9. Difyへのアクセス
- ブラウザで`http://localhost:8080/install`にアクセス
- 初期セットアップを完了

## トラブルシューティング

### 問題1: 短縮名が解決されない
**症状**: `Error: short-name "postgres:15-alpine" did not resolve to an alias`

**解決方法**:
```bash
# 手順3の短縮名解決の設定を再実行
sudo tee /etc/containers/registries.conf <<EOF
[registries.search]
registries = ['docker.io']
EOF
```

### 問題2: コンテナが起動しない
**症状**: コンテナが作成されない、または起動に失敗する

**解決方法**:
```bash
# 既存のコンテナとネットワークをクリーンアップ
podman-compose down
podman stop $(podman ps -q) 2>/dev/null || true
podman rm $(podman ps -aq) 2>/dev/null || true

# ネットワークを再作成
podman network rm dify_main_network 2>/dev/null || true
podman network create dify_main_network

# 再実行
podman-compose up -d
```

### 問題3: ポートが使用できない
**症状**: `Error: port is already in use`

**解決方法**:
```bash
# 使用中のポートを確認
netstat -tlnp | grep :8080

# 必要に応じてポート番号を変更
sed -i 's/EXPOSE_NGINX_PORT=8080/EXPOSE_NGINX_PORT=8081/' .env
```

### 問題4: plugin_daemonのネットワーク接続エラー
**症状**: `plugin_daemon`が`dify_main_network`に接続されない、または他のサービスと通信できない

**解決方法**:
```bash
# 1. 現在のネットワーク接続を確認
podman ps --format "table {{.Names}}\t{{.Networks}}"

# 2. plugin_daemonが適切なネットワークに接続されているか確認
# 期待される結果: dify_main_network, docker_default

# 3. 問題がある場合は、docker-compose.yamlの設定を確認
# plugin_daemonセクションに以下が含まれているか確認:
# networks:
#   - dify_main_network
#   - default

# 4. 設定を修正後、サービスを再起動
podman-compose down
podman-compose up -d
```

### 問題5: サービス間の通信エラー
**症状**: APIサービスがデータベースやRedisに接続できない

**解決方法**:
```bash
# 1. ネットワークの詳細を確認
podman network inspect dify_main_network

# 2. 各サービスのネットワーク接続を確認
podman inspect <container_name> | grep -A 10 "Networks"

# 3. 必要に応じてネットワークを再作成
podman network rm dify_main_network
podman network create dify_main_network
podman-compose up -d
```

## ネットワーク構成

### 使用されるネットワーク
- **dify_main_network**: メインアプリケーション用（外部作成）
- **ssrf_proxy_network**: SSRF保護用（内部作成）
- **default**: デフォルトブリッジネットワーク

### サービスとネットワークの対応
- **api, worker, worker_beat**: `dify_main_network`, `ssrf_proxy_network`, `default`
- **web, db, redis, nginx**: `dify_main_network`
- **sandbox, ssrf_proxy**: `ssrf_proxy_network`
- **plugin_daemon**: `dify_main_network`, `default` ⭐ **重要: 最新の設定**

### ネットワーク設定の確認方法
```bash
# 全サービスのネットワーク接続状況を確認
podman ps --format "table {{.Names}}\t{{.Networks}}"

# 特定のサービスのネットワーク詳細を確認
podman inspect <container_name> | grep -A 15 "Networks"

# ネットワークの詳細情報を確認
podman network inspect dify_main_network
```

## よく使用するコマンド

### サービスの管理
```bash
# 全サービスを起動
podman-compose up -d

# 全サービスを停止
podman-compose down

# 特定のサービスを起動
podman-compose up -d db redis

# サービスの状態確認
podman-compose ps

# サービスの再起動
podman-compose restart api
```

### ログの確認
```bash
# 全サービスのログ
podman-compose logs

# 特定サービスのログ
podman-compose logs api

# リアルタイムログ
podman-compose logs -f
```

### ネットワークの管理
```bash
# ネットワーク一覧
podman network ls

# ネットワークの詳細
podman network inspect dify_main_network

# ネットワークの削除
podman network rm dify_main_network
```

### トラブルシューティング用コマンド
```bash
# コンテナの詳細情報確認
podman inspect <container_name>

# ネットワーク接続テスト
podman exec <container_name> ping <target_host>

# ポート使用状況確認
netstat -tlnp | grep :8080

# ログの詳細確認
podman-compose logs --tail=100 <service_name>
```

## 環境変数の主要設定

### ポート設定
- `EXPOSE_NGINX_PORT=8080`: NginxのHTTPポート
- `EXPOSE_NGINX_SSL_PORT=8443`: NginxのHTTPSポート

### データベース設定
- `DB_HOST=db`: PostgreSQLホスト
- `DB_PORT=5432`: PostgreSQLポート
- `REDIS_HOST=redis`: Redisホスト
- `REDIS_PORT=6379`: Redisポート

### ベクターストア設定
- `VECTOR_STORE=weaviate`: デフォルトのベクターストア

### Plugin Daemon設定
- `PLUGIN_DAEMON_PORT=5002`: プラグインデーモンの内部ポート
- `EXPOSE_PLUGIN_DEBUGGING_PORT=5003`: プラグインのデバッグポート（外部公開）

## 注意事項

1. **Rootlessモード**: 1024未満のポートは使用できません
2. **ネットワーク**: `dify_main_network`は事前に作成する必要があります
3. **短縮名**: registries.confの設定が重要です
4. **依存関係**: データベース（db, redis）が起動してから他のサービスが起動します
5. **Plugin Daemon**: `dify_main_network`と`default`の両方のネットワークに接続する必要があります
