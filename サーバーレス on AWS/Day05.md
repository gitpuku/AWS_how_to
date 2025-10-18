# Day5-1: S3 静的ウェブサイトホスティングの設定

## 概要

S3 の静的ウェブサイトホスティング機能を使用して、シンプルウェザーニュースアドミンサイトをホスティングします。Vue.js で構築されたフロントエンドの雛形を GitHub から取得し、S3 バケットにデプロイして動作確認を行います。

## 前提条件

- AWS マネジメントコンソールにアクセスできること
- Day04 までのハンズオンが完了していること
- DynamoDB にデータが登録されていること

## 手順

### 1. S3 バケットの作成

1. AWS マネジメントコンソールの検索窓に「**S3**」と入力し、S3 サービスに移動
2. 「**バケットを作成**」をクリック
3. バケット名を入力（例: `aws-serverless-10days-simple-weather-news-admin-[あなたの名前]`）
   - ⚠️ **注意**: バケット名は世界中で一意である必要があります
   - 末尾に名前やユニークな文字列を追加してください
4. 「**このバケットのブロックパブリックアクセス設定**」セクションで:
   - 「**パブリックアクセスをすべてブロック**」の**チェックを外す**
   - 警告が表示されるので、「**現在の設定により、このバケットとバケット内のオブジェクトが公開される可能性があることを承認します**」に**チェックを入れる**
5. その他の設定はデフォルトのまま、「**バケットを作成**」をクリック

### 2. バケットポリシーの設定

1. 作成したバケットのリンクをクリックして、バケットの詳細画面に移動
2. 「**アクセス許可**」タブをクリック
3. 「**バケットポリシー**」セクションで「**編集**」をクリック
4. 以下の JSON をコピーして貼り付け:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::Bucket-Name/*"]
    }
  ]
}
```

5. **`Bucket-Name`** の部分を、作成した S3 バケット名に置き換える
   - 💡 バケット名は画面上部の「バケット ARN」から確認できます。最後に`/*`が必要です。
6. 「**変更の保存**」をクリック
7. 「正常に編集されました」と表示されることを確認

### 3. 静的ウェブサイトホスティングの有効化

1. 「**プロパティ**」タブをクリック
2. 一番下までスクロールし、「**静的ウェブサイトホスティング**」セクションで「**編集**」をクリック
3. 「**静的ウェブサイトホスティング**」で「**有効にする**」を選択
4. 「**インデックスドキュメント**」に `index.html` と入力
5. 「**エラードキュメント**」に `index.html` と入力
6. 「**変更の保存**」をクリック
7. 「静的ウェブサイトホスティングが有効となりました」と表示されることを確認

### 4. AWS CloudShell の起動

1. AWS マネジメントコンソールのロゴを**右クリック**し、「**新しいタブで開く**」を選択
2. 検索窓に「**CloudShell**」と入力し、CloudShell サービスをクリック
3. CloudShell が起動するまで待機
4. （オプション）画面設定の変更:
   - 右上の歯車アイコンをクリック
   - フォントサイズやテーマ(明るい/暗い)を好みに応じて変更

### 5. vim エディタの初期設定

1. CloudShell で以下のコマンドを実行:

```bash
vim ~/.vimrc
```

2. `i` キーを押して挿入モード(INSERT)に切り替え
3. 以下の設定を入力:

```
set tabstop=2
```

4. `ESC` キーを押して挿入モードを終了
5. `:wq` と入力して `Enter` を押し、保存して終了

#### vim の基本操作(参考)

- **挿入モードに入る**: `i` キーを押す
- **挿入モードを終了**: `ESC` キーを押す
- **保存して終了**: `ESC` → `:wq` → `Enter`
- **保存せずに終了**: `ESC` → `:q!` → `Enter`
- **画面のクリア**: `clear` コマンドを実行

### 6. GitHub リポジトリのクローン

1. CloudShell で以下のコマンドを実行し、GitHub リポジトリをクローン:

```bash
git clone https://github.com/ketancho/aws-serverless-10days.git
```

2. クローンが完了したことを確認

### 7. 作業用ディレクトリの作成とファイルのコピー

1. 作業用ディレクトリを作成:

```bash
mkdir work-simple-weather-news-admin
```

2. Day05-01 の雛形ファイルを作業用ディレクトリにコピー:

```bash
cp -r aws-serverless-10days/Day05/simple_weather_admin/Day05-01/* work-simple-weather-news-admin/
```

3. 作業用ディレクトリに移動:

```bash
cd work-simple-weather-news-admin
```

4. ファイルが正しくコピーされたか確認:

```bash
ls
```

以下のファイルが表示されることを確認:

- `index.html`
- `list.html`
- `detail.html`
- `js/` ディレクトリ(`config.js`, `auth.js` を含む)

### 8. ファイルの概要確認(参考情報)

#### index.html

- サインアップ・サインイン画面
- 現時点ではダミー実装(Day06 で Cognito と連携予定)
- メールアドレスとパスワードを入力すると `list.html` に遷移

#### list.html

- 都市一覧と天気情報を表示
- 5 つの都市(札幌、東京、名古屋、大阪、博多)の情報を表示
- 現時点ではダミーデータを表示(Day5-2 で API 連携予定)

#### detail.html

- 都市の詳細情報を表示
- 天気情報の更新機能(Day5-4 で実装予定)

#### js/auth.js

- Cognito 連携用のコード(Day06 で使用)

#### js/config.js

- API Gateway エンドポイントなどの設定ファイル(Day5-2 で編集)

### 9. S3 バケットへのファイルアップロード

1. 以下のコマンドで S3 バケットにファイルを同期:

```bash
aws s3 sync . s3://your-bucket-name/
```

⚠️ **`your-bucket-name`** を、手順 1 で作成した実際のバケット名に置き換えてください

2. アップロードが完了し、以下のようなメッセージが表示されることを確認:

```
upload: ./index.html to s3://your-bucket-name/index.html
upload: ./list.html to s3://your-bucket-name/list.html
upload: ./detail.html to s3://your-bucket-name/detail.html
upload: ./js/auth.js to s3://your-bucket-name/js/auth.js
upload: ./js/config.js to s3://your-bucket-name/js/config.js
```

### 10. S3 バケットでのファイル確認

1. S3 バケットの「**オブジェクト**」タブを開き、ページをリロード
2. 以下のファイルとフォルダがアップロードされていることを確認:
   - `index.html`
   - `list.html`
   - `detail.html`
   - `js/` フォルダ(`auth.js` と `config.js` を含む)

### 11. 静的ウェブサイトの動作確認

1. S3 バケットの「**プロパティ**」タブを開く
2. 一番下までスクロールし、「**静的ウェブサイトホスティング**」セクションを確認
3. 「**バケットウェブサイトエンドポイント**」に表示されている URL をクリック
4. ブラウザで「シンプルウェザーニュースアドミン」のサインイン画面が表示されることを確認

### 12. サイトの動作テスト

1. メールアドレス欄に適当なメールアドレス形式の文字列を入力(例: `test@example.com`)
   - ⚠️ `@` マークを含む必要があります(バリデーションチェックのため)
2. パスワード欄に任意の文字列を入力
3. 「**サインイン**」ボタンをクリック
4. 都市一覧画面(`list.html`)に遷移することを確認
5. 各都市(札幌、東京、名古屋、大阪、博多)の情報が表示されることを確認
6. 任意の都市をクリックし、詳細画面(`detail.html`)に遷移することを確認

### 13. 現時点の動作の確認と理解

現時点では以下の点に注意:

- 📌 **サインイン機能**: 実際の認証は行われず、どのような入力でも次の画面に遷移します(Day06 で Cognito と連携予定)
- 📌 **天気情報**: DynamoDB のデータではなく、ダミーデータが表示されています(Day5-2 で API 連携予定)
- 📌 **更新機能**: まだ実装されていません(Day5-4 で実装予定)

DynamoDB のテーブルを確認すると、実際のデータと画面の表示が異なることがわかります。次のセクション(Day5-2)で GET API と連携し、実際のデータを取得できるようにします。

---

## トラブルシューティング

### 途中でエラーが発生した場合のリセット手順

もし作業途中でエラーが発生し、特定の終了断面からやり直したい場合は、以下のコマンドを使用してください:

```bash
# ホームディレクトリに移動
cd ~

# 作業ディレクトリの内容を削除
rm -r work-simple-weather-news-admin/*

# 指定した終了断面のファイルをコピー
# 例: Day5-2 終了断面に戻したい場合は Day05-02-end を指定
cp -r aws-serverless-10days/Day05/simple_weather_admin/Day05-02-end/* work-simple-weather-news-admin/

# 作業ディレクトリに移動
cd work-simple-weather-news-admin

# S3 バケットに同期
aws s3 sync . s3://your-bucket-name/
```

⚠️ **注意**: 終了断面に戻すと `js/config.js` の設定が初期化されるため、API Gateway エンドポイントなどを再度設定する必要があります。

---

## CloudShell 上での作業コマンド(まとめ)

### vim の初期設定

```bash
vim ~/.vimrc
```

i を押し、挿入モードにした上で、下記のように記入し、 ESC + :wq + Enter で保存

```
set tabstop=2
```

### GitHub からフロントエンドコードの雛形を取得し、作業用ディレクトリに配置、S3 バケットにホスティング

```bash
git clone https://github.com/ketancho/aws-serverless-10days.git
mkdir work-simple-weather-news-admin
cp -r aws-serverless-10days/Day05/simple_weather_admin/Day05-01/* work-simple-weather-news-admin/
cd work-simple-weather-news-admin
aws s3 sync . s3://your-bucket-name/
```

### もし上手く動かなくなった場合、終了断面に戻したい場合に使用するコマンド

```bash
cd ~
rm -r work-simple-weather-news-admin/*
# DayNN-NN-end の NN 部分を具体的な数字に置き換えてください
# 例: Day5-2終了断面に戻したい場合は Day05-02-end
cp -r aws-serverless-10days/Day05/simple_weather_admin/DayNN-NN-end/* work-simple-weather-news-admin/
# ⚠️ js/config.js の設定が消えてしまうので、API Gateway エンドポイントなどを改めて設定してください
```

# Day5-2: API Gateway との連携(GET API)

## 概要

フロントエンドから API Gateway を呼び出して、DynamoDB に格納されている実際の天気予報データを取得・表示できるようにします。CORS(Cross-Origin Resource Sharing)の設定を行い、異なるドメイン間での API 呼び出しを可能にします。

## 前提条件

- Day5-1 までの手順が完了していること
- S3 静的ウェブサイトホスティングが設定されていること
- API Gateway のエンドポイントが作成されていること

## 手順

### 1. 作業ディレクトリへの移動

CloudShell で以下のコマンドを実行し、作業ディレクトリに移動します。

```bash
cd ~/work-simple-weather-news-admin
```

💡 **補足**: ホームディレクトリにいる場合は、上記のコマンドで作業ディレクトリに移動できます。

### 2. API Gateway エンドポイントの設定

#### 2.1. config.js ファイルの編集

1. CloudShell で `config.js` ファイルを開きます:

```bash
vim js/config.js
```

2. 新しいタブで AWS マネジメントコンソールを開き、API Gateway の画面に移動:

   - 検索窓に「**API Gateway**」と入力してサービスを開く
   - 「**simple-weather-news-api**」をクリック
   - 「**シンプルウェザー限定**」をクリック

3. 「**デフォルトのエンドポイント**」の URL をコピー

4. CloudShell に戻り、以下の操作を実行:

   - `i` キーを押して挿入モードに入る
   - `apiBaseUrl` の値として、コピーした URL を貼り付け
   - ⚠️ **注意**: URL の末尾のスラッシュ(`/`)は不要です。削除してください

5. 以下のように編集:

```javascript
const apiBaseUrl =
  "https://your-api-id.execute-api.ap-northeast-1.amazonaws.com";
```

6. `ESC` キーを押してから `:wq` と入力し、`Enter` を押して保存

#### 2.2. list.html ファイルの編集

1. `list.html` ファイルを開きます:

```bash
vim list.html
```

2. ファイルの先頭(1 行目)に移動します:

   - vim で `gg` と入力すると先頭に移動できます

3. `<script>` タグで囲まれている部分を下にスクロールして探します

4. **ダミーデータのコメントアウト**:
   - **Day5-2** のコメント部分を探す
   - 札幌から博多までの固定データ(配列の要素 5 つ)をコメントアウト
   - `i` キーを押して挿入モードに入る
   - ダミーデータの前に `/*` を追加
   - ダミーデータの後ろに `*/` を追加

コメントアウトする箇所:

```javascript
/*
 { cityId: 1, cityName: "札幌", weatherName: "くもり", rainfallProbability: 50 },
 { cityId: 13, cityName: "東京", weatherName: "雨", rainfallProbability: 90 },
 { cityId: 23, cityName: "名古屋", weatherName: "晴れ", rainfallProbability: 10 },
 { cityId: 27, cityName: "大阪", weatherName: "雨", rainfallProbability: 90 },
 { cityId: 40, cityName: "博多", weatherName: "雨", rainfallProbability: 100 }
*/
```

5. **API 呼び出しのコメントアウト解除**:
   - `mounted()` メソッド内の `this.fetchWeatherData()` の呼び出しを探す
   - この行がコメントアウトされている場合は、コメントを解除
   - コメントアウトを解除することで、画面起動時に API を呼び出すようになります

コメントアウトを解除する箇所:

```javascript
mounted() {
  this.fetchWeatherData(); // この行のコメントを解除
},
```

💡 **補足**: `fetchWeatherData()` メソッドは、API Gateway のエンドポイント(`/roll`)を呼び出し、レスポンスを処理する機能がすでに実装されています。

6. `ESC` キーを押してから `:wq` と入力し、`Enter` を押して保存

### 3. S3 バケットへのファイル同期

編集したファイルを S3 バケットにアップロードします。

1. CloudShell で以下のコマンドを実行:

```bash
aws s3 sync . s3://your-bucket-name/
```

⚠️ **`your-bucket-name`** を実際のバケット名に置き換えてください

2. 変更されたファイルのみがアップロードされることを確認:

```
upload: ./js/config.js to s3://your-bucket-name/js/config.js
upload: ./list.html to s3://your-bucket-name/list.html
```

💡 **補足**: 上矢印キー(↑)を押すと、過去に実行したコマンドを呼び出せます。ただし、CloudShell のセッションが切れた場合は履歴が消えるため、再度コマンドを入力してください。

### 4. ブラウザでの動作確認(エラー確認)

1. S3 バケットの「**プロパティ**」タブを開く
2. 一番下までスクロールし、「**静的ウェブサイトホスティング**」セクションの「**バケットウェブサイトエンドポイント**」をクリック
3. ダミーのサインイン:

   - メールアドレス欄: `@` マークを含む任意の文字列を入力
   - パスワード欄: 1 文字以上の任意の文字列を入力
   - 「**サインイン**」ボタンをクリック

4. 天気予報一覧画面が表示されない場合、エラーを確認します

#### 4.1. デベロッパーツールでのエラー確認

1. ブラウザのデベロッパーツールを開く:

   - **Chrome の場合**: `F12` または `Ctrl + Shift + I` (Windows) / `Cmd + Option + I` (Mac)
   - または、メニューから「**表示**」→「**開発/管理**」→「**デベロッパーツール**」
   - ショートカット: `Ctrl + Shift + J` (Windows) / `Cmd + Option + J` (Mac)

2. 「**Console**」タブを確認

3. 以下のようなエラーメッセージが表示されているはずです:

```
ブロックされました: CORS ポリシー
Access-Control-Allow-Origin ヘッダーがない
```

💡 **CORS エラーの意味**:

- **CORS (Cross-Origin Resource Sharing)**: 異なるドメイン間でのリソース共有を制御するセキュリティ機能
- S3 静的ウェブサイトホスティングのドメインと API Gateway のドメインが異なるため、CORS 設定が必要
- この設定を行わないと、悪意のあるサイトが勝手に他のサイトのデータを取得するのを防ぐためにブラウザがリクエストをブロックします

#### 4.2. ブラウザキャッシュのクリア(トラブルシューティング)

もし CORS エラー以外のエラーが表示されている場合、ブラウザのキャッシュが原因の可能性があります。

**スーパーリロード(強制リロード)の方法**:

- **Chrome (Windows)**: `Ctrl + Shift + R` または `Shift + F5`
- **Chrome (Mac)**: `Cmd + Shift + R`
- **Firefox (Windows)**: `Ctrl + F5`
- **Firefox (Mac)**: `Cmd + Shift + R`

キャッシュを使わずにリロードすることで、最新のファイルが読み込まれます。

### 5. API Gateway での CORS 設定

#### 5.1. CORS 設定画面を開く

1. API Gateway の画面に戻る
2. 「**simple-weather-news-api**」が選択されていることを確認
3. 左側のメニューから「**CORS**」をクリック

#### 5.2. CORS 設定の追加

1. 「**設定**」ボタンをクリック

2. 「**アクセスコントロール-許可-オリジン**」欄に以下を入力:

```
*
```

3. ⚠️ **重要**: 「**追加**」ボタンをクリック

   - この操作を忘れると設定が反映されません

4. 「**アクセスコントロール-許可-ヘッダー**」欄にも以下を入力:

```
*
```

5. 「**追加**」ボタンをクリック

6. 「**アクセスコントロール-許可-メソッド**」で以下の 2 つを選択:

   - ✅ **GET**
   - ✅ **PUT**

7. 「**保存**」ボタンをクリック

8. 「無事に更新ができました」というメッセージが表示されることを確認

💡 **CORS 設定の意味**:

- **Access-Control-Allow-Origin**: どのオリジン(ドメイン)からのリクエストを許可するか。`*` は全てのオリジンを許可
- **Access-Control-Allow-Headers**: どのリクエストヘッダーを許可するか。`*` は全てのヘッダーを許可
- **Access-Control-Allow-Methods**: どの HTTP メソッドを許可するか。今回は GET と PUT を使用

⚠️ **セキュリティに関する注意**:
本番環境では `*` ではなく、特定のオリジンのみを許可することを推奨します。

### 6. 動作確認

#### 6.1. フロントエンドでの確認

1. S3 静的ウェブサイトホスティングの URL をブラウザでリロード:

   - 通常のリロード: `F5` または `Ctrl + R` (Windows) / `Cmd + R` (Mac)
   - スーパーリロード: `Ctrl + Shift + R` (Windows) / `Cmd + Shift + R` (Mac)

2. ダミーのサインインを実行

3. 天気予報一覧画面に **全国の天気予報データが表示される** ことを確認:

   - 札幌
   - 東京
   - 名古屋
   - 大阪
   - 博多

4. デベロッパーツールのコンソールでエラーが消えていることを確認

#### 6.2. DynamoDB との値の一致確認

1. AWS マネジメントコンソールで DynamoDB の画面に移動

2. 「**テーブル**」→「**simple-weather-news-table**」をクリック

3. 「**項目を探索**」をクリック

4. 「**項目をスキャンまたはクエリする**」の「**スキャン**」ボタンをクリック

5. DynamoDB のデータと、フロントエンドに表示されているデータが一致していることを確認:
   - 各都市の天気
   - 降水確率

💡 **補足**: EventBridge のスケジュール(5 分ごと)によって、データが更新されている可能性があります。もしデータが一致しない場合は、両方の画面をリロードして最新のデータを確認してください。

### 7. 詳細画面の動作確認

1. 都市一覧画面で、任意の都市(例: 札幌)をクリック

2. 詳細画面(`detail.html`)に遷移することを確認

3. ⚠️ **注意**: 現時点では詳細画面はまだ修正されていないため、古いダミーデータが表示されます
   - 一覧画面と異なる天気が表示される場合があります
   - 次のセクション(Day5-3)で `detail.html` を修正します

---

## トラブルシューティング

### CORS エラーが解消されない場合

- API Gateway の CORS 設定を確認:
  - アクセスコントロール-許可-オリジン: `*` が追加されているか
  - アクセスコントロール-許可-ヘッダー: `*` が追加されているか
  - アクセスコントロール-許可-メソッド: GET と PUT が選択されているか
  - 「**追加**」ボタンを押し忘れていないか
  - 「**保存**」ボタンをクリックしたか

### データが表示されない場合

1. デベロッパーツールのコンソールでエラーを確認
2. `js/config.js` の API エンドポイントが正しいか確認
3. `list.html` で `fetchWeatherData()` の呼び出しがコメントアウトされていないか確認
4. S3 バケットに最新のファイルがアップロードされているか確認(`aws s3 sync` を再実行)
5. ブラウザのスーパーリロードを実行

### vim の編集に失敗した場合

- 保存せずに終了: `ESC` → `:q!` → `Enter`
- GitHub の終了断面に戻す:

```bash
cd ~
rm -r work-simple-weather-news-admin/*
cp -r aws-serverless-10days/Day05/simple_weather_admin/Day05-02-end/* work-simple-weather-news-admin/
cd work-simple-weather-news-admin
```

⚠️ **注意**: 終了断面に戻した場合は、`js/config.js` の API Gateway エンドポイントを再度設定してください。

---

## CloudShell 上での作業コマンド(まとめ)

```bash
cd ~/work-simple-weather-news-admin
vim js/config.js
vim list.html
aws s3 sync . s3://your-bucket-name/
```

💡 **参考**: Day5-2 の最終断面については、`aws-serverless-10days/Day05/simple_weather_admin/Day05-02-end` 以下に格納されています。

# Day5-3

参考：Day5-3 の最終断面については、Day06/simple_weather_admin/Day05-03-end 以下に格納してあります

## CloudShell 上での作業コマンド

```bash
cd ~/work-simple-weather-news-admin
vim detail.html
aws s3 sync . s3://your-bucket-name/
```

# Day5-4

## put_city_weather_function の実装例

```py
import json
import boto3

dynamodb_client = boto3.client('dynamodb')

def lambda_handler(event, context):

    if 'cityId' in event['pathParameters']:
        city_id = int(event['pathParameters']['cityId'])
    else:
        return {
            'isBase64Encoded': False,
            'statusCode': 404,
            'body': json.dumps({'error': 'city_id is required'}, ensure_ascii=False),
            'headers': {
                'content-type': 'application/json'
            }
        }

    city_name = findCityName(city_id)
    if city_name == 'ERROR':
        return {
            'isBase64Encoded': False,
            'statusCode': 404,
            'body': json.dumps({'error': 'Invalid cityId'}, ensure_ascii=False),
            'headers': {
                'content-type': 'application/json'
            }
        }

    body = json.loads(event['body'])
    weather_id = int(body['weatherId'])
    weather_name = findWeatherName(weather_id)

    rainfall_probability = int(body['rainfallProbability'])

    dynamodb_client.put_item(
        TableName='simple-weather-news-table',
        Item={
            'CityId': {
                'N': str(city_id),
            },
            'CityName': {
                'S': city_name,
            },
            'WeatherId': {
                'N': str(weather_id),
            },
            'WeatherName': {
                'S': weather_name,
            },
            'RainfallProbability': {
                'N': str(rainfall_probability),
            },
        },
    )

    return {
        "isBase64Encoded": False,
        "statusCode": 200,
        "body": json.dumps('Update: ' + str(city_id), ensure_ascii=False),
        "headers": {
            "content-type": "application/json"
        }
    }

def findCityName(city_id):
    city_map = {1: '札幌', 13: '東京', 23: '名古屋', 27: '大阪', 40: '博多'}
    return city_map.get(city_id, 'ERROR')

def findWeatherName(weather_id):
    weather_map = {2: '晴れ', 4: 'くもり', 12: '雨'}
    return weather_map.get(weather_id, 'ERROR')
```

## put_city_weather_function の Test Event 例

```json
{
  "pathParameters": {
    "cityId": "1"
  },
  "body": "{\"weatherId\": 12, \"rainfallProbability\": 100}",
  "headers": {
    "Content-Type": "application/json"
  },
  "httpMethod": "PUT",
  "resource": "/{cityId}"
}
```

## PUT API のテストコマンド

```bash
curl -X PUT \
  your-api-gateway-url/1 \
  -H "Content-Type: application/json" \
  -d '{"weatherId": 2, "rainfallProbability": 0}'
```

# Day5-5

参考：Day5-5 の最終断面については、Day06/simple_weather_admin/Day05-05-end 以下に格納してあります

## CloudShell 上での作業コマンド

```bash
cd ~/work-simple-weather-news-admin
vim detail.html
aws s3 sync . s3://your-bucket-name/
```
