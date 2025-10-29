# Day7-2

## Step Functions ステートマシンの作成

### 1. Step Functions の画面に移動

1. AWS マネジメントコンソールの左上の検索欄に「step functions」と入力
2. Step Functions のサービスページに移動
3. ハンバーガーメニューから「ステートマシン」を選択
4. 「ステートマシンの作成」をクリック

### 2. ステートマシンの基本設定

1. 「空白から作成」を選択（デフォルトのまま）
2. ステートマシン名に「SimpleWeatherNewsStateMachine」と入力
3. 右下の「続行」をクリック

### 3. ワークフローの設定

1. 三角マークを押して、画面を広く使えるように調整
2. 左側の「アクション」からテキスト窓に「DynamoDB Scan」と入力して →「DynamoDB Scan」を選択
3. ドラッグアンドドロップでワークフロー画面に配置

### 4. DynamoDB Scan の設定

1. 配置したスキャンアクションをクリックして選択（青くハイライト）
2. 右側の設定パネルで「引数と出力」タブをクリック

#### 4-1. boto3 ドキュメントによる必須パラメータの確認

DynamoDB Scan で必要なパラメータを確認するため、以下の手順で boto3 ドキュメントを参照：

1. ブラウザで「boto3 dynamodb scan」と検索
2. Python boto3 のドキュメントページを開く
3. **Request Syntax**セクションで必須項目を確認：
   - `TableName` が `[REQUIRED]` として表示されている
   - 最低限このパラメータを設定すれば実行可能

#### 4-2. テーブル名の確認と設定

1. AWS コンソールの左上の AWS ロゴを右クリック → 「新しいタブで開く」
2. 新しいタブで DynamoDB コンソールを開く
3. テーブル一覧で「simple-weather-news-table」が存在することを確認
4. テーブル名「simple-weather-news-table」をコピー
5. Step Functions の画面に戻り、引数の「TableName」フィールドにペースト

### 5. ステートマシンの作成

1. 右上の「作成」ボタンをクリック
2. IAM ロールの警告が表示される（DynamoDB アクセス権限が必要）
3. 「確認」をクリックしてステートマシンを作成

### 6. 実行テスト（エラー確認）

1. 右上の「実行」ボタンをクリック
2. 入力画面でそのまま「実行開始」をクリック
3. エラーが発生し、スキャンアクションが赤く表示される
4. 赤い部分をクリックして、右側でエラー詳細を確認
   - DynamoDB スキャンの権限がないエラーが表示される

### 7. IAM ロールの権限設定

1. ステートマシンの詳細画面に戻る
2. 上部の「IAM ロール」リンクをクリック
3. IAM ロール設定画面で以下を実行：
   - 「許可を追加」→「ポリシーをアタッチ」をクリック
   - 検索欄に「DynamoDB」と入力
   - 「AmazonDynamoDBReadOnlyAccess」を選択
   - 「許可を追加」をクリック

### 8. 再実行テスト

1. IAM 設定反映のため、20-30 秒待機
2. ステートマシンに戻り、再度「実行」をクリック
3. 「実行開始」をクリック
4. 今度は緑色で成功することを確認

### 9. 実行結果の確認

1. 緑色になったスキャンアクションをクリック
2. 右側で以下を確認：
   - **入力**: スタートから受け取った入力データ
   - **定義**: DynamoDB スキャンのタスク定義
   - **出力**: スキャン結果のデータ（DynamoDB テーブルの内容）
3. 下部の「イベント履歴」でタスクの実行ログを確認
   - スケジュール → 開始 → 成功の流れを確認

### 注意点

- エラーが発生した場合は、失敗したアクションが赤く表示される
- 赤い部分をクリックして詳細を確認し、権限やパラメータを修正する
- IAM ロール変更後は、設定反映のため少し時間を置いてから実行する

# Day7-3

## Bedrock InvokeModel の追加とワークフロー設定

### 1. ステートマシンの編集

1. SimpleWeatherNewsStateMachine のページで右上の「編集」ボタンをクリック
2. ステートマシンの編集画面に移動

### 2. Bedrock InvokeModel の追加

1. 左側のアクションの検索バーに「Bedrock invoke」と入力
2. 2 つの InvokeModel が表示される
   - Runtime が付いていない方を選択（通常の InvokeModel）
3. ドラッグアンドドロップで DynamoDB Scan の下に配置

### 3. Bedrock InvokeModel の基本設定

1. 配置した Bedrock InvokeModel をクリックしてフォーカスを当てる
2. 設定を下にスクロール
3. 「ベッドロックのモデル識別子 呼び出す基盤モデルを決定します」の項目を確認
4. プルダウンメニューをクリック
5. 検索窓に「haiku」と入力
6. 先ほど有効化した Claude の haiku を選択してクリック

### 4. 引数と出力の設定

1. 「引数と出力」タブに移動
2. テキスト入力エリアを広く使用するために画面を調整

### 5. Text 部分の編集

1. JSON の Text フィールド（12 行目あたり）の string 部分を編集
2. string を削除して `{%%}` と入力
3. 右側に表示される鉛筆マークをクリック
4. 「JSON の中身を編集」画面が表示される

### 6. ステート入力の設定

1. `$` マークを入力（使用可能な変数の補完が表示される）
2. `states` を選択
3. 完全な形：`$states.input`
   - これは前のステート（Scan）の出力がインプットとして渡されることを意味
4. `$states.input.Items` と入力
   - Scan の出力の Items 部分を取得

### 7. テキスト内容の構成

```
"この天気予報データから、天気予報のアナウンス原稿を作りたいです。「今日の天気は全国的に〜」から始めて、150文字前後の原稿を作ってください。途中で途切れないようにしてください。天気予報データ：" & $states.input.Items
```

- 文字列と変数を連結するには `&` を使用（スペース AND スペース）
- 先頭の固定文字列 + 実際のデータ（Items）を組み合わせ

### 8. 変更の適用

1. 「変更を適用して閉じる」をクリック
2. ポップアップが消えて入力内容が反映される

### 9. 注意点：JSON エスケープ

- Text フィールドでダブルクォーテーションを使用している
- 内部で使用するダブルクォーテーションはバックスラッシュでエスケープされる
- JSON を直接更新する際はエスケープに注意

### 10. ステートマシンの保存

1. 右上の「保存」ボタンをクリック
2. ステートマシンが正常に保存される

### 11. IAM ロールの権限追加

1. ステートマシンの詳細画面から「IAM ロール」をクリック
2. IAM ロール設定画面で以下を実行：
   - 「許可を追加」→「ポリシーをアタッチ」をクリック
   - 検索欄に「bedrock」と入力
   - 「AmazonBedrockLimitedAccess」を選択してチェック
   - 「許可を追加」をクリック

### 12. 実行テスト

1. ステートマシン(フローが描画されてるページ)に戻り「実行」をクリック
2. 入力は不要なので「実行を開始」をクリック
3. 実行フローの確認：
   - Scan → Bedrock InvokeModel の順で実行
   - 両方が緑色で成功することを確認

### 13. 実行結果の確認

1. 緑色になった Bedrock InvokeModel をクリック
2. 出力結果を確認：
   - **入力**: Scan からの出力データ
   - **出力**: 生成された天気予報アナウンス文言

#### 出力例

```
全国的に曇りの傾向です。
博多が40%といった状況で、全体的に太陽が見えにくい状況となりそうです。
雨具の準備もしてくださいね。
```

### 14. データ更新での再テスト

1. 別タブで simple_weather_admin を開く
2. 天気データを「晴れ」に更新（例：名古屋を晴れに変更）
3. 全国的に晴れの状況を作成
4. ステートマシンで「新しい実行」→「実行開始」
5. 更新されたデータに基づく新しいアナウンス文言を確認

#### 更新後の出力例

```
今日の天気は全国的に晴れる見込みです。
日差しが強いので日焼け対策をしっかりしてくださいね。
```

### 重要なポイント

1. **ステート間のデータ流れ**

   - 前のステートの出力が次のステートの入力になる
   - `$states.input` でアクセス可能

2. **JSON 形式での入力設定**

   - `{% %}` 形式でステート入力を参照
   - 文字列連結は `&` 演算子を使用

3. **生成 AI の特性**
   - 同じデータでも返答が変わる可能性がある
   - 現在のデータ状況に応じた適切な内容が生成される

### 参考: Bedrock モデルパラメータ JSON 全文

※ 一部の `"` が `/"` にエスケープされていることにご注意ください

```json
{
  "anthropic_version": "bedrock-2023-05-31",
  "max_tokens": 200,
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "{% \"この天気予報データから、天気予報のアナウンス原稿を作りたいです。「今日の天気は全国的に〜」から始めて、150文字前後の原稿を作ってください。途中で途切れないようにしてください。天気予報データ:\" & $states.input.Items %}"
        }
      ]
    }
  ]
}
```

## トラブルシューティング

### Bedrock.ResourceNotFoundException エラーの対処法

以下のエラーメッセージが表示される場合：

```
Model use case details have not been submitted for this account. Fill out the Anthropic use case details form before using the model.
```

**対処手順：**

1. AWS マネジメントコンソールで「Amazon Bedrock」サービスを開く
2. 左側メニューから「Model catalog」をクリック
3. 「Anthropic」セクションで Claude 3 Haiku を選択
4. 「View model details」をクリック
5. 利用用途フォームに以下の情報を記入：
   - **Use case description**: 「Weather forecast announcement generation for educational purposes」
   - **Industry**: 「Education」または「Technology」
   - **Geographic region**: 「Japan」
   - **Expected monthly usage**: 「Low (under 1,000 requests)」
6. フォームを提出
7. 承認まで 15 分〜数時間待機
8. 承認後、Step Functions で再実行

# Day7-4

## Parallel フローの設定と TranslateText の追加

### 1. ステートマシンの編集

1. SimpleWeatherNewsStateMachine のページで「編集」をクリック
2. ステートマシンの編集画面に移動

### 2. Parallel フローの配置

1. 左側の「フロー」タブをクリック
2. 「Parallel」をドラッグアンドドロップでワークフローに配置
   - DynamoDB Scan → Bedrock InvokeModel の後に配置

### 3. Parallel の左側ブランチ設定（プレースホルダー）

1. Parallel の左側に「Pass」ステートを配置
   - 左側の「フロー」タブから「Pass」を選択
   - 左側のブランチにドラッグアンドドロップ
   - ※ これは何もしないプレースホルダーとして設置（後のハンズオンで置き換え予定）

### 4. Parallel の右側ブランチ設定（TranslateText）

1. 左側の「アクション」タブをクリック
2. 検索欄に「translate」と入力
3. 「TranslateText」を選択
4. 右側のブランチにドラッグアンドドロップ

### 5. TranslateText の設定

1. TranslateText ステートをクリックして選択
2. 右側の「引数と出力」タブをクリック
3. 以下のパラメータを設定：
   - **SourceLanguageCode**: `ja`（日本語から）
   - **TargetLanguageCode**: `en`（英語へ）

### 6. Text パラメータの設定

1. Bedrock の出力結果を確認するため、マネジメントコンソールの新しいタブを開く
2. Step Functions の実行履歴から前回の実行結果を確認
3. Bedrock InvokeModel の出力構造を確認：

   ```
   出力 → Body → content → text
   ```

4. TranslateText の Text フィールドに以下を設定：
   - `Mydata` を削除して`{%%}`と入力
   - 鉛筆マークをクリックして編集
   - `$states.input.Body.content.text` と入力
   - 「変更を適用して閉じる」をクリック

### 7. ステートマシンの保存

1. 右上の「保存」ボタンをクリック
2. ステートマシンの変更が保存される

### 8. IAM ロールの権限追加

1. ステートマシン詳細画面で「IAM ロール」をクリック
2. IAM ロール画面で以下を実行：
   - 「許可を追加」→「ポリシーをアタッチ」をクリック
   - 検索欄に「translate」と入力
   - 「TranslateReadOnly」を選択してチェック
   - 「許可を追加」をクリック

### 9. 実行テスト

1. ステートマシンに戻り「実行」をクリック
2. 入力はそのまま（空）で「実行開始」をクリック
3. 実行結果の確認：
   - InvokeModel → Parallel の順で実行される
   - Parallel 内で Pass と TranslateText が並列実行される
   - TranslateText は軽量なタスクのため高速で完了

### 10. 実行結果の確認

1. 緑色になった TranslateText をクリック
2. 下部の出力を確認：
   - "TODAY IS THE EASE..." から始まる英訳されたテキストを確認
3. 重要な確認ポイント：
   - Bedrock InvokeModel の出力が Pass と TranslateText の両方に入力として渡されている
   - Parallel が受け取った入力を各ブランチに正しく分配している

### 注意点

- Parallel フローでは、各ブランチが同じ入力を受け取る
- TranslateText は軽量なタスクのため並列実行の効果が見えにくいが、左側のブランチを発展させると並列処理の効果が分かりやすくなる
- 空のブランチがあるとエラーになるため、プレースホルダーとして Pass を配置

# Day7-5

## AWS CLI を使用した S3 バケット作成と設定

### はじめに

この操作では、Amazon Polly で天気予報アナウンスの音源 MP3 ファイルを作成するための出力先 S3 バケットを用意します。このバケットは Polly タスクの出力先として使用し、同時に Day8 以降で構築する「シンプルウェザーニュースサイト」の利用者用サイトのホスティング用バケットとしても使用します。

### 1. AWS CloudShell の起動

1. AWS マネジメントコンソールの左上のロゴをクリック
2. 新しいタブを開いて AWS マネジメントコンソールにアクセス
3. CloudShell のサービスページに移動
4. CloudShell が起動するまで待機（初回起動時は時間がかかる場合があります）

### 2. AWS CLI コマンドの準備

以下のコマンドブロックをコピーして準備します。`BUCKET_NAME` の値は世界でユニークなバケット名に変更する必要があります。

```bash
BUCKET_NAME="aws-serverless-10days-simple-weather-news"
# 1. S3バケット作成
aws s3api create-bucket --bucket $BUCKET_NAME --create-bucket-configuration LocationConstraint=ap-northeast-1
# 2. パブリックアクセスブロック設定を無効化
aws s3api put-public-access-block --bucket $BUCKET_NAME --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
# 3. 静的ウェブサイトホスティング有効化
aws s3api put-bucket-website --bucket $BUCKET_NAME --website-configuration '{"IndexDocument":{"Suffix":"list.html"},"ErrorDocument":{"Key":"list.html"}}'
# 4. バケットポリシー設定
aws s3api put-bucket-policy --bucket $BUCKET_NAME --policy "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Sid\":\"PublicReadGetObject\",\"Effect\":\"Allow\",\"Principal\":\"*\",\"Action\":\"s3:GetObject\",\"Resource\":\"arn:aws:s3:::$BUCKET_NAME/*\"}]}"
# 5. voice-news フォルダ作成
aws s3api put-object --bucket $BUCKET_NAME --key voice-news/
```

### 3. バケット名の設定とコマンド実行

1. CloudShell にコマンドブロックを貼り付ける前に、バケット名を編集
2. `BUCKET_NAME="aws-serverless-10days-simple-weather-news"` の部分を世界でユニークな名前に変更
   - 例：`BUCKET_NAME="your-name-simple-weather-news"`
   - 例：`BUCKET_NAME="your-company-weather-app-bucket"`
3. 全てのコマンドを CloudShell に貼り付け
4. **Enter キーを押して最後のコマンドまで実行**

### 4. 各コマンドの詳細説明

#### 4-1. S3 バケット作成コマンド

```bash
aws s3api create-bucket --bucket $BUCKET_NAME --create-bucket-configuration LocationConstraint=ap-northeast-1
```

- **コマンド**: `aws s3api create-bucket`
- **必須パラメータ**: `--bucket` （作成するバケットの名前）
- **オプション**: `--create-bucket-configuration LocationConstraint=ap-northeast-1` （東京リージョンに明示的に作成）

#### 4-2. パブリックアクセスブロック設定の無効化

```bash
aws s3api put-public-access-block --bucket $BUCKET_NAME --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

- **目的**: 利用者ページとして公開するため、パブリックアクセスブロックを無効化
- **設定内容**: 全てのパブリックアクセス制限を false に設定

#### 4-3. 静的ウェブサイトホスティングの設定

```bash
aws s3api put-bucket-website --bucket $BUCKET_NAME --website-configuration '{"IndexDocument":{"Suffix":"list.html"},"ErrorDocument":{"Key":"list.html"}}'
```

- **設定内容**:
  - **IndexDocument**: `list.html` （通常の `index.html` の代わり）
  - **ErrorDocument**: `list.html` （エラー時も同じページを表示）
- **理由**: 利用者ページにはログイン画面がなく、`list.html` と `detail.html` のみ使用

#### 4-4. バケットポリシーの設定

```bash
aws s3api put-bucket-policy --bucket $BUCKET_NAME --policy "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Sid\":\"PublicReadGetObject\",\"Effect\":\"Allow\",\"Principal\":\"*\",\"Action\":\"s3:GetObject\",\"Resource\":\"arn:aws:s3:::$BUCKET_NAME/*\"}]}"
```

- **設定内容**: 全てのユーザーがバケット内のオブジェクトを読み取り可能
- **JSON エスケープ**: ダブルクォーテーションがバックスラッシュでエスケープされている

#### 4-5. voice-news フォルダの作成

```bash
aws s3api put-object --bucket $BUCKET_NAME --key voice-news/
```

- **目的**: Polly で作成する音声ファイルを格納するフォルダを事前作成
- **S3 の仕様**: S3 ではフォルダーもオブジェクトも全て `put-object` コマンドで作成

### 5. AWS CLI ドキュメントの活用方法

#### 5-1. コマンドの調べ方

1. ブラウザで「aws s3api」や「aws s3api create-bucket」で検索
2. AWS CLI コマンドリファレンスのドキュメントページを参照
3. **Request Syntax** セクションで必須パラメータを確認
   - 大括弧 `[]` で囲まれていないパラメータが必須項目
   - 例：`create-bucket` では `--bucket` が必須

#### 5-2. ドキュメントの読み方

- AWS CLI のドキュメントは boto3 SDK のドキュメントと構造が似ている
- サンプルコードを参考にトライアンドエラーで学習することが重要
- パラメータの詳細は各コマンドのドキュメントページで確認

### 6. 作成結果の確認

1. AWS マネジメントコンソールで新しいタブを開く
2. S3 サービスページに移動
3. 作成したバケットが一覧に表示されることを確認
4. バケット内に `voice-news/` フォルダが作成されていることを確認

### 7. バケット設定の確認

#### 7-1. 静的ウェブサイトホスティングの確認

1. 作成したバケットをクリック
2. 「プロパティ」タブを選択
3. 一番下の「静的ウェブサイトホスティング」が**有効**になっていることを確認
4. エンドポイント URL が表示される（現時点ではファイルがないため空白ページ）

#### 7-2. アクセス許可の確認

1. 「アクセス許可」タブを選択
2. **パブリックアクセスブロック**: 全て **オフ** になっていることを確認
3. **バケットポリシー**: JSON ポリシーが正しく設定されていることを確認

### 8. AWS CLI の利点

- **効率性**: マネジメントコンソールでの手動操作と比較して高速
- **再現性**: コマンドとして記録され、同じ設定を繰り返し実行可能
- **自動化**: スクリプト化してインフラストラクチャの自動構築が可能
- **学習効果**: AWS のサービス構造をより深く理解

### 9. 今後の活用

このバケットは以下の用途で使用されます：

1. **音声ファイル格納**: Amazon Polly で生成した MP3 ファイルの保存先
2. **ウェブサイトホスティング**: Day8 以降で構築する利用者向けサイトのホスティング
3. **静的コンテンツ配信**: HTML、CSS、JavaScript ファイルの公開

### 注意点

- **バケット名の一意性**: S3 バケット名は世界で一意である必要があります
- **リージョン設定**: `ap-northeast-1`（東京リージョン）を明示的に指定
- **セキュリティ**: パブリックアクセスを許可しているため、機密情報は格納しない
- **コスト**: 使用しないファイルは定期的に削除してストレージコストを管理

### トラブルシューティング

#### バケット名の重複エラー

```
BucketAlreadyExists: The requested bucket name is not available
```

**対処法**: バケット名をより具体的でユニークな名前に変更

#### 権限エラー

```
AccessDenied: User is not authorized to perform this operation
```

**対処法**: AWS CLI の認証情報が正しく設定されているか確認

#### リージョンエラー

```
IllegalLocationConstraintException: The unspecified location constraint is incompatible
```

**対処法**: `--create-bucket-configuration LocationConstraint=ap-northeast-1` が正しく指定されているか確認

# Day7-6

## Step Functions に Amazon Polly を統合する

### はじめに

前回作成した S3 バケットを使用して、Step Functions ワークフローに Amazon Polly（音声合成サービス）を統合します。まず Amazon Polly の基本的な使い方を体験してから、Step Functions での実装を行います。

### 1. Amazon Polly の動作確認

#### 1-1. Amazon Polly コンソールへのアクセス

1. AWS マネジメントコンソールの左上の検索欄に「polly」と入力
2. Amazon Polly のサービスページに移動
3. 「Polly を試す」ボタンをクリック

#### 1-2. テキスト音声変換の体験

1. **入力テキスト**欄に以下のテキストを入力：
   ```
   こんにちはAWSサーバーレス10DAYSへようこそ
   ```
2. 「音声を聞く」ボタンをクリック
3. 女性の声（みずき）で音声が再生されることを確認

#### 1-3. 音声の調整

1. **数字の読み方調整**:

   - 「10DAYS」が「テンデイズ」と読まれる場合
   - テキストを「10 日間」に変更すると自然な読み上げになる

2. **音声の変更**:
   - 「Voice ID」プルダウンから「Takumi」（男性の声）を選択
   - 再度「音声を聞く」をクリックして違いを確認

### 2. Step Functions の編集準備

1. 新しいタブで AWS マネジメントコンソールを開く
2. Step Functions のサービスページに移動
3. 「SimpleWeatherNewsStateMachine」をクリック
4. 右上の「編集」ボタンをクリック

### 3. StartSpeechSynthesisTask の追加

#### 3-1. アクションの検索と配置

1. 左側の「アクション」タブをクリック
2. 検索欄に「polly」と入力
3. 「StartSpeechSynthesisTask」を選択
4. ドラッグアンドドロップで Parallel の左側ブランチ（Pass の下）に配置

#### 3-2. プレースホルダーの削除

1. Pass ステート（プレースホルダー）を選択
2. キーボードの「Delete」キーを押して削除
3. StartSpeechSynthesisTask が左側ブランチのメインタスクになることを確認

### 4. StartSpeechSynthesisTask の設定

#### 4-1. 基本設定

1. StartSpeechSynthesisTask をクリックして選択
2. 右側の「引数と出力」タブをクリック
3. 各パラメータを以下のように設定

#### 4-2. OutputFormat の設定

1. **OutputFormat**: `"mp3"`
   - 音声ファイルの出力形式を MP3 に指定

#### 4-3. OutputS3BucketName の設定

1. 新しいタブで S3 コンソールを開く
2. Day7-5 で作成したバケット名をコピー
   - 例：`aws-serverless-10days-simple-weather-news`
3. **OutputS3BucketName** にバケット名をペースト

#### 4-4. OutputS3KeyPrefix の設定

1. **OutputS3KeyPrefix** に以下を設定：
   ```
   "voice-news/"
   ```
   - 音声ファイルを `voice-news/` フォルダ内に格納

#### 4-5. VoiceId の設定

1. **VoiceId**: `"Takumi"`
   - 日本語男性音声の「Takumi」を使用

#### 4-6. Text の設定

1. **Text** フィールドで `"MyData"` を削除
2. `{%%}` と入力
3. 鉛筆マークをクリックして編集画面を開く
4. `$states.input.Body.content.text` と入力
5. 「変更を適用して閉じる」をクリック

**設定理由**:

- TranslateText と同じ入力データを使用
- Bedrock InvokeModel の出力テキストを音声に変換

### 5. IAM ロールの権限追加

#### 5-1. 必要な権限の追加

1. 新しいタブでステートマシンの詳細画面を開く
2. 「IAM ロール」リンクをクリック
3. 「許可を追加」→「ポリシーをアタッチ」をクリック

#### 5-2. Polly 権限の追加

1. 検索欄に「polly」と入力
2. 「AmazonPollyFullAccess」にチェックを入れる
3. **検索欄の文字列をクリア**

#### 5-3. S3 権限の追加

1. 検索欄に「s3」と入力
2. 「AmazonS3FullAccess」にチェックを入れる
3. 2 つのポリシーが選択されていることを確認
4. 「許可を追加」をクリック

### 6. ステートマシンの保存と実行

#### 6-1. 保存

1. Step Functions の編集画面に戻る
2. 右上の「保存」ボタンをクリック

#### 6-2. 実行テスト

1. 右上の「実行」ボタンをクリック
2. 入力はそのまま（空）で「実行開始」をクリック
3. 実行プロセスの確認：
   - DynamoDB Scan → Bedrock InvokeModel → Parallel（StartSpeechSynthesisTask と TranslateText）

### 7. 実行結果の確認

#### 7-1. Step Functions での確認

1. StartSpeechSynthesisTask が緑色で成功していることを確認
2. StartSpeechSynthesisTask をクリック
3. 出力を確認：
   - **OutputUri**: S3 に作成されたファイルの場所が表示される

#### 7-2. S3 での確認

1. S3 コンソールで作成したバケットを開く
2. `voice-news/` フォルダを確認
3. MP3 ファイルが作成されていることを確認
4. ファイルをクリックして「オブジェクト URL」をコピー
5. ブラウザの新しいタブで URL を開いて音声を再生

**期待される結果**:
「今日の天気は全国的に概して晴れ間が見られますが、一部地域では曇りや小雨の可能性があります。」のような天気予報アナウンスが Takumi の声で再生される

### 8. 固定ファイル名での音声ファイル作成

#### 8-1. 課題の説明

現在の実装では、Polly が自動生成するファイル名（ランダム）でファイルが作成されます。Day8 以降で利用者サイトを構築する際に、固定のファイル名で音声ファイルにアクセスしたいため、CopyObject を使用してファイル名を統一します。

**目標**: `voice-news/voice-news.mp3` という固定ファイル名でアクセスできるようにする

#### 8-2. CopyObject ステートの追加

1. 左側の「アクション」タブで検索欄に「s3 copy」と入力
2. 「CopyObject」を選択
3. ドラッグアンドドロップで StartSpeechSynthesisTask の下に配置

#### 8-3. CopyObject の設定

1. CopyObject をクリックして選択
2. 「引数と出力」タブで以下を設定：

**Bucket**:

```json
"your-bucket-name"
```

※ StartSpeechSynthesisTask と同じバケット名を設定

**Key**:

```json
"voice-news/voice-news.mp3"
```

※ 固定のファイル名を指定

**CopySource**:

1. `"MyData"` を削除して `{%%}` と入力
2. 鉛筆マークをクリック
3. `$states.input.SynthesisTask.OutputUri` と入力
4. 「変更を適用して閉じる」をクリック

### 9. 再実行とエラーの確認

#### 9-1. 実行

1. ステートマシンを保存
2. 「実行」→「実行開始」をクリック

#### 9-2. エラーの発生と原因

実行すると CopyObject でエラーが発生します：

**エラーメッセージ**: `NoSuchKey: The specified key does not exist`

**エラーの原因**:

1. **非同期処理の特性**: StartSpeechSynthesisTask は非同期タスク
2. **タイミング問題**: Polly にタスクを投げた直後にレスポンスが返される
3. **ファイル作成の遅延**: 実際の MP3 ファイル作成には時間がかかる
4. **コピー失敗**: ファイルがまだ存在しない段階で CopyObject が実行される

#### 9-3. エラー詳細の確認

1. 赤くなった CopyObject をクリック
2. エラー詳細で「NoSuchKey」エラーを確認
3. 一方で、S3 を確認すると実際にはファイルが作成されている

### 10. 非同期処理の理解

#### 10-1. StartSpeechSynthesisTask の動作

```
Step Functions → Polly: "この音声ファイルを作って"
Polly → Step Functions: "OK、作っておくね（OutputUri を返す）"
Step Functions: "すぐに次のタスク（CopyObject）を実行"
実際のファイル作成: まだ進行中...
```

#### 10-2. 解決方法の予告

この問題は次の動画（Day7-7）で解決します。解決方法には以下が含まれます：

1. **待機メカニズム**: ファイル作成完了まで待機
2. **ポーリング**: 定期的にファイル存在をチェック
3. **条件分岐**: ファイルが存在する場合のみコピーを実行

### 参考: 完成時の JSON 設定

#### StartSpeechSynthesisTask の引数 JSON

```json
{
  "OutputFormat": "mp3",
  "OutputS3BucketName": "your-bucket-name",
  "OutputS3KeyPrefix": "voice-news/",
  "Text": "{% $states.input.Body.content.text %}",
  "VoiceId": "Takumi"
}
```

#### CopyObject の引数 JSON

```json
{
  "Bucket": "your-bucket-name",
  "CopySource": "{% $states.input.SynthesisTask.OutputUri %}",
  "Key": "voice-news/voice-news.mp3"
}
```

### 重要なポイント

1. **非同期処理の理解**: AWS サービスの多くは非同期で動作する
2. **タイミング制御**: 非同期タスクの完了を適切に待機する必要がある
3. **エラーハンドリング**: 想定されるエラーを理解し、適切に対処する
4. **ファイル管理**: 固定ファイル名により利用者サイトでの音声再生が容易になる

### トラブルシューティング

#### IAM 権限エラー

**エラー**: `User: ... is not authorized to perform: polly:StartSpeechSynthesisTask`

**対処法**:

1. IAM ロールに `AmazonPollyFullAccess` が正しく追加されているか確認
2. 権限反映のため 1-2 分待機してから再実行

#### S3 権限エラー

**エラー**: `AccessDenied: Access Denied`

**対処法**:

1. IAM ロールに `AmazonS3FullAccess` が正しく追加されているか確認
2. バケット名が正しく設定されているか確認

#### バケット名エラー

**エラー**: `NoSuchBucket: The specified bucket does not exist`

**対処法**: OutputS3BucketName に正しいバケット名が設定されているか確認

# Day7-7

## Step Functions のポーリング機能とフロー制御の実装

### はじめに

前回までで Amazon Polly の音声合成タスクを実装しましたが、非同期処理のため MP3 ファイルの作成完了を待たずに CopyObject を実行してエラーが発生していました。この動画では、Choice state と Wait state を使用してポーリング機能を実装し、タスクの完了を適切に待ってからファイルコピーを行う仕組みを構築します。

### 1. 実行時間の調整による一時的解決策の確認

#### 1-1. Wait state の位置変更

1. Step Functions の編集画面で Wait state を選択
2. Wait state を StartSpeechSynthesisTask の後、CopyObject の前に移動
3. フローの確認：StartSpeechSynthesisTask → Wait → CopyObject

#### 1-2. 実行テスト

1. ステートマシンを保存
2. 「実行」→「実行開始」をクリック
3. 実行プロセスの確認：
   - Scan → Bedrock InvokeModel → Parallel（StartSpeechSynthesisTask → Wait(5 秒) → CopyObject と TranslateText）
4. Wait 中にタスクが非同期で処理され、5 秒後にファイルが完成している状態を確認

#### 1-3. 結果の確認

1. 全てのステートが緑色で成功することを確認
2. S3 バケットを更新日時でソートし、`voice-news.mp3` ファイルが作成されていることを確認
3. ファイルサイズが同じであることでコピーが成功していることを確認

### 2. 固定待機時間方式の課題分析

#### 2-1. 課題の説明

現在の実装では 5 秒の固定待機を使用していますが、以下の問題があります：

1. **テキスト長による処理時間の変動**: 長いテキストほど音声生成に時間がかかる
2. **過剰な待機時間**: 短いテキストでも 5 秒待機する必要がある
3. **不足する待機時間**: 長いテキストの場合、5 秒では足りない可能性がある
4. **非効率性**: 最適な待機時間を予測できない

#### 2-2. 改善すべき理由

固定待機時間は「ケースバイケースだとは思いつつも、完璧とは言えない、やや美しくはない」実装であり、より動的で効率的な解決策が必要です。

### 3. GetSpeechSynthesisTask の追加

#### 3-1. タスクの検索と配置

1. 左側の「アクション」タブで検索欄に「GetSpeech」と入力
2. 「GetSpeechSynthesisTask」を選択
3. ドラッグアンドドロップで Wait の前に配置
4. フロー：StartSpeechSynthesisTask → GetSpeechSynthesisTask → Wait → CopyObject

#### 3-2. GetSpeechSynthesisTask の設定

1. GetSpeechSynthesisTask をクリックして選択
2. 「引数と出力」タブで以下を設定

#### 3-3. TaskId パラメータの設定

1. **引数と出力**を確認：このタスクは TaskId を要求
2. StartSpeechSynthesisTask の出力構造を確認：

   - 実行履歴から前回の StartSpeechSynthesisTask の出力を確認
   - `SynthesisTask` の配下に `TaskId` が存在することを確認

3. **TaskId** フィールドの設定：
   - `"MyData"` を削除して `{%%}` と入力
   - 鉛筆マークをクリック
   - `$states.input.SynthesisTask.TaskId` と入力
   - 「変更を適用して閉じる」をクリック

### 4. 実行テストとタスクステータスの確認

#### 4-1. 保存と実行

1. ステートマシンを保存
2. 「実行」→「実行開始」をクリック

#### 4-2. GetSpeechSynthesisTask の出力確認

1. GetSpeechSynthesisTask が緑色で完了後、クリックして選択
2. 出力を確認：
   - **TaskStatus**: "scheduled" または "inProgress" が表示される
   - まだ "completed" ではないことを確認

#### 4-3. タスクステータスの理解

Amazon Polly のタスクステータスには以下があります：

- **scheduled**: タスクがスケジュールされた状態
- **inProgress**: 処理中
- **completed**: 完了
- **failed**: 失敗

### 5. Choice State の実装

#### 5-1. Choice State の追加

1. 左側の「フロー」タブをクリック
2. 「Choice」を選択
3. ドラッグアンドドロップで GetSpeechSynthesisTask の下に配置

#### 5-2. 分岐フローの設計

1. **左側の分岐**: タスクが完了している場合 → CopyObject へ
2. **右側の分岐**: タスクが未完了の場合 → Wait → GetSpeechSynthesisTask へ（ポーリング）

#### 5-3. フローの再接続

1. CopyObject を Choice の左側に接続
2. Wait を Choice の右側に接続
3. Wait の「次の状態」を GetSpeechSynthesisTask に設定（ループ構造の作成）

### 6. Choice State の条件設定

#### 6-1. ルールの編集

1. Choice をクリックして選択
2. 右側の設定パネルで矢印ボタンをクリック
3. 「条件を追加」をクリック

#### 6-2. 条件の設定

1. **変数の指定**:

   - 変数名フィールドに `$.SynthesisTask.TaskStatus` と入力
   - これは GetSpeechSynthesisTask の出力からタスクステータスを参照

2. **比較条件の設定**:

   - 演算子: `Is equal to`（等しい）を選択
   - データタイプ: `String`（文字列）を選択
   - 値: `completed` と入力

3. **条件の保存**:
   - 「条件を保存する」をクリック

#### 6-3. 条件の意味

この条件は以下を表します：

```
IF (TaskStatus == "completed") THEN
    左側の分岐（CopyObject）を実行
ELSE
    右側の分岐（Wait → 再度 GetSpeechSynthesisTask）を実行
```

### 7. ポーリング機能のテスト

#### 7-1. 事前準備

1. S3 バケットで既存の `voice-news.mp3` ファイルを削除
   - チェックを入れて「削除」→「完全に削除」をクリック

#### 7-2. 実行テスト

1. ステートマシンを保存
2. 「実行」→「実行開始」をクリック
3. 実行フローの観察：
   - StartSpeechSynthesisTask → GetSpeechSynthesisTask → Choice
   - Choice で条件判定 → Wait（5 秒）→ GetSpeechSynthesisTask → Choice
   - 条件が満たされるまでループ
   - 最終的に CopyObject が実行される

#### 7-3. ポーリング動作の確認

実行中に以下の動作が確認できます：

1. **1 回目の GetSpeechSynthesisTask**: TaskStatus が "scheduled" のため Wait へ
2. **Wait**: 5 秒間待機
3. **2 回目の GetSpeechSynthesisTask**: TaskStatus が "completed" のため CopyObject へ
4. **CopyObject**: ファイルのコピーが成功

### 8. 実行ログの詳細分析

#### 8-1. ログの確認方法

1. 実行完了後、下部の「イベント履歴」を確認
2. 各ステートの実行順序と出力を確認

#### 8-2. 典型的な実行ログ

```
1. StartSpeechSynthesisTask: タスクを開始
2. GetSpeechSynthesisTask (1回目): TaskStatus = "scheduled"
3. Choice: 条件不一致、右側分岐へ
4. Wait: 5秒待機
5. GetSpeechSynthesisTask (2回目): TaskStatus = "completed"
6. Choice: 条件一致、左側分岐へ
7. CopyObject: ファイルコピー成功
```

### 9. 最終確認

#### 9-1. S3 ファイルの確認

1. S3 バケットを更新
2. `voice-news/voice-news.mp3` ファイルが作成されていることを確認
3. ファイルにアクセスして音声が正常に再生されることを確認

#### 9-2. 処理効率の確認

この実装により以下が実現されました：

1. **動的な待機**: タスクの完了状況に応じた待機
2. **効率的な処理**: 不要な待機時間の削減
3. **堅牢性**: 処理時間が変動しても確実に動作
4. **拡張性**: 他の非同期タスクにも応用可能

### 10. Step Functions のハンズオン完了

#### 10-1. 実装された機能

1. **DynamoDB Scan**: 天気データの取得
2. **Bedrock InvokeModel**: AI による天気予報アナウンス生成
3. **並列処理**: 音声合成と翻訳の同時実行
4. **Amazon Polly**: 日本語音声ファイルの生成
5. **Amazon Translate**: 英語翻訳
6. **ポーリング機能**: 非同期タスクの完了待機
7. **S3 ファイル管理**: 固定ファイル名での音声ファイル保存

#### 10-2. ワークフローの全体像

```
DynamoDB Scan
↓
Bedrock InvokeModel (天気予報文言生成)
↓
Parallel
├─ StartSpeechSynthesisTask → GetSpeechSynthesisTask → Choice → Wait (ループ) → CopyObject
└─ TranslateText (英語翻訳)
```

### 11. 今後の活用方法

#### 11-1. 手動実行での利用

現在の実装では定期実行の仕組みは含まれていません。今後のハンズオンでは：

1. 画面更新時に手動でステートマシンを実行
2. 最新の天気データに基づく音声ファイルを生成
3. 利用者サイトで更新された音声を聴取

#### 11-2. 実装パターンの応用

このポーリング実装パターンは以下にも応用できます：

- **AWS Batch**: バッチジョブの完了待機
- **Amazon Textract**: 文書解析の完了待機
- **AWS CodeBuild**: ビルドプロセスの完了待機
- **その他の非同期 AWS サービス**: 任意の非同期処理の管理

### 12. Choice State の条件設定 (参考)

実装した条件の JSON 表現：

```json
{
  "Variable": "$.SynthesisTask.TaskStatus",
  "StringEquals": "completed"
}
```

この条件により、TaskStatus が "completed" の場合のみ CopyObject が実行され、そうでなければ Wait → GetSpeechSynthesisTask のループが継続されます。

### 重要なポイント

1. **非同期処理の制御**: AWS の多くのサービスは非同期であり、適切な完了待機が必要
2. **ポーリングパターン**: Choice + Wait + ループによる動的な状態監視
3. **効率性と堅牢性**: 固定待機時間よりも柔軟で効率的な処理
4. **Step Functions の威力**: 複雑なワークフローを視覚的に設計・管理

### トラブルシューティング

#### Choice State の条件が動作しない

**症状**: 常に同じ分岐に進む

**確認項目**:

1. 変数パス（`$.SynthesisTask.TaskStatus`）が正しいか
2. 比較値（`completed`）が正確か
3. データタイプ（String）が適切か

#### ループが無限に続く

**症状**: Wait → GetSpeechSynthesisTask が終わらない

**対処法**:

1. Amazon Polly のタスクが実際に実行されているか AWS コンソールで確認
2. IAM 権限（Polly、S3）が正しく設定されているか確認
3. S3 バケット名、出力パスが正確か確認

#### CopyObject でエラーが発生

**エラー**: `NoSuchKey`

**対処法**:

1. GetSpeechSynthesisTask の出力で TaskStatus が "completed" になっているか確認
2. StartSpeechSynthesisTask の OutputUri が正しく生成されてるか確認
3. S3 バケットとオブジェクトの権限設定を確認
