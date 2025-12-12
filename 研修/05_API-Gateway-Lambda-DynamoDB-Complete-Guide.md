# AWS API Gateway + Lambda + DynamoDB 完全学習ガイド

> 本資料はラボ 5 を基に、実務で即使える知識を抽出した学習ノートです。  
> ラボ環境が利用できなくなった後も、この手順に従って自分で環境を再現・実験できます。

---

## 📊 全体アーキテクチャ

```
┌─────────────┐       ┌──────────────┐       ┌──────────┐       ┌──────────┐
│   Web App   │──────>│ API Gateway  │──────>│ Lambda   │──────>│DynamoDB  │
│  (React)    │ REST  │   /notes     │       │ 関数     │       │  テーブル │
└─────────────┘       └──────────────┘       └──────────┘       └──────────┘
```

### コンポーネント別の役割

| コンポーネント  | 役割                                                     |
| --------------- | -------------------------------------------------------- |
| **API Gateway** | REST API のエンドポイント提供、リクエスト検証、CORS 処理 |
| **Lambda**      | ビジネスロジック実装（データの取得・作成・更新）         |
| **DynamoDB**    | ノートデータの永続化、ユーザー別の管理                   |

---

## 🗄️ DynamoDB テーブル設計（基盤となる重要情報）

### テーブル仕様

| 項目                   | 値                         |
| ---------------------- | -------------------------- |
| **テーブル名**         | `Notes` （環境変数で注入） |
| **パーティションキー** | `UserId` (String)          |
| **ソートキー**         | `NoteId` (Number)          |

### テーブル内のデータ構造

| 列名     | データ型 | 説明                             | 例                          |
| -------- | -------- | -------------------------------- | --------------------------- |
| `UserId` | String   | ユーザーを一意に識別             | `"student"`                 |
| `NoteId` | Number   | 同じユーザー内でのノート識別番号 | `1`, `2`, `3`               |
| `Note`   | String   | ノートの本文内容                 | `"APIGatewayの設定方法..."` |

### DynamoDB のキー設計を理解する

- **パーティションキー (`UserId`)** = ユーザーごとにデータを分割
- **ソートキー (`NoteId`)** = 同じユーザーの中で複数のノートを区別

→ 結果：1 ユーザーが複数のノートを管理できる

---

## ⚙️ Lambda 関数の実装と役割

### 1️⃣ ListFunction（GET リクエスト用）

**用途：** ノートの一覧を取得

**エンドポイント：** `GET /notes`

#### 処理ロジック

| パターン           | 条件                        | 処理                                       |
| ------------------ | --------------------------- | ------------------------------------------ |
| **全件取得**       | `UserId` が指定されていない | `scan()` でテーブル全体をスキャン          |
| **ユーザー別取得** | `UserId` が指定されている   | `query()` で該当ユーザーのみを検索（高速） |

#### 完成版コード

```python
from __future__ import print_function
import boto3
import os
from boto3.dynamodb.conditions import Key

dynamoDBResource = boto3.resource('dynamodb')

def lambda_handler(event, context):
    print(event)
    ddbTable = os.environ['TABLE_NAME']
    databaseItems = getDatabaseItems(dynamoDBResource, ddbTable, event)
    return databaseItems

def getDatabaseItems(dynamoDBResource, ddbTable, event):
    print("getDatabaseItems Function")
    table = dynamoDBResource.Table(ddbTable)

    # UserId が指定されている場合は、そのユーザーのノートのみ取得
    if "UserId" in event:
        UserId = event['UserId']
        records = table.query(KeyConditionExpression=Key("UserId").eq(UserId))
    else:
        # UserId が指定されていない場合は、全ノートを取得
        records = table.scan()

    # 返すのは Items 配列のみ
    return records["Items"]
```

#### ポイント

✅ `query()` は高速（キーで検索）  
✅ `scan()` は遅い（全テーブルをスキャン）  
✅ 環境変数 `TABLE_NAME` を使用してテーブル名を外部化

---

### 2️⃣ CreateUpdateFunction（POST リクエスト用）

**用途：** ノートの作成または更新

**エンドポイント：** `POST /notes`

#### 処理ロジック

`put_item()` を使用することで、**UPSERT（存在しなければ作成、あれば更新）** を実現

| シナリオ                 | 動作                     |
| ------------------------ | ------------------------ |
| 新しい `UserId + NoteId` | **新規作成**（INSERT）   |
| 既存の `UserId + NoteId` | **上書き更新**（UPDATE） |

#### 完成版コード

```python
from __future__ import print_function
import boto3
import os

dynamoDBResource = boto3.resource('dynamodb')

def lambda_handler(event, context):
    print(event)

    UserId = event["UserId"]
    NoteId = event['NoteId']
    Note = event['Note']
    ddbTable = os.environ['TABLE_NAME']

    newNoteId = upsertItem(dynamoDBResource, ddbTable, UserId, NoteId, Note)
    return newNoteId

def upsertItem(dynamoDBResource, ddbTable, UserId, NoteId, Note):
    print('upsertItem Function')
    table = dynamoDBResource.Table(ddbTable)
    table.put_item(
        Item={
            'UserId': UserId,
            'NoteId': int(NoteId),
            'Note': Note
        }
    )
    return NoteId
```

#### ポイント

✅ `put_item()` = UPSERT の標準的な実装方法  
✅ `int(NoteId)` で型変換（DynamoDB では型が厳密）  
✅ リクエストから 3 つの値を抽出して保存

---

## 🔌 API Gateway の設定手順

### 基本設定

| 設定項目     | 値                   |
| ------------ | -------------------- |
| **API 名**   | `PollyNotesAPI`      |
| **リソース** | `/notes`             |
| **メソッド** | `GET`, `POST`        |
| **統合先**   | 対応する Lambda 関数 |

### 1️⃣ CORS 設定（必須）

**目的：** Web アプリから API へのアクセスを許可

```
✅ 以下を有効化：
  - GET, POST を許可
  - すべてのオリジン (*) からのアクセスを許可
  - Authorization ヘッダーを許可
```

**なぜ必要？**

- Web ブラウザのセキュリティポリシー（Same-Origin Policy）を緩和
- React の Web アプリから別ドメインの API へリクエスト可能にする

---

### 2️⃣ GET /notes の設定

#### Lambda 統合

- **統合先：** `list-function`
- **統合タイプ：** Lambda Function

#### マッピングテンプレート（リクエスト）

**目的：** API Gateway が受け取ったリクエストを Lambda に渡す形に変換

```json
{
  "UserId": "student"
}
```

**なぜ必要？**

- クライアント側のリクエストにはデータがない（GET なので）
- サーバー側で固定値 `"student"` をセットして、そのユーザーのノートを取得

#### マッピングテンプレート（レスポンス）

**目的：** Lambda の返り値から必要なデータだけを抽出

Velocity Template Language (VTL) を使用：

```vtl
#set($output = [])
#foreach($item in $input.path('$'))
    #set($map = {
        "NoteId": $item.NoteId,
        "Note": "$item.Note"
    })
    $output.add($map)
#end
$output
```

**処理の流れ：**

1. `$input.path('$')` で Lambda の返り値（配列）を取得
2. `#foreach` で各アイテムをループ
3. `UserId` を除いて、`NoteId` と `Note` だけを抽出
4. 新しい配列を構築して返す

---

### 3️⃣ POST /notes の設定

#### Lambda 統合

- **統合先：** `createUpdate-function`
- **統合タイプ：** Lambda Function

#### リクエスト検証（Model 作成）

**目的：** クライアントから正しい形式のデータが送られてきたかを検証

JSON Schema を定義：

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "UserId": {
      "type": "string"
    },
    "NoteId": {
      "type": "number"
    },
    "Note": {
      "type": "string"
    }
  },
  "required": ["UserId", "NoteId", "Note"]
}
```

**効果：**

- 必須フィールドが欠けていると 400 エラーを返す
- 型が異なると 400 エラーを返す
- Lambda 関数がリクエストデータの妥当性を信頼できる

---

### 4️⃣ デプロイ

#### ステージの作成と公開

```
1. ステージ名を作成：dev, test, prod など
2. 「デプロイ」ボタンをクリック
3. 「Invoke URL」が自動生成される

例：https://abcd1234.execute-api.ap-northeast-1.amazonaws.com/dev/notes
```

#### テスト方法

**curl で GET をテスト：**

```bash
curl https://<your-api-id>.execute-api.ap-northeast-1.amazonaws.com/dev/notes
```

**curl で POST をテスト：**

```bash
curl -X POST https://<your-api-id>.execute-api.ap-northeast-1.amazonaws.com/dev/notes \
  -H "Content-Type: application/json" \
  -d '{"UserId":"student","NoteId":1,"Note":"APIの勉強をしている"}'
```

---

## 🔄 ラボ後の環境再現方法

ラボ環境が利用できなくなった後も、以下の方法で同じ環境を構築・実験できます。

### 方法 A：AWS 無料枠で本番環境を再現（最も実務的）

#### 必要な作業

1. **DynamoDB テーブルを作成**

   - テーブル名：`Notes`
   - パーティションキー：`UserId` (String)
   - ソートキー：`NoteId` (Number)

2. **Lambda 関数を 2 つ作成**

   - `list-function` （ListFunction のコード）
   - `createUpdate-function` （CreateUpdateFunction のコード）
   - 環境変数 `TABLE_NAME=Notes` を設定

3. **API Gateway REST API を構築**
   - `/notes` リソース作成
   - GET, POST メソッド統合
   - CORS を有効化
   - ステージへデプロイ

#### 無料枠の範囲

| サービス    | 無料枠                       |
| ----------- | ---------------------------- |
| Lambda      | 100 万回/月                  |
| DynamoDB    | 25GB + 読取/書込ユニット無料 |
| API Gateway | 100 万 API 呼び出し          |

**結論：** このアーキテクチャなら**完全に無料**で動きます

---

### 方法 B：ローカルで SAM + LocalStack を使う（最強のデバッグ環境）

**メリット：**

- 💰 お金がかからない
- ⚡ Lambda, API Gateway, DynamoDB がローカルで完全に動く
- 🐛 デバッグが非常に簡単
- 📦 本番デプロイと同じコードを検証可能

#### セットアップ手順

1. **前提条件のインストール**

   ```bash
   # Docker のインストール（必須）
   # https://www.docker.com/

   # AWS SAM CLI のインストール
   pip install aws-sam-cli
   ```

2. **LocalStack の起動**

   ```bash
   docker run -d -p 4566:4566 localstack/localstack
   ```

3. **SAM テンプレート（template.yml）の作成**

   ```yaml
   AWSTemplateFormatVersion: "2010-09-09"
   Transform: AWS::Serverless-2016-10-31

   Parameters:
     TableName:
       Type: String
       Default: Notes

   Resources:
     NotesTable:
       Type: AWS::DynamoDB::Table
       Properties:
         TableName: !Ref TableName
         BillingMode: PAY_PER_REQUEST
         AttributeDefinitions:
           - AttributeName: UserId
             AttributeType: S
           - AttributeName: NoteId
             AttributeType: N
         KeySchema:
           - AttributeName: UserId
             KeyType: HASH
           - AttributeName: NoteId
             KeyType: RANGE

     ListFunction:
       Type: AWS::Serverless::Function
       Properties:
         CodeUri: list-function/
         Handler: app.lambda_handler
         Runtime: python3.9
         Environment:
           Variables:
             TABLE_NAME: !Ref TableName
         Policies:
           - DynamoDBCrudPolicy:
               TableName: !Ref TableName

     CreateUpdateFunction:
       Type: AWS::Serverless::Function
       Properties:
         CodeUri: createUpdate-function/
         Handler: app.lambda_handler
         Runtime: python3.9
         Environment:
           Variables:
             TABLE_NAME: !Ref TableName
         Policies:
           - DynamoDBCrudPolicy:
               TableName: !Ref TableName

     NotesApi:
       Type: AWS::Serverless::Api
       Properties:
         StageName: dev
         Cors: "'*'"
         Auth:
           DefaultAuthorizer: NONE

     ListApiResource:
       Type: AWS::ApiGateway::Resource
       Properties:
         RestApiId: !Ref NotesApi
         ParentId: !GetAtt NotesApi.RootResourceId
         PathPart: notes

     ListMethod:
       Type: AWS::ApiGateway::Method
       Properties:
         RestApiId: !Ref NotesApi
         ResourceId: !Ref ListApiResource
         HttpMethod: GET
         AuthorizationType: NONE
         Integration:
           Type: AWS_PROXY
           IntegrationHttpMethod: POST
           Uri: !Sub
             - arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${LambdaArn}/invocations
             - LambdaArn: !GetAtt ListFunction.Arn
   ```

4. **デプロイして実行**

   ```bash
   sam deploy --guided
   ```

5. **ローカルテスト**
   ```bash
   curl http://localhost:4566/restapis/<api-id>/dev/_user_request_/notes
   ```

---

### 方法 C：最小構成で DynamoDB + Lambda だけ実験

**用途：** Web フロントエンドなしで、バックエンドの動作確認だけしたい

#### DynamoDB へのアイテム追加

```bash
aws dynamodb put-item \
  --table-name Notes \
  --item '{"UserId":{"S":"student"},"NoteId":{"N":"1"},"Note":{"S":"勉強中"}}'
```

#### DynamoDB からのデータ取得

```bash
aws dynamodb query \
  --table-name Notes \
  --key-condition-expression "UserId = :userId" \
  --expression-attribute-values '{":userId":{"S":"student"}}'
```

#### Lambda をローカルで実行

```bash
sam local invoke ListFunction --event event.json
```

---

## 📚 このラボで学ぶべき重要ポイント

### 1. **アーキテクチャパターン**

REST API → Lambda → DynamoDB は、AWS における**最も一般的で標準的**なアーキテクチャです。

本ラボで学んだパターンは、実務でそのまま応用できます。

### 2. **DynamoDB の NoSQL 設計**

- パーティションキー + ソートキー のコンセプト
- Query と Scan の性能差の理解
- UPSERT（put_item）の活用

### 3. **CORS の理解**

- Web アプリとバックエンドが異なるドメインで動く場合の必須知識
- ブラウザのセキュリティモデルを理解することで、問題解決が容易になる

### 4. **マッピングテンプレート（VTL）**

- API Gateway でのリクエスト/レスポンス変換
- サーバーサイドでのデータフォーマット調整の基本

### 5. **環境変数による外部化**

- `TABLE_NAME` など、環境ごとに変わる設定値を外部から注入
- コードと設定の分離が、本番運用の基本

### 6. **Lambda の役割の明確化**

- 1 関数 = 1 責務の原則
- ListFunction と CreateUpdateFunction が完全に独立している設計

### 7. **ステージデプロイ**

- dev, test, prod などの複数環境を同じ API Gateway で管理
- 本番リリース前のテストステージが重要

---

## 💡 よくある質問と答え

### Q: DynamoDB と RDS（関係型 DB）はどう違う？

| 項目               | DynamoDB                     | RDS (MySQL など)             |
| ------------------ | ---------------------------- | ---------------------------- |
| **モデル**         | NoSQL（キー・バリュー）      | SQL（テーブル）              |
| **スケーリング**   | 自動スケール                 | 手動スケール                 |
| **参考整合性**     | 結果整合性                   | ACID 準拠                    |
| **向いている用途** | 高トラフィック、単純なクエリ | 複雑な検索、トランザクション |

**本ラボの場合：** DynamoDB は「ユーザーごとのノート」という単純な データ構造のため、NoSQL で十分です。

### Q: Lambda の実行時間に制限はある？

最大 15 分です。ただし、本ラボでは API リクエストに対する即座の応答なので、まったく問題になりません。

### Q: DynamoDB のコスト は？

- **無料枠：** 25GB のストレージ + 読取/書込ユニット無料
- **このアーキテクチャの月額：** ほぼ 0 円（無料枠で十分）

### Q: 複数ステージ（dev, test, prod）を運用する場合？

API Gateway は 1 つ で複数のステージを管理できます。ステージごとに異なる Lambda 関数やテーブルを指す環境変数を設定可能です。

---

## 📖 関連リソース

- **[AWS Lambda 公式ドキュメント](https://docs.aws.amazon.com/lambda/)**
- **[DynamoDB 開発者ガイド](https://docs.aws.amazon.com/dynamodb/)**
- **[API Gateway REST API 作成ガイド](https://docs.aws.amazon.com/apigateway/)**
- **[AWS SAM](https://aws.amazon.com/jp/serverless/sam/)** - Infrastructure as Code でのデプロイ自動化

---

**最終更新：2025 年 12 月**
