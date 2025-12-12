## 1. DynamoDB テーブル作成の基本

● テーブル構成

テーブル名：Notes

パーティションキー（HASH）：UserId（文字列）

ソートキー（RANGE）：NoteId（数値 or 文字列 → 今回は数値）

プロビジョニングモード：Provisioned

ReadCapacityUnits：5

WriteCapacityUnits：5

## 2. Python（Boto3）で DynamoDB テーブルを作成する流れ

① config.ini から設定値を読む

tableName

partitionKey

sortKey

readCapacity

writeCapacity
→ 実務でも「設定値はコードに直接書かず config に分離する」ことは超重要。

② DynamoDB クライアントを作成する
client = boto3.client('dynamodb')

これは AWS SDK と通信するためのオブジェクト。

③ create_table() を呼んでテーブルを作成

構造は以下の形が基本。

response = ddbClient.create_table(
AttributeDefinitions=[
{'AttributeName': 'UserId', 'AttributeType': 'S'},
{'AttributeName': 'NoteId', 'AttributeType': 'N'},
],
KeySchema=[
{'AttributeName': 'UserId', 'KeyType': 'HASH'},
{'AttributeName': 'NoteId', 'KeyType': 'RANGE'},
],
ProvisionedThroughput={
'ReadCapacityUnits': 5,
'WriteCapacityUnits': 5,
},
TableName='Notes'
)

④ テーブルが「ACTIVE」になるまで Waiter で待つ

非同期処理なので、作った直後はまだ利用不可。

waiter = ddbClient.get_waiter('table_exists')
waiter.wait(TableName=tableName)

Waiter は DynamoDB 利用時に便利。
（実務でも "作成直後に describe すると NOT FOUND になる" 問題が避けられる）

⑤ describe_table で状態を表示
client.describe_table(TableName='Notes')

## 3. テーブルへのデータ投入（loadData.py）

ポイント：リソース（resource）版 DynamoDB を使う

DynamoDB では

client（低レベル API）

resource（高レベル API）

の 2 種類がある。

client → create_table など細かい操作用
resource → put_item / query / scan に便利

put_item の基本

今回正しい形は以下：

table.put_item(
Item={
'UserId': note["UserId"],
'NoteId': int(note["NoteId"]),
'Note': note["Note"]
}
)

DynamoDB の項目は（Key + 追加データ）という形式で保存される。

JSON → Python dict → テーブルに投入する流れ

notes.json を読み込む

1 件ずつループ

put_item を呼ぶ

## 4. データのクエリ（queryData.py の重要点）

（※後半は途中のため想定で要点だけ整理）

DynamoDB の基本クエリは パーティションキー必須
table.query(
KeyConditionExpression=Key('UserId').eq('testuser')
)

プロジェクション式（ProjectionExpression）

必要な項目だけを取得できる。

ProjectionExpression="NoteId, Note"

Scan より Query のほうが圧倒的に高速
（※Partition key が使えるときは必ず Query を使う）

## 5. DynamoDB で覚えておくべき重要ポイント

🔹 1. Primary Key 設計が最重要（パフォーマンスに直結）

パーティションキーでデータが分散される

ソートキーで並び順・範囲検索ができる

🔹 2. create_table は非同期 → Waiter 必須
🔹 3. put_item は上書きするので注意

→ UpdateItem を使うと部分更新できる

🔹 4. Query は PartitionKey 必須、SortKey 条件も組み合わせ可能
Key('UserId').eq('student') & Key('NoteId').begins_with("00")

🔹 5. Scan は全件検索で重い → 実務ではほぼ使わない

## ✔ まとめ：この研修で身につく実践力

DynamoDB のテーブル構造設計

Python（boto3）でのテーブル作成方法

Waiter によるリソースの安定化

JSON データの取り込み → DynamoDB への put_item

Query と ProjectionExpression の基本

この内容だけ覚えておけば、
簡単な CRUD（作成・読み取り・更新・削除）は DynamoDB で問題なく扱えるようになります。
