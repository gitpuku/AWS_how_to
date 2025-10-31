# Day10：AWS CLI、AWS SDK を使ってプログラマブルに AWS を操作する

## 目標

- AWS CLI を使ってコマンドラインから AWS リソースを操作する
- AWS SDK（boto3）を使って Python から AWS リソースを操作する
- EC2 インスタンスの停止・AMI 作成・起動を CLI 化する
- S3 バケットの作成とファイル操作をプログラムで実行する
- RDS のデータ更新バッチを Python で作成する

## 概要

これまでマネジメントコンソール（GUI）を使って様々な AWS サービスを操作してきましたが、AWS の真価は API ベースでプログラマブルに操作できることにあります。Day10 では以下を学習します：

1. **AWS CLI**：コマンドラインから AWS を操作
2. **AWS SDK**：プログラミング言語から AWS を操作（今回は Python 用の boto3 を使用）

## 事前準備

- Day9 までの環境が構築済みであること
- CloudShell が利用可能であること
- VPC、EC2、RDS、CloudFront が設定済みであること

## 1. AWS CLI による AWS リソース操作

### 1.1 AWS CLI の基本

- AWS CLI はコマンドラインから AWS サービスを操作するツール
- CloudShell には既にインストール済み
- コマンド形式：`aws [サービス名] [アクション] [オプション]`

### 1.2 EC2 インスタンスの停止

#### 手順

1. **マネジメントコンソールで EC2 画面を開く**

   - 現在の状態を確認（Web-1A: 実行中、Web-1C: 停止済み）

2. **AWS CLI ドキュメントの調べ方**

   ```bash
   # 「aws cli ec2」で検索
   # ドキュメントのExamplesセクションを参照する
   ```

3. **インスタンス停止コマンド**

   ```bash
   aws ec2 stop-instances --instance-ids i-xxxxxxxxxxxxxxxxx
   ```

   - `i-xxxxxxxxxxxxxxxxx`：Web-1A のインスタンス ID に置き換える
   - マネジメントコンソールからインスタンス ID をコピーする

4. **結果確認**
   - JSON レスポンスが返る
   - PreviousState: running → CurrentState: stopping
   - マネジメントコンソールで停止中になることを確認

### 1.3 AMI（Amazon Machine Image）の作成

#### 手順

1. **ドキュメント調査**

   ```bash
   # 「aws cli ec2 create-image」で検索
   # create-imageコマンドのドキュメントを確認
   ```

2. **AMI 作成コマンド**

   ```bash
   aws ec2 create-image --instance-id i-xxxxxxxxxxxxxxxxx --name "web-ami-from-cli"
   ```

   - `--instance-id`：停止した Web-1A のインスタンス ID
   - `--name`：AMI の名前を指定

3. **結果確認**
   - マネジメントコンソール > EC2 > AMI でステータス確認
   - 「保留」→「利用可能」になるまで待機

### 1.4 EC2 インスタンスの起動

#### 手順

1. **起動コマンド**

   ```bash
   aws ec2 start-instances --instance-ids i-xxxxxxxxxxxxxxxxx
   ```

2. **結果確認**
   - ステータスチェック 2/2 になるまで待機
   - マネジメントコンソールで実行中になることを確認

### 1.5 S3 バケットの作成

#### 手順

1. **ドキュメント調査**

   ```bash
   # 「aws s3 cli」で検索
   # mbコマンド（make bucket）を確認
   ```

2. **バケット作成コマンド**

   ```bash
   aws s3 mb s3://udemy-aws-14days-batch-[ユニーク文字列]
   ```

   - バケット名はグローバルでユニークにする必要がある
   - 日付や名前を付けてユニークにする

3. **結果確認**
   - S3 マネジメントコンソールでバケットが作成されていることを確認

## 2. AWS SDK（boto3）による開発

### 2.1 事前準備

#### CSV ファイルの準備

1. **posts.csv ファイルをダウンロードして編集**
   - 補足資料から posts.csv をダウンロード
   - CloudFront のディストリビューションドメイン名を確認
   - CSV ファイル内の 5 箇所のドメイン名を自分の CloudFront ドメインに置換
   - 編集後、S3 バケットにアップロード

#### バッチサーバーの作成

1. **EC2 インスタンス作成**

   - 名前：batch
   - サブネット：public-1c
   - キーペア：udemy-aws-14days
   - セキュリティグループ：新規作成
     - 名前：batch-security-group-sg
     - SSH（22）：任意の場所から

2. **高度なネットワーク設定**

   - プライベート IP アドレス：10.0.2.50

3. **ユーザーデータ**
   ```bash
   #!/bin/bash
   yum update -y
   yum install -y git
   ```

### 2.2 セキュリティグループの設定

#### DB 用セキュリティグループの修正

1. **EC2 > セキュリティグループ > db-sg を選択**
2. **インバウンドルールを編集**
3. **ルールを追加**
   - タイプ：MySQL/Aurora
   - ソース：batch-sg（batch 用セキュリティグループ）
4. **ルールを保存**

### 2.3 バッチサーバーへの SSH 接続とセットアップ

#### 接続と Python 環境確認

```bash
# CloudShellから接続
ssh -i udemy-aws-14days.pem ec2-user@[バッチサーバのパブリックIP]

# Pythonバージョン確認
python3 -V
```

#### 必要なパッケージのインストール

```bash
# pipのインストール
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python3 get-pip.py

# boto3のインストール
pip install boto3

# MySQL connector のインストール
pip install mysql-connector-python
```

### 2.4 AWS SDK を使った S3 ファイルダウンロード

#### Python スクリプトの作成

1. **ファイル作成**

   ```bash
   vi get_posts_from_s3.py
   ```

2. **コード実装**

   ```python
   import boto3

   # S3クライアントの作成
   s3_client = boto3.client('s3')

   # ファイルのダウンロード
   s3_client.download_file(
       'udemy-aws-14days-batch-[ユニーク文字列]',  # バケット名
       'posts.csv',                                    # S3上のファイル名
       'posts.csv'                                     # ローカルファイル名
   )
   ```

#### 認証エラーの対処

1. **IAM アクセスキーの作成**

   - IAM > ユーザー > [自分のユーザー] を選択
   - セキュリティ認証情報タブ
   - アクセスキーを作成
   - 用途：EC2 上のアプリケーションコード
   - 説明：udemy-aws-14days-batch

2. **AWS 認証情報の設定**
   ```bash
   aws configure
   ```
   - AWS Access Key ID: [作成したアクセスキー]
   - AWS Secret Access Key: [シークレットアクセスキー]
   - Default region name: ap-northeast-1
   - Default output format: json

#### スクリプトの実行

```bash
python3 get_posts_from_s3.py
```

### 2.5 RDS データ更新バッチの作成

#### GitHub からサンプルコードの取得

```bash
git clone https://github.com/[リポジトリ]/udemy-aws-14days.git
cd udemy-aws-14days/day10
cp update_posts_using_csv.py .
```

#### RDS エンドポイントの設定

1. **ファイル編集**

   ```bash
   vi update_posts_using_csv.py
   ```

2. **RDS エンドポイントの置換**
   - RDS > DB インスタンス > エンドポイントをコピー
   - スクリプト内の RDS_ENDPOINT を自分のエンドポイントに置換

#### データ更新バッチの実行

1. **実行前の状態確認**

   - シンプルブログで現在のデータを確認

2. **バッチ実行**

   ```bash
   python3 update_posts_using_csv.py
   ```

3. **結果確認**
   - シンプルブログを更新して新しいレコードが追加されていることを確認

### 2.6 セキュリティ対策

#### アクセスキーの削除

1. **IAM > ユーザー > セキュリティ認証情報**
2. **作成したアクセスキーを無効化**
3. **アクセスキーを削除**

> **重要**: アクセスキーは機密情報です。GitHub などにアップロードしないよう注意してください。

## 3. まとめと振り返り

### 学習内容

1. **AWS CLI**

   - EC2 インスタンスの停止・起動
   - AMI の作成
   - S3 バケットの作成
   - ドキュメントを参照しながらコマンドを調べる方法

2. **AWS SDK（boto3）**
   - S3 からのファイルダウンロード
   - RDS データの更新バッチ作成
   - 認証設定の重要性

### 重要なポイント

- **コマンドは覚える必要はない**：調べ方を覚えることが重要
- **ドキュメントの Examples を活用**する
- **アクセスキーの取り扱いに注意**する
- **次回（Day11）ではより安全な IAM ロールを学習**する

## 4. プラスアルファチャレンジ

### チャレンジ 1：ローカル PC での AWS CLI

- ローカル PC に AWS CLI をインストール
- `aws configure`でクレデンシャル設定
- 今回使用していないコマンドを試してみる

### チャレンジ 2：boto3 の他のメソッド

- `download_file`以外の S3 操作メソッドを試す
- 例：ファイル削除、リスト取得など
- boto3 ドキュメントを参照して実装

## 5. 次回予告

Day11 では、アクセスキーを使わずにより安全に AWS リソースにアクセスする方法（IAM ロール）について学習します。

## 注意事項

- アクセスキーは機密情報として適切に管理する
- 不要になったリソースは削除してコストを抑制する
- セキュリティグループの設定は最小権限の原則に従う
