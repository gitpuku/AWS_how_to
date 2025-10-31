# Day13：AWS Code サービス + GitHub を使って CI/CD 環境を構築する

## 学習目標

AWS Code シリーズサービスと GitHub を組み合わせて、継続的インテグレーション・継続的デリバリー（CI/CD）環境を構築し、アプリケーションの自動化されたビルドとデプロイを実現する。

## CI/CD とは

CI/CD（継続的インテグレーション・継続的デリバリー）は、Web アプリケーション開発において日々のアップデートを効率的にリリースするための仕組みです。新しい機能の追加や不具合修正を継続的に行い、エンドユーザーに価値を届け続けるために必要不可欠な技術です。

## AWS Code シリーズの概要

### 使用するサービス

1. **AWS CodeBuild** - ソースコードのビルドとテスト環境を提供
2. **AWS CodeDeploy** - ビルド済み成果物を実行環境にデプロイ
3. **AWS CodePipeline** - 一連の工程を自動化し、CI/CD 環境全体を統合管理

### 全体のワークフロー

```
開発者 → GitHubにPush → CodePipeline → CodeBuild → CodeDeploy → EC2インスタンス
```

## 事前準備

### GitHub アカウントの準備

1. GitHub アカウントを作成（未作成の場合）
2. プライベートリポジトリを作成
3. Personal Access Token を生成

### GitHub リポジトリの作成

1. GitHub にログインし、新しいリポジトリを作成

   - リポジトリ名：`udemy-aws-14days-simple-blog`
   - プライベートリポジトリとして作成

2. Personal Access Token の生成
   - GitHub の設定 → Developer settings → Personal access tokens
   - スコープ：`repo`にチェック
   - **重要：トークンは安全に保管すること**

## 実習手順

### 1. ソースコードの準備

#### CloudShell での作業

1. 既存のシンプルブログのソースコードを修正

```bash
vim src/index.php
```

2. タイトルを「シンプルブログ Day13」に変更

3. GitHub にプッシュ

```bash
git init
git add -A
git commit -m "First commit"
git branch -M main
git remote add origin [GitHubリポジトリURL]
git push -u origin main
```

### 2. AWS CodeBuild の設定

#### S3 バケットの作成

1. S3 コンソールでバケットを作成
   - バケット名：`udemy-aws-14days-codebuild-artifacts`（ユニークな名前に変更）
   - バケットのバージョニングを有効化

#### buildspec.yml の編集

1. CloudShell で buildspec.yml を編集

```yaml
version: 0.2
phases:
  build:
    commands:
      - aws deploy push --application-name udemy-aws-14days-simple-blog --s3-location s3://[バケット名]/sample.zip --region ap-northeast-1
artifacts:
  files:
    - "**/*"
  name: sample.zip
  base-directory: src
```

2. アプリケーション名と S3 バケット名を自分の環境に合わせて修正

3. GitHub にプッシュ

```bash
git add -A
git commit -m "Update buildspec.yml"
git push origin main
```

#### CodeBuild プロジェクトの作成

1. CodeBuild コンソールで「プロジェクトを作成する」
2. プロジェクト名：`udemy-aws-14days-simple-blog`
3. ソースプロバイダー：GitHub
4. GitHub に接続（初回のみ）
   - CodeConnections で新しい接続を作成
   - 接続名：`udemy-aws-14days-connection`
   - リポジトリアクセス権限を設定
5. リポジトリを選択：作成したプライベートリポジトリ
6. ブランチ：main
7. 環境設定：推奨されている最新バージョンを選択
8. ビルドスペックファイルを使用する
9. アーティファクト：Amazon S3
   - バケット名：作成した S3 バケットを選択
   - 名前空間タイプ：ビルド ID

#### IAM ロールの権限追加

1. IAM コンソールで CodeBuild のサービスロールを確認
2. `CodeDeployDeployerAccess`ポリシーをアタッチ

### 3. AWS CodeDeploy の設定

#### EC2 インスタンス用 IAM ロールの作成

1. IAM コンソールで新しいロールを作成

   - サービス：EC2
   - ポリシー：`AmazonS3FullAccess`
   - ロール名：`udemy-aws-14days-web-role`

2. 既存の Web サーバー EC2 インスタンスにロールをアタッチ
   - Web1A、Web1C の両方に設定

#### EC2 インスタンスの準備

1. Web1C インスタンスを起動（停止している場合）
2. ターゲットグループに Web1C を追加登録

#### CodeDeploy エージェントのインストール

各 EC2 インスタンス（Web1A、Web1C）に SSH 接続してエージェントをインストール：

```bash
# Web1A、Web1Cの両方で実行
sudo yum install ruby -y
wget https://aws-codedeploy-ap-northeast-1.s3.ap-northeast-1.amazonaws.com/latest/install
chmod +x ./install
sudo ./install auto
```

#### CodeDeploy 用 IAM ロールの作成

1. IAM コンソールで新しいロールを作成
   - サービス：AWS CodeDeploy
   - ユースケース：CodeDeploy
   - ロール名：`udemy-aws-14days-codedeploy-role`

#### CodeDeploy アプリケーションの作成

1. CodeDeploy コンソールで「アプリケーションを作成」

   - アプリケーション名：`udemy-aws-14days-simple-blog`
   - プラットフォーム：EC2/オンプレミス

2. デプロイグループを作成
   - デプロイグループ名：`udemy-aws-14days-simple-blog-deploy-group`
   - サービスロール：作成した CodeDeploy ロール
   - デプロイタイプ：インプレース
   - 環境設定：EC2 インスタンス
   - キー：Name、値：Web1A、Web1C（両方指定）
   - デプロイ設定：OneAtATime（一台ずつデプロイ）
   - ロードバランサー：ターゲットグループを選択

### 4. AWS CodePipeline の設定

#### パイプラインの作成

1. CodePipeline コンソールで「パイプラインを作成する」
2. パイプライン名：`udemy-aws-14days-code-pipeline`
3. 各ステージの設定：

**ソースステージ**

- ソースプロバイダー：GitHub（アプリ経由）
- 接続：作成済みの接続を選択
- リポジトリ名：作成したリポジトリ
- ブランチ：main

**ビルドステージ**

- プロバイダー：AWS CodeBuild
- プロジェクト名：作成した CodeBuild プロジェクト

**テストステージ**

- スキップ

**デプロイステージ**

- プロバイダー：AWS CodeDeploy
- アプリケーション：作成した CodeDeploy アプリケーション
- デプロイグループ：作成したデプロイグループ

### 5. 動作確認

#### パイプラインの実行確認

1. パイプライン作成後、自動的に初回実行が開始される
2. 全てのステージが緑色になることを確認

#### ソースコード変更によるパイプライン起動確認

1. CloudShell で index.php を修正

```bash
vim src/index.php
# タイトルを「シンプルブログ コードシリーズ」に変更
```

2. GitHub にプッシュ

```bash
git add -A
git commit -m "Update for code series"
git push origin main
```

3. 自動的にパイプラインが起動することを確認
4. CloudFront ディストリビューションで変更が反映されることを確認

## 重要なポイント

### デプロイ方式

- **OneAtATime**: 一台ずつデプロイすることで、サービス停止時間を最小化
- ロードバランサーから一台を切り離し → デプロイ → 復帰 → 次の台 という流れ

### buildspec.yml

- CodeBuild での処理内容を定義
- ビルドコマンドやアーティファクト設定を記述

### appspec.yml

- CodeDeploy でのデプロイ内容を定義
- ファイル配置先やデプロイ前後の処理を記述

### セキュリティ

- 各サービス間の連携に適切な IAM ロール設定が必要
- Personal Access Token は機密情報として厳重に管理

## トラブルシューティング

### よくある問題

1. **権限エラー**: IAM ロールやポリシーの設定を確認
2. **デプロイ時間が長い**: ALB ターゲットグループの登録解除遅延設定を確認
3. **CodeDeploy エージェントエラー**: エージェントのインストール状況を確認

## プラスアルファチャレンジ

### 1. 設定項目の理解深化

- 今回使用しなかった設定や選択肢について調査
- 各設定の意味合いと使用場面を理解

### 2. 承認プロセスの追加

```
CodePipelineの編集 → ステージを編集 → アクショングループを追加 → 承認アクション
```

### 3. 高度な設定への挑戦

- Blue/Green デプロイの設定
- テストステージの追加
- 複数環境への対応

## まとめ

Day13 では、AWS Code シリーズと GitHub を組み合わせた CI/CD 環境を構築しました。これにより、ソースコードの変更から本番環境への自動デプロイまでの一連の流れを自動化することができました。

CI/CD は、アプリケーションエンジニアとインフラエンジニア双方の知識が必要な分野であり、実務においても重要度の高い技術です。今回の経験を通じて、CI/CD 環境構築の基礎知識と実践力を身に付けることができました。

**学習のポイント**

- 各 Code サービスの役割と連携方法の理解
- GitHub と AWS サービスの統合方法
- 自動化によるデプロイ効率の向上
- セキュリティを考慮した権限設定

次回 Day14 では、Infrastructure as Code（IaC）について学習し、インフラ構築の自動化に挑戦します。
