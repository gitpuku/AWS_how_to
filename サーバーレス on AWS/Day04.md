# API Gateway と Lambda 関数の統合ハンズオン

# Day4-2

## HTTP API の AWS Lambda プロキシ統合の return 文

```json
{
  "isBase64Encoded": false,
  "statusCode": 200,
  "body": "{ \"message\": \"Hello from Lambda!\" }",
  "headers": {
    "content-type": "application/json"
  }
}
```

## Day4-2 終了断面の get_all_weather_function

```py
import json

def lambda_handler(event, context):
    # 固定の天気予報データ
    weather_data = [
        {
            "cityId": 1,
            "cityName": "札幌",
            "weatherId": 12,
            "weatherName": "雨",
            "rainfallProbability": 90
        },
        {
            "cityId": 13,
            "cityName": "東京",
            "weatherId": 4,
            "weatherName": "くもり",
            "rainfallProbability": 70
        },
        {
            "cityId": 23,
            "cityName": "名古屋",
            "weatherId": 4,
            "weatherName": "くもり",
            "rainfallProbability": 50
        },
        {
            "cityId": 27,
            "cityName": "大阪",
            "weatherId": 4,
            "weatherName": "くもり",
            "rainfallProbability": 30
        },
        {
            "cityId": 40,
            "cityName": "博多",
            "weatherId": 2,
            "weatherName": "晴れ",
            "rainfallProbability": 10
        }
    ]

    return {
        "isBase64Encoded": False,
        "statusCode": 200,
        "body": json.dumps(weather_data, ensure_ascii=False),
        "headers": {
            "Content-Type": "application/json"
        }
    }
```

# Day4-3 天気予報一覧 JSON を取得する Web API（GET /all）を構築する

## ハンズオン手順

### 1. Lambda 関数の作成

#### 1.1 基本設定

1. Lambda のコンソール画面に移動
2. 「関数の作成」をクリック
3. 以下の設定で関数を作成：
   - 関数名：`get-all-weather-function`
   - ランタイム：Python 3.13
   - その他：デフォルト設定

#### 1.2 IAM ロールの設定

1. 作成した関数の「設定」タブを選択
2. 「アクセス権限」からロール名をクリック
3. 「許可を追加」→「ポリシーをアタッチ」を選択
4. 検索窓で「DynamoDB」と入力
5. `AmazonDynamoDBReadOnlyAccess` を選択して追加

### 2. API Gateway の設定

#### 2.1 API の作成

1. API Gateway のコンソール画面に移動
2. 「API の作成」をクリック
3. HTTP API の「構築」をクリック
4. API 名に `simple-weather-news-api` を入力
5. 「次へ」を 3 回クリックし、「作成」を選択

#### 2.2 ルートの設定

1. ルートを選択し「作成」をクリック
2. 以下の設定でルートを作成：
   - メソッド：GET
   - リソースパス：/all
3. 作成した GET メソッドをクリック
4. 「統合をアタッチ」→「統合を作成してアタッチ」を選択
5. 統合タイプで「Lambda 関数」を選択
6. リージョンが「ap-northeast-1」（東京）であることを確認
7. Lambda 関数 `get_city_weather_function` を選択して作成

### 3. 動作確認

1. API Gateway のコンソールで「デフォルトのエンドポイント」の URL をコピー
2. ブラウザで `[エンドポイントURL]/all` にアクセス
3. JSON データが返却されることを確認

# Day4-3 終了断面の get_all_weather_function

```py
import json
import boto3

dynamodb_client = boto3.client('dynamodb')

def lambda_handler(event, context):
    response = dynamodb_client.scan(
        TableName='simple-weather-news-table',
    )
    items = response['Items']

    weather_data = []

    for item in items:
        weather_item = {
            "cityId": int(item['CityId']['N']),
            "cityName": item['CityName']['S'],
            "weatherId": int(item['WeatherId']['N']),
            "weatherName": item['WeatherName']['S'],
            "rainfallProbability": int(item['RainfallProbability']['N'])
        }
        weather_data.append(weather_item)

    weather_data.sort(key=lambda x: x['cityId'])

    return {
        "isBase64Encoded": False,
        "statusCode": 200,
        "body": json.dumps(weather_data, ensure_ascii=False),
        "headers": {
            "content-type": "application/json"
        }
    }
```

## Day4-4（このセクションの目的）

この動画パート（Day4-4）では、パスパラメータを使って特定都市の天気情報だけを返す Web API を作成します。

ポイント：リクエストパスの末尾に含まれる cityId（例：/city/13 の 13）を Lambda に渡し、その ID を使って DynamoDB の `get_item` を実行し、該当データを返却します。以下の手順に沿って進めてください。

### 手順（要点）

1. Lambda 関数を作成

   - 関数名：`get_city_weather_function`
   - ランタイム：Python 3.13

2. IAM ロールに DynamoDB 参照権限を付与

   - 関数画面の「設定」→「アクセス権限」でロール名をクリック
   - 「許可を追加」→「ポリシーをアタッチ」→ `AmazonDynamoDBReadOnlyAccess` を追加

3. 初期実装（固定 ID で検証）
   - まずはコード内で `city_id = "1"` のように固定して実装し、動作確認を行います。
   - 必要な import とクライアント作成：

```py
import json
import boto3

dynamodb_client = boto3.client('dynamodb')
```

    - `get_item` の例：

```py
response = dynamodb_client.get_item(
     TableName='simple-weather-news-table',
     Key={ 'CityId': { 'N': city_id } }
)
```

    - `print(response)` を入れて CloudWatch で `response['Item']` の構造を確認します。

以下は Day4-4 の終了断面コードです：

```py
import json
import boto3

dynamodb_client = boto3.client('dynamodb')

def lambda_handler(event, context):

    city_id = "1"

    response = dynamodb_client.get_item(
        TableName='simple-weather-news-table',
        Key={
            'CityId': {
                'N': city_id
            }
        }
    )

    if 'Item' not in response:
        return {
            "isBase64Encoded": False,
            "statusCode": 404,
            "body": json.dumps({"error": "指定された都市が見つかりません"}, ensure_ascii=False),
            "headers": {
                "content-type": "application/json"
            }
        }

    item = response['Item']
    weather_detail = {
        "cityId": int(item['CityId']['N']),
        "cityName": item['CityName']['S'],
        "weatherId": int(item['WeatherId']['N']),
        "weatherName": item['WeatherName']['S'],
        "rainfallProbability": int(item['RainfallProbability']['N'])
    }

    return {
        "isBase64Encoded": False,
        "statusCode": 200,
        "body": json.dumps(weather_detail, ensure_ascii=False),
        "headers": {
            "content-type": "application/json"
        }
    }
```

# Day4-5

Day4-5 パスパラメータを使った個別都市情報取得 API の完成

## Day4-5 の目的

Day4-4 で作成した `get_city_weather_function` の cityId を決め打ちから、API Gateway のパスパラメータから取得する形に変更し、実際に API Gateway と統合して動作確認を行います。

## ハンズオン手順

### 1. Lambda 関数の編集画面に戻る

1. Lambda のコンソールを開く
2. `get_city_weather_function` を選択
3. 「コード」タブを表示（コードエディタを最大化すると見やすい）

### 2. テストイベントの作成（API Gateway からのイベント構造を確認）

#### 2.1 テストイベントのテンプレートを確認

1. 「Test」クリックし、「Create test event」を選択
2. 「Template - optional」のプルダウンをクリック
3. 「API Gateway HTTP API」を選択
4. テンプレート JSON が表示される
   - このテンプレートは API Gateway HTTP API が Lambda に渡す JSON の構造を示しています
   - 下の方に `"pathParameters"` というセクションがあることを確認

#### 2.2 pathParameters の構造を確認

テンプレート内の `pathParameters` セクションに以下のような記述があります：

```json
"pathParameters": {
  "parameter1": "value1"
}
```

parameter1 を`cityId`、値を `13` に変更する
`pathParameters`をコピーして、イベントの後ろに追加します：

```py
city_id = event['pathParameters'] ['cityId']
```

⚠️ **重要な注意点**：

- キー名は **`cityId`** とする（`c` は小文字、`I` と `d` は大文字）
- IDE の補完機能が `city-id` や `city_id` を提案することがありますが、それらは使わない
- API Gateway のリソースパスで定義する名前と完全に一致させる必要があります

#### 2.3 テストイベントの保存

1. イベント名を「`APIGWTest`」などとする
2. 以下の内容でテストイベントを作成：

```json
{
  "pathParameters": {
    "cityId": "13"
  }
}
```

3. 「Save」をクリック
4. テストイベント作成画面を閉じる

### 3. Lambda 関数コードの修正

```py
import json
import boto3

dynamodb_client = boto3.client('dynamodb')

def lambda_handler(event, context):

    city_id = event['pathParameters']['cityId']

    response = dynamodb_client.get_item(
        TableName='simple-weather-news-table',
        Key={
            'CityId': {
                'N': city_id
            }
        }
    )

    if 'Item' not in response:
        return {
            "isBase64Encoded": False,
            "statusCode": 404,
            "body": json.dumps({"error": "指定された都市が見つかりません"}, ensure_ascii=False),
            "headers": {
                "content-type": "application/json"
            }
        }

    item = response['Item']
    weather_detail = {
        "cityId": int(item['CityId']['N']),
        "cityName": item['CityName']['S'],
        "weatherId": int(item['WeatherId']['N']),
        "weatherName": item['WeatherName']['S'],
        "rainfallProbability": int(item['RainfallProbability']['N'])
    }

    return {
        "isBase64Encoded": False,
        "statusCode": 200,
        "body": json.dumps(weather_detail, ensure_ascii=False),
        "headers": {
            "content-type": "application/json"
        }
    }
```

#### 3.3 コードのデプロイ

1. 「Deploy」ボタンをクリック
2. デプロイ完了を確認

### 4. Lambda 関数のテスト実行

#### 4.1 テストの実行

1. 「Test」タブで先ほど作成した「APIGWTest」を選択
2. 三角の「Invoke」ボタンをクリック
3. 実行結果を確認
   - cityId: 13（東京）のデータが返却されることを確認

#### 4.2 別のパスパラメータでテスト

1. テストイベントの編集（鉛筆マーク）をクリック
2. `"cityId": "40"` に変更
3. 「Save」→「Invoke」を実行
4. 博多のデータが返却されることを確認

#### 4.3 存在しない ID でテスト（異常系）

1. テストイベントの編集をクリック
2. `"cityId": "99"` に変更
3. 「Save」→「Invoke」を実行
4. 404 エラーと「指定された都市が見つかりません」メッセージが返却されることを確認

### 5. API Gateway との統合

#### 5.1 API Gateway の画面を開く

1. API Gateway のコンソールに移動
2. `simple-weather-news-api` を選択

#### 5.2 新しいルートの作成

現在は `/all` の GET メソッドのみが存在するので、個別都市取得用のルートを追加します。

1. 「Routes」（ルート）セクションで「Create」をクリック
2. 以下の設定でルートを作成：
   - **メソッド**：`GET`
   - **リソースパス**：`/{cityId}`
     - ⚠️ **重要**：波括弧 `{}` で囲むことでパスパラメータとして扱われます
     - ⚠️ **重要**：パラメータ名は `cityId`（`c` は小文字、`Id` は大文字）と正確に入力
     - Lambda 関数内で参照する名前と一致させる必要があります
3. 「Create」をクリック

#### 5.3 Lambda 統合の設定

1. 作成した `GET /{cityId}` ルートをクリック
2. 「統合をアタッチ」をクリック
3. 「統合を作成してアタッチ」を選択
4. 以下の設定で統合を作成：
   - **統合タイプ**：Lambda 関数
   - **リージョン**：ap-northeast-1（東京）
   - **Lambda 関数**：`get_city_weather_function` を選択
     - ⚠️ **注意**：`get_all_weather_function` ではなく `get_city_weather_function` を選ぶ
5. 「作成」をクリック

### 6. API の動作確認

#### 6.1 エンドポイント URL の取得

1. API Gateway のコンソールで `simple-weather-news-api` を選択
2. 「デフォルトのエンドポイント」の URL をコピー

#### 6.2 ブラウザでの動作確認

以下の URL にブラウザでアクセスして動作を確認します：

**1. ルートパス（エラー確認）**

```
[エンドポイントURL]/
```

→ エラーが返却される（ルートへの GET は定義していないため）

**2. 全件取得（既存機能）**

```
[エンドポイントURL]/all
```

→ 全都市の天気情報の配列が返却される

**3. 東京の天気（cityId: 13）**

```
[エンドポイントURL]/13
```

→ 東京の天気情報のみが返却される

```json
{
  "cityId": 13,
  "cityName": "東京",
  "weatherId": 4,
  "weatherName": "くもり",
  "rainfallProbability": 70
}
```

**4. 大阪の天気（cityId: 27）**

```
[エンドポイントURL]/27
```

→ 大阪の天気情報のみが返却される

**5. 存在しない都市（cityId: 99）**

```
[エンドポイントURL]/99
```

→ 404 エラーと「指定された都市が見つかりません」が返却される

## Day4-5 完成時の構成

### API Gateway のルート構成

| メソッド | パス        | Lambda 関数               | 機能                     |
| -------- | ----------- | ------------------------- | ------------------------ |
| GET      | `/all`      | get_all_weather_function  | 全都市の天気情報を取得   |
| GET      | `/{cityId}` | get_city_weather_function | 指定都市の天気情報を取得 |

### 完成した API の仕様

1. **GET /all**

   - 全都市の天気情報を配列で返却
   - DynamoDB の `scan` を使用

2. **GET /{cityId}**
   - パスパラメータで指定された cityId の天気情報を返却
   - DynamoDB の `get_item` を使用
   - 存在しない cityId の場合は 404 エラーを返却

## Day4-5 終了断面の get_city_weather_function

コード内の重要な変更箇所：

```py
city_id = event['pathParameters']['cityId']
```

この 1 行により、API Gateway から渡されるパスパラメータを Lambda 関数で取得できるようになります。

## テスト用 JSON（正常系）

```json
{
  "pathParameters": {
    "cityId": "13"
  }
}
```

## テスト用 JSON（異常系）

```json
{
  "pathParameters": {
    "cityId": "99"
  }
}
```

## Day4 全体のまとめ

Day4 のハンズオンで以下を実現しました：

1. **API Gateway HTTP API の作成**

   - `simple-weather-news-api` という名前の HTTP API を作成

2. **Lambda 関数を 2 つ作成**

   - `get_all_weather_function`：全件取得用
   - `get_city_weather_function`：個別取得用

3. **API Gateway と Lambda の統合**

   - 2 つのルート（`/all` と `/{cityId}`）を作成し、それぞれ対応する Lambda 関数と統合

4. **DynamoDB との連携**

   - `scan` オペレーションで全件取得
   - `get_item` オペレーションでキー指定取得

5. **RESTful な API の実装**
   - パスパラメータを使った動的なルーティング
   - 適切な HTTP ステータスコード（200, 404）の返却

これで、サーバーレスアーキテクチャを使った実用的な Web API が完成しました。

````

## テスト用 JSON（正常系）

```json
{
  "pathParameters": {
    "cityId": "13"
  }
}
````

## テスト用 JSON（異常系）

```json
{
  "pathParameters": {
    "cityId": "99"
  }
}
```
