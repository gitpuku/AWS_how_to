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

3. 「**デフォルトのエンドポイント**」の URL をコピー

4. CloudShell に戻り、以下の操作を実行:

   - `i` キーを押して挿入モードに入る
   - `apiBaseUrl` の値として、コピーした URL を貼り付け
   - ⚠️ **注意**: URL の末尾のスラッシュ(`/`)は不要です。削除してください

5. 以下のように編集:

```javascript
const CONFIG = {
  API_BASE_URL: "https://your-api-id.execute-api.ap-northeast-1.amazonaws.com", // API Gatway のデフォルトエンドポイントを入力
  COGNITO: {
    USER_POOL_ID: "ap-northeast-1_XXXXXXXXX", // User Pool ID を入力
    CLIENT_ID: "XXXX", // Client ID を入力
    REGION: "ap-northeast-1",
  },
};
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

# Day5-3: 詳細画面での GET API 連携

## 概要

天気予報詳細画面(`detail.html`)から GET API(`/roll/{cityId}`)を呼び出して、選択した都市の最新情報を表示できるようにします。Day5-2 では一覧画面のみを修正しましたが、このセクションでは詳細画面も実際のデータを表示するように修正します。

## 前提条件

- Day5-2 までの手順が完了していること
- 一覧画面で API 連携が正常に動作していること
- API Gateway の CORS 設定が完了していること

## 現在の状況

- ✅ 一覧画面(`list.html`): 実際の天気予報データが表示される
- ❌ 詳細画面(`detail.html`): まだダミーデータが表示されている
- ❌ 更新機能: まだ実装されていない(Day5-4 以降で対応)

詳細画面をクリックした際、ダミーデータではなく DynamoDB の最新データを取得して表示するように修正します。

## 手順

### 1. 作業ディレクトリへの移動

CloudShell で以下のコマンドを実行し、作業ディレクトリに移動します。

```bash
cd ~/work-simple-weather-news-admin
```

### 2. detail.html ファイルの編集

#### 2.1. ファイルを開く

1. CloudShell で `detail.html` ファイルを開きます:

```bash
vim detail.html
```

2. ファイルの下方にスクロールして、`<script>` タグで囲まれている JavaScript コードの部分を探します

#### 2.2. ダミーデータのコメントアウト

1. 以下のような箇所を探します:

```javascript
cities : [
 cityId: 1, cityName: "札幌", weatherName: "くもり", rainfallProbability: 50, weatherId: 4 },
 cityId: 13, cityName: "東京", weatherName: "雨", rainfallProbability: 90, weatherId: 12 },
 cityId: 23, cityName: "名古屋", weatherName: "晴れ", rainfallProbability: 10, weatherId: 2 },
 cityId: 27, cityName: "大阪", weatherName: "雨", rainfallProbability: 90, weatherId: 12 },
 cityId: 40, cityName: "博多", weatherName: "雨", rainfallProbability: 100, weatherId: 12 }
];
```

2. `i` キーを押して挿入モードに入ります

3. ダミーデータをコメントアウトします:
   - 配列の中身(都市情報 5 つ)を `/* */` で囲む
   - 配列自体は空の配列 `[]` として残す

修正後:

```javascript
cities: [
  /*
 cityId: 1, cityName: "札幌", weatherName: "くもり", rainfallProbability: 50, weatherId: 4 },
 cityId: 13, cityName: "東京", weatherName: "雨", rainfallProbability: 90, weatherId: 12 },
 cityId: 23, cityName: "名古屋", weatherName: "晴れ", rainfallProbability: 10, weatherId: 2 },
 cityId: 27, cityName: "大阪", weatherName: "雨", rainfallProbability: 90, weatherId: 12 },
 cityId: 40, cityName: "博多", weatherName: "雨", rainfallProbability: 100, weatherId: 12 }
 */
];
```

#### 2.3. ダミー実装のコメントアウト

1. 下方にスクロールして、`mounted()` メソッドの中を探します

2. 以下のようなダミーデータを使って表示する実装がある箇所を探します:

```javascript
this.currentCity =
  this.cities.find((city) => city.cityId === this.cityId) || {};
```

この部分をコメントアウトします:

#### 2.4. API 呼び出しのコメントアウト解除

1. `mounted()` メソッド内の下部に、以下のような `fetchCityData()` の呼び出しがコメントアウトされている箇所を探します:

```javascript
// await this.fetchCityData();
```

2. この行のコメントを解除します:

💡 **fetchCityData() メソッドの動作**:

- URL パラメータから `cityId` を取得(例: `detail.html?cityId=13`)
- API Gateway のエンドポイント(`/roll/{cityId}`)を呼び出し
- レスポンスデータを画面に表示

例:

- 東京をクリック → `/roll/13` を呼び出し
- 博多をクリック → `/roll/40` を呼び出し

#### 2.5. ファイルを保存

1. `ESC` キーを押して挿入モードを終了
2. `:wq` と入力して `Enter` を押し、保存して終了

### 3. S3 バケットへのファイル同期

編集したファイルを S3 バケットにアップロードします。

```bash
aws s3 sync . s3://your-bucket-name/
```

⚠️ **`your-bucket-name`** を実際のバケット名に置き換えてください

以下のような出力が表示されることを確認:

```
upload: ./detail.html to s3://your-bucket-name/detail.html
```

### 4. 動作確認

#### 4.1. ブラウザでの確認

1. S3 静的ウェブサイトホスティングの URL をブラウザで開く

2. スーパーリロードを実行:

   - **Chrome (Windows)**: `Ctrl + Shift + R`
   - **Chrome (Mac)**: `Cmd + Shift + R`

3. ダミーのサインインを実行

4. 天気予報一覧画面で任意の都市(例: 名古屋)をクリック

5. 詳細画面に遷移し、画面遷移時に裏で API が呼び出される

6. 都市の最新情報が表示されることを確認:
   - 都市名
   - 天気(晴れ、くもり、雨)
   - 降水確率

💡 **補足**: 画面遷移時に一瞬ローディングが発生することがあります。これは API を呼び出してデータを取得しているためです。

#### 4.2. 複数の都市での確認

1. 一覧画面に戻る(ブラウザの戻るボタンを使用)

2. 別の都市(例: 博多)をクリック

3. 詳細画面で博多の最新情報が表示されることを確認

4. DynamoDB のデータと一致していることを確認:
   - AWS マネジメントコンソールで DynamoDB の画面を開く
   - `simple-weather-news-table` のアイテムをスキャン
   - 該当都市のデータと画面表示が一致しているか確認

### 5. 現時点での実装状況の確認

#### 実装済み

- ✅ 一覧画面での GET API 呼び出し(`/roll`)
- ✅ 詳細画面での GET API 呼び出し(`/roll/{cityId}`)
- ✅ 画面遷移時のデータ取得

#### 未実装(次のセクション以降で対応)

- ❌ 天気予報更新機能(Day5-4 で実装)
  - 現時点では「更新」ボタンをクリックしても、見た目だけで実際には更新されません
  - 更新用の PUT API(`/roll/{cityId}`)をまだ作成していないため

💡 **次のステップ**:

- Day5-4: PUT API の作成(天気予報データを更新する Lambda 関数と API Gateway ルートの設定)
- Day5-5: 詳細画面から PUT API を呼び出す実装

---

## トラブルシューティング

### 詳細画面でデータが表示されない場合

1. デベロッパーツールのコンソールを開いてエラーを確認:

   - `F12` または `Ctrl + Shift + I` (Windows) / `Cmd + Option + I` (Mac)
   - 「**Console**」タブを確認

2. CORS エラーが表示されている場合:

   - API Gateway の CORS 設定を再確認
   - 「アクセスコントロール-許可-オリジン」と「アクセスコントロール-許可-ヘッダー」に `*` が追加されているか
   - GET メソッドが許可されているか

3. API エンドポイントが間違っている場合:

   - `js/config.js` の `apiBaseUrl` が正しいか確認
   - API Gateway の「デフォルトのエンドポイント」と一致しているか

4. `fetchCityData()` が呼び出されていない場合:

   - `detail.html` の `mounted()` メソッドで `this.fetchCityData();` のコメントが外れているか確認

5. ブラウザキャッシュが原因の場合:
   - スーパーリロードを実行(`Ctrl + Shift + R` など)
   - それでも解決しない場合は、ブラウザの履歴とキャッシュを完全にクリア

### vim の編集に失敗した場合

- 保存せずに終了: `ESC` → `:q!` → `Enter`
- GitHub の終了断面に戻す:

```bash
cd ~
rm -r work-simple-weather-news-admin/*
cp -r aws-serverless-10days/Day05/simple_weather_admin/Day05-03-end/* work-simple-weather-news-admin/
cd work-simple-weather-news-admin
```

⚠️ **注意**: 終了断面に戻した場合は、`js/config.js` の API Gateway エンドポイントを再度設定してください。

---

## CloudShell 上での作業コマンド(まとめ)

```bash
cd ~/work-simple-weather-news-admin
vim detail.html
aws s3 sync . s3://your-bucket-name/
```

💡 **参考**: Day5-3 の最終断面については、`aws-serverless-10days/Day05/simple_weather_admin/Day05-03-end` 以下に格納されています。

# Day5-4: PUT API の作成と連携

## 概要

天気予報データを更新するための PUT API を作成します。Lambda 関数を実装し、API Gateway に PUT メソッドのルートを追加して、フロントエンドから天気予報データを更新できるようにします。このセクションでは PUT API の作成と API Gateway 経由でのテストまでを行います。

## 前提条件

- Day5-3 までの手順が完了していること
- GET API が正常に動作していること
- DynamoDB テーブルが作成されていること

## 要件

### API 仕様

- **エンドポイント**: `PUT /roll/{cityId}`
- **パスパラメータ**: `cityId` (更新対象の都市 ID)
- **リクエストボディ**:
  ```json
  {
    "weatherId": 12,
    "rainfallProbability": 100
  }
  ```
- **機能**: 指定された都市の天気予報データを DynamoDB に更新

### weatherId と天気の対応

- `2`: 晴れ
- `4`: くもり
- `12`: 雨

## 手順

### 1. Lambda 関数の作成

#### 1.1. Lambda サービスを開く

1. AWS マネジメントコンソールの検索窓に「**Lambda**」と入力し、Lambda サービスに移動
2. 新しいタブで開いておくと作業しやすくなります(ロゴを右クリック → 「**新しいタブで開く**」)

#### 1.2. 関数を作成

1. 「**関数の作成**」をクリック
2. 以下の内容で設定:
   - **関数名**: `put_city_weather_function`
   - **ランタイム**: `Python 3.13`
3. 「**関数の作成**」をクリック
4. 関数が作成されるまで 10 秒程度待機

### 2. Lambda 関数への権限付与

Lambda 関数が DynamoDB にデータを書き込めるように、適切な権限を付与します。

#### 2.1. IAM ロールに権限を追加

1. Lambda 関数の画面で「**設定**」タブをクリック
2. 左側メニューから「**アクセス権限**」をクリック
3. 「**ロール名**」のリンク(青字の部分)をクリック
   - 新しいタブで IAM コンソールが開きます
4. 「**許可を追加**」→「**ポリシーをアタッチ**」をクリック
5. 検索窓に「**dynamodb**」と入力
6. 「**AmazonDynamoDBFullAccessV2**」を選択
   - ⚠️ V2 のポリシーを選択してください
7. 「**許可を追加**」をクリック

💡 **権限について**:

- 本番環境では最小権限の原則に従い、特定のテーブルのみへの書き込み権限を付与することを推奨します
- ハンズオンでは簡易的に FullAccess を使用しています

### 3. Lambda 関数のコード実装

#### 3.1. コードの準備

補足資料に用意されている以下のコードをコピーします:

```python
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

#### 3.2. コードの貼り付け

1. Lambda 関数の「**コード**」タブに戻る
2. エディタ内の既存のコードを**すべて選択して削除**
   - `Ctrl + A` (Windows) / `Cmd + A` (Mac) → `Delete`
3. 上記のコードを貼り付け

#### 3.3. コードの解説

画面を拡大表示にして、コードの動作を確認しましょう。

##### パスパラメータから cityId を取得

```python
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
```

- `event['pathParameters']` から `cityId` を取得
- パスパラメータに `cityId` がない場合はエラーを返す

##### 都市名の取得

```python
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
```

- `findCityName()` メソッドで `cityId` から都市名を取得
- Day02 で実装したものと同じロジック
- 無効な `cityId` の場合はエラーを返す

##### リクエストボディからデータを取得

```python
body = json.loads(event['body'])
weather_id = int(body['weatherId'])
weather_name = findWeatherName(weather_id)
rainfall_probability = int(body['rainfallProbability'])
```

- `event['body']` から `weatherId` と `rainfallProbability` を取得
- `findWeatherName()` メソッドで `weatherId` から天気名を取得
- `int()` で囲んでいますが、今回は数値として扱うので必須ではありません

##### findWeatherName メソッド

```python
def findWeatherName(weather_id):
    weather_map = {2: '晴れ', 4: 'くもり', 12: '雨'}
    return weather_map.get(weather_id, 'ERROR')
```

- `findCityName()` とほぼ同じ実装
- `weatherId` に対応する天気名を返す
  - `2` → 晴れ
  - `4` → くもり
  - `12` → 雨
  - それ以外 → ERROR

##### DynamoDB へのデータ書き込み

```python
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
```

- `simple-weather-news-table` テーブルにアイテムを書き込み
- 今まで Day02 や Day03 で実装してきたものと同じ形式

#### 3.4. コードのデプロイ

1. 画面上部の「**Deploy**」ボタンをクリック
2. 「変更が正常にデプロイされました」と表示されることを確認

### 4. Lambda 関数のテスト

#### 4.1. テストイベントの作成

1. 「**Test**」ボタンをクリック
2. 「**Create new event**」を選択(初回のみ)
3. 以下の内容で設定:
   - **イベント名**: `test`(デフォルトのまま)
   - **イベント JSON**: 以下の内容に置き換え

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

💡 **テストイベントの内容**:

- **pathParameters**: `cityId` に `1`(札幌)を指定
- **body**: `weatherId` に `12`(雨)、`rainfallProbability` に `100`(降水確率 100%)を指定
- 実行すると、札幌のデータが「雨・降水確率 100%」に更新される

#### 4.2. テストの実行

1. 「**Save**」をクリックしてテストイベントを保存
2. 「**Test**」または「**Invoke**」ボタンをクリック
3. 実行結果を確認:
   - **Execution result**: succeeded(緑色)
   - **Response**: `"Update: 1"` が返ってくる

#### 4.3. DynamoDB での確認

1. DynamoDB の画面を開く
2. 「**テーブル**」→「**simple-weather-news-table**」をクリック
3. 「**項目を探索**」→「**スキャン**」をクリック
4. 札幌(CityId: 1)のデータが以下のように更新されていることを確認:
   - **WeatherName**: 雨
   - **RainfallProbability**: 100

💡 **補足**: 札幌は晴れだったため、雨に変わったことがわかりやすく確認できます

### 5. API Gateway への統合

#### 5.1. API Gateway のルート画面に戻る

1. API Gateway の画面を開く
2. 「**simple-weather-news-api**」を選択
3. 現在以下の 2 つのルートがあることを確認:
   - `GET /roll`
   - `GET /roll/{cityId}`

⚠️ **注意**: CORS 設定画面になっている場合は、左側メニューから「**ルート**」をクリックしてルート一覧に戻ってください

#### 5.2. PUT メソッドのルートを作成

1. 「**作成**」ボタンをクリック
2. 以下の内容で設定:
   - **メソッド**: `PUT`
   - **パス**: `/{cityId}`
3. 「**作成**」をクリック

💡 **ルートの意味**:

- `PUT /roll/{cityId}`: 指定した都市の天気予報データを更新
- 例: `PUT /roll/1` → 札幌のデータを更新

#### 5.3. 作成されたルートの確認

1. 「無事にルートが作成されました」というメッセージが表示される
2. ルート一覧に `PUT /{cityId}` が追加されていることを確認
3. `/{cityId}` の下に `PUT` が表示されている

### 6. Lambda 統合の設定

#### 6.1. 統合をアタッチ

1. `PUT /{cityId}` ルートをクリック
2. 「**統合をアタッチする**」をクリック
3. 「**統合を作成してアタッチ**」をクリック

#### 6.2. 統合の設定

1. 以下の内容で設定:
   - **統合タイプ**: `Lambda 関数`
   - **リージョン**: `ap-northeast-1`(東京リージョン)
   - **Lambda 関数**: `put` と入力して検索
2. 検索結果に 2 つの関数が表示される:
   - `put_city_weather_function` (今回作成したもの)
   - `put_city_weather_function_random` (Day03 で作成したランダム更新用)
3. **`put_city_weather_function`** を選択
   - ⚠️ **注意**: ランダム更新用の関数ではなく、今回作成した関数を選択してください
4. 「**作成**」をクリック

#### 6.3. 統合の確認

1. 「無事に統合ができました」というメッセージが表示される
2. ルートに Lambda 関数が統合されたことを確認

### 7. CloudShell から API Gateway のテスト

#### 7.1. テストコマンドの準備

ブラウザから PUT リクエストを送信することはできないため、CloudShell から `curl` コマンドを使用してテストします。

補足資料に用意されている以下のコマンドをコピーします:

```bash
curl -X PUT \
  your-api-gateway-url/1 \
  -H "Content-Type: application/json" \
  -d '{"weatherId": 2, "rainfallProbability": 0}'
```

#### 7.2. CloudShell を開く

1. AWS マネジメントコンソールで CloudShell を開く
2. もし CloudShell のセッションが切れている場合はリロードして再接続

#### 7.3. コマンドの編集

1. コピーしたコマンドを CloudShell に貼り付け

   - 複数行のテキストを貼り付ける場合、「**複数行のテキストを安全に貼り付けます**」というダイアログが表示される
   - ダイアログが表示されない場合は、過去に「今後表示しない」を選択した可能性があります。その場合はテキストエディタで編集してください

2. 「**編集**」をクリックして、コマンドを修正:

   - `your-api-gateway-url` の部分を、実際の API Gateway のエンドポイント URL に置き換え

3. API Gateway のエンドポイント URL を取得:

   - API Gateway の画面に戻る
   - 「**デフォルトのエンドポイント**」の URL をコピー
   - 例: `https://xxxxxxxxxx.execute-api.ap-northeast-1.amazonaws.com`

4. CloudShell のダイアログに戻り、URL を貼り付け:

```bash
curl -X PUT \
  https://xxxxxxxxxx.execute-api.ap-northeast-1.amazonaws.com/1 \
  -H "Content-Type: application/json" \
  -d '{"weatherId": 2, "rainfallProbability": 0}'
```

💡 **コマンドの内容**:

- `PUT /1`: 札幌(cityId: 1)のデータを更新
- `weatherId: 2`: 晴れに変更
- `rainfallProbability: 0`: 降水確率 0%に変更

#### 7.4. コマンドの実行

1. 「**貼り付け**」ボタンをクリック
2. `Enter` キーを押してコマンドを実行
3. レスポンスを確認:
   - 改行が正しく入っていなくても問題ありません
   - `"Update: 1"` というレスポンスが返ってくることを確認

#### 7.5. DynamoDB での確認

1. DynamoDB の画面を開く
2. `simple-weather-news-table` テーブルの「**項目を探索**」画面をリロード
3. 札幌(CityId: 1)のデータが以下のように更新されていることを確認:
   - **WeatherName**: 晴れ
   - **RainfallProbability**: 0

💡 **補足**: 先ほど Lambda 関数のテストで「雨・100%」にしたデータが、「晴れ・0%」に更新されました

### 8. API 動作確認のまとめ

#### 8.1. 成功したこと

- ✅ Lambda 関数の作成とコード実装
- ✅ DynamoDB への書き込み権限の付与
- ✅ Lambda 関数単体でのテスト
- ✅ API Gateway への PUT メソッドの追加
- ✅ Lambda 統合の設定
- ✅ CloudShell から API Gateway 経由での PUT リクエスト送信
- ✅ DynamoDB のデータ更新

#### 8.2. 次のステップ

PUT API が正常に動作することを確認できました。次のセクション(Day5-5)では、フロントエンドの詳細画面から PUT API を呼び出して、画面から天気予報データを更新できるようにします。

---

## トラブルシューティング

### Lambda 関数のテストが失敗する場合

1. **権限エラー**: DynamoDB へのアクセス権限が付与されているか確認

   - 「**設定**」→「**アクセス権限**」→ ロール名をクリック
   - `AmazonDynamoDBFullAccessV2` がアタッチされているか確認

2. **コードエラー**: コードが正しく貼り付けられているか確認

   - インデントが崩れていないか
   - 全角スペースが混入していないか

3. **テーブル名エラー**: DynamoDB のテーブル名が正しいか確認
   - コード内の `TableName='simple-weather-news-table'` が正しいか

### API Gateway のテストが失敗する場合

1. **統合エラー**: 正しい Lambda 関数を選択しているか確認

   - `put_city_weather_function` を選択しているか
   - ランダム更新用の関数を選択していないか

2. **エンドポイント URL エラー**: URL が正しいか確認

   - API Gateway の「デフォルトのエンドポイント」と一致しているか
   - 末尾にスラッシュ(`/`)が余分についていないか

3. **CloudShell のコマンドエラー**: JSON の形式が正しいか確認
   - シングルクォート(`'`)とダブルクォート(`"`)の使い分け
   - JSON 内ではダブルクォートを使用

### DynamoDB のデータが更新されない場合

1. Lambda 関数のテストを再実行して、関数自体が正常に動作するか確認
2. CloudShell のレスポンスでエラーメッセージが返っていないか確認
3. DynamoDB の画面をリロードして、最新のデータを表示

---

## 補足資料

### put_city_weather_function の完全なコード

```python
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

### put_city_weather_function の Test Event

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

### PUT API のテストコマンド

```bash
curl -X PUT \
  your-api-gateway-url/1 \
  -H "Content-Type: application/json" \
  -d '{"weatherId": 2, "rainfallProbability": 0}'
```

# Day5-5

参考：Day5-5 の最終断面については、Day06/simple_weather_admin/Day05-05-end 以下に格納してあります

## 概要

詳細画面の更新ボタンから PUT API を呼び出して、天気予報データを更新できるようにします。Day5-4 で作成した PUT API をフロントエンドと連携させ、シンプルウェザーニュースアドミンサイトの基本機能を完成させます。

## 前提条件

- Day5-4 までの手順が完了していること
- PUT API が正常に動作していること
- API Gateway の CORS 設定が完了していること

## 現在の状況

- ✅ GET API との連携: 一覧画面・詳細画面でデータ取得が可能
- ✅ PUT API の作成: API Gateway 経由でデータ更新が可能
- ❌ 更新ボタンの実装: 現在はダミー実装(アラートを表示するのみ)

詳細画面で天気や降水確率を変更して「更新」ボタンをクリックしても、見た目だけで実際にはデータが更新されません。このセクションでは、更新ボタンをクリックした時に PUT API を呼び出してデータを更新できるように実装します。

## 手順

### 1. 作業ディレクトリへの移動

CloudShell で以下のコマンドを実行し、作業ディレクトリに移動します。

```bash
cd ~/work-simple-weather-news-admin
```

### 2. detail.html ファイルの編集

#### 2.1. ファイルを開く

1. CloudShell で `detail.html` ファイルを開きます:

```bash
vim detail.html
```

2. ファイルの下方にスクロールして、`updateWeather()` メソッドを探します
   - Day5-3 で修正した `fetchCityData()` の下にあります

#### 2.2. ダミー実装のコメントアウト

1. `updateWeather()` メソッド内の以下の 2 行を探します:

```javascript
alert("天気予報データを更新しました!(ダミー)");
this.$router.go(-1);
```

2. `i` キーを押して挿入モードに入ります

3. この 2 行をコメントアウトします:

```javascript
// alert("天気予報データを更新しました!(ダミー)");
// this.$router.go(-1);
```

💡 **補足**: この実装は、アラートを表示して前のページに戻るだけのダミー実装です

#### 2.3. API 呼び出しのコメントアウト解除

1. そのまま下方にスクロールして、以下のコメントを探します:

```
// ↓ 以降Day5-5でコメントアウトを外す
```

2. この直後に、コメントアウトされた `try-catch` ブロックがあります

3. **コメントアウトの先頭を削除**:

   - `try {` の前にある `/*` を削除

4. さらに下方にスクロールして、**コメントアウトの末尾を削除**:
   - `alert("更新中にエラーが発生しました。");` の下にある `*/` を削除

修正後のコードは以下のようになります:

```javascript
try {
  const response = await fetch(`${apiBaseUrl}/${this.cityId}`, {
    method: "PUT",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      weatherId: this.currentCity.weatherId,
      rainfallProbability: this.currentCity.rainfallProbability,
    }),
  });

  if (!response.ok) {
    throw new Error("API呼び出しに失敗しました。");
  }

  alert("天気予報データを更新しました!");
  this.$router.go(-1);
} catch (error) {
  console.error("更新エラー:", error);
  alert("更新中にエラーが発生しました。");
}
```

💡 **コードの解説**:

- **method: "PUT"**: HTTP メソッドを PUT に指定
- **`${apiBaseUrl}/${this.cityId}`**: API エンドポイント(`PUT /roll/{cityId}`)を呼び出し
- **body**: 選択された天気 ID(`weatherId`)と降水確率(`rainfallProbability`)を JSON 形式で送信
- **成功時**: アラートを表示して前のページ(一覧画面)に戻る
- **エラー時**: エラーメッセージを表示

#### 2.4. ファイルを保存

1. `ESC` キーを押して挿入モードを終了
2. `:wq` と入力して `Enter` を押し、保存して終了

### 3. S3 バケットへのファイル同期

編集したファイルを S3 バケットにアップロードします。

```bash
aws s3 sync . s3://your-bucket-name/
```

⚠️ **`your-bucket-name`** を実際のバケット名に置き換えてください

以下のような出力が表示されることを確認:

```
upload: ./detail.html to s3://your-bucket-name/detail.html
```

### 4. 動作確認

#### 4.1. ブラウザでの確認

1. S3 静的ウェブサイトホスティングの URL をブラウザで開く

2. 通常のリロードを実行:

   - **Windows**: `F5` または `Ctrl + R`
   - **Mac**: `Cmd + R`

3. ダミーのサインインを実行

4. 天気予報一覧画面で任意の都市(例: 札幌)をクリック

#### 4.2. 天気予報の更新テスト(札幌)

1. 札幌の詳細画面で、天気を「**くもり**」に変更

2. 降水確率を「**50**」に変更

3. 「**更新**」ボタンをクリック

4. ⚠️ **もしダミーのアラートが表示される場合**:

   - ブラウザのキャッシュが原因です
   - **スーパーリロード**を実行:
     - **Chrome (Windows)**: `Ctrl + Shift + R`
     - **Chrome (Mac)**: `Cmd + Shift + R`
   - 再度、くもり・50%に変更して「更新」ボタンをクリック

5. 「**天気予報データを更新しました!**」というアラートが表示される

6. 「OK」をクリックすると、一覧画面に戻る

7. 一覧画面で札幌のデータが更新されていることを確認:
   - 天気: くもり
   - 降水確率: 50%

#### 4.3. 別の都市での更新テスト(東京)

1. 一覧画面で「**東京**」をクリック

2. 東京の詳細画面で、天気を「**晴れ**」に変更

3. 降水確率を「**0**」に変更

4. 「**更新**」ボタンをクリック

5. 「天気予報データを更新しました!」というアラートが表示される

6. 一覧画面に戻り、東京のデータが更新されていることを確認:
   - 天気: 晴れ
   - 降水確率: 0%

#### 4.4. DynamoDB での確認

1. AWS マネジメントコンソールで DynamoDB の画面を開く

2. `simple-weather-news-table` テーブルの「**項目を探索**」画面をリロード

3. 更新したデータが DynamoDB に反映されていることを確認:

   - 札幌(CityId: 1): くもり・50%
   - 東京(CityId: 13): 晴れ・0%

4. フロントエンドに表示されているデータと DynamoDB のデータが一致していることを確認

### 5. 実装完了の確認

#### Day5 で実装した機能のまとめ

- ✅ **S3 静的ウェブサイトホスティング**: フロントエンドのホスティング
- ✅ **GET API 連携(一覧)**: 全国の天気予報データを取得・表示
- ✅ **GET API 連携(詳細)**: 特定都市の天気予報データを取得・表示
- ✅ **PUT API 作成**: 天気予報データを更新する Lambda 関数と API Gateway ルート
- ✅ **PUT API 連携(詳細)**: 詳細画面から天気予報データを更新

#### 現在の課題

⚠️ **セキュリティの問題**:

- URL さえわかれば、誰でもサイトにアクセスできる
- 認証なしで天気予報データを更新できる
- 本番環境では重大なセキュリティリスク

💡 **次のステップ**:
Day06 では、Amazon Cognito を使用して認証・認可の仕組みを実装します。ユーザーを作成しないとサイトにアクセスしたり、データを更新できないようにします。

---

## トラブルシューティング

### 更新ボタンをクリックしてもデータが更新されない場合

1. **ブラウザキャッシュが原因**:

   - スーパーリロードを実行(`Ctrl + Shift + R` など)
   - それでも解決しない場合は、ブラウザの履歴とキャッシュを完全にクリア

2. **デベロッパーツールでエラーを確認**:

   - `F12` または `Ctrl + Shift + I` (Windows) / `Cmd + Option + I` (Mac)
   - 「**Console**」タブでエラーメッセージを確認

3. **CORS エラーが表示される場合**:

   - API Gateway の CORS 設定を確認
   - アクセスコントロール-許可-メソッドに **PUT** が追加されているか確認
   - Day5-2 で設定した CORS 設定を再確認

4. **API エンドポイントが間違っている場合**:

   - `js/config.js` の `apiBaseUrl` が正しいか確認
   - API Gateway の「デフォルトのエンドポイント」と一致しているか

5. **コメントアウトの解除漏れ**:
   - `detail.html` の `updateWeather()` メソッドで `try-catch` ブロックのコメントが外れているか確認
   - `/*` と `*/` が完全に削除されているか確認

### ダミーのアラートが表示される場合

- ブラウザのキャッシュが原因です
- スーパーリロードを実行してください:
  - **Chrome (Windows)**: `Ctrl + Shift + R`
  - **Chrome (Mac)**: `Cmd + Shift + R`
  - **Firefox (Windows)**: `Ctrl + F5`
  - **Firefox (Mac)**: `Cmd + Shift + R`

### vim の編集に失敗した場合

- 保存せずに終了: `ESC` → `:q!` → `Enter`
- GitHub の終了断面に戻す:

```bash
cd ~
rm -r work-simple-weather-news-admin/*
cp -r aws-serverless-10days/Day05/simple_weather_admin/Day05-05-end/* work-simple-weather-news-admin/
cd work-simple-weather-news-admin
```

⚠️ **注意**: 終了断面に戻した場合は、`js/config.js` の API Gateway エンドポイントを再度設定してください。

---

## CloudShell 上での作業コマンド(まとめ)

```bash
cd ~/work-simple-weather-news-admin
vim detail.html
aws s3 sync . s3://your-bucket-name/
```

💡 **参考**: Day5-5 の最終断面については、`aws-serverless-10days/Day05/simple_weather_admin/Day05-05-end` 以下に格納されています。

---

## Day5 のまとめ

### 実装した内容

Day5 では、S3 静的ウェブサイトホスティングとフロントエンドの実装を行いました:

1. **Day5-1**: S3 静的ウェブサイトホスティングの設定
2. **Day5-2**: GET API との連携(一覧画面)と CORS 設定
3. **Day5-3**: GET API との連携(詳細画面)
4. **Day5-4**: PUT API の作成と Lambda 統合
5. **Day5-5**: PUT API との連携(更新機能)

### アーキテクチャ

```
[S3 静的ウェブサイトホスティング]
         ↓ (HTTP リクエスト)
[API Gateway] (CORS 設定済み)
         ↓
[Lambda 関数]
  ├── get_all_city_weather_function (GET /roll)
  ├── get_city_weather_function (GET /roll/{cityId})
  └── put_city_weather_function (PUT /roll/{cityId})
         ↓
[DynamoDB] (simple-weather-news-table)
```

### 次のステップ(Day06)

Day06 では、Amazon Cognito を使用して認証・認可の仕組みを実装します:

- ユーザープールの作成
- サインアップ・サインイン機能の実装
- 認証トークンを使った API 呼び出し
- 認証なしではサイトにアクセスできないようにする

これにより、セキュアな管理者サイトを構築します。

5-5 までの最終的な detail.html

```html
<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>天気予報更新 - Simple Weather Admin</title>
    <script src="https://sdk.amazonaws.com/js/aws-cognito-sdk.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/amazon-cognito-identity-js@6.3.12/dist/amazon-cognito-identity.min.js"></script>
    <script src="js/config.js"></script>
    <script src="js/auth.js"></script>
    <script src="https://unpkg.com/vue@3/dist/vue.global.js"></script>
    <script src="https://cdn.tailwindcss.com"></script>
  </head>

  <body class="bg-gray-100 min-h-screen">
    <div id="app">
      <header class="bg-blue-600 text-white p-4 shadow-md">
        <div class="container mx-auto flex justify-between items-center">
          <h1 class="text-2xl font-bold">Simple Weather Admin</h1>
          <button
            @click="logout"
            class="bg-blue-500 hover:bg-blue-700 px-4 py-2 rounded"
          >
            ログアウト
          </button>
        </div>
      </header>

      <main class="container mx-auto p-6">
        <div class="max-w-md mx-auto bg-white rounded-lg shadow-md p-8">
          <div class="flex items-center justify-between mb-6">
            <button
              @click="goBack"
              class="text-blue-600 hover:text-blue-800 text-sm flex items-center"
            >
              ← 一覧に戻る
            </button>
            <h2 class="text-2xl font-bold text-gray-800">天気予報更新</h2>
            <div class="w-20"></div>
          </div>
          <div class="mb-6 text-center">
            <div v-if="currentCity.cityName" class="text-6xl mb-2">
              {{ getWeatherIcon(currentCity.weatherName) }}
            </div>
            <h3
              v-if="currentCity.cityName"
              class="text-3xl font-bold text-gray-800"
            >
              {{ currentCity.cityName }}
            </h3>
          </div>

          <div class="mb-6 p-4 bg-gray-50 rounded-lg">
            <h4 class="text-lg font-semibold text-gray-700 mb-2">現在の天気</h4>
            <p class="text-gray-600">天気: {{ currentCity.weatherName }}</p>
            <p class="text-gray-600">
              降水確率: {{ currentCity.rainfallProbability }}%
            </p>
          </div>

          <form @submit.prevent="updateWeather">
            <div class="mb-4">
              <label
                for="weather"
                class="block text-sm font-medium text-gray-700 mb-2"
              >
                天気
              </label>
              <select
                id="weather"
                v-model="selectedWeatherName"
                class="w-full p-3 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
              >
                <option value="晴れ">☀️ 晴れ</option>
                <option value="くもり">☁️ くもり</option>
                <option value="雨">☂️ 雨</option>
              </select>
            </div>

            <div class="mb-6">
              <label
                for="rainfall"
                class="block text-sm font-medium text-gray-700 mb-2"
              >
                降水確率
              </label>
              <select
                id="rainfall"
                v-model="selectedRainfallProbability"
                class="w-full p-3 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
              >
                <option
                  v-for="percent in rainfallOptions"
                  :key="percent"
                  :value="percent"
                >
                  {{ percent }}%
                </option>
              </select>
            </div>

            <button
              type="submit"
              class="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 px-4 rounded-md transition duration-200"
            >
              更新
            </button>
          </form>
        </div>
      </main>
    </div>

    <script>
      const { createApp } = Vue;

      createApp({
        data() {
          return {
            cityId: null,
            currentCity: {},
            selectedWeatherName: "",
            selectedRainfallProbability: 0,
            cities: [
              // Day5-3 のタイミングでコメントアウトする
              /*
              { cityId: 1, cityName: "札幌", weatherName: "くもり", rainfallProbability: 50, weatherId: 4 },
              { cityId: 13, cityName: "東京", weatherName: "雨", rainfallProbability: 90, weatherId: 12 },
              { cityId: 23, cityName: "名古屋", weatherName: "晴れ", rainfallProbability: 10, weatherId: 2 },
              { cityId: 27, cityName: "大阪", weatherName: "雨", rainfallProbability: 90, weatherId: 12 },
              { cityId: 40, cityName: "博多", weatherName: "雨", rainfallProbability: 100, weatherId: 12 } 
	  */
            ],
            rainfallOptions: [0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 100],
          };
        },
        mounted() {
          /* Day6-6 のタイミングでコメントアウトを外す（ここから） */
          /*
        if (!authService.isAuthenticated()) {
          window.location.href = 'index.html';
          return;
        }
        */
          /* Day6-6 のタイミングでコメントアウトを外す（ここまで） */

          this.initializePage();
        },
        methods: {
          async initializePage() {
            // URLからcityIdを取得
            const urlParams = new URLSearchParams(window.location.search);
            this.cityId = parseInt(urlParams.get("cityId"));

            // 該当都市のデータを取得 Day5-3 のタイミングでコメントアウトする
            //this.currentCity = this.cities.find(city => city.cityId === this.cityId) || {};
            // API呼び出しで都市データを取得 Day5-3 のタイミングでコメントアウトを外す
            await this.fetchCityData();

            // フォームの初期値を設定
            this.selectedWeatherName = this.currentCity.weatherName || "晴れ";
            this.selectedRainfallProbability =
              this.currentCity.rainfallProbability || 0;
          },
          async fetchCityData() {
            try {
              /* Day5 までの実装, Day6-6 でコメントアウト（ここから） */
              const response = await fetch(
                `${CONFIG.API_BASE_URL}/${this.cityId}`
              );
              /* Day5 までの実装, Day6-6 でコメントアウト（ここまで） */

              /* Day6-6 でコメントアウトを外す（ここから） */
              /*
            const response = await fetch(`${CONFIG.API_BASE_URL}/${this.cityId}`, {
              headers: {
                'Authorization': `Bearer ${authService.getAccessToken()}`
              }
            });
            if (response.status === 401) {
              // 認証エラー時はログアウトしてログイン画面に戻る
              authService.signOut();
              window.location.href = 'index.html';
              return;
            }
            */
              /* Day6-6 でコメントアウトを外す（ここまで） */

              const data = await response.json();
              this.currentCity = data;
            } catch (error) {
              console.error("都市データ取得エラー:", error);
            }
          },
          async updateWeather() {
            // weatherNameからweatherIdを決定
            const weatherId = this.getWeatherIdFromName(
              this.selectedWeatherName
            );

            // ダミー実装：実際はAPI呼び出し ここから Day5-5 でコメントアウトする
            alert(
              `${this.currentCity.cityName}の天気を${this.selectedWeatherName}（降水確率${this.selectedRainfallProbability}%）に更新しました！`
            );
            this.goBack();
            // ここまで

            /* Day5-5 でコメントアウトを外す(ここから) */

            try {
              const response = await fetch(
                `${CONFIG.API_BASE_URL}/${this.cityId}`,
                {
                  method: "PUT",
                  headers: {
                    "Content-Type": "application/json",
                    // Day6-7 のタイミングでコメントアウトを外す（ここから）
                    //'Authorization': `Bearer ${authService.getAccessToken()}`,
                    // Day6-7 のタイミングでコメントアウトを外す（ここまで）
                  },
                  body: JSON.stringify({
                    weatherId: weatherId,
                    rainfallProbability: this.selectedRainfallProbability,
                  }),
                }
              );

              // Day6-7 のタイミングでコメントアウトを外す（ここから）
              //if (response.status === 401) {
              //  authService.signOut();
              //  window.location.href = 'index.html';
              //  return;
              //}
              // Day6-7 のタイミングでコメントアウトを外す（ここまで）

              if (response.ok) {
                alert(`${this.currentCity.cityName}の天気を更新しました！`);
                this.goBack();
              } else {
                console.error("更新に失敗しました:", response.status);
                alert("更新に失敗しました。");
              }
            } catch (error) {
              console.error("API呼び出しエラー:", error);
              alert("更新中にエラーが発生しました。");
            }

            /* Day5-5 でコメントアウトを外す(ここまで) */
          },
          goBack() {
            window.location.href = "list.html";
          },
          logout() {
            /* Day6-7 からの実装（ここから） */
            /*
          authService.signOut();
          window.location.href = 'index.html';
          */
            /* Day6-7 からの実装（ここまで） */
          },
          getWeatherIcon(weatherName) {
            const icons = {
              晴れ: "☀️",
              くもり: "☁️",
              雨: "☂️",
            };
            return icons[weatherName] || "❓";
          },
          getWeatherIdFromName(weatherName) {
            const weatherMap = {
              晴れ: 2,
              くもり: 4,
              雨: 12,
            };
            return weatherMap[weatherName] || 2;
          },
        },
      }).mount("#app");
    </script>
  </body>
</html>
```
