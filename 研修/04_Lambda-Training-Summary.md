# 📘 AWS Lambda 研修まとめノート（重要ポイントだけ抜粋）

## 1. Lambda 関数の基本構築手順（Python）

### ① Lambda 関数の作成（GUI）

```
Lambda → 「関数を作成」

名前：dictate-function

ランタイム：Python 3.11

IAM ロール：lambdaPollyRole を使用
→ DynamoDB / S3 / Polly / CloudWatch Logs へのアクセスが付与されたロール。
```

### ② Lambda の特徴

- AWS コンソールでコード編集可能（3MB 未満の場合）
- Lambda コンテナ内の書き込み可能領域は /tmp のみ
- 外部サービスへは AWS SDK（boto3）でアクセス

---

## 2. dictate-function の処理フロー

この Lambda は「DynamoDB のテキストを音声化し、S3 に mp3 を置いて署名付き URL を返す」関数。

### 🔄 全体の 3 ステップ

1. DynamoDB からノートのテキスト取得
2. Polly で音声合成し、/tmp に MP3 を保存
3. S3 にアップロード → 署名付き URL を返す

> 📌 この流れは サーバレスでオンデマンド音声生成の基本構造。

---

## 3. 環境変数設定（超重要）

Lambda では外部値は環境変数で渡す。

```bash
aws lambda update-function-configuration \
--function-name dictate-function \
--environment Variables="{MP3_BUCKET_NAME=<bucket>, TABLE_NAME=Notes}"
```

### 使用環境変数：

| 変数名          | 説明                           |
| --------------- | ------------------------------ |
| MP3_BUCKET_NAME | MP3 を保存する S3 バケット     |
| TABLE_NAME      | DynamoDB のテーブル名（Notes） |

---

## 4. Lambda のコード構造（学びになるポイント）

研修の app.py 構造のポイントはこれ：

### ① クライアントの初期化はハンドラー外で行う（ベストプラクティス）

```python
dynamo = boto3.resource('dynamodb')
polly = boto3.client('polly')
s3 = boto3.client('s3')
```

**理由：**
Lambda はコンテナ再利用されるため、外で作ると高速化する。

---

## 5. DynamoDB から Note を取得（TODO1）

**正解コード：**

```python
return record['Item']['Note']
```

**理由：**
返すべきなのは項目全体ではなく Note 属性だけ。

---

## 6. Polly で MP3 を生成（TODO2）

**正解コード：**

```python
pollyResponse = pollyClient.synthesize_speech(
    OutputFormat='mp3',
    Text=text,
    VoiceId=VoiceId
)
```

**ポイント：**

- OutputFormat='mp3' を指定しないと失敗
- 返るのはバイナリストリーム → `with open('/tmp/xxx.mp3','wb')`

---

## 7. S3 に MP3 をアップロード（TODO3）

**正解コード：**

```python
s3Client.upload_file(
    filePath,
    mp3Bucket,
    f"{UserId}/{NoteId}.mp3"
)
```

**ポイント：**

- upload_file() を使う（put_file は存在しない）
- S3 Key（パス）にはユーザー ID/ノート ID を使用する構造が多い

---

## 8. 署名付き URL の生成

**典型的な boto3 コード：**

```python
url = s3Client.generate_presigned_url(
    ClientMethod='get_object',
    Params={'Bucket': mp3Bucket, 'Key': f"{UserId}/{NoteId}.mp3"},
    ExpiresIn=3600  # 1時間
)
```

✔ バケットやオブジェクトをパブリックにせずアクセスを渡せる  
✔ 一般的な API サービスのダウンロードリンクでよく使う手法

---

## 9. /tmp の扱い

Lambda では書き込みできるのは /tmp のみ：

- **容量：** 最大 512MB
- **生存期間：** 関数実行後、コンテナが再利用される場合はファイルが 残る

これは PolIy→MP3 保存 に必須の知識。

---

## 10. 学びの要点まとめ（超重要だけ）

1. Lambda では boto3 クライアントは関数外で初期化する
2. 外部値・設定は環境変数で管理
3. DynamoDB は Table.get_item() → Item → 属性取り出し
4. Polly は OutputFormat が必須
5. /tmp しか書き込めない
6. S3 は upload_file() が基本
7. ダウンロード URL は generate_presigned_url を使う
8. IAM ロールが全てのサービス連携の鍵

---

## 11. AWS CDK で環境を構築する（VSCode + EC2）

### 11.1 AWS CDK とは何か（初心者向け）

**AWS CDK（Cloud Development Kit）** は、プログラミング言語を使って AWS リソースを定義・構築するツールです。

#### 📚 従来のやり方 vs CDK のやり方

```
【従来の方法】
AWS マネジメントコンソール → ポチポチ → DynamoDB 作成
                         → ポチポチ → S3 バケット作成
                         → ポチポチ → IAM ロール作成
                         → ポチポチ → Lambda 関数作成

問題：手動で時間がかかる、ミスしやすい、再現性がない

【CDK のやり方】
app.py に環境全体をコード化 → cdk deploy → 自動で全リソース作成

利点：コード化、再利用可能、バージョン管理、高速化
```

#### 🔑 CDK の基本概念

| 用語          | 説明                             | 例                                     |
| ------------- | -------------------------------- | -------------------------------------- |
| **App**       | CDK プロジェクト全体             | cdk init app で作成される app.py       |
| **Stack**     | AWS のリソースをまとめるグループ | PollyNotesStack = 1 つの独立したセット |
| **Construct** | AWS リソース 1 つ 1 つ           | lambda.Function, dynamodb.Table など   |

#### 📊 階層構造

```
App（プロジェクト全体）
└── Stack（PollyNotesStack）
    ├── Construct（DynamoDB Table）
    ├── Construct（S3 Bucket）
    ├── Construct（IAM Role）
    └── Construct（Lambda Function）
```

#### CDK vs CloudFormation

```
CloudFormation（JSON/YAML）：
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Resources": {
    "NotesTable": {
      "Type": "AWS::DynamoDB::Table",
      "Properties": { ... }
    }
  }
}
→ 長くて書きにくい

CDK（Python）：
notes_table = dynamodb.Table(
    self, "NotesTable",
    table_name="Notes",
    partition_key=dynamodb.Attribute(...)
)
→ シンプルで読みやすい、プログラミング言語の力を活用
```

**CDK = CloudFormation の上位レイヤー**（自動的に CloudFormation テンプレートに変換される）

#### ✅ CDK を使うメリット

| メリット               | 説明                                     |
| ---------------------- | ---------------------------------------- |
| **書きやすい**         | Python/TypeScript で直感的に記述         |
| **再利用可能**         | コードをコピペで別プロジェクトに流用可能 |
| **プログラミング機能** | ループ、関数、条件分岐などが使える       |
| **冪等性**             | 何度実行しても同じ結果（同じ状態になる） |
| **バージョン管理**     | Git で構成変更を追跡可能                 |
| **自動クリーンアップ** | `cdk destroy` で全リソース削除           |
| **テスト可能**         | インフラをテストコードで検証可能         |

#### 🔄 CDK のワークフロー

```
1. cdk init → プロジェクト初期化
2. requirements.txt に必要なモジュール追加
3. lib/xxxxx_stack.py に Construct 実装
4. cdk synth → CloudFormation テンプレート生成（確認用）
5. cdk deploy → AWS にデプロイ
6. アプリ実行・テスト
7. コード修正 → cdk deploy で更新
8. cdk destroy → 全リソース削除
```

#### 💡 初心者が理解すべき 3 つのポイント

```
① Construct（部品）を理解する
   lambda_.Function() = Lambda 関数という部品
   dynamodb.Table() = DynamoDB テーブルという部品

② 部品を Stack に追加する
   Stack = 関連する部品をまとめるコンテナ

③ Stack を App で実行する
   App = Stack を実行し、CloudFormation テンプレートを生成
```

---

### 11.2 CDK セットアップと実装

ラボ環境が使えなくなった場合、AWS CDK で同じインフラを自動構築できます。

### 構築対象リソース

- ✅ DynamoDB テーブル（Notes）
- ✅ S3 バケット（MP3 保存用）
- ✅ IAM ロール（lambdaPollyRole）
- ✅ Lambda 関数（dictate-function ほか）
- ✅ EC2 インスタンス（VSCode 環境）

### 準備（EC2 SSH 接続後）

```bash
# Node.js のインストール確認
node --version
npm --version

# AWS CDK のインストール
npm install -g aws-cdk

# AWS CLI のインストール確認
aws --version

# AWS 認証情報の設定
aws configure
```

### CDK プロジェクト初期化

```bash
# プロジェクトフォルダの作成
mkdir polly-notes-cdk
cd polly-notes-cdk

# CDK プロジェクト初期化（Python）
cdk init app --language python

# 依存関係のインストール
source .venv/bin/activate
pip install -r requirements.txt
```

### CDK スタック実装例

**lib/polly-notes-stack.py** の実装：

```python
from aws_cdk import (
    aws_dynamodb as dynamodb,
    aws_s3 as s3,
    aws_iam as iam,
    aws_lambda as lambda_,
    core,
    Duration,
    RemovalPolicy,
)
import os

class PollyNotesStack(core.Stack):
    def __init__(self, scope: core.Construct, id: str, **kwargs) -> None:
        super().__init__(scope, id, **kwargs)

        # ===== DynamoDB テーブル（Notes）=====
        notes_table = dynamodb.Table(
            self, "NotesTable",
            table_name="Notes",
            partition_key=dynamodb.Attribute(
                name="UserId",
                type=dynamodb.AttributeType.STRING
            ),
            sort_key=dynamodb.Attribute(
                name="NoteId",
                type=dynamodb.AttributeType.STRING
            ),
            billing_mode=dynamodb.BillingMode.PAY_PER_REQUEST,
            removal_policy=core.RemovalPolicy.DESTROY
        )

        # ===== S3 バケット（MP3 保存用）=====
        mp3_bucket = s3.Bucket(
            self, "MP3Bucket",
            versioned=False,
            removal_policy=RemovalPolicy.DESTROY,
            auto_delete_objects=True
        )

        # ===== IAM ロール（Lambda 実行用）=====
        lambda_role = iam.Role(
            self, "LambdaPollyRole",
            role_name="lambdaPollyRole",
            assumed_by=iam.ServicePrincipal("lambda.amazonaws.com")
        )

        # DynamoDB アクセス権限
        lambda_role.add_to_policy(iam.PolicyStatement(
            actions=[
                "dynamodb:GetItem",
                "dynamodb:PutItem",
                "dynamodb:UpdateItem",
                "dynamodb:DeleteItem",
                "dynamodb:Query",
                "dynamodb:Scan"
            ],
            resources=[notes_table.table_arn]
        ))

        # S3 アクセス権限
        lambda_role.add_to_policy(iam.PolicyStatement(
            actions=[
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            resources=[mp3_bucket.bucket_arn + "/*"]
        ))

        # Polly アクセス権限
        lambda_role.add_to_policy(iam.PolicyStatement(
            actions=[
                "polly:SynthesizeSpeech"
            ],
            resources=["*"]
        ))

        # CloudWatch Logs アクセス権限
        lambda_role.add_to_policy(iam.PolicyStatement(
            actions=[
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            resources=["arn:aws:logs:*:*:*"]
        ))

        # ===== Lambda 関数（dictate-function）=====
        # app.py は EC2 の dictate-function フォルダから参照
        dictate_function = lambda_.Function(
            self, "DictateFunction",
            function_name="dictate-function",
            runtime=lambda_.Runtime.PYTHON_3_11,
            code=lambda_.Code.from_asset(
                os.path.join(os.path.dirname(__file__), "../", "functions/dictate-function")
            ),
            handler="app.lambda_handler",
            role=lambda_role,
            timeout=core.Duration.seconds(60),
            memory_size=256,
            environment={
                "TABLE_NAME": "Notes",
                "MP3_BUCKET_NAME": mp3_bucket.bucket_name
            }
        )

        # 出力情報
        core.CfnOutput(
            self, "DynamoDBTableName",
            value=notes_table.table_name,
            export_name="NotesTableName"
        )

        core.CfnOutput(
            self, "S3BucketName",
            value=mp3_bucket.bucket_name,
            export_name="MP3BucketName"
        )

        core.CfnOutput(
            self, "LambdaRoleArn",
            value=lambda_role.role_arn,
            export_name="LambdaPollyRoleArn"
        )
```

### CDK コード詳細説明

#### 🔍 Stack クラスの構造

```python
class PollyNotesStack(core.Stack):
    def __init__(self, scope: core.Construct, id: str, **kwargs) -> None:
        super().__init__(scope, id, **kwargs)

        # ここに Construct（リソース）を追加していく
```

**説明：**

- `scope`：親の Stack（通常は App）
- `id`：この Stack を識別する名前（AWS では `id` ベースの ID が付く）
- `super().__init__`：親クラスの初期化

#### 🔍 Construct（リソース）の作成パターン

**パターン 1：DynamoDB テーブル**

```python
notes_table = dynamodb.Table(
    self,                           # このスタックに追加
    "NotesTable",                   # 識別子（CDK 内用）
    table_name="Notes",             # AWS での実際の名前
    partition_key=dynamodb.Attribute(
        name="UserId",              # ユーザー ID で分割
        type=dynamodb.AttributeType.STRING
    ),
    sort_key=dynamodb.Attribute(
        name="NoteId",              # ノート ID でソート
        type=dynamodb.AttributeType.STRING
    ),
    billing_mode=dynamodb.BillingMode.PAY_PER_REQUEST,
    removal_policy=core.RemovalPolicy.DESTROY  # cdk destroy で削除
)
```

**パターン 2：S3 バケット**

```python
mp3_bucket = s3.Bucket(
    self, "MP3Bucket",
    versioned=False,                # バージョニング不要
    removal_policy=RemovalPolicy.DESTROY,  # 自動削除
    auto_delete_objects=True        # 内容物も一緒に削除
)
```

**パターン 3：IAM ロール**

```python
lambda_role = iam.Role(
    self, "LambdaPollyRole",
    role_name="lambdaPollyRole",    # AWS での名前
    assumed_by=iam.ServicePrincipal("lambda.amazonaws.com")  # Lambda が引き受け可能
)

# DynamoDB アクセス権限を追加
lambda_role.add_to_policy(iam.PolicyStatement(
    actions=[                        # どの操作を許可するか
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query",
        "dynamodb:Scan"
    ],
    resources=[notes_table.table_arn]  # どのリソースに対してか
))
```

#### 🔍 変数の参照

CDK では、作成したリソースの情報を別のリソースから参照できます：

```python
# テーブルの ARN（Amazon Resource Name）を参照
resources=[notes_table.table_arn]

# バケット名を参照
environment={
    "MP3_BUCKET_NAME": mp3_bucket.bucket_name
}

# ロールの ARN を参照
role=lambda_role
```

#### 🔍 出力情報（CfnOutput）

デプロイ完了後に値を出力します：

```python
core.CfnOutput(
    self, "DynamoDBTableName",      # 識別子
    value=notes_table.table_name,   # 出力する値
    export_name="NotesTableName"    # 他の Stack から参照可能にする
)
```

**出力例：**

```
Outputs:
DynamoDBTableName = Notes
S3BucketName = pollynotescdk-mp3bucket-abc123xyz
LambdaRoleArn = arn:aws:iam::123456789012:role/lambdaPollyRole
```

### CDK デプロイ手順

```bash
# AWS アカウント初期化（初回のみ）
cdk bootstrap

# スタック確認
cdk synth

# スタック デプロイ
cdk deploy --require-approval never

# 確認
aws dynamodb describe-table --table-name Notes
aws s3 ls
aws iam get-role --role-name lambdaPollyRole
```

### 💡 各コマンドの説明

| コマンド        | 意味                            | 用途                                                        |
| --------------- | ------------------------------- | ----------------------------------------------------------- |
| `cdk bootstrap` | CDK 用の初期セットアップ        | 初回のみ必要（CloudFormation デプロイ用の S3 バケット作成） |
| `cdk init`      | 新規プロジェクト作成            | `cdk init app --language python`                            |
| `cdk synth`     | CloudFormation テンプレート生成 | デプロイ前の確認（`cdk.out/` フォルダに JSON が生成される） |
| `cdk deploy`    | AWS にデプロイ                  | 実際にリソースを作成                                        |
| `cdk destroy`   | 全リソース削除                  | 不要になったリソースを一括削除                              |
| `cdk diff`      | 差分確認                        | 現在のコードと AWS の状態を比較                             |

### 🔄 実装フロー（推奨）

1. EC2 SSH 接続 → VSCode 起動
2. 既存コード確認（app.py など）
3. CDK プロジェクト初期化
4. スタック実装（上記参考）
5. `cdk deploy` でインフラ構築
6. Lambda 関数テスト
7. 必要に応じてコード修正 → `cdk deploy` で更新

---

### 🔄 実装フロー（推奨）

1. EC2 SSH 接続 → VSCode 起動
2. 既存コード確認（app.py など）
3. CDK プロジェクト初期化
4. スタック実装（上記参考）
5. `cdk deploy` でインフラ構築
6. Lambda 関数テスト
7. 必要に応じてコード修正 → `cdk deploy` で更新

---

### 11.3 初心者向け：CDK の実装ステップバイステップ

#### ステップ 1：プロジェクト初期化

```bash
# フォルダ作成
mkdir polly-notes-cdk
cd polly-notes-cdk

# CDK プロジェクト初期化
cdk init app --language python

# ファイル構成
polly-notes-cdk/
├── app.py               # メインファイル
├── polly_notes/         # スタック定義フォルダ
│   └── polly_notes_stack.py
├── requirements.txt     # Python パッケージリスト
├── .venv/              # 仮想環境
└── cdk.json            # CDK 設定
```

#### ステップ 2：Stack を実装（コピペで使用可能）

**polly_notes/polly_notes_stack.py**

```python
from aws_cdk import (
    aws_dynamodb as dynamodb,
    aws_s3 as s3,
    aws_iam as iam,
    aws_lambda as lambda_,
    core,
    RemovalPolicy,
)
import os

class PollyNotesStack(core.Stack):

    def __init__(self, scope: core.Construct, id: str, **kwargs) -> None:
        super().__init__(scope, id, **kwargs)

        # ===== リソース 1: DynamoDB テーブル =====
        notes_table = dynamodb.Table(
            self, "NotesTable",
            table_name="Notes",
            partition_key=dynamodb.Attribute(
                name="UserId",
                type=dynamodb.AttributeType.STRING
            ),
            sort_key=dynamodb.Attribute(
                name="NoteId",
                type=dynamodb.AttributeType.STRING
            ),
            billing_mode=dynamodb.BillingMode.PAY_PER_REQUEST,
            removal_policy=RemovalPolicy.DESTROY
        )

        # ===== リソース 2: S3 バケット =====
        mp3_bucket = s3.Bucket(
            self, "MP3Bucket",
            versioned=False,
            removal_policy=RemovalPolicy.DESTROY,
            auto_delete_objects=True
        )

        # ===== リソース 3: IAM ロール =====
        lambda_role = iam.Role(
            self, "LambdaPollyRole",
            role_name="lambdaPollyRole",
            assumed_by=iam.ServicePrincipal("lambda.amazonaws.com")
        )

        # DynamoDB 権限追加
        lambda_role.add_to_policy(iam.PolicyStatement(
            actions=["dynamodb:*"],
            resources=[notes_table.table_arn]
        ))

        # S3 権限追加
        lambda_role.add_to_policy(iam.PolicyStatement(
            actions=["s3:*"],
            resources=[mp3_bucket.bucket_arn + "/*"]
        ))

        # Polly 権限追加
        lambda_role.add_to_policy(iam.PolicyStatement(
            actions=["polly:SynthesizeSpeech"],
            resources=["*"]
        ))

        # ===== リソース 4: Lambda 関数 =====
        dictate_function = lambda_.Function(
            self, "DictateFunction",
            function_name="dictate-function",
            runtime=lambda_.Runtime.PYTHON_3_11,
            code=lambda_.Code.from_asset("functions/dictate-function"),
            handler="app.lambda_handler",
            role=lambda_role,
            timeout=core.Duration.seconds(60),
            memory_size=256,
            environment={
                "TABLE_NAME": "Notes",
                "MP3_BUCKET_NAME": mp3_bucket.bucket_name
            }
        )

        # ===== 出力情報 =====
        core.CfnOutput(self, "TableName", value=notes_table.table_name)
        core.CfnOutput(self, "BucketName", value=mp3_bucket.bucket_name)
        core.CfnOutput(self, "RoleArn", value=lambda_role.role_arn)
```

#### ステップ 3：App で Stack を実行

**app.py**（自動生成されるが修正）

```python
#!/usr/bin/env python3
import aws_cdk as cdk
from polly_notes.polly_notes_stack import PollyNotesStack

app = cdk.App()
PollyNotesStack(app, "PollyNotesStack")

app.synth()
```

#### ステップ 4：デプロイ

```bash
# 仮想環境有効化
source .venv/bin/activate

# 初回のみ
cdk bootstrap

# デプロイ
cdk deploy

# 削除時
cdk destroy
```

#### 💡 デプロイの流れ

```
CDK コード（Python）
    ↓ cdk synth で変換 ↓
CloudFormation テンプレート（JSON）
    ↓ cdk deploy で送信 ↓
AWS CloudFormation
    ↓ 自動展開 ↓
実際のリソース（DynamoDB、S3、IAM、Lambda）
```

---

### 11.4 デプロイ後の動作確認

# Lambda 関数をテスト

aws lambda invoke \
 --function-name dictate-function \
 --payload '{"UserId": "test", "NoteId": "1", "VoiceId": "Joanna"}' \
 response.json

cat response.json

````

### CDK コマンド集

```bash
# スタックの差分表示
cdk diff

# スタック削除（リソース削除）
cdk destroy

# 特定スタックのみデプロイ
cdk deploy PollyNotesStack

# 出力情報表示
cdk output
````

### 📌 CDK のメリット

✅ **インフラ as Code**：全リソースをコードで管理  
✅ **冪等性**：何度実行しても同じ結果  
✅ **再現性**：別の AWS アカウント/リージョンで再構築可能  
✅ **バージョン管理**：Git で構成管理  
✅ **自動クリーンアップ**：`cdk destroy` でリソース削除

### 🔄 実装フロー（推奨）

1. EC2 SSH 接続 → VSCode 起動
2. 既存コード確認（app.py など）
3. CDK プロジェクト初期化
4. スタック実装（上記参考）
5. `cdk deploy` でインフラ構築
6. Lambda 関数テスト
7. 必要に応じてコード修正 → `cdk deploy` で更新

---

## 12. トラブルシューティング

### Lambda デプロイ時の注意点

```bash
# app.py を Zip 化する際、関数ディレクトリ内で実行
cd ~/environment/dictate-function
zip dictate-function.zip app.py

# Lambda コード更新（マネジメントコンソール以外）
aws lambda update-function-code \
  --function-name dictate-function \
  --zip-file fileb://dictate-function.zip
```

### 環境変数が反映されない場合

```bash
# 環境変数の確認
aws lambda get-function-configuration --function-name dictate-function | grep Variables

# 環境変数の更新
aws lambda update-function-configuration \
  --function-name dictate-function \
  --environment Variables="{TABLE_NAME=Notes, MP3_BUCKET_NAME=your-bucket}"
```

### Lambda のタイムアウト

Polly の音声合成は時間がかかるため、Lambda のタイムアウトを増やす：

```bash
aws lambda update-function-configuration \
  --function-name dictate-function \
  --timeout 60 \
  --memory-size 256
```

### S3 署名付き URL の確認

```bash
# URL の有効期限確認（ExpiresIn = 3600 秒 = 1時間）
aws s3 presign s3://bucket-name/UserId/NoteId.mp3 --expires-in 3600
```

完成版`app.py`

```python
# This lambda function will get a note text from DynamoDB,
# convert the text to speech using Polly, save it as an MP3 file,
# upload to S3, and return a pre-signed URL for accessing the file.

from __future__ import print_function
import boto3
import os
from contextlib import closing

# Initialize AWS clients (Best practice: outside handler)
dynamoDBResource = boto3.resource('dynamodb')
pollyClient = boto3.client('polly')
s3Client = boto3.client('s3', endpoint_url='https://s3.' +
                        os.environ['AWS_REGION'] + '.amazonaws.com')


def lambda_handler(event, context):

    # Log debug information
    print(event)

    # Extract the user parameters
    UserId = event["UserId"]
    NoteId = event["NoteId"]
    VoiceId = event['VoiceId']

    # Environment variables
    mp3Bucket = os.environ['MP3_BUCKET_NAME']
    ddbTable = os.environ['TABLE_NAME']

    # 1. Get note text from DynamoDB
    text = getNote(dynamoDBResource, ddbTable, UserId, NoteId)

    # 2. Generate MP3 file using Polly
    filePath = createMP3File(pollyClient, text, VoiceId, NoteId)

    # 3. Upload MP3 to S3 and return signed URL
    signedURL = hostFileOnS3(s3Client, filePath, mp3Bucket, UserId, NoteId)

    return signedURL


def getNote(dynamoDBResource, ddbTable, UserId, NoteId):
    print("getNote Function")

    table = dynamoDBResource.Table(ddbTable)
    record = table.get_item(
        Key={
            'UserId': UserId,
            'NoteId': int(NoteId)
        }
    )

    # TODO 1 (completed): Return only the Note text
    return record['Item']['Note']


def createMP3File(pollyClient, text, VoiceId, NoteId):
    print("createMP3File Function")

    # TODO 2 (completed): Convert text to speech
    pollyResponse = pollyClient.synthesize_speech(
        OutputFormat='mp3',
        Text=text,
        VoiceId=VoiceId
    )

    if "AudioStream" in pollyResponse:
        postId = str(NoteId)
        filePath = os.path.join("/tmp/", postId + ".mp3")

        with closing(pollyResponse["AudioStream"]) as stream:
            with open(filePath, "wb") as file:
                file.write(stream.read())

    return filePath


def hostFileOnS3(s3Client, filePath, mp3Bucket, UserId, NoteId):
    print("hostFileOnS3 Function")

    # TODO 3 (completed): Upload file to S3
    objectKey = f"{UserId}/{NoteId}.mp3"
    s3Client.upload_file(filePath, mp3Bucket, objectKey)

    # Remove local file (security best practice)
    os.remove(filePath)

    # Generate presigned URL
    url = s3Client.generate_presigned_url(
        ClientMethod='get_object',
        Params={
            'Bucket': mp3Bucket,
            'Key': objectKey
        }
    )

    return url

```
