# Day05：リレーショナル DB のマネージドサービス (Amazon RDS) - Next.js 版

## 学習目標

EC2 上の MySQL から Amazon RDS（マネージドデータベースサービス)に移行し、バックアップ・復元機能を理解する。

## 📝 PHP 版からの変更点

**主な変更点：**
- 環境変数 (`.env`) で接続先を管理
- RDS エンドポイントへの接続先変更のみで対応可能
- PHP の接続設定変更と同様の手順

---

## 前提条件

- Day04 (Next.js版) を完了していること
- EC2 上の MySQL でアプリケーションが動作していること

---

## 1. DB サブネットグループの作成

### 1.1 DB サブネットグループの作成

1. RDS コンソール → 「サブネットグループ」→「DB サブネットグループを作成」

**設定内容：**
- **名前**: `udemy-aws-14days-db-subnet-group`
- **説明**: DB subnet group for RDS
- **VPC**: `udemy-aws-14days-vpc`
- **アベイラビリティーゾーン**:
  - `ap-northeast-1a` → `udemy-aws-14days-private-subnet-1a` を選択
  - `ap-northeast-1c` → `udemy-aws-14days-private-subnet-1c` を選択

2. 「作成」をクリック

---

## 2. RDS インスタンスの作成

### 2.1 RDS インスタンスの作成開始

1. RDS コンソール → 「データベース」→「データベースの作成」

### 2.2 エンジン設定

- **エンジンのタイプ**: MySQL
- **エンジンのバージョン**: MySQL 8.4.3（または利用可能な最新版）
- **テンプレート**: 無料利用枠

### 2.3 設定

- **DB インスタンス識別子**: `udemy-aws-14days-db`
- **マスターユーザー名**: `admin`
- **マスターパスワード**: `Admin!1234`（自動生成でも可）

### 2.4 インスタンス設定

- **DB インスタンスクラス**: バースト可能クラス → `db.t3.micro`

### 2.5 ストレージ

- **ストレージタイプ**: 汎用 SSD (gp3)
- **割り当てられたストレージ**: 20 GiB
- **ストレージの自動スケーリング**: 有効（オプション）

### 2.6 接続

- **VPC**: `udemy-aws-14days-vpc`
- **DB サブネットグループ**: `udemy-aws-14days-db-subnet-group`
- **パブリックアクセス**: なし
- **VPC セキュリティグループ**: 既存のものを選択
  - `udemy-aws-14days-db-sg` を選択
  - default は削除
- **アベイラビリティーゾーン**: `ap-northeast-1a`

### 2.7 追加設定

- **最初のデータベース名**: `simpleblog_db`
- **バックアップ**:
  - 自動バックアップ: 有効
  - バックアップ保持期間: 7 日
- **暗号化**: 有効（デフォルト）
- **ログのエクスポート**: 必要に応じて設定

### 2.8 データベースの作成

1. 「データベースの作成」をクリック
2. 作成完了まで約 5〜10 分待機
3. ステータスが「利用可能」になることを確認

---

## 3. EC2 の MySQL データを RDS にマイグレーション

### 3.1 EC2 の MySQL データをダンプ

```bash
# web-1a にログイン
ssh -i udemy-aws-14days.pem ec2-user@[web-1aのパブリックIP]

# データベースのダンプ
mysqldump -h 10.0.101.20 -u simpleblog_user -p simpleblog_db > simpleblog_backup.sql
# パスワード: User!1234
```

### 3.2 RDS にデータをインポート

```bash
# RDS のエンドポイントを確認
# RDS コンソール → データベース → udemy-aws-14days-db → 「接続とセキュリティ」タブ
# エンドポイント例: udemy-aws-14days-db.xxxxx.ap-northeast-1.rds.amazonaws.com

# RDS に接続してデータベースとテーブルを作成
mysql -h [RDSエンドポイント] -u admin -p
# パスワード: Admin!1234
```

```sql
-- データベースは既に作成済み
USE simpleblog_db;

-- テーブル作成
CREATE TABLE posts (
  id INT AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(255) NOT NULL,
  content TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- アプリケーション用ユーザー作成
CREATE USER 'simpleblog_user'@'%' IDENTIFIED BY 'User!1234';
GRANT ALL PRIVILEGES ON simpleblog_db.* TO 'simpleblog_user'@'%';
FLUSH PRIVILEGES;

EXIT;
```

```bash
# ダンプファイルをインポート（データのみ）
# テーブル作成は上記で実施済みのため、INSERT文のみ抽出するか、全体をインポート
mysql -h [RDSエンドポイント] -u simpleblog_user -p simpleblog_db < simpleblog_backup.sql
# パスワード: User!1234

# データ確認
mysql -h [RDSエンドポイント] -u simpleblog_user -p
USE simpleblog_db;
SELECT * FROM posts;
EXIT;
```

---

## 4. Next.js アプリケーションの接続先変更

### 4.1 環境変数の更新

```bash
# web-1a で .env を編集
cd ~/simple-blog
vim .env
```

**更新内容：**
```bash
DB_HOST=[RDSエンドポイント]
DB_USER=simpleblog_user
DB_PASSWORD=User!1234
DB_NAME=simpleblog_db
INSTANCE_ID=web-1a
```

例：
```bash
DB_HOST=udemy-aws-14days-db.c1234567890.ap-northeast-1.rds.amazonaws.com
DB_USER=simpleblog_user
DB_PASSWORD=User!1234
DB_NAME=simpleblog_db
INSTANCE_ID=web-1a
```

### 4.2 アプリケーションの再起動

```bash
# Docker コンテナを再起動
docker compose down
docker compose up -d

# ログで接続状態を確認
docker compose logs -f
```

### 4.3 動作確認

1. ブラウザで `http://[web-1aのパブリックIP]:3000` にアクセス
2. RDS から取得したデータが表示されることを確認
3. 新規投稿ができることを確認

---

## 5. スナップショット機能

### 5.1 手動スナップショットの作成

1. RDS コンソール → 「データベース」
2. `udemy-aws-14days-db` を選択
3. 「アクション」→「スナップショットの取得」
4. **スナップショット名**: `udemy-aws-14days-db-snapshot-01`
5. 「スナップショットの取得」をクリック

### 5.2 スナップショットからの復元

1. RDS コンソール → 「スナップショット」
2. 作成したスナップショットを選択
3. 「アクション」→「スナップショットの復元」

**復元設定：**
- **DB インスタンス識別子**: `udemy-aws-14days-db-restored`
- その他の設定は元のインスタンスと同様
- **VPC セキュリティグループ**: `udemy-aws-14days-db-sg`

4. 「DB インスタンスの復元」をクリック
5. 復元完了まで待機

### 5.3 復元したデータベースへの接続テスト

```bash
# 新しいエンドポイントで接続確認
mysql -h [復元したRDSのエンドポイント] -u simpleblog_user -p
USE simpleblog_db;
SELECT * FROM posts;
EXIT;
```

### 5.4 アプリケーションの接続先を復元 DB に変更（オプション）

```bash
# .env を編集
vim ~/simple-blog/.env

# DB_HOST を復元した RDS のエンドポイントに変更
# 保存後
docker compose down
docker compose up -d
```

### 5.5 動作確認後、元に戻す

```bash
# .env を元の RDS エンドポイントに戻す
vim ~/simple-blog/.env
docker compose down
docker compose up -d
```

### 5.6 復元したインスタンスの削除

1. RDS コンソール → 「データベース」
2. `udemy-aws-14days-db-restored` を選択
3. 「アクション」→「削除」
4. 最終スナップショットの作成: スキップ
5. 確認チェックを入れて「削除」

---

## 6. 自動バックアップの設定確認

### 6.1 自動バックアップの設定

1. RDS コンソール → 「データベース」→ `udemy-aws-14days-db`
2. 「メンテナンスとバックアップ」タブ
3. バックアップ設定を確認：
   - **自動バックアップ**: 有効
   - **バックアップ保持期間**: 7 日
   - **バックアップウィンドウ**: 優先するバックアップウィンドウなし（または指定）

### 6.2 自動バックアップからの復元

1. RDS コンソール → 「自動バックアップ」
2. 対象のバックアップを選択
3. 「復元」をクリック
4. ポイントインタイムリカバリで特定の時点に復元可能

---

## 7. EC2 の MySQL データベースサーバーの停止（オプション）

RDS への移行が完了したら、EC2 の MySQL サーバー（db-1a）は不要になります。

```bash
# EC2 コンソールで db-1a を停止または削除
```

---

## 📝 トラブルシューティング

### RDS に接続できない

```bash
# セキュリティグループの確認
# - db-sg がポート 3306 を web-sg から許可しているか
# - RDS のセキュリティグループが正しく設定されているか

# VPC エンドポイント確認
# - RDS インスタンスがプライベートサブネット内にあるか
# - サブネットグループが正しく設定されているか

# 接続テスト
mysql -h [RDSエンドポイント] -u simpleblog_user -p
```

### "Access denied" エラー

```sql
-- RDS に admin でログインして確認
mysql -h [RDSエンドポイント] -u admin -p

-- ユーザーと権限を確認
SELECT user, host FROM mysql.user;
SHOW GRANTS FOR 'simpleblog_user'@'%';

-- 権限が不足している場合は再付与
GRANT ALL PRIVILEGES ON simpleblog_db.* TO 'simpleblog_user'@'%';
FLUSH PRIVILEGES;
```

### アプリケーションがデータを取得できない

```bash
# Docker コンテナのログを確認
docker compose logs

# .env が正しく読み込まれているか確認
cat ~/simple-blog/.env

# 環境変数を反映して再起動
docker compose down
docker compose up -d
```

---

## まとめ

Day5 (Next.js版) では以下を実施しました：

### 学習したキーワード

- Amazon RDS
- マネージドデータベース
- DB サブネットグループ
- スナップショット
- バックアップと復元
- ポイントインタイムリカバリ
- マルチ AZ 配置

### 実施した内容

1. RDS インスタンスの作成
2. EC2 MySQL から RDS へのマイグレーション
3. Next.js アプリケーションの接続先変更（環境変数のみ）
4. スナップショットの作成と復元
5. 自動バックアップ設定の確認

### PHP 版との主な違い

| 項目 | PHP 版 | Next.js 版 (Docker) |
|------|--------|---------------------|
| 接続設定変更 | PHP ファイル内を直接編集 | .env を編集 |
| 設定管理 | ハードコード | 環境変数 |
| 再起動 | Apache 再起動 | docker compose restart |
| 接続方式 | mysqli | mysql2 (コネクションプール) |

### RDS のメリット

- **自動バックアップ**: 7 日間の自動バックアップ
- **マルチ AZ**: 高可用性構成（オプション）
- **自動パッチ適用**: メンテナンスウィンドウで自動更新
- **スケーラビリティ**: リードレプリカで読み取り負荷分散
- **監視**: CloudWatch との統合

**次回の Day06 では、ELB を使用した負荷分散を実装します。**
