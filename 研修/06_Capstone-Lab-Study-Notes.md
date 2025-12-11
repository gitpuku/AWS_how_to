# AWS 研修ノート：ラボ 6 キャップストーン（FastAPI エンジニア向け解説版）

## 🎯 このラボで学ぶこと（重要ポイント）

- **Amazon Cognito** でユーザー認証を実装
- **JWT トークン** を使った API 認証の仕組み
- **API Gateway** が JWT を検証して Lambda を呼び出すまでの流れ
- **Swagger 定義書（YAML）** を使って API Gateway を自動構築
- フロントエンド（React SPA）から認証付き API を呼び出す方法

---

## 📖 このノートについて

あなたは **FastAPI** で API 開発を経験していますね？

このノートは「**Swagger の定義書の書き方**」と「**AWS での API 実装との違い**」をわかりやすく説明します。

**FastAPI vs AWS API Gateway**

| 項目           | FastAPI                       | AWS API Gateway              |
| -------------- | ----------------------------- | ---------------------------- |
| API の定義方法 | Python 装飾子（`@app.get()`） | YAML 設定ファイル（Swagger） |
| 認証ロジック   | FastAPI コード内に記述        | Cognito に委譲（自動）       |
| CORS 設定      | コード内で設定                | YAML に記述                  |
| デプロイ       | Uvicorn サーバー起動          | AWS が自動実行               |

---

## 📘 まず理解：「Swagger 定義」って何？

### FastAPI でのエンドポイント定義

あなたが FastAPI で書く場合：

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/notes/search")
def search_notes(text: str):
    """テキストで note を検索"""
    return {"results": []}
```

### 同じものを Swagger（YAML）で書く

**同じエンドポイント情報** を YAML という**設定ファイル形式** で書いたのが「Swagger 定義書」：

```yaml
paths:
  /notes/search:
    get:
      parameters:
        - name: text
          in: query
          required: true
          type: string
      responses:
        "200":
          description: Success
```

### 🔍 「YAML」ってなに？

- **テキストベースの設定ファイル形式**
- インデント（スペース）で階層を表現
- JSON より人間が読みやすい
- Docker、Kubernetes などでもよく使われる

```yaml
# YAML の基本
user: # キー（オブジェクト）
  name: Taro # キーと値をコロンで区切る
  age: 30
email_list: # 配列
  - user1@example.com
  - user2@example.com
```

### 💡 Swagger 定義書の役割

この YAML ファイルを **AWS API Gateway にインポート** すると、自動で以下が構築されます：

- ✅ エンドポイント作成
- ✅ パラメータ検証ルール
- ✅ CORS 設定
- ✅ Lambda 関数との連携
- ✅ 認証設定（Cognito）

**手動で AWS コンソールをポチポチいじる代わりに、YAML を書くだけで完成** ← これがメリット！

---

## 📘 1. Amazon Cognito：AWS の認証専門サービス

### 何をするのか？

**Cognito = AWS の認証専門サービス**

- ユーザー登録・管理
- パスワード認証
- **JWT（JSON Web Token）** 発行

### FastAPI での認証との違い

**FastAPI の場合：**

```python
from fastapi import Depends
from fastapi.security import HTTPBearer

security = HTTPBearer()

@app.get("/api/notes")
def get_notes(credentials = Depends(security)):
    # 自分でトークン検証ロジックを書く
    token = credentials.credentials
    if not validate_token(token):  # 自作関数
        raise HTTPException(status_code=401)
    return get_user_notes(user_id)
```

認証ロジック **すべてを自分で実装** する必要があります。

**AWS Cognito を使う場合：**

```
① ユーザーが Cognito にログイン
   ↓
② Cognito から JWT トークンを受け取る
   ↓
③ フロントが API 呼び出し時に JWT をヘッダーに付ける
   ↓
④ API Gateway が自動で「このトークン本物？」と Cognito に確認
   ↓
⑤ 本物なら Lambda を実行 / 偽物なら 401 を返却
```

**大きな違い：認証は AWS が全部やってくれる** → コードが簡潔！

### ✔️ 作成する項目

| 項目               | 説明                                 | 備考                                |
| ------------------ | ------------------------------------ | ----------------------------------- |
| **User Pool**      | ユーザーを管理する場所               | `PollyNotesPool` という名前をつける |
| **App Client**     | アプリが Cognito と通信するための ID | `PollyNotesWeb` という名前          |
| パスワードポリシー | 学習用なので簡単 OK                  | 「最小 8 文字」など                 |
| メール検証         | 学習なので不要                       | 無効化                              |

### ⚠️ 絶対にメモする（後で使う）

Cognito 作成後、以下の 3 つの情報は **Swagger に埋め込む** ため、テキストファイルにコピペして保管：

```
1. User Pool ID    例: ap-northeast-1_abc123xyz
2. User Pool ARN   例: arn:aws:cognito-idp:ap-northeast-1:123456789:userpool/ap-northeast-1_abc123xyz
3. Client ID       例: 5g8h9i0jk1l2m3n4o5p6q7r
```

---

## 📘 2. CLI でユーザーを登録してテスト

### 2 ステップのフロー

#### ステップ 1：ユーザーサインアップ

```bash
aws cognito-idp sign-up \
  --client-id YOUR_CLIENT_ID \
  --username student \
  --password Passw0rd!
```

#### ステップ 2：管理者が承認

```bash
aws cognito-idp admin-confirm-sign-up \
  --user-pool-id YOUR_USER_POOL_ID \
  --username student
```

### ✔️ JWT トークンの確認

ラボで提供される Web ページ（testLoginWebsite）でログインして、JavaScript コンソールで JWT を確認：

```javascript
// Cognito SDK でログイン後
const idToken = cognitoUser.getIdToken().getJwtToken();

// idToken は Base64 エンコードされたテキスト
// 例: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
// （デコードするとユーザー情報が入ってる）
```

**重要：この JWT がセキュリティキー**

- JWT を持っていない → API 実行できない
- JWT を持っている → API 実行できる

---

## 📘 3. API Gateway が JWT を自動検証

### FastAPI での認証フローとの違い

**FastAPI の場合：**

```python
# 毎回、認証ロジックを書く
@app.get("/api/notes")
def get_notes(credentials = Depends(security)):
    token = credentials.credentials
    user_id = validate_and_decode_token(token)  # ← コードが必要
    return get_user_notes(user_id)
```

**AWS API Gateway + Cognito の場合：**

```
リクエスト到着
   ↓
API Gateway が「Authorization ヘッダーの JWT 本物？」と自動確認
   ↓
確認OK → Lambda 実行（既にユーザー情報が含まれている）
確認NG → 401 Unauthorized を返却（Lambda は実行されない）
```

**ポイント：Lambda に到達する前に API Gateway がフィルタリングしてくれる**

### Swagger で「認証必須」と宣言

エンドポイントに以下を追加すると JWT 認証が必須になります：

```yaml
paths:
  /notes/search:
    get:
      security:
        - PollyNotesPool: [] # ← これで「Cognito 認証が必須」と宣言
```

### Swagger に認証設定を記述

Swagger ファイルの最初の方に以下を追加します：

```yaml
securityDefinitions:
  PollyNotesPool:
    type: "apiKey"
    name: "Authorization"
    in: "header"
    x-amazon-apigateway-authtype: "cognito_user_pools"
    x-amazon-apigateway-authorizer:
      type: "cognito_user_pools"
      providerARNs:
        - "arn:aws:cognito-idp:ap-northeast-1:123456789:userpool/ap-northeast-1_abc123xyz"
        # ↑ さっき記録した User Pool ARN をここに貼り付ける
```

### 🔍 各項目の意味

| 項目                             | 説明                                |
| -------------------------------- | ----------------------------------- |
| `type: apiKey`                   | ヘッダーのトークンで認証            |
| `name: Authorization`            | ヘッダー名は `Authorization` を期待 |
| `in: header`                     | ヘッダーから読み込む                |
| `x-amazon-apigateway-authorizer` | AWS 拡張：Cognito に確認を任せる    |

---

## 📘 4. Lambda がユーザー情報を受け取る

### JWT から「今のユーザーは誰？」を取得

API Gateway が JWT を検証したら、**Lambda はそのユーザー情報を受け取る** ことができます。

Swagger の **requestTemplates** セクションで以下のように書きます：

```yaml
x-amazon-apigateway-integration:
  requestTemplates:
    application/json: |
      {
        "UserId": "$context.authorizer.claims['cognito:username']",
        "text": "$input.params('text')"
      }
```

### VTL（Velocity Template Language）について

`$context` や `$input` は **VTL** という簡単なテンプレート言語です。API Gateway から Lambda へのデータ変換に使います。

| 式                                               | 意味                         |
| ------------------------------------------------ | ---------------------------- |
| `$context.authorizer.claims['cognito:username']` | JWT から取得したユーザー名   |
| `$input.params('text')`                          | クエリパラメータ `text` の値 |

### Lambda 関数が受け取るデータ

```python
def lambda_handler(event, context):
    user_id = event['UserId']      # "student" など
    search_text = event['text']    # "meeting" など

    # ビジネスロジック実装
    results = search_in_dynamodb(user_id, search_text)
    return results
```

---

## 📘 5. Swagger で Lambda を呼び出す設定

### AWS 拡張：x-amazon-apigateway-integration

このセクションで「API Gateway が Lambda をどう呼び出すか」を定義します。

```yaml
paths:
  /notes/search:
    get:
      security:
        - PollyNotesPool: [] # ← Cognito 認証必須

      x-amazon-apigateway-integration:
        httpMethod: "POST"
        uri: "arn:aws:apigateway:ap-northeast-1:lambda:path/2015-03-31/functions/arn:aws:lambda:ap-northeast-1:123456789:function:search-function/invocations"
        # ↑ Lambda 関数の ARN

        requestTemplates:
          application/json: |
            {
              "UserId": "$context.authorizer.claims['cognito:username']",
              "text": "$input.params('text')"
            }
```

### 各項目の説明

| 項目                 | 説明                                                     |
| -------------------- | -------------------------------------------------------- |
| `httpMethod: "POST"` | Lambda を呼び出す際は POST を使用（Lambda 側は関係ない） |
| `uri`                | Lambda 関数の ARN。どの Lambda を呼び出すかを指定        |
| `requestTemplates`   | API Gateway から Lambda への「入力データ」の形式         |

---

## 📘 6. CORS 設定（ブラウザからの API 呼び出しを許可）

### CORS とは？

ブラウザ（React SPA）が別ドメインの API を呼び出す場合、**CORS（Cross-Origin Resource Sharing）** という許可が必要です。

Swagger では `options` メソッドで自動設定：

```yaml
paths:
  /notes/search:
    get:
      # GET の定義...

    options:
      x-amazon-apigateway-integration:
        responses:
          default:
            statusCode: "200"
            responseParameters:
              method.response.header.Access-Control-Allow-Methods: "'GET,OPTIONS'"
              method.response.header.Access-Control-Allow-Headers: "'*'"
              method.response.header.Access-Control-Allow-Origin: "'*'"
        type: "mock" # Lambda を実行しない
```

**ポイント：**

- `type: "mock"` → Lambda を呼び出さず、API Gateway が直接応答
- `Access-Control-Allow-Origin: '*'` → どのドメインからでもアクセス OK（学習用）

---

## 📘 7. React（フロントエンド）での実装

### 実装フロー

```
① Cognito Hosted UI でログイン
② Cognito から JWT を受け取る
③ React で API を呼び出し（JWT をヘッダーに付ける）
④ API Gateway が JWT を自動検証
⑤ Lambda が実行される
```

### コード例（JavaScript / React）

```javascript
// 1. Cognito にログイン（既に実装済みと仮定）
const idToken = cognitoUser.getIdToken().getJwtToken();

// 2. API を呼び出し（JWT をヘッダーに付ける）
fetch(
  "https://xxx.execute-api.ap-northeast-1.amazonaws.com/Prod/notes/search?text=meeting",
  {
    method: "GET",
    headers: {
      Authorization: idToken, // ← JWT を Authorization ヘッダーに入れる
    },
  }
)
  .then((response) => response.json())
  .then((data) => {
    console.log("検索結果:", data);
  })
  .catch((error) => console.error("エラー:", error));
```

**ポイント：**

- `Authorization: <JWT>` → API Gateway が JWT を取得して Cognito に確認
- 確認 OK → Lambda 実行 → 結果を返す

---

## 📚 8. ラボ終了後：自分の環境で継続学習

### ✔️ (A) AWS 無料枠で再構築

ラボが終了しても、**AWS 無料枠だけで同じ環境を完全再現できます。**

**必要な AWS サービス:**

| サービス        | 用途               | 無料枠                           |
| --------------- | ------------------ | -------------------------------- |
| **API Gateway** | エンドポイント管理 | 月 100 万リクエスト              |
| **Lambda**      | ロジック実行       | 月 100 万実行回数                |
| **DynamoDB**    | データ保存         | オンデマンド課金（少量なら無料） |
| **S3**          | ファイル保存       | 1 年間無料                       |
| **Cognito**     | 認証管理           | 月 5 万ユーザーまで無料          |

**メリット：**

- 既にコードを持っているので、サービス作成だけで動く
- 本番環境に近い環境で学習できる

### ✔️ (B) CDK（Infrastructure as Code）で自動化

時間があれば AWS CDK を使うと効率的：

```python
# CDK で以下を全部自動構築可能
from aws_cdk import (
    aws_cognito,
    aws_apigateway,
    aws_lambda,
    aws_dynamodb,
    aws_s3,
    core
)

# Cognito / API Gateway / Lambda / DynamoDB / S3 を自動作成
```

**メリット：**

- 再現性が高い（コード = インフラの定義書）
- 削除 → 再作成が簡単
- Swagger は CDK でインポート可能

### ✔️ (C) ローカル開発（AWS アカウント不要）

ラボがなくても **ローカルで** 開発・テスト可能：

#### 1️⃣ Lambda をローカルで実行

```bash
# AWS SAM CLI を使用
sam local start-api

# http://localhost:3000/notes/search?text=test でテスト可能
```

#### 2️⃣ JWT をローカルで自作

Cognito の代わりに JWT 発行ライブラリを使う：

**Python の場合：**

```python
from jose import jwt

secret = "my-secret-key"
payload = {
    "cognito:username": "student",
    "sub": "12345"
}

token = jwt.encode(payload, secret, algorithm="HS256")
print(token)  # JWT トークンが出力される
```

**Node.js の場合：**

```javascript
const jwt = require("jsonwebtoken");

const secret = "my-secret-key";
const payload = {
  "cognito:username": "student",
  sub: "12345",
};

const token = jwt.sign(payload, secret, { algorithm: "HS256" });
console.log(token); // JWT トークンが出力される
```

**開発時は ダミー JWT でも十分です。**

### ✔️ (D) 最小構成で再現

自分の AWS アカウントで以下を作成するだけで再現完了：

```
Cognito User Pool
  ↓
API Gateway (REST API)
  ↓
Lambda (3 functions)
  ↓
DynamoDB + S3
```

---

## 📝 最重要ポイント：4 つの技術の連携

このラボの核は、以下 4 つの技術がどう連携するかを理解することです。

| AWS サービス    | 役割                           | 責務                                 |
| --------------- | ------------------------------ | ------------------------------------ |
| **Cognito**     | 認証                           | ユーザー管理・JWT 生成               |
| **API Gateway** | リクエスト受付＆フィルタリング | JWT 検証・Lambda 呼び出し・CORS 処理 |
| **Lambda**      | ビジネスロジック               | 検索・音声生成・削除処理             |
| **Swagger**     | 設定の効率化                   | YAML で API Gateway を自動構築       |

### 💡 あなたが持っている Swagger 定義ファイルは実務級

- **AWS 経験者が書いた本物のテンプレート**
- **自分の環境にそのままデプロイ可能**
- **プロダクション環境への応用も容易**

このファイルをマスターすれば、実務でもすぐに使えます。

---

## 🔗 参考リファレンス

- 【ファイル】[PollyNotesAPI-swagger.yaml](./environment/swagger/PollyNotesAPI-swagger.yaml)
- 【前のノート】[05_API-Gateway-Lambda-DynamoDB-Complete-Guide.md](./05_API-Gateway-Lambda-DynamoDB-Complete-Guide.md)
- 【公式】[AWS Cognito ドキュメント](https://docs.aws.amazon.com/ja_jp/cognito/)
- 【公式】[API Gateway ドキュメント](https://docs.aws.amazon.com/ja_jp/apigateway/)
- 【公式】[OpenAPI 2.0 仕様](https://swagger.io/specification/v2/)
