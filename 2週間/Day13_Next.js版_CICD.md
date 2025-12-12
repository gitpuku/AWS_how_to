# Day13：AWS Code サービス + GitHub を使って CI/CD 環境を構築する - Next.js 版

## 学習目標

AWS Code シリーズサービスと GitHub を組み合わせて、Next.js アプリケーションの CI/CD 環境を構築し、自動ビルド・デプロイを実現する。

## 📝 PHP 版からの変更点

**主な変更点：**
- Docker イメージのビルドとプッシュ
- ECR (Elastic Container Registry) にイメージを保存
- デプロイ時に Docker Compose で コンテナを再起動
- CodeBuild で Docker イメージをビルド

---

## 前提条件

- Day03〜Day06 (Next.js版) を完了していること
- GitHub アカウントを持っていること
- CloudFront と ALB が正常に動作していること

---

## 1. GitHub リポジトリの準備

### 1.1 GitHub リポジトリの作成

1. GitHub にログインし、新しいリポジトリを作成
   - リポジトリ名：`aws-simple-blog-nextjs`
   - プライベートリポジトリとして作成
   - README, .gitignore は追加しない

### 1.2 ローカルリポジトリの準備

```bash
# web-1a にログイン
ssh -i udemy-aws-14days.pem ec2-user@[web-1aのパブリックIP]

cd ~/simple-blog

# Git の初期化（まだしていない場合）
git init
git branch -M main

# .gitignore を作成
cat > .gitignore << 'EOF'
node_modules/
.next/
.env
*.log
.DS_Store
EOF

# 初回コミット
git add .
git commit -m "Initial commit: Next.js simple blog"

# GitHub リポジトリと接続
git remote add origin https://github.com/[your-username]/aws-simple-blog-nextjs.git
git push -u origin main
```

---

## 2. S3 バケットの作成

### 2.1 アーティファクト用 S3 バケットの作成

1. S3 コンソール → 「バケットを作成」

**設定内容：**
- **バケット名**: `udemy-aws-14days-nextjs-artifacts` (ユニークな名前に変更)
- **リージョン**: ap-northeast-1
- **バージョニング**: 有効化
- その他はデフォルト

---

## 3. buildspec.yml の作成

### 3.1 buildspec.yml を作成

```bash
# ローカル環境または web-1a で作成
cd ~/simple-blog

cat > buildspec.yml << 'EOF'
version: 0.2

phases:
  pre_build:
    commands:
      - echo "Logging in to Docker Hub..."
      - echo "Building Docker image..."
  build:
    commands:
      - echo "Building application image"
      - docker build -t simple-blog:latest .
      - echo "Saving Docker image as tar"
      - docker save simple-blog:latest -o simple-blog.tar
      - echo "Creating deployment package"
      - mkdir -p deploy
      - mv simple-blog.tar deploy/
      - cp docker-compose.yml deploy/
      - cp appspec.yml deploy/
      - cp -r scripts deploy/
  post_build:
    commands:
      - echo "Build completed on `date`"
      - cd deploy
      - zip -r ../simple-blog-deploy.zip .
      - cd ..
      - aws deploy push --application-name udemy-aws-14days-nextjs-blog --s3-location s3://udemy-aws-14days-nextjs-artifacts/simple-blog-deploy.zip --region ap-northeast-1

artifacts:
  files:
    - '**/*'
  base-directory: deploy
  name: simple-blog-deploy.zip
EOF
```

**重要**: `s3://udemy-aws-14days-nextjs-artifacts` の部分を、実際に作成したバケット名に変更してください。

---

## 4. appspec.yml の作成

### 4.1 appspec.yml を作成

```bash
cat > appspec.yml << 'EOF'
version: 0.0
os: linux

files:
  - source: /
    destination: /home/ec2-user/simple-blog-deploy

hooks:
  BeforeInstall:
    - location: scripts/before_install.sh
      timeout: 300
      runas: root
  AfterInstall:
    - location: scripts/after_install.sh
      timeout: 300
      runas: ec2-user
  ApplicationStart:
    - location: scripts/application_start.sh
      timeout: 300
      runas: ec2-user
  ApplicationStop:
    - location: scripts/application_stop.sh
      timeout: 300
      runas: ec2-user
EOF
```

---

## 5. デプロイスクリプトの作成

### 5.1 scripts ディレクトリを作成

```bash
mkdir -p scripts
```

### 5.2 application_stop.sh

```bash
cat > scripts/application_stop.sh << 'EOF'
#!/bin/bash
set -e

echo "Stopping Docker containers..."
cd ~/simple-blog

# Docker コンテナを停止
if [ -f "docker-compose.yml" ]; then
  docker compose down || true
fi

echo "Containers stopped successfully"
EOF

chmod +x scripts/application_stop.sh
```

### 5.3 before_install.sh

```bash
cat > scripts/before_install.sh << 'EOF'
#!/bin/bash
set -e

echo "Preparing for installation..."

# .env ファイルのバックアップ
if [ -f "/home/ec2-user/simple-blog/.env" ]; then
  echo "Backing up .env..."
  cp /home/ec2-user/simple-blog/.env /tmp/.env.backup
fi

echo "Before install completed"
EOF

chmod +x scripts/before_install.sh
```

### 5.4 after_install.sh

```bash
cat > scripts/after_install.sh << 'EOF'
#!/bin/bash
set -e

echo "Running after install..."
cd /home/ec2-user/simple-blog-deploy

# Docker イメージをロード
if [ -f "simple-blog.tar" ]; then
  echo "Loading Docker image..."
  docker load -i simple-blog.tar
fi

# ファイルを本番ディレクトリにコピー
cp docker-compose.yml ~/simple-blog/
cp -r scripts ~/simple-blog/ || true

# .env ファイルを復元
if [ -f "/tmp/.env.backup" ]; then
  echo "Restoring .env..."
  cp /tmp/.env.backup ~/simple-blog/.env
  rm /tmp/.env.backup
fi

echo "After install completed"
EOF

chmod +x scripts/after_install.sh
```

### 5.5 application_start.sh

```bash
cat > scripts/application_start.sh << 'EOF'
#!/bin/bash
set -e

echo "Starting Docker containers..."
cd ~/simple-blog

# Docker Compose でコンテナを起動
docker compose up -d

# ヘルスチェック
echo "Waiting for application to start..."
sleep 10

if curl -f http://localhost:3000 > /dev/null 2>&1; then
  echo "Application started successfully"
else
  echo "Warning: Application health check failed"
  docker compose logs
  exit 1
fi

echo "Application start completed"
EOF

chmod +x scripts/application_start.sh
```

---

## 6. GitHub にプッシュ

```bash
# すべてのファイルをコミット
git add buildspec.yml appspec.yml scripts/
git commit -m "Add CI/CD configuration files"
git push origin main
```

---

## 7. AWS CodeBuild の設定

### 7.1 CodeBuild プロジェクトの作成

1. CodeBuild コンソール → 「プロジェクトを作成する」

**設定内容：**
- **プロジェクト名**: `udemy-aws-14days-nextjs-blog`
- **ソースプロバイダー**: GitHub
- **リポジトリ**: GitHub の接続を使用（CodeConnections 経由）
  - 接続名：`udemy-aws-14days-github`
  - リポジトリ：`aws-simple-blog-nextjs`
  - ブランチ：`main`

**環境設定：**
- **環境イメージ**: マネージド型イメージ
- **オペレーティングシステム**: Amazon Linux
- **ランタイム**: Standard
- **イメージ**: aws/codebuild/amazonlinux2-x86_64-standard:5.0 (または最新)
- **イメージのバージョン**: 最新のイメージを常に使用する
- **環境タイプ**: Linux EC2
- **サービスロール**: 新しいサービスロール

**Buildspec:**
- **ビルド仕様**: buildspec ファイルを使用する

**アーティファクト：**
- **タイプ**: Amazon S3
- **バケット名**: `udemy-aws-14days-nextjs-artifacts`
- **名前空間タイプ**: ビルド ID
- **アーティファクトのパッケージ化**: Zip

2. 「プロジェクトを作成」をクリック

### 7.2 CodeBuild サービスロールに権限追加

1. IAM コンソール → 「ロール」
2. CodeBuild が作成したサービスロールを検索（例：`codebuild-udemy-aws-14days-nextjs-blog-service-role`）
3. 「許可を追加」→「ポリシーをアタッチ」
4. `AWSCodeDeployDeployerAccess` を検索してアタッチ

---

## 8. AWS CodeDeploy の設定

### 8.1 EC2 インスタンス用 IAM ロールの確認

Day03 で作成した `udemy-aws-14days-web-role` に以下のポリシーがアタッチされていることを確認：
- `AmazonS3FullAccess`（または必要な S3 バケットへのアクセス権限）

両インスタンス（web-1a、web-1c）にこのロールがアタッチされていることを確認。

### 8.2 CodeDeploy エージェントのインストール（未インストールの場合）

```bash
# web-1a と web-1c の両方で実行
ssh -i udemy-aws-14days.pem ec2-user@[インスタンスのパブリックIP]

# CodeDeploy エージェントのインストール
sudo yum install ruby -y
wget https://aws-codedeploy-ap-northeast-1.s3.ap-northeast-1.amazonaws.com/latest/install
chmod +x ./install
sudo ./install auto

# サービスの起動確認
sudo systemctl status codedeploy-agent
```

### 8.3 CodeDeploy 用 IAM ロールの作成

1. IAM コンソール → 「ロール」→「ロールを作成」
2. **信頼されたエンティティタイプ**: AWS サービス
3. **ユースケース**: CodeDeploy → CodeDeploy
4. ロール名: `udemy-aws-14days-codedeploy-role`
5. 「ロールを作成」をクリック

### 8.4 CodeDeploy アプリケーションの作成

1. CodeDeploy コンソール → 「アプリケーション」→「アプリケーションを作成」

**設定内容：**
- **アプリケーション名**: `udemy-aws-14days-nextjs-blog`
- **コンピューティングプラットフォーム**: EC2/オンプレミス

2. 「アプリケーションの作成」をクリック

### 8.5 デプロイグループの作成

1. 作成したアプリケーション内で「デプロイグループを作成」

**設定内容：**
- **デプロイグループ名**: `udemy-aws-14days-nextjs-deploy-group`
- **サービスロール**: `udemy-aws-14days-codedeploy-role`
- **デプロイタイプ**: インプレース
- **環境設定**: Amazon EC2 インスタンス
  - **キー**: Name
  - **値**: web-1a, web-1c（両方）
- **デプロイ設定**: CodeDeployDefault.OneAtATime
- **ロードバランサー**: 有効
  - **Application Load Balancer または Network Load Balancer を選択**
  - **ターゲットグループ**: `udemy-aws-14days-tg`

**詳細オプション：**
- **登録解除の遅延**: 300 秒（デフォルト、またはターゲットグループの設定に合わせる）

2. 「デプロイグループを作成」をクリック

---

## 9. AWS CodePipeline の設定

### 9.1 パイプラインの作成

1. CodePipeline コンソール → 「パイプラインを作成する」

**パイプライン設定：**
- **パイプライン名**: `udemy-aws-14days-nextjs-pipeline`
- **サービスロール**: 新しいサービスロール

### 9.2 ソースステージ

- **ソースプロバイダー**: GitHub（バージョン 2）
- **接続**: 既存の接続を選択（CodeBuild で作成したもの）
- **リポジトリ名**: `aws-simple-blog-nextjs`
- **ブランチ名**: `main`
- **出力アーティファクト形式**: CodePipeline のデフォルト

### 9.3 ビルドステージ

- **ビルドプロバイダー**: AWS CodeBuild
- **リージョン**: アジアパシフィック（東京）
- **プロジェクト名**: `udemy-aws-14days-nextjs-blog`

### 9.4 デプロイステージ

- **デプロイプロバイダー**: AWS CodeDeploy
- **リージョン**: アジアパシフィック（東京）
- **アプリケーション名**: `udemy-aws-14days-nextjs-blog`
- **デプロイグループ**: `udemy-aws-14days-nextjs-deploy-group`

### 9.5 パイプラインの作成

1. 「パイプラインを作成する」をクリック
2. 初回実行が自動的に開始される
3. 全てのステージが成功することを確認

---

## 10. 動作確認

### 10.1 初回デプロイの確認

1. CodePipeline コンソールで全ステージが成功していることを確認
2. ブラウザで ALB または CloudFront の URL にアクセス
3. アプリケーションが正常に表示されることを確認

### 10.2 コード変更による自動デプロイのテスト

```bash
# ローカル環境または web-1a で
cd ~/simple-blog

# app/layout.tsx を編集
vim app/layout.tsx
```

タイトルを変更：

```typescript
export const metadata: Metadata = {
  title: "シンプルブログ - CI/CD対応",
  description: "AWS 14日間チャレンジ - Next.js版 with CI/CD",
};
```

```bash
# コミットしてプッシュ
git add app/layout.tsx
git commit -m "Update title for CI/CD test"
git push origin main
```

### 10.3 パイプライン実行の確認

1. CodePipeline が自動的に起動することを確認
2. Source → Build → Deploy の順に実行される
3. 全ステージが成功することを確認
4. ブラウザでアプリケーションにアクセスし、変更が反映されていることを確認

---

## 📝 トラブルシューティング

### ビルドエラー: "npm: command not found"

```yaml
# buildspec.yml の環境設定を確認
# Amazon Linux 2 以降のイメージを使用しているか確認
# runtime: standard 5.0 以降を使用
```

### デプロイエラー: CodeDeploy エージェントが起動していない

```bash
# EC2 インスタンスで確認
sudo systemctl status codedeploy-agent

# 起動していない場合
sudo systemctl start codedeploy-agent
sudo systemctl enable codedeploy-agent
```

### デプロイ後にアプリが起動しない

```bash
# EC2 インスタンスで確認
cd ~/simple-blog
docker compose ps
docker compose logs

# .env が存在するか確認
ls -la ~/simple-blog/.env

# 手動で起動テスト
docker compose up -d
```

### ヘルスチェック失敗

```bash
# アプリケーションが起動しているか確認
curl http://localhost:3000

# Docker コンテナのログを確認
cd ~/simple-blog
docker compose logs

# ポート確認
sudo netstat -tulpn | grep 3000
```

---

## まとめ

Day13 (Next.js版) では以下を実施しました：

### 学習したキーワード

- CI/CD パイプライン
- AWS CodeBuild
- AWS CodeDeploy
- AWS CodePipeline
- buildspec.yml
- appspec.yml
- **Docker イメージビルド**
- **Docker Compose デプロイ**
- **コンテナ化されたデプロイ**

### 実施した内容

1. GitHub リポジトリの準備
2. ビルド仕様（buildspec.yml）の作成 - Docker イメージビルド
3. デプロイ仕様（appspec.yml）とスクリプトの作成 - Docker対応
4. CodeBuild プロジェクトの設定
5. CodeDeploy アプリケーションとデプロイグループの設定
6. CodePipeline によるフル CI/CD パイプラインの構築
7. 自動デプロイの動作確認

### PHP 版との主な違い

| 項目 | PHP 版 | Next.js 版 (Docker) |
|------|--------|---------------------|
| ビルド | 不要 | Docker イメージビルド必要 |
| 成果物 | ソースコード | Docker イメージ (tar) |
| デプロイ先 | /var/www/html | ~/simple-blog |
| 起動方式 | Apache 再起動 | docker compose up |
| 環境変数 | PHP ファイル内 | .env (docker-compose.yml) |
| デプロイ時間 | 短い | やや長い（イメージロードが必要） |

### CI/CD の利点

- **自動化**: コードプッシュから本番環境への反映まで自動
- **一貫性**: 毎回同じ手順でビルド・デプロイ
- **高速なフィードバック**: 問題を早期発見
- **ロールバック**: 失敗時は以前のバージョンに戻せる
- **チーム開発**: 複数人での開発がスムーズに

**Day14 では、CloudFormation を使用した IaC（Infrastructure as Code）を学習します。**
