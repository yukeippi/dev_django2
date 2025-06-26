# デプロイメント設定ガイド

このドキュメントでは、GitHub ActionsとEC2を使用したDjangoアプリケーションの自動デプロイ設定について説明します。

## 目次

1. [概要](#概要)
2. [前提条件](#前提条件)
3. [EC2サーバー設定](#ec2サーバー設定)
4. [GitHub Secrets設定](#github-secrets設定)
5. [デプロイフロー](#デプロイフロー)
6. [運用方法](#運用方法)
7. [トラブルシューティング](#トラブルシューティング)

## 概要

このプロジェクトは以下の機能を持つ自動デプロイシステムを提供します：

- **自動テスト**: PostgreSQLを使用したテスト実行
- **自動デプロイ**: developブランチへのpushまたはタグpush時の自動デプロイ
- **ロールバック機能**: 問題発生時の自動・手動ロールバック
- **ヘルスチェック**: デプロイ後の自動健全性確認
- **バージョン管理**: 複数バージョンの保持と切り替え

## 前提条件

- AWS EC2インスタンス（Ubuntu 22.04 LTS推奨）
- GitHubリポジトリへのアクセス権限
- ドメイン名（オプション）
- SSL証明書（HTTPS使用時）

## EC2サーバー設定

### 1. 初期設定

EC2インスタンスにSSH接続し、以下のスクリプトを実行してください：

```bash
# リポジトリをクローン
git clone <your-repository-url>
cd <repository-name>

# サーバー初期設定スクリプトを実行
sudo bash deploy/server_setup.sh
```

### 2. 必要なソフトウェア

以下のパッケージが自動的にインストールされます：

- **Python 3.11+**: アプリケーション実行環境
- **PostgreSQL 15**: データベース
- **Nginx**: Webサーバー・リバースプロキシ
- **Git**: ソースコード管理
- **UFW**: ファイアウォール
- **Certbot**: SSL証明書管理（Let's Encrypt）

### 3. ディレクトリ構造

```
/var/www/
├── diary/                    # 現在のアプリケーション（シンボリックリンク）
├── releases/                 # リリースバージョン管理
│   ├── v1.0.0/
│   ├── v1.0.1/
│   └── develop-20241226-143000/
├── shared/                   # 共有ファイル
│   ├── .env                  # 環境変数
│   ├── logs/                 # ログファイル
│   ├── media/                # アップロードファイル
│   └── static/               # 静的ファイル
├── scripts/                  # デプロイスクリプト
│   ├── deploy-with-rollback.sh
│   └── health-check.sh
├── current_version           # 現在のバージョン記録
├── previous_version          # 前のバージョン記録
└── deploy.log               # デプロイログ
```

### 4. データベース設定

PostgreSQLの設定：

```sql
-- データベースとユーザーの作成
CREATE DATABASE diary_production;
CREATE USER diary_user WITH PASSWORD 'your_secure_password_here';
ALTER ROLE diary_user SET client_encoding TO 'utf8';
ALTER ROLE diary_user SET default_transaction_isolation TO 'read committed';
ALTER ROLE diary_user SET timezone TO 'Asia/Tokyo';
GRANT ALL PRIVILEGES ON DATABASE diary_production TO diary_user;
```

### 5. 環境変数設定

`/var/www/shared/.env`ファイルを編集：

```bash
# Django設定
SECRET_KEY=your_secret_key_here
DJANGO_SETTINGS_MODULE=config.settings_production

# データベース設定
POSTGRES_DB=diary_production
POSTGRES_USER=diary_user
POSTGRES_PASSWORD=your_secure_password_here
POSTGRES_HOST=localhost
POSTGRES_PORT=5432

# サーバー設定
DOMAIN_NAME=your-domain.com
SERVER_IP=your_server_ip

# HTTPS設定（SSL証明書使用時）
USE_HTTPS=False
```

### 6. Systemdサービス設定

Gunicornサービスファイル（`/etc/systemd/system/diary-gunicorn.service`）：

```ini
[Unit]
Description=Diary Django Application (Gunicorn)
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=notify
User=www-data
Group=www-data
WorkingDirectory=/var/www/diary
Environment=DJANGO_SETTINGS_MODULE=config.settings_production
EnvironmentFile=/var/www/diary/.env
ExecStart=/var/www/diary/venv/bin/gunicorn --config gunicorn.conf.py config.wsgi:application
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed
TimeoutStopSec=5
PrivateTmp=true
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

サービスの有効化：

```bash
sudo systemctl daemon-reload
sudo systemctl enable diary-gunicorn
sudo systemctl start diary-gunicorn
```

### 7. Nginx設定

Nginxサイト設定ファイル（`/etc/nginx/sites-available/diary`）：

```nginx
server {
    listen 80;
    server_name your-domain.com www.your-domain.com;
    
    # セキュリティヘッダー
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;
    
    # 静的ファイルの配信
    location /static/ {
        alias /var/www/diary/static/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
    
    # メディアファイルの配信
    location /media/ {
        alias /var/www/diary/media/;
        expires 1y;
        add_header Cache-Control "public";
    }
    
    # Djangoアプリケーションへのプロキシ
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # タイムアウト設定
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
        
        # バッファリング設定
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }
    
    # ファイルアップロードサイズ制限
    client_max_body_size 10M;
    
    # ログ設定
    access_log /var/log/nginx/diary_access.log;
    error_log /var/log/nginx/diary_error.log;
}
```

サイトの有効化：

```bash
sudo ln -s /etc/nginx/sites-available/diary /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### 8. ファイアウォール設定

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 'Nginx Full'
sudo ufw --force enable
```

### 9. デプロイスクリプトの配置

```bash
# スクリプトディレクトリの作成
sudo mkdir -p /var/www/scripts

# デプロイスクリプトのコピー
sudo cp deploy/deploy-with-rollback.sh /var/www/scripts/
sudo cp deploy/health-check.sh /var/www/scripts/

# 実行権限の付与
sudo chmod +x /var/www/scripts/*.sh
```

## GitHub Secrets設定

GitHubリポジトリの Settings > Secrets and variables > Actions で以下のSecretsを設定してください：

### 必須Secrets

| Secret名 | 説明 | 例 |
|----------|------|-----|
| `EC2_HOST` | EC2インスタンスのIPアドレスまたはドメイン名 | `203.0.113.1` または `your-domain.com` |
| `EC2_USERNAME` | SSH接続用ユーザー名 | `ubuntu` |
| `EC2_SSH_KEY` | SSH秘密鍵（RSA形式） | `-----BEGIN RSA PRIVATE KEY-----...` |

### SSH鍵の生成と設定

1. **EC2側でSSH鍵ペアを生成**：
```bash
sudo -u www-data ssh-keygen -t rsa -b 4096 -f /var/www/.ssh/id_rsa -N ""
```

2. **公開鍵をauthorized_keysに追加**：
```bash
sudo -u www-data cat /var/www/.ssh/id_rsa.pub >> /home/ubuntu/.ssh/authorized_keys
```

3. **秘密鍵をGitHub Secretsに設定**：
```bash
sudo cat /var/www/.ssh/id_rsa
```
この内容を`EC2_SSH_KEY`として設定

## デプロイフロー

### 自動デプロイのトリガー

1. **developブランチへのpush**: 開発版の自動デプロイ
2. **タグのpush** (`v*`): リリース版の自動デプロイ
3. **手動実行**: GitHub ActionsのWorkflow Dispatchから実行

### デプロイプロセス

```mermaid
graph TD
    A[コードpush] --> B[テスト実行]
    B --> C{テスト成功?}
    C -->|Yes| D[バージョン決定]
    C -->|No| E[デプロイ停止]
    D --> F[EC2にSSH接続]
    F --> G[デプロイスクリプト実行]
    G --> H[ヘルスチェック]
    H --> I{ヘルスチェック成功?}
    I -->|Yes| J[デプロイ完了]
    I -->|No| K[自動ロールバック]
    K --> L[ロールバック完了]
```

### バージョン命名規則

- **タグpush**: `v1.0.0` → バージョン名: `v1.0.0`
- **developブランチ**: 自動生成 → バージョン名: `develop-20241226-143000`
- **手動実行**: 指定可能 → バージョン名: 指定した値

## 運用方法

### 手動デプロイ

GitHub ActionsのWorkflow Dispatchから実行：

1. GitHubリポジトリの「Actions」タブを開く
2. 「Deploy with Rollback Support」ワークフローを選択
3. 「Run workflow」をクリック
4. パラメータを設定：
   - **Action**: `deploy`
   - **Version**: デプロイするバージョン（オプション）

### 手動ロールバック

問題が発生した場合のロールバック：

1. GitHub ActionsのWorkflow Dispatchから実行
2. パラメータを設定：
   - **Action**: `rollback`
   - **Version**: ロールバック先のバージョン（オプション、未指定時は前のバージョン）

### サーバー上での直接操作

```bash
# デプロイ済みバージョンの確認
sudo bash /var/www/scripts/deploy-with-rollback.sh list

# 手動ロールバック
sudo bash /var/www/scripts/deploy-with-rollback.sh rollback v1.0.0

# ヘルスチェック実行
sudo bash /var/www/scripts/health-check.sh
```

### ログの確認

```bash
# デプロイログ
tail -f /var/www/deploy.log

# アプリケーションログ
sudo journalctl -u diary-gunicorn -f

# Nginxログ
sudo tail -f /var/log/nginx/diary_access.log
sudo tail -f /var/log/nginx/diary_error.log
```

## トラブルシューティング

### よくある問題と解決方法

#### 1. SSH接続エラー

**症状**: GitHub ActionsでSSH接続に失敗する

**原因と解決方法**:
- SSH鍵の設定確認
- EC2のセキュリティグループでSSH（ポート22）が許可されているか確認
- `EC2_HOST`、`EC2_USERNAME`の値を確認

#### 2. デプロイスクリプトエラー

**症状**: デプロイスクリプトの実行に失敗する

**解決方法**:
```bash
# スクリプトの権限確認
ls -la /var/www/scripts/

# 権限がない場合は付与
sudo chmod +x /var/www/scripts/*.sh

# ディレクトリの所有者確認
sudo chown -R www-data:www-data /var/www/
```

#### 3. データベース接続エラー

**症状**: アプリケーションがデータベースに接続できない

**解決方法**:
```bash
# PostgreSQLサービス状態確認
sudo systemctl status postgresql

# データベース接続テスト
sudo -u postgres psql -c "\l"

# 環境変数ファイル確認
sudo cat /var/www/shared/.env
```

#### 4. Nginx/Gunicornエラー

**症状**: Webサイトにアクセスできない

**解決方法**:
```bash
# サービス状態確認
sudo systemctl status nginx
sudo systemctl status diary-gunicorn

# ポート確認
sudo netstat -tuln | grep -E ':80|:8000'

# ログ確認
sudo journalctl -u diary-gunicorn --no-pager
sudo tail /var/log/nginx/error.log
```

#### 5. ヘルスチェック失敗

**症状**: デプロイ後のヘルスチェックに失敗する

**解決方法**:
```bash
# 手動ヘルスチェック実行
sudo bash /var/www/scripts/health-check.sh

# 個別チェック実行
sudo bash /var/www/scripts/health-check.sh --services
sudo bash /var/www/scripts/health-check.sh --http
sudo bash /var/www/scripts/health-check.sh --database
```

### 緊急時の対応

#### 完全なサービス停止

```bash
# 全サービス停止
sudo systemctl stop diary-gunicorn
sudo systemctl stop nginx

# 前のバージョンに緊急ロールバック
sudo bash /var/www/scripts/deploy-with-rollback.sh rollback

# サービス再開
sudo systemctl start diary-gunicorn
sudo systemctl start nginx
```

#### ディスク容量不足

```bash
# 古いリリースの手動削除
sudo rm -rf /var/www/releases/old-version-name

# ログファイルのローテーション
sudo logrotate -f /etc/logrotate.conf
```

### 監視とメンテナンス

#### 定期的な確認項目

- ディスク使用量: `df -h`
- メモリ使用量: `free -h`
- サービス状態: `sudo systemctl status diary-gunicorn nginx postgresql`
- ログサイズ: `du -sh /var/log/`

#### 推奨メンテナンス

- 週次: ログファイルの確認とローテーション
- 月次: システムアップデートとセキュリティパッチ適用
- 四半期: 古いリリースバージョンのクリーンアップ

---

## 参考情報

- [Django Deployment Checklist](https://docs.djangoproject.com/en/stable/howto/deployment/checklist/)
- [Nginx Configuration](https://nginx.org/en/docs/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
