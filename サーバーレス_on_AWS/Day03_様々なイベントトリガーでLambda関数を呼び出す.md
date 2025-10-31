Day3 のハンズオンに関する情報を掲載します。
動画にあわせて手を動かしていただくことを推奨していますが、もし「同じ手順を実施したはずなのに動かない..」となったときように、コピー＆ペーストしていただく用途で使っていただければと考えています。

# Day3-3 put_random_weather_function の修正

```py
    # return 文の前に追加する
    print('Update: ' + city_name + ' to ' + weather_name)
```

# Day3-5 「設計図の使用」からコピーができなかった方向けの Get S3 object Python3.12 コード

```py
import json
import urllib.parse
import boto3

print('Loading function')

s3 = boto3.client('s3')


def lambda_handler(event, context):
    #print("Received event: " + json.dumps(event, indent=2))

    # Get the object from the event and show its content type
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = urllib.parse.unquote_plus(event['Records'][0]['s3']['object']['key'], encoding='utf-8')
    try:
        response = s3.get_object(Bucket=bucket, Key=key)
        print("CONTENT TYPE: " + response['ContentType'])
        return response['ContentType']
    except Exception as e:
        print(e)
        print('Error getting object {} from bucket {}. Make sure they exist and your bucket is in the same region as this function.'.format(key, bucket))
        raise e

```

# Day3-6 update_weather_from_csv_function

```py
import json
import urllib.parse
import boto3

print('Loading function')

s3 = boto3.client('s3')
dynamodb_client = boto3.client('dynamodb') ###

def lambda_handler(event, context):
    #print("Received event: " + json.dumps(event, indent=2))

    # Get the object from the event and show its content type
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = urllib.parse.unquote_plus(event['Records'][0]['s3']['object']['key'], encoding='utf-8')
    try:
        response = s3.get_object(Bucket=bucket, Key=key)
        #print("CONTENT TYPE: " + response['ContentType'])

        ###
        weather_info_list = response['Body'].read().decode('utf-8').splitlines()
        for weather_info in weather_info_list:
            weather_info = weather_info.split(',')
            dynamodb_client.put_item(
                TableName='simple-weather-news-table',
                Item={
                    'CityId': {
                        'N': weather_info[0],
                    },
                    'CityName': {
                        'S': weather_info[1],
                    },
                    'WeatherId': {
                        'N': weather_info[2],
                    },
                    'WeatherName': {
                        'S': weather_info[3],
                    },
                    'RainfallProbability': {
                        'N': weather_info[4],
                    },
                },
            )
        ###

        return response['ContentType']
    except Exception as e:
        print(e)
        print('Error getting object {} from bucket {}. Make sure they exist and your bucket is in the same region as this function.'.format(key, bucket))
        raise e
```

## 実行前の IAM ロール権限設定

## 問題

現時点では、Lambda 関数には S3 の読み取り権限しか付与されていません。
そのため、DynamoDB へデータを書き込もうとすると「アクセス拒否エラー」になります。

## 対応手順

1. Lambda 関数画面の「設定 > アクセス権限」タブを開きます。
2. 「実行ロール」欄のロール名をクリックして IAM コンソールを開きます。
3. 「許可ポリシー」の右にある 「許可を追加」 をクリック。
4. 「ポリシーをアタッチ」 を選択します。
5. 検索欄に DynamoDB と入力。
6. 一覧から AmazonDynamoDBFullAccess（バージョン V2）を選択。
7. 下部の「許可を追加」をクリック。
8. これで Lambda 関数が DynamoDB にアクセス可能になります。

## 動作確認

## 手順

1. S3 コンソールを開き、前回作成したバケットを選択します。

2. weather.csv ファイルをドラッグ＆ドロップで再アップロードします。  
   既に同名ファイルがある場合は上書きで OK です。
3. アップロード完了後、Lambda 関数が自動的にトリガーされます。
4. Lambda 関数画面の「モニタリング > CloudWatch ログを表示」をクリック。
5. 最新ログストリームを確認し、エラーが出ていないことを確認します。
6. DynamoDB コンソールを開き、SIMPLE ウェザーニューステーブル の「項目の探索」を開きます。
7. データが CSV と同じ内容（ID・都市名・天気・降水確率）で登録されていれば成功です。

## 注意点と補足

- Lambda 実行直後に DynamoDB の反映が見えない場合、30 秒〜1 分 程度待ちます。

- もし EventBridge（5 分おきの定期実行）が動いている場合、
  タイミングによって値が一部上書きされることがあります。
  → 再度 CSV をアップロードすると整合します。

- S3 トリガーや EventBridge トリガーは、Lambda の起動条件を制御する重要要素です。
  今後は他のトリガー（API Gateway、SNS など）も確認してみると理解が深まります。

# Day3 のまとめ

## 落ち穂拾い: DynamoDB テーブルへの一括書き込みについて

### Table.batch_writer

https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/programming-with-python.html

### 一括取り込みソリューション

https://aws.amazon.com/jp/blogs/news/implementing-bulk-csv-ingestion-to-amazon-dynamodb/
