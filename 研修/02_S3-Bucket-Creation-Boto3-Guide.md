# Amazon S3 バケット作成の流れ（Python / Boto3）

## 📌 学ぶべき重要ポイント

- Boto3 の基本的な使い方（client 生成）
- バケット名の要件（グローバル一意・小文字 etc）
- リージョンごとの作成方法（us-east-1 だけ特別）
- Waiter を利用したリソース作成完了待ち
- head_bucket によるバケット存在チェック

## 1.1 Boto3 クライアントの作成

```python
import boto3
s3 = boto3.client("s3")
```

## 1.2 バケット名の検証（存在チェック）

既存バケットの有無を確認する標準手法
→ head_bucket を呼ぶ
→ 存在しなければ例外が発生する

```python
def verifyBucketName(s3Client, bucket):
    try:
        s3Client.head_bucket(Bucket=bucket)
        # 存在する → 名前として使えない
        return False
    except:
        # 存在しない → OK
        return True
```

## 1.3 バケット作成（リージョン対応）

us-east-1 は LocationConstraint を指定してはいけない特例

他リージョンは LocationConstraint が必須

```python
def createBucket(s3Client, name, current_region):
    if current_region == 'us-east-1':
        return s3Client.create_bucket(Bucket=name)
    else:
        return s3Client.create_bucket(
            Bucket=name,
            CreateBucketConfiguration={
                'LocationConstraint': current_region
            }
        )
```

## 1.4 Waiter を使用して「作成完了」を待つ

AWS リソースは作成直後アクセスできないことがあるため、Waiter を使う。

```python
def verifyBucket(s3Client, bucket):
    waiter = s3Client.get_waiter('bucket_exists')
    waiter.wait(Bucket=bucket)
```

## 2. S3 へのオブジェクトアップロード

### 📌 学ぶべき重要ポイント

- upload_file / put_object の違い
- メタデータ付きアップロードの方法
- ファイルパスとバケットの関係
- 詳細な権限が必要（IAM 設定）

### 2.1 基本のアップロード（メタデータ付き）

```python
def uploadObject(s3Client, bucket):
    s3Client.upload_file(
        Filename='notes.csv',
        Bucket=bucket,
        Key='notes.csv',
        ExtraArgs={
            'Metadata': {
                'uploaded-by': 'training-script'
            }
        }
    )
```

upload_file はファイル →S3 の高レベル API で、
自動リトライなどが付いているため推奨。

## 3. 構成ファイル（config.ini）

学習ポイントとして覚えるべきは：

```
bucket_name = notes-bucket-12345
region = us-west-2
```

config.ini を使ってコードを設定ファイル化する
→ ハードコーディングしない練習

どの言語でも共通の良い設計習慣

## 4. スクリプト構成（推奨パターン）

AWS SDK を使ったスクリプトの理想的構成：

1. import
2. 関数定義
3. 設定ファイル読み込み
4. クライアント生成
5. main() 実行
6. try/except によるエラーハンドリング

これは現場でも使われるベストプラクティス。

## 5. 実務的な学習ポイントまとめ

### 🔑 最重要

- S3 バケットはグローバル一意
- us-east-1 は LocationConstraint 不要
- waiter を使うと安定する
- head_bucket で存在チェックができる
- upload_file は高レベル API

## 6. サンプル：S3 バケット作成 & CSV アップロードの最小コード

学習の "おさらい用コード" として使えます。

```python
import boto3
import configparser

def main():
    config = configparser.ConfigParser()
    config.read('config.ini')

    bucket = config['DEFAULT']['bucket_name']
    region = config['DEFAULT']['region']

    s3 = boto3.client('s3')

    # バケット名チェック
    try:
        s3.head_bucket(Bucket=bucket)
        print("Bucket already exists.")
        return
    except:
        pass

    # バケット作成
    if region == "us-east-1":
        s3.create_bucket(Bucket=bucket)
    else:
        s3.create_bucket(
            Bucket=bucket,
            CreateBucketConfiguration={'LocationConstraint': region}
        )

    # Waiter
    waiter = s3.get_waiter('bucket_exists')
    waiter.wait(Bucket=bucket)

    print("Bucket created:", bucket)

    # CSV アップロード
    s3.upload_file(
        'notes.csv',
        bucket,
        'notes.csv',
        ExtraArgs={'Metadata': {'uploaded-by': 'training'}}
    )

    print("CSV uploaded!")

if __name__ == "__main__":
    main()
```

## 📘 以上の内容を押さえておけば

ラボがなくても AWS S3 × Boto3 の基礎はしっかり復習できます。
特に **バケット作成の仕様差（リージョン）** と
**waiter の重要性** は実務でも非常に役立ちます。
