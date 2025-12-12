# Day06：ELB を用いて Web レイヤの冗長化 - Next.js 版

## 学習目標

Application Load Balancer (ALB) を使用して複数の Web サーバーに負荷を分散し、高可用性を実現する。

## 📝 PHP 版からの変更点

**主な変更点：**
- ポート 80 → ポート 3000 に変更
- Apache 設定 → Docker Compose の環境変数 `INSTANCE_ID` でインスタンス識別
- ヘルスチェックパスは `/` のまま使用可能

---

## 前提条件

- Day05 (Next.js版) を完了していること
- RDS に接続した Next.js アプリケーションが動作していること

---

## 1. AMI の作成

### 1.1 web-1a の AMI 作成

1. EC2 コンソール → 「インスタンス」
2. `web-1a` を選択
3. 「アクション」→「イメージとテンプレート」→「イメージを作成」

**設定内容：**
- **イメージ名**: `udemy-aws-14days-web-ami`
- **イメージの説明**: Next.js web server with Docker, Docker Compose, and MySQL client
- **再起動しない**: チェックを入れる（本番環境では外す）

4. 「イメージを作成」をクリック
5. AMI が利用可能になるまで数分待機

---

## 2. セキュリティグループの更新

### 2.1 ALB 用セキュリティグループの作成

1. VPC コンソール → 「セキュリティグループ」→「セキュリティグループを作成」

**設定内容：**
- **セキュリティグループ名**: `udemy-aws-14days-alb-sg`
- **説明**: Security group for Application Load Balancer
- **VPC**: `udemy-aws-14days-vpc`

**インバウンドルール：**
| タイプ | プロトコル | ポート範囲 | ソース | 説明 |
|--------|-----------|-----------|--------|------|
| HTTP | TCP | 80 | 0.0.0.0/0 | Allow HTTP from anywhere |

2. 「セキュリティグループを作成」をクリック

### 2.2 Web サーバー用セキュリティグループの更新

1. `udemy-aws-14days-web-sg` を選択
2. 「インバウンドのルール」タブ → 「インバウンドのルールを編集」
3. 既存のルールを以下に変更：

**更新後のインバウンドルール：**
| タイプ | プロトコル | ポート範囲 | ソース | 説明 |
|--------|-----------|-----------|--------|------|
| SSH | TCP | 22 | 0.0.0.0/0 | SSH access |
| カスタム TCP | TCP | 3000 | udemy-aws-14days-alb-sg | Next.js from ALB |

---

## 3. 2台目の Web サーバーの起動

### 3.1 AMI から web-1c インスタンスを作成

1. EC2 コンソール → 「AMI」→ 作成した AMI を選択
2. 「AMI から起動」をクリック

**設定内容：**
- **名前**: `web-1c`
- **インスタンスタイプ**: `t2.micro`
- **キーペア**: `udemy-aws-14days`
- **ネットワーク設定**:
  - **VPC**: `udemy-aws-14days-vpc`
  - **サブネット**: `udemy-aws-14days-public-subnet-1c`
  - **パブリック IP の自動割り当て**: 有効化
  - **セキュリティグループ**: `udemy-aws-14days-web-sg`
- **高度な詳細**:
  - **プライマリ IP**: `10.0.2.10`

3. 「インスタンスを起動」をクリック

### 3.2 インスタンス識別のための環境変数設定

#### web-1a の設定

```bash
# web-1a にログイン
ssh -i udemy-aws-14days.pem ec2-user@[web-1aのパブリックIP]

# .env を確認・更新
cd ~/simple-blog
vim .env
```

```bash
DB_HOST=[RDSエンドポイント]
DB_USER=simpleblog_user
DB_PASSWORD=User!1234
DB_NAME=simpleblog_db
INSTANCE_ID=web-1a
```

```bash
# Docker コンテナを再起動
docker compose down
docker compose up -d
```

#### web-1c の設定

```bash
# web-1c にログイン
ssh -i udemy-aws-14days.pem ec2-user@[web-1cのパブリックIP]

# .env を更新
cd ~/simple-blog
vim .env
```

```bash
DB_HOST=[RDSエンドポイント]
DB_USER=simpleblog_user
DB_PASSWORD=User!1234
DB_NAME=simpleblog_db
INSTANCE_ID=web-1c
```

```bash
# Docker コンテナが起動しているか確認
docker compose ps

# 起動していない場合
cd ~/simple-blog
docker compose up -d

# ログ確認
docker compose logs -f
```

### 3.3 両インスタンスの動作確認

```bash
# web-1a
http://[web-1aのパブリックIP]:3000
# → "Instance: web-1a" と表示されることを確認

# web-1c
http://[web-1cのパブリックIP]:3000
# → "Instance: web-1c" と表示されることを確認
```

---

## 4. ターゲットグループの作成

### 4.1 ターゲットグループ作成

1. EC2 コンソール → 「ロードバランシング」→「ターゲットグループ」
2. 「ターゲットグループの作成」をクリック

### 4.2 基本設定

- **ターゲットタイプ**: インスタンス
- **ターゲットグループ名**: `udemy-aws-14days-tg`
- **プロトコル**: HTTP
- **ポート**: 3000
- **VPC**: `udemy-aws-14days-vpc`
- **プロトコルバージョン**: HTTP1

### 4.3 ヘルスチェック設定

- **ヘルスチェックプロトコル**: HTTP
- **ヘルスチェックパス**: `/`
- **詳細ヘルスチェック設定**:
  - **正常のしきい値**: 2
  - **非正常のしきい値**: 2
  - **タイムアウト**: 5 秒
  - **間隔**: 10 秒（ハンズオン用に短縮）
  - **成功コード**: 200

### 4.4 ターゲット登録

1. 「次へ」をクリック
2. `web-1a` と `web-1c` の両方を選択
3. 「以下を保留中として含める」をクリック
4. 「ターゲットグループの作成」をクリック

---

## 5. Application Load Balancer の作成

### 5.1 ALB の作成開始

1. EC2 コンソール → 「ロードバランサー」→「ロードバランサーの作成」
2. 「Application Load Balancer」の「作成」をクリック

### 5.2 基本設定

- **ロードバランサー名**: `udemy-aws-14days-alb`
- **スキーム**: インターネット向け
- **IP アドレスタイプ**: IPv4

### 5.3 ネットワークマッピング

- **VPC**: `udemy-aws-14days-vpc`
- **マッピング**:
  - `ap-northeast-1a` → `udemy-aws-14days-public-subnet-1a`
  - `ap-northeast-1c` → `udemy-aws-14days-public-subnet-1c`

### 5.4 セキュリティグループ

- `udemy-aws-14days-alb-sg` を選択
- default は削除

### 5.5 リスナーとルーティング

**デフォルトリスナー：**
- **プロトコル**: HTTP
- **ポート**: 80
- **デフォルトアクション**: `udemy-aws-14days-tg` にフォワード

### 5.6 ロードバランサーの作成

1. 「ロードバランサーの作成」をクリック
2. プロビジョニング完了まで数分待機
3. ステータスが「Active」になることを確認

---

## 6. 動作確認

### 6.1 ALB の DNS 名を取得

1. EC2 コンソール → 「ロードバランサー」
2. `udemy-aws-14days-alb` を選択
3. DNS 名をコピー（例：udemy-aws-14days-alb-123456789.ap-northeast-1.elb.amazonaws.com）

### 6.2 ブラウザでアクセス

```
http://[ALBのDNS名]
```

### 6.3 負荷分散の確認

1. ページを複数回リロード
2. 「Instance: web-1a」と「Instance: web-1c」が交互に表示されることを確認
3. データベースの内容が両方のインスタンスで同じであることを確認

### 6.4 ヘルスチェックの確認

1. EC2 コンソール → 「ターゲットグループ」→ `udemy-aws-14days-tg`
2. 「ターゲット」タブ
3. 両方のインスタンスが「healthy」であることを確認

---

## 7. 高可用性のテスト

### 7.1 1台のインスタンスを停止

```bash
# web-1c を停止
# EC2 コンソール → web-1c を選択 → インスタンスの状態 → インスタンスを停止
```

### 7.2 動作確認

1. ブラウザで ALB の DNS にアクセス
2. 「Instance: web-1a」のみが表示されることを確認
3. アプリケーションが正常に動作し続けることを確認

### 7.3 ターゲットグループのヘルスチェック確認

1. ターゲットグループで web-1c のステータスが「unhealthy」になることを確認
2. ALB が自動的に健全なターゲット（web-1a）にのみトラフィックを送信

### 7.4 インスタンスを再起動

```bash
# web-1c を起動
# EC2 コンソール → web-1c を選択 → インスタンスの状態 → インスタンスを開始
```

### 7.5 復旧確認

1. ターゲットグループで web-1c のステータスが「healthy」に戻ることを確認
2. ブラウザで再度アクセスし、負荷分散が再開されることを確認

---

## 8. セキュリティの強化（オプション）

### 8.1 Web サーバーへの直接アクセスを制限

現在の構成では、ALB 経由だけでなく、EC2 の パブリック IP:3000 でも直接アクセス可能です。セキュリティを強化するには：

**オプション 1: セキュリティグループで制限（推奨）**

`udemy-aws-14days-web-sg` のインバウンドルールを以下のみに変更：

| タイプ | プロトコル | ポート範囲 | ソース | 説明 |
|--------|-----------|-----------|--------|------|
| SSH | TCP | 22 | 0.0.0.0/0 | SSH access |
| カスタム TCP | TCP | 3000 | udemy-aws-14days-alb-sg | Next.js from ALB only |

**オプション 2: 環境変数でヘッダーチェック**

ALB から送信される特定のヘッダーをチェックする実装を追加することも可能です（より高度な設定）。

---

## 📝 トラブルシューティング

### ALB にアクセスできない

```bash
# セキュリティグループの確認
# - alb-sg がポート 80 を 0.0.0.0/0 から許可しているか
# - web-sg がポート 3000 を alb-sg から許可しているか

# ターゲットグループの確認
# - ヘルスチェックが成功しているか
# - ポート番号が 3000 になっているか
```

### ヘルスチェックが失敗する

```bash
# Docker コンテナが起動しているか確認
docker compose ps

# ポート 3000 でリッスンしているか確認
sudo netstat -tulpn | grep 3000

# コンテナのログを確認
docker compose logs

# セキュリティグループを一時的に緩和してテスト
# web-sg: ポート 3000 を 0.0.0.0/0 から許可して直接アクセステスト
```

### Docker コンテナが起動していない

```bash
# Docker コンテナの状態を確認
docker compose ps

# 起動していない場合
cd ~/simple-blog
docker compose up -d

# ログを確認
docker compose logs -f

# 完全に再構築
docker compose down
docker compose up -d --build
```

---

## まとめ

Day6 (Next.js版) では以下を実施しました：

### 学習したキーワード

- Application Load Balancer (ALB)
- ターゲットグループ
- ヘルスチェック
- AMI（Amazon Machine Image）
- 負荷分散
- 高可用性
- 冗長化

### 実施した内容

1. Web サーバーの AMI 作成
2. 2台目の Web サーバーを別 AZ に配置
3. ALB とターゲットグループの作成
4. 負荷分散の動作確認
5. 高可用性テスト

### PHP 版との主な違い

| 項目 | PHP 版 | Next.js 版 (Docker) |
|------|--------|---------------------|
| ポート | 80 | 3000 |
| インスタンス識別 | PHP ファイル編集 | Docker Compose 環境変数 INSTANCE_ID |
| プロセス管理 | systemd (httpd) | Docker Compose |
| ヘルスチェック | /index.php | / (ルート) |
| 起動確認 | systemctl status | docker compose ps |
| 再起動 | systemctl restart | docker compose restart |

### ALB のメリット

- **負荷分散**: 複数のターゲットに自動でトラフィックを分散
- **ヘルスチェック**: 異常なターゲットを自動検出・除外
- **高可用性**: 複数 AZ での冗長構成
- **スケーラビリティ**: ターゲット数を簡単に増減可能
- **SSL/TLS 終端**: HTTPS 化も ALB で簡単に実現（Day08 以降）

**次回の Day07 では、S3 を使用した静的コンテンツの配信を学習します。**
