# AWS CI/CD 作業手順書

## 1. GitHub リポジトリの作成

### 1.1 概要

GitHub にリポジトリ(アプリケーションのコードを保存する場所)を作成します。ここにコードをコミットすることで、イベントトリガーとして CI/CD パイプラインが起動します。

### 1.2 前提条件

- GitHub アカウントを持っていること
- GitHub アカウントにログイン済みであること

### 1.3 リポジトリ作成手順

#### 1.3.1 リポジトリ作成画面へアクセス

1. GitHub アカウントにログインした状態で、以下の URL にアクセスします
   ```
   https://github.com/new
   ```
   ※ログインしていない場合は、ログイン画面が表示されますのでログインしてください

#### 1.3.2 リポジトリの設定

1. **Owner(オーナー)**: 自分のアカウント名を選択します
2. **Repository name(リポジトリ名)**: `ecs-learning-course` と入力します
3. **Description(説明)**: 任意項目のため、省略可能です
4. **Public/Private**: `Private` を選択します
   - Private: 権限のある人(作成者本人)のみ閲覧可能
   - Public: 誰でも閲覧可能
     ※後から Public に変更することも可能です
5. 以下のオプションは**すべてチェックを外します**(後から追加可能):
   - □ Add a README file
   - □ Add .gitignore
   - □ Choose a license

#### 1.3.3 リポジトリの作成

1. `Create repository` ボタンをクリックします
2. リポジトリが作成されたことを確認します

---

## 2. ローカル環境へのクローン

### 2.1 リポジトリ URL のコピー

1. 作成したリポジトリのページで、リポジトリ URL をコピーします
   - ページ上部に表示されている HTTPS URL をコピー

### 2.2 ターミナルでのクローン操作

#### 2.2.1 作業ディレクトリへ移動

```bash
cd ~/Desktop
```

※お好みのディレクトリに移動してください

#### 2.2.2 リポジトリをクローン

```bash
git clone <コピーしたURL>
```

例:

```bash
git clone https://github.com/your-account/ecs-learning-course.git
```

#### 2.2.3 クローン完了の確認

1. ディレクトリが作成されたことを確認

   ```bash
   ls
   ```

2. クローンしたディレクトリ内に移動

   ```bash
   cd ecs-learning-course
   ```

3. 現時点では空のディレクトリであることを確認
   ```bash
   ls
   ```

---

## 3. 必要な AWS リソースの整理

### 3.1 概要

CI/CD パイプラインを構築する前に、必要な AWS リソースを整理し、新規作成が必要なものと既存のものを確認します。

### 3.2 リソース一覧

#### 3.2.1 ECS 関連リソース

| リソース           | 名前                 | 状態         | 備考                       |
| ------------------ | -------------------- | ------------ | -------------------------- |
| **ECS クラスター** | `my-app-cluster`     | 既存         | 過去のセクションで作成済み |
| **ECS サービス**   | `my-app-api-service` | **【新規】** | 今回のセクションで作成     |
| **タスク定義**     | `my-app-api`         | **【新規】** | 今回のセクションで作成     |
| **ECR リポジトリ** | `my-app-api`         | **【新規】** | 今回のセクションで作成     |

#### 3.2.2 ネットワーク・セキュリティ関連リソース

| リソース                     | 名前                             | 状態         | 備考                       |
| ---------------------------- | -------------------------------- | ------------ | -------------------------- |
| **VPC**                      | `my-workspace-vpc`               | 既存         | 過去のセクションで作成済み |
| **サブネット(パブリック)**   | `my-workspace-subnet-public1-a`  | 既存         | 過去のセクションで作成済み |
| **サブネット(プライベート)** | `my-workspace-subnet-private1-a` | 既存         | 過去のセクションで作成済み |
| **セキュリティグループ**     | `my-app-api-sg`                  | **【新規】** | 今回のセクションで作成     |

#### 3.2.3 IAM 関連リソース

| リソース             | 名前                   | 状態 | 備考                       |
| -------------------- | ---------------------- | ---- | -------------------------- |
| **タスク実行ロール** | `ecsTaskExecutionRole` | 既存 | 過去のセクションで作成済み |

### 3.3 前提条件の確認

> **重要**: 以下のリソースが既に存在していることを前提として、このコースは進めていきます。もし削除されている場合は、「AWS_ECS 作成手順書.md」を参照して再作成してください。

**既存リソースの確認項目:**

- [ ] ECS クラスター `my-app-cluster` が存在する
- [ ] VPC `my-workspace-vpc` が存在する
- [ ] パブリックサブネット `my-workspace-subnet-public1-a` が存在する
- [ ] プライベートサブネット `my-workspace-subnet-private1-a` が存在する
- [ ] タスク実行ロール `ecsTaskExecutionRole` が存在する

### 3.4 作業の流れ

これから、以下の順序で**【新規】**リソースを作成していきます:

1. タスク定義 `my-app-api` の作成(一時的なダミー設定)
2. ECR リポジトリ `my-app-api` の作成
3. ECS サービス `my-app-api-service` の作成

> **注意**: 本来 ECS サービスを作成する際にはタスク定義が必要ですが、タスク定義の作成時には ECR のイメージ URI が必要になります。しかし、ECR リポジトリはまだ作成していないため、イメージ URI が存在しません。
>
> そのため、以下の手順で進めます:
>
> 1. タスク定義を**ダミーのイメージ URI**で作成
> 2. ECR リポジトリを作成
> 3. ECS サービスを作成
> 4. 後ほど CI/CD パイプラインで正しいイメージ URI に更新

---

## 4. タスク定義の作成

### 4.1 概要

ECS サービスを作成する前に、タスク定義を作成します。タスク定義には、コンテナの設定(イメージ URI、CPU、メモリなど)を定義します。

> **重要**: この段階では ECR リポジトリがまだ存在しないため、一時的に AWS が公開している NGINX のイメージを使用します。後ほど CI/CD パイプラインで正しいイメージに更新されます。

### 4.2 手順

#### 4.2.1 ECS コンソールへアクセス

1. AWS マネジメントコンソールで「ECS」を検索し、「Elastic Container Service」を選択します

#### 4.2.2 タスク定義作成画面へ移動

1. 左側のメニューから「タスク定義」をクリックします
2. 「新しいタスク定義の作成」ボタンをクリックします

#### 4.2.3 タスク定義の基本設定

**タスク定義ファミリー名:**

```
my-app-api
```

**インフラストラクチャの要件:**

- **起動タイプ**: `AWS Fargate` にチェック
- **オペレーティングシステム/アーキテクチャ**:

  - OS: `Linux`
  - アーキテクチャ: `X86_64`

  > **補足**: `X86_64`は Intel チップ、`ARM64`は ARM チップです。要件に応じて選択してください。このコースでは`X86_64`を使用します。

**タスクサイズ:**

- **CPU**: `0.5 vCPU`
  - 最小の`0.25 vCPU`だと Spring Boot を動かすには不十分なため、`0.5 vCPU`を選択します
- **メモリ**: `1 GB`
  - 最小の`1 GB`を選択します

> **注意**: これらのリソース設定は後ほど CI/CD パイプラインで更新されるため、ここでの設定値は一時的なものです。

**タスクロール:**

- 空欄のまま(設定不要)

**タスク実行ロール:**

- デフォルトのまま(`ecsTaskExecutionRole`が自動選択されます)

#### 4.2.4 コンテナの定義

画面を下にスクロールし、「コンテナ」セクションで以下を設定します。

**コンテナ-1:**

**基本設定:**

- **名前**: `dummy`
  - 一時的なダミーコンテナなので`dummy`と命名します
  - 後ほど CI/CD で正しい名前に更新されます
- **イメージ URI**:
  ```
  public.ecr.aws/nginx/nginx:stable-perl
  ```
  - AWS が公開している NGINX イメージを一時的に使用します
  - ECR リポジトリ作成後、CI/CD で正しい Spring Boot のイメージ URI に更新されます

**ポートマッピング:**

- デフォルトのまま(後ほど CI/CD で更新されます)

**環境変数:**

- 設定不要

**ログ収集:**

- **ログ収集の使用** にチェックを入れます
  - トラブルシューティング時にログが必要になるため、有効化します

**その他の設定:**

- デフォルトのまま

#### 4.2.5 タスク定義の作成

1. 画面下部の「作成」ボタンをクリックします
2. タスク定義 `my-app-api` のリビジョン `1` が作成されます

---

## 5. ECR リポジトリの作成

### 5.1 概要

Docker イメージを保存するためのプライベートなコンテナレジストリ(ECR リポジトリ)を作成します。

### 5.2 手順

#### 5.2.1 ECR リポジトリ作成画面へ移動

1. ECS コンソールの左側メニューから「リポジトリ」をクリックします
2. 「リポジトリを作成」ボタンをクリックします

#### 5.2.2 リポジトリの設定

**可視性設定:**

- 「プライベート」を選択します
  - 公開する必要がないため、プライベートリポジトリとします

**リポジトリ名:**

```
my-app-api
```

**その他の設定:**

- デフォルトのまま(変更不要)

#### 5.2.3 リポジトリの作成

1. 「リポジトリを作成」ボタンをクリックします
2. リポジトリ `my-app-api` が作成されます
3. 現時点ではイメージは空の状態です

---

## 6. ECS サービスの作成

### 6.1 概要

タスク定義と ECR リポジトリの準備が完了したので、ECS サービスを作成します。ECS サービスは、タスク定義に基づいてコンテナを起動し、管理します。

### 6.2 手順

#### 6.2.1 クラスター選択

1. ECS コンソールで「クラスター」をクリックします
2. クラスター一覧から `my-app-cluster` をクリックします

#### 6.2.2 サービス作成画面へ移動

1. 「サービス」タブで「作成」ボタンをクリックします

#### 6.2.3 環境設定

**コンピューティング設定:**

- **コンピューティングオプション**: `起動タイプ`
- **起動タイプ**: `FARGATE`
- **プラットフォームバージョン**: `LATEST`

**アプリケーションタイプ:**

- `サービス` を選択

**タスク定義:**

- **ファミリー**: `my-app-api`
- **リビジョン**: `1 (latest)`

**サービス名:**

```
my-app-api-service
```

**サービスタイプ:**

- `レプリカ` を選択

**必要なタスク:**

```
1
```

- 1 つのタスク(コンテナ)のみ起動します
- 複数指定すると、同じタスク定義を使用した複数のコンテナが起動します

#### 6.2.4 デプロイ設定

**デプロイオプション:**

- デフォルトの `ローリングアップデート` のまま
  - Blue/Green デプロイは設定が複雑で費用もかかるため、今回は使用しません

**サービスコネクト:**

- デフォルトのまま(設定不要)

**サービス検出:**

- デフォルトのまま(設定不要)

#### 6.2.5 ネットワーキング設定

**VPC:**

- `my-workspace-vpc` を選択します

**サブネット:**

- **パブリックサブネット 2 つ**にチェックを入れます
- プライベートサブネットのチェックは外します
  - パブリックサブネットを使用することで、インターネットからのアクセスを可能にします

**セキュリティグループ:**

1. 「新しいセキュリティグループの作成」を選択します
2. 以下の情報を入力:

**セキュリティグループ名:**

```
my-app-api-sg
```

**セキュリティグループの説明:**

```
For my-app-api-service
```

**インバウンドルール:**

- **タイプ**: `HTTP`
- **プロトコル**: `TCP`
- **ポート範囲**: `80`
- **ソース**: `Anywhere (0.0.0.0/0)`

  > **セキュリティに関する注意**:
  >
  > - より厳格にする場合は、カスタムで自分の IP アドレスのみを許可することを推奨します
  > - 今回は設定を簡略化するため、Anywhere(どこからでもアクセス可能)に設定します

**パブリック IP:**

- `オン` になっていることを確認します
  - インターネットからアクセスするために必要です

#### 6.2.6 その他の設定

**ロードバランシング:**

- `なし` を選択

**サービスのオートスケーリング:**

- デフォルトのまま(設定不要)

**タグ:**

- デフォルトのまま(設定不要)

#### 6.2.7 サービスの作成

1. 「作成」ボタンをクリックします
2. デプロイが開始されます

#### 6.2.8 デプロイ完了の確認

**デプロイ状態の確認:**

1. サービス作成画面で「正常にデプロイされました」と表示されるまで待ちます(数分かかります)
2. ECS コンソールの「クラスター」→「my-app-cluster」→「サービス」タブで、`my-app-api-service` が表示されることを確認します

---

## 7. 動作確認(ダミーコンテナの確認)

### 7.1 概要

作成した ECS サービスが正常に動作しているか確認します。この段階ではまだダミーの NGINX コンテナが動作している状態です。

### 7.2 手順

#### 7.2.1 タスクの確認

1. ECS コンソールで「クラスター」→「my-app-cluster」→「サービス」→「my-app-api-service」をクリックします
2. 「タスク」タブをクリックします
3. タスク一覧に 1 つのタスクが表示され、ステータスが「実行中(RUNNING)」になっていることを確認します

#### 7.2.2 コンテナの確認

1. 表示されているタスクの ID をクリックします
2. 「コンテナ」セクションで、コンテナ名 `dummy` のステータスが「実行中(RUNNING)」になっていることを確認します

#### 7.2.3 パブリック IP の確認とアクセス

1. タスク詳細画面の「ネットワーキング」セクションで、「パブリック IP」が割り当てられていることを確認します
2. パブリック IP をコピーします
3. ブラウザで以下の URL にアクセスします:

   ```
   http://<パブリックIP>
   ```

   例:

   ```
   http://54.123.45.67
   ```

#### 7.2.4 確認結果

ブラウザに「**Welcome to nginx!**」と表示されれば、ECS サービスが正常に動作しています。

**なぜ NGINX が表示されるのか:**

- タスク定義 `my-app-api` のリビジョン 1 で、NGINX のイメージ URI(`public.ecr.aws/nginx/nginx:stable-perl`)を指定しているため
- NGINX コンテナは起動時に 80 番ポートで「Welcome to nginx!」ページを表示します
- まだ Spring Boot の WEB API コンテナは作成していないため、この状態は正常です

**次のステップ:**

- この後、Spring Boot の WEB API アプリケーションをローカルで準備し、CI/CD パイプラインを構築します
- CI/CD パイプラインにより、タスク定義が正しいイメージ URI に更新され、Spring Boot アプリケーションが動作するようになります

---

## 8. Spring Boot WEB API のセットアップ

### 8.1 概要

Spring Boot を使用した WEB API アプリケーションをローカル環境にセットアップします。このコースは Spring Boot の学習が目的ではないため、すでに作成済みのコードを使用します。

> **注意**:
>
> - 今回は Spring Boot を使用しますが、Ruby on Rails、Django、Next.js など他の Web フレームワークでも同じ考え方が適用できます
> - 実務では自分の環境に応じて応用してください

### 8.2 参考リポジトリ

このセクションで使用するコードは、以下の GitHub リポジトリで公開されています:

```
https://github.com/kentny/ecs-learning-course
```

### 8.3 手順

#### 8.3.1 ソースコードのダウンロード

1. 上記の GitHub リポジトリにアクセスします
2. ページ右下の「**Tags**」をクリックします
3. `cicd-hello-api` というタグを探します
4. タグ名の右側にある **ZIP アイコン** をクリックしてダウンロードします

#### 8.3.2 ダウンロードファイルの解凍

1. ダウンロードされた ZIP ファイルをダブルクリックして解凍します
2. 解凍すると `cicd-section` というディレクトリが含まれています

#### 8.3.3 ファイルの配置

**ターミナルで操作する場合:**

1. 以前作成した GitHub リポジトリをクローンしたディレクトリに移動します:

   ```bash
   cd ~/Desktop/ecs-learning-course
   ```

   ※クローンした場所に応じて適宜変更してください

2. 現在のディレクトリ構造を確認します:

   ```bash
   ls
   ```

   現時点では空のはずです。

3. ダウンロードフォルダから `cicd-section` ディレクトリを移動します:

   ```bash
   mv ~/Downloads/ecs-learning-course-cicd-hello-api/cicd-section .
   ```

4. ディレクトリが移動されたことを確認します:

   ```bash
   ls
   ```

   `cicd-section` ディレクトリが表示されれば成功です。

5. ダウンロードフォルダの不要なファイルを削除します:
   ```bash
   rm -rf ~/Downloads/ecs-learning-course-cicd-hello-api
   rm ~/Downloads/ecs-learning-course-cicd-hello-api.zip
   ```

#### 8.3.4 VS Code でプロジェクトを開く

1. VS Code を起動します
2. メニューから「ファイル」→「フォルダーを開く」を選択します
3. `ecs-learning-course/cicd-section/api` ディレクトリを選択します
   - 例: `/home/<ユーザー名>/Desktop/ecs-learning-course/cicd-section/api`
4. 「OK」ボタンをクリックします

> **補足**: コマンドラインから直接開く場合:
>
> ```bash
> code ~/Desktop/ecs-learning-course/cicd-section/api
> ```

#### 8.3.5 プロジェクト構造の確認

VS Code でプロジェクトが開いたら、以下のファイル・ディレクトリが存在することを確認します:

```
api/
├── Dockerfile          ← Docker用の設定ファイル
├── build.gradle        ← Gradleビルド設定
├── settings.gradle
└── src/
    ├── main/
    │   ├── java/
    │   │   └── controller/
    │   │       └── HelloController.java  ← WEB APIのコントローラー
    │   └── resources/
    └── test/
```

**重要なファイル:**

**Dockerfile:**

- コンテナイメージをビルドするための設定ファイル
- Spring Boot アプリケーションを実行する環境を定義

**HelloController.java:**

- `/api/hello` エンドポイントに GET リクエストが来た際に、固定文字列 `"Hello"` を返すシンプルな API
- Spring Boot の文法で実装されています

### 8.4 ローカル環境での動作確認

#### 8.4.1 概要

ローカルマシンで Spring Boot API を起動し、正常に動作することを確認します。実行環境として Docker コンテナを使用します。

> **なぜ Docker を使用するのか:**
>
> - Java 実行環境(JDK 21)や Gradle を直接 PC にインストールする必要がない
> - 環境の差異による問題を回避できる
> - 実際に AWS ECS で動作させる環境と同じ構成で確認できる

#### 8.4.2 Dockerfile の確認

`Dockerfile` の内容を簡単に確認します:

```dockerfile
# Java 21とGradleを含むベースイメージを使用
FROM gradle:jdk21

# 作業ディレクトリを設定
WORKDIR /app

# プロジェクトファイルをコンテナにコピー
COPY . .

# Spring Bootアプリケーションを起動
CMD ["./gradlew", "bootRun"]
```

**解説:**

- `gradle:jdk21` イメージを使用(Java 21 + Gradle 環境)
- `/app` を作業ディレクトリに設定
- プロジェクトファイル一式をコンテナ内にコピー
- `./gradlew bootRun` コマンドで Spring Boot を起動

#### 8.4.3 Docker イメージのビルド

1. VS Code でターミナルを開きます:

   - メニューから「ターミナル」→「新しいターミナル」を選択

2. 現在のディレクトリが `api` であることを確認します:

   ```bash
   pwd
   ```

3. Docker イメージをビルドします:

   ```bash
   docker image build -t hello-world-api .
   ```

   **コマンドの説明:**

   - `docker image build`: Docker イメージをビルド
   - `-t hello-world-api`: イメージに `hello-world-api` という名前(タグ)を付ける
   - `.`: カレントディレクトリをビルドコンテキストとして使用(Dockerfile を含む)

4. ビルドが完了するまで待ちます(初回は数分かかります)

   - `gradle:jdk21` イメージのダウンロード
   - 依存関係のダウンロード
   - プロジェクトのビルド

5. ビルド完了後、イメージが作成されたことを確認します:

   ```bash
   docker image ls
   ```

   以下のように `hello-world-api` が表示されれば成功です:

   ```
   REPOSITORY          TAG       IMAGE ID       CREATED          SIZE
   hello-world-api     latest    xxxxxxxxxxxx   30 seconds ago   xxx MB
   ```

#### 8.4.4 Docker コンテナの起動

1. ビルドしたイメージからコンテナを起動します:

   ```bash
   docker container run --rm -p 8080:8080 hello-world-api
   ```

   **コマンドの説明:**

   - `docker container run`: コンテナを起動
   - `--rm`: コンテナ停止時に自動的にコンテナを削除
   - `-p 8080:8080`: ホストの 8080 番ポートとコンテナの 8080 番ポートをマッピング
     - Spring Boot はデフォルトで 8080 番ポートでリッスンします
   - `hello-world-api`: 使用するイメージ名

2. 起動処理が開始されます(1〜2 分かかります)

   - 依存関係のダウンロード
   - アプリケーションの起動

3. 以下のような Spring ロゴと起動完了メッセージが表示されれば成功です:

   ```
     .   ____          _            __ _ _
    /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
   ( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
    \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
     '  |____| .__|_| |_|_| |_\__, | / / / /
    =========|_|==============|___/=/_/_/_/

   ... (起動ログ)
   ```

#### 8.4.5 API の動作確認

1. コンテナが起動したまま、新しいブラウザタブを開きます

2. 以下の URL にアクセスします:

   ```
   http://localhost:8080/api/hello
   ```

3. ブラウザに以下のように表示されれば成功です:
   ```
   Hello
   ```

**動作確認のポイント:**

- エンドポイント `/api/hello` にアクセス
- レスポンスとして固定文字列 `"Hello"` が返される
- これは `HelloController.java` で実装されたシンプルな API です

#### 8.4.6 コンテナの停止

1. ターミナルに戻り、`Ctrl + C` を押してコンテナを停止します
2. `--rm` オプションを指定しているため、コンテナは自動的に削除されます

---

## 9. Docker コンテナイメージの最適化

### 9.1 概要

前回の講義では、単一の`Dockerfile`を使用して Spring Boot アプリケーションのビルドと実行を行いました。しかし、この方法ではイメージサイズが大きくなり、デプロイ時間も長くなります。

本セクションでは、**マルチステージビルド**という技術を使用して、Docker イメージを最適化し、サイズを削減します。

### 9.2 Java アプリケーションのビルドと実行

#### 9.2.1 Java の特徴

Java は、Python や JavaScript などのインタープリタ型言語とは異なり、**コンパイルが必要**な言語です。

**ビルドプロセス:**

1. **複数の Java ソースファイル(.java)** → **コンパイル** → **1 つの JAR ファイル**
2. JAM ファイルを実行することでアプリケーションが起動される

> **例:**
>
> - 本コースのプロジェクトには複数の Java ファイルが存在します
> - Gradle というビルドツールを使用して、これらのファイルを 1 つの JAR ファイルにパッケージ化します
> - 最終的な成果物は「1 つの JAR ファイル」です

#### 9.2.2 前回の Dockerfile の問題点

前回の講義で使用した`Dockerfile`:

```dockerfile
FROM gradle:jdk21

WORKDIR /app

COPY . .

CMD [ "./gradlew", "bootRun" ]
```

**この方法の問題点:**

| 問題点                       | 説明                                                                                                     |
| ---------------------------- | -------------------------------------------------------------------------------------------------------- |
| **イメージサイズが大きい**   | `gradle:jdk21`イメージには、ビルドツールや多くの依存関係が含まれているため、イメージサイズが大きい       |
| **デプロイ時間が長い**       | コンテナ起動時に毎回`bootRun`コマンドが実行され、ビルドが行われるため起動に時間がかかる                  |
| **不要なファイルが含まれる** | ビルド完了後、ソースコード(.java ファイル)やビルドツール(Gradle)は実行に不要だが、イメージに含まれている |

**結果:**

- イメージサイズが非常に大きい(例:740MB)
- コンテナの起動が遅い
- ストレージ効率が悪い

#### 9.2.3 マルチステージビルドの概念

マルチステージビルドは、複数の`FROM`文を使用して、異なる段階で異なるベースイメージを使用する技術です。

**利点:**

1. **ビルド段階**と**実行段階**を分離
2. ビルド段階の成果物(JAR ファイル)だけを実行段階にコピー
3. ビルドに必要なファイルやツールを最終イメージから除外
4. **イメージサイズを大幅に削減**(例:740MB → 524MB)

**概念図:**

```
ステージ1(ビルド):                    ステージ2(実行):
┌─────────────────────┐              ┌──────────────────┐
│ gradle:jdk21        │              │ openjdk:21-slim  │
├─────────────────────┤              ├──────────────────┤
│ ・Gradleツール     │              │ ・Java実行環境   │
│ ・Javaコンパイラ   │              │ (最小限のみ)    │
│ ・ビルド環境       │              │                  │
│ ・ソースコード     │              │                  │
│ ・その他多数       │              │                  │
└─────────────────────┘              └──────────────────┘
           ↓                                   ↑
     ./gradlew build                   (JARファイル
      (JARファイル作成)                  を実行)
           ↓                                   ↑
      JARファイル ──────────────→ コピー
```

**イメージサイズの比較:**

| イメージ          | 用途       | サイズ         | 説明                           |
| ----------------- | ---------- | -------------- | ------------------------------ |
| `gradle:jdk21`    | ビルド環境 | 約 800MB       | ビルドツール等を含む           |
| `openjdk:21-slim` | 実行環境   | 約 80MB        | Java 実行環境のみ              |
| **削減効果**      | -          | **約 90%削減** | 最終イメージが大幅に小さくなる |

### 9.3 マルチステージ Dockerfile の実装

#### 9.3.1 改良された Dockerfile

マルチステージビルドを使用した新しい`Dockerfile`:

```dockerfile
# =====================================
# ステージ1: ビルドステージ
# =====================================
FROM gradle:jdk21 AS build

WORKDIR /app

COPY . /app

RUN chmod +x ./gradlew && \
    ./gradlew build

# =====================================
# ステージ2: 実行ステージ
# =====================================
FROM openjdk:21

# ビルドステージから成果物(JARファイル)をコピー
COPY --from=build /app/build/libs/my-app-0.0.1-SNAPSHOT.jar ./my-app.jar

# アプリケーションを実行
CMD [ "java", "-jar", "my-app.jar" ]
```

#### 9.3.2 Dockerfile の詳細説明

**ステージ 1: ビルドステージ(6 行目～ 12 行目)**

```dockerfile
FROM gradle:jdk21 AS build
```

- `gradle:jdk21` イメージをベースに使用
- `AS build` でこのステージに「build」という名前を付ける
- 後のステージで参照する際に使用

```dockerfile
WORKDIR /app
COPY . /app
RUN chmod +x ./gradlew && \
    ./gradlew build
```

- 作業ディレクトリ `/app` を作成
- ローカルのすべてのファイルをコンテナの `/app` にコピー
- `chmod +x ./gradlew`: Gradle ラッパーに実行権限を付与
- `./gradlew build` でプロジェクトをビルド
  - 複数の Java ファイルをコンパイル
  - 1 つの JAR ファイル(`build/libs/my-app-0.0.1-SNAPSHOT.jar`)を作成

> **重要**: 前回のように`bootRun`ではなく、`build`コマンドを使用
>
> - `bootRun`: ビルド + 実行(イメージ内で実行)
> - `build`: ビルド結果をファイルとして出力(JAR ファイル作成)

**ステージ 2: 実行ステージ(15 行目～ 21 行目)**

```dockerfile
FROM openjdk:21
```

- 実行用に`openjdk:21` イメージを使用
- ビルドツールやその他の不要なものは含まない

```dockerfile
COPY --from=build /app/build/libs/my-app-0.0.1-SNAPSHOT.jar ./my-app.jar
```

- `--from=build` で「build」ステージから取得
- ビルドステージで作成された JAR ファイルをコピー
- `/app/build/libs/my-app-0.0.1-SNAPSHOT.jar` : ビルドステージ内でのパス
- `./my-app.jar` : 実行ステージ内でのコピー先パス(名前変更)

```dockerfile
CMD [ "java", "-jar", "my-app.jar" ]
```

- Java コマンドでアプリケーションを起動
- `-jar` オプション: JAR ファイルを実行
- `my-app.jar` : 実行する JAR ファイル

### 9.4 Dockerfile の更新

#### 9.4.1 既存の Dockerfile を確認

1. VS Code で`api`ディレクトリが開いている状態を確認します

2. ファイルエクスプローラーから`Dockerfile`をクリックして開きます

3. 現在の内容を確認します:

```dockerfile
FROM gradle:jdk21

WORKDIR /app

COPY . /app

CMD [ "./gradlew", "bootRun" ]
```

#### 9.4.2 Dockerfile を更新

1. 既存の Dockerfile の内容を全て削除します

2. マルチステージビルド版に置き換えます:

```dockerfile
# =====================================
# ステージ1: ビルドステージ
# =====================================
FROM gradle:jdk21 AS build

WORKDIR /app

COPY . /app

RUN chmod +x ./gradlew && \
    ./gradlew build

# =====================================
# ステージ2: 実行ステージ
# =====================================
FROM openjdk:21

COPY --from=build /app/build/libs/my-app-0.0.1-SNAPSHOT.jar ./my-app.jar

CMD [ "java", "-jar", "my-app.jar" ]
```

3. ファイルを保存します(`Ctrl + S`)

#### 9.4.3 Dockerfile の保存確認

- ファイルタブにファイル名が表示され、変更マークがなければ保存完了です

### 9.5 ビルドイメージサイズの比較

#### 9.5.1 マルチステージビルド版のビルド

1. ターミナルを開きます:

   - メニューから「ターミナル」→「新しいターミナル」を選択
   - または`Ctrl + Shift + `(バッククォート)

2. 現在のディレクトリが`api`であることを確認します:

```bash
cd /path/to/ecs-learning-course/cicd-section/api
```

3. マルチステージビルド版のイメージをビルドします:

```bash
docker image build -t hello-world-api-v2 .
```

> **コマンド解説:**
>
> - `docker image build`: Docker イメージをビルド
> - `-t hello-world-api-v2`: イメージ名を `hello-world-api-v2` に設定
> - `.`: カレントディレクトリを対象(更新された Dockerfile を使用)

4. ビルド完了までしばらく待ちます(初回は数分かかります)

#### 9.5.2 ビルド完了確認

ビルド完了後、以下のコマンドでイメージ一覧を表示します:

```bash
docker image ls
```

**出力例:**

```
REPOSITORY               TAG       IMAGE ID       CREATED          SIZE
hello-world-api-v2      latest    xxxxxxxxxxxx   10 seconds ago   524MB
hello-world-api         latest    xxxxxxxxxxxx   1 day ago        740MB
```

**サイズ削減の確認:**

| イメージ                   | サイズ | 削減率         |
| -------------------------- | ------ | -------------- |
| `hello-world-api`(前回)    | 740MB  | -              |
| `hello-world-api-v2`(今回) | 524MB  | **約 29%削減** |

> **差分の原因:**
>
> - ビルド段階の Gradle ツール、コンパイラ、ソースコードが最終イメージから除外
> - 実行に必要な Java 実行環境(`openjdk:21-slim`)のみが含まれる

#### 9.5.3 イメージの実行確認

新しくビルドしたイメージが正常に動作することを確認します:

1. コンテナを起動します:

```bash
docker container run --rm -p 8080:8080 hello-world-api-v2
```

2. Spring Boot が起動するまで待ちます(約 30 秒～ 1 分)

3. 起動ログに Spring ロゴが表示されれば成功です:

```
     .   ____          _            __ _ _
    /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
   ( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
    \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
     '  |____| .__|_| |_|_| |_\__, | / / / /
    =========|_|==============|___/=/_/_/_/
```

#### 9.5.4 API の動作確認

1. 新しいブラウザタブを開きます

2. 以下の URL にアクセスします:

```
http://localhost:8080/api/hello
```

3. ブラウザに以下のように表示されることを確認します:

```
Hello
```

> **確認ポイント:**
>
> - エンドポイント `/api/hello` が正常に動作
> - マルチステージビルド版のイメージが正常に起動
> - アプリケーションが期待通りに動作

#### 9.5.5 起動速度の比較

**起動時間の改善:**

| イメージ                   | 起動時間     | 理由                                |
| -------------------------- | ------------ | ----------------------------------- |
| `hello-world-api`(前回)    | 1 ～ 2 分    | コンテナ起動時にビルド処理を実行    |
| `hello-world-api-v2`(今回) | 30 秒～ 1 分 | ビルド済みの JAR ファイルのみを実行 |

> **改善効果:**
>
> - 起動時間が約 50 ～ 60%高速化
> - 理由: ビルド処理がイメージビルド時に完了しているため

#### 9.5.6 コンテナの停止

1. ターミナルに戻ります

2. `Ctrl + C` を押してコンテナを停止します

3. `--rm` オプションのため、コンテナは自動的に削除されます

### 9.6 マルチステージビルドの利点まとめ

#### 9.6.1 ファイルサイズの削減

| 項目           | 前回  | 今回  | 削減                |
| -------------- | ----- | ----- | ------------------- |
| イメージサイズ | 740MB | 524MB | **216MB 削減(29%)** |

#### 9.6.2 起動速度の改善

- **前回:** コンテナ起動時にビルド処理を実行(1 ～ 2 分)
- **今回:** ビルド済み JAR を実行(30 秒～ 1 分)
- **改善率:** 約 50 ～ 60%高速化

#### 9.6.3 セキュリティの向上

| 項目                 | 説明                                                   |
| -------------------- | ------------------------------------------------------ |
| **ソースコード除外** | 最終イメージに Java ソースコードが含まれない           |
| **ビルドツール除外** | Gradle やその他のビルドツールが含まれない              |
| **攻撃対象面積削減** | 不要なソフトウェアが含まれないため、脆弱性のリスク低減 |

#### 9.6.4 本番環境への適用

マルチステージビルドの特性は、本番環境へのデプロイに特に効果的です:

- **ストレージ効率:** AWS などのクラウド環境でストレージコストを削減
- **デプロイ時間:** イメージサイズが小さいため、プルしやすく、起動が高速
- **スケーラビリティ:** 複数インスタンスのデプロイが迅速

---

## 10. GitHub Actions の設定

### 10.1 概要

GitHub Actions は、GitHub 上でコードの変更(プッシュ、プルリクエストなど)をトリガーに自動的にビルド、テスト、デプロイなどの処理を実行する CI/CD(継続的インテグレーション/継続的デプロイメント)ツールです。

このセクションでは、GitHub Actions を使用して自動的に以下の処理を実行できるよう設定を行います:

- コードの変更検出
- Docker イメージのビルド
- AWS ECR へのプッシュ
- ECS へのデプロイ

### 10.2 GitHub Actions の基本構造

GitHub Actions パイプラインは以下の要素で構成されています:

#### 10.2.1 イベント(Event)

- **定義**: パイプラインの開始トリガーとなる GitHub 上の動き
- **例**:
  - `push`: コミットがプッシュされた時
  - `pull_request`: プルリクエストが作成された時
  - `issues`: Issue が開かれた時
  - その他の GitHub イベント

#### 10.2.2 ランナー(Runner)

- **定義**: CI/CD パイプラインを実行するマシン
- **例**:
  - `ubuntu-latest`: Ubuntu の最新バージョン
  - `windows-latest`: Windows の最新バージョン
  - `macos-latest`: macOS の最新バージョン
  - セルフホストランナー(自身で用意したマシン)

#### 10.2.3 ジョブ(Job)

- **定義**: ランナー上で実行される処理の単位
- **特性**:
  - 複数個定義可能
  - デフォルトでは並列実行される(依存関係がなければ同時に実行)
  - それぞれ異なるランナーで独立して実行
  - ジョブ間の相互干渉がない

#### 10.2.4 ステップ(Step)

- **定義**: ジョブ内で実行される個別の処理
- **特性**:
  - ジョブ内では順序が保証される(上から順に実行)
  - 同じランナー上で実行されるため、前のステップの出力を次のステップで利用可能
  - ターミナルコマンドを順次実行するのと同じイメージ

#### 10.2.5 GitHub Actions の全体像

```
┌─────────────────────────────────────────┐
│ イベント(Event)                          │
│ 例: Push, Pull Request, Issue          │
└────────────────┬────────────────────────┘
                 │ トリガー
                 ▼
┌─────────────────────────────────────────┐
│ ランナー(Runner)                         │
│ Ubuntu, Windows, macOS など             │
└────────────────┬────────────────────────┘
                 │
       ┌─────────┴──────────┐
       ▼                    ▼
┌──────────────┐    ┌──────────────┐
│  ジョブ 1    │    │  ジョブ 2    │
│┌──────────┐ │    │┌──────────┐ │
││ステップ 1│ │    ││ステップ 1│ │
│└──────────┘ │    │└──────────┘ │
│┌──────────┐ │    │┌──────────┐ │
││ステップ 2│ │    ││ステップ 2│ │
│└──────────┘ │    │└──────────┘ │
└──────────────┘    └──────────────┘
    (並列実行)         (並列実行)
```

### 10.3 ワークフローファイルの作成

#### 10.3.1 ディレクトリ構造の準備

GitHub Actions を使用するには、リポジトリの特定の場所にワークフローファイルを配置する必要があります。

1. VS Code で `ecs-learning-course` ディレクトリを開きます

   - メニューから **File → Open Folder** を選択
   - `ecs-learning-course` ディレクトリを選択

2. ディレクトリが開かれたら、以下の信頼確認ダイアログが表示される場合があります

   > **Do you trust the authors of the files in this folder?**

   このメッセージに対して、**Yes, I trust the authors** をクリックして進めます

3. 新規フォルダを作成します

   - エクスプローラーの最上部で **New Folder** アイコンをクリック(またはマウス右クリックで「New Folder」を選択)
   - フォルダ名を **`.github`** と入力します(先頭のドット `.` は必須です)

4. `.github` フォルダ内に新規フォルダを作成します

   - `.github` フォルダを展開して選択
   - **New Folder** アイコンをクリック
   - フォルダ名を **`workflows`** と入力します

> **GitHub Actions 規約:**
>
> GitHub Actions が自動的にワークフローを認識するには、ファイルの配置場所が固定されています:
>
> - ディレクトリ構造: `/.github/workflows/`
> - ファイル形式: YAML ファイル(`.yml` または `.yaml` 拡張子)
>
> この命名と構造は GitHub により決定されているため、変更することはできません。

#### 10.3.2 ワークフローファイルの作成

1. `workflows` フォルダ内に新規ファイルを作成します

   - `workflows` フォルダを選択した状態で **New File** をクリック
   - ファイル名を入力します: **`cicd.yml`**

   > **ファイル名について:**
   >
   > - 拡張子は `.yml` または `.yaml` である必要があります
   > - ファイル名の前半部分(`ci-cd`)は任意です(GitHub が自動的に認識します)
   > - わかりやすい名前を付けることをお勧めします

2. ファイルが開かれると、YAML ファイルの編集が可能になります

### 10.4 ワークフローの基本設定

#### 10.4.1 ワークフロー定義の記述

作成した `cicd.yml` ファイルに、以下の内容を記述します:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  hello-job:
    runs-on: ubuntu-latest

    steps:
      - run: echo Hello
      - run: echo "GitHub Actions!"

  goodbye-job:
    runs-on: ubuntu-latest

    steps:
      - run: echo "Goodbye GitHub Actions!"
```

#### 10.4.2 設定内容の詳細説明

**1. パイプラインの名前**

```yaml
name: CI/CD Pipeline
```

- このワークフロー全体の識別名です
- GitHub の Actions ページに表示されます
- わかりやすい名前を付けてください

**2. トリガーイベント(on セクション)**

```yaml
on:
  push:
    branches:
      - main
```

- **`on`**: このワークフローが実行されるトリガーを定義
- **`push`**: コミットがプッシュされた時にトリガー
- **`branches`**: ブランチの指定
  - `- main`: `main` ブランチへのプッシュが対象
  - 他のブランチも対象にする場合: `- main` と `- develop` など複数記述可能

**複数イベントを指定する例:**

```yaml
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
```

このように記述することで、プッシュ時とプルリクエスト時の両方がトリガーになります。

**3. ジョブの定義(jobs セクション)**

```yaml
jobs:
  hello-job:
    runs-on: ubuntu-latest
    steps: ...
```

- **`jobs`**: 複数のジョブを定義
- **`hello-job`**: ジョブの識別名(任意で命名可能)
- **`runs-on`**: このジョブを実行するランナーを指定
  - `ubuntu-latest`: Ubuntu の最新版を使用
  - 他の選択肢: `windows-latest`, `macos-latest` など
- **`steps`**: このジョブ内で実行するステップのリスト

**4. ステップの定義(steps セクション)**

```yaml
steps:
  - run: echo Hello
  - run: echo "GitHub Actions!"
```

- **`-` (ハイフン)**: YAML のリスト表記(複数のステップを並べる)
- **`run`**: 実行するコマンド
  - 通常のターミナルコマンドと同じ

> **ステップの`name` フィールドについて:**
>
> `name` フィールドはオプションです。省略した場合、GitHub Actions は `run` コマンドの内容をステップ名として表示します。
>
> - `name` を指定した場合: ステップ名として指定した文字列が表示される
> - `name` を省略した場合: `run` コマンドの内容が自動的にステップ名として使用される

**このワークフローの実行フロー:**

1. `main` ブランチへのコミットプッシュをトリガー
2. `hello-job` と `goodbye-job` が並列実行開始
3. `hello-job` 内で:
   - ステップ 1: `echo Hello` を実行して "Hello" を出力
   - ステップ 2: `echo "GitHub Actions!"` を実行して "GitHub Actions!" を出力
4. `goodbye-job` 内で:
   - ステップ 1: `echo "Goodbye GitHub Actions!"` を実行
5. 両ジョブの完了を待つ
6. パイプライン終了

### 10.5 ワークフローファイルの保存とプッシュ

#### 10.5.1 ファイルの保存

1. `cicd.yml` ファイルを保存します

   - キーボード: `Ctrl + S`(Windows/Linux) または `Cmd + S`(macOS)

#### 10.5.2 Git コマンドでコミット・プッシュ

1. VS Code 内のターミナルを開きます

   - メニュー: **Terminal → New Terminal** または `Ctrl + `` (バッククォート)

2. `ecs-learning-course` ディレクトリにいることを確認します

   ```bash
   pwd
   ```

3. Git のステータスを確認します

   ```bash
   git status
   ```

   **出力例:**

   ```
   On branch main

   No commits yet

   Untracked files:
     (use "git add <file>..." to include in what will be committed)
           .github/

   nothing added to commit but untracked files exist (use "git add" to track)
   ```

   > `.github` ディレクトリが `Untracked files`(追跡されていないファイル)として表示されます

4. ファイルをステージします

   ```bash
   git add .github
   ```

   または、すべてのファイルを追加する場合:

   ```bash
   git add .
   ```

5. ステータスを確認して、ステージされたことを確認します

   ```bash
   git status
   ```

   **出力例:**

   ```
   On branch main

   No commits yet

   Changes to be committed:
     (use "git rm --cached <file>..." to unstage)
           new file:   .github/workflows/cicd.yml
   ```

6. コミットを作成します

   ```bash
   git commit -m "Add GitHub Actions CI/CD pipeline"
   ```

   **出力例:**

   ```
   [main (root-commit) xxxxxxxx] Add GitHub Actions CI/CD pipeline
    1 file changed, 30 insertions(+)
    create mode 100644 .github/workflows/cicd.yml
   ```

7. リポジトリにプッシュします

   ```bash
   git push origin main
   ```

   **出力例:**

   ```
   Enumerating objects: 4, done.
   Counting objects: 100% (4/4), done.
   Delta compression: 100% (2/2), done.
   Writing objects: 100% (4/4), 389 bytes | 389.00 KiB/s, done.
   Total 4 (delta 0), reused 0 (delta 0), received 0 (pack-reused 0)
   To https://github.com/your-account/ecs-learning-course.git
    * [new branch]      main -> main
   ```

### 10.6 GitHub Actions の実行確認

#### 10.6.1 Actions ページへアクセス

1. GitHub のリポジトリページにアクセスします

2. ページ上部のタブメニューから **Actions** をクリックします

   > **画面例:**
   >
   > リポジトリページには以下のタブが表示されます:
   > `Code | Issues | Pull requests | Actions | Projects | ...`

#### 10.6.2 ワークフロー実行の確認

1. Actions ページが表示されます

   > **初回の表示:**
   > プッシュしたコミットメッセージが表示され、右側に **緑色のチェックマーク(✓)** が付いている場合、パイプラインが正常に完了しています。

2. コミットメッセージをクリックして、詳細ページを表示します

   **詳細ページの構成:**

   - 左側: ジョブ一覧

     - `hello-job`
     - `goodbye-job`

   - 右側: ログ表示領域

#### 10.6.3 ジョブの実行ログを確認

1. `hello-job` をクリックすると、そのジョブ内で実行されたステップが表示されます

   **表示内容:**

   ```
   ✓ Set up job
   ✓ Run echo Hello
   ✓ Run echo "GitHub Actions!"
   ✓ Complete job
   ```

   > `name` フィールドを指定していないため、ステップ名は `run` コマンドの内容が使用されます。

2. **Run echo Hello** のステップをクリックして詳細を確認します

   **ログ内容:**

   ```
   Run echo Hello
   Hello
   ```

3. **Run echo "GitHub Actions!"** のステップをクリックして詳細を確認します

   **ログ内容:**

   ```
   Run echo "GitHub Actions!"
   GitHub Actions!
   ```

#### 10.6.4 goodbye-job の確認

1. 左側のジョブ一覧から `goodbye-job` をクリックします

2. このジョブ内のステップを確認します

   **表示内容:**

   ```
   ✓ Set up job
   ✓ Run echo "Goodbye GitHub Actions!"
   ✓ Complete job
   ```

3. **Run echo "Goodbye GitHub Actions!"** ステップの詳細を確認します

   **ログ内容:**

   ```
   Run echo "Goodbye GitHub Actions!"
   Goodbye GitHub Actions!
   ```

#### 10.6.5 実行結果の確認ポイント

| 確認項目             | 期待値                      | 確認方法                         |
| -------------------- | --------------------------- | -------------------------------- |
| **パイプライン結果** | 緑色のチェックマーク(✓)表示 | Actions ページのトップで確認     |
| **ジョブ数**         | 2 つのジョブが表示される    | 詳細ページ左側のジョブ一覧で確認 |
| **ジョブ実行順序**   | 並列実行(完了時刻が近い)    | 各ジョブの完了時刻を比較         |
| **ステップ出力**     | 正しい文字列が出力される    | 各ステップのログで確認           |

### 10.7 GitHub Actions の実行仕組みの確認

#### 10.7.1 イベントとトリガーの関係

**今回のワークフロー:**

```yaml
on:
  push:
    branches:
      - main
```

このトリガー定義により:

- `main` ブランチへの`push`(コミット)があった時点でイベントが発火
- GitHub Actions が自動的にワークフローを実行開始
- ユーザーが手動で実行を開始する必要はない

#### 10.7.2 ジョブの並列実行確認

ワークフローで複数のジョブを定義した場合:

```yaml
jobs:
  hello-job: ...
  goodbye-job: ...
```

**実行パターン:**

- デフォルトでは `hello-job` と `goodbye-job` は**並列実行**
- 依存関係がなければ、同時に実行される
- それぞれ異なるランナーで独立して動作

**実行時間の比較:**

| パターン     | 実行時間 | 説明                                        |
| ------------ | -------- | ------------------------------------------- |
| **並列実行** | 30 秒    | 2 つのジョブが同時に実行(今回のケース)      |
| **直列実行** | 60 秒    | ジョブ 1 終了 → ジョブ 2 実行(もし依存関係) |

#### 10.7.3 ステップ内での順序保証

同一ジョブ内のステップは必ず順序が保証されます:

```yaml
steps:
  - name: Step 1
    run: echo "First"

  - name: Step 2
    run: echo "Second"

  - name: Step 3
    run: echo "Third"
```

実行結果:

```
First
Second
Third
```

## 必ずこの順番で実行されます(並列化されない)。

## 11. Web API デプロイパイプラインの作成

### 11.1 概要

Spring Boot の Web API アプリケーションを ECS にデプロイするための GitHub Actions パイプラインを作成します。

### 11.2 前提条件

- GitHub リポジトリに `cicd-section/api` ディレクトリ配下に Spring Boot のコードが存在すること
- 前のセクションで基本的な GitHub Actions の動作を確認済みであること

### 11.3 ワークフローファイルの編集

#### 11.3.1 ファイルの準備

1. 前回作成した `.github/workflows/cicd.yml` ファイルを開きます

#### 11.3.2 ワークフロー名の変更

```yaml
name: Web API デプロイパイプライン
```

**設定の意図:**

- パイプラインの目的を明確にするため

#### 11.3.3 トリガー設定の変更

従来の `on: [push]` を以下のように変更します:

```yaml
on:
  push:
    paths:
      - "cicd-section/api/**"
```

**設定の意図:**

- `cicd-section/api` ディレクトリ配下のファイルに変更があった場合のみワークフローを実行
- `**` はワイルドカードで、サブディレクトリを含めたすべてのファイルを対象とする
- Spring Boot のコード以外の変更では、このパイプラインは実行されない(無駄な実行を防ぐ)

#### 11.3.4 環境変数の定義

`on` セクションの後に、環境変数を定義します:

```yaml
env:
  AWS_REGION: us-west-2
  ECS_CLUSTER: my-app-cluster
  ECS_SERVICE: my-app-api-service
  ECR_REPOSITORY: my-app-api
  ECS_TASK_DEFINITION_API: cicd-section/.aws/task-definition.json
```

**各環境変数の説明:**

| 変数名                    | 説明                               | 備考                               |
| ------------------------- | ---------------------------------- | ---------------------------------- |
| `AWS_REGION`              | 使用する AWS リージョン            | 今回は `us-west-2`(オレゴン)を使用 |
| `ECS_CLUSTER`             | ECS クラスター名                   | 事前に作成済みのクラスター名       |
| `ECS_SERVICE`             | ECS サービス名                     | 事前に作成済みのサービス名         |
| `ECR_REPOSITORY`          | ECR リポジトリ名                   | 事前に作成済みのリポジトリ名       |
| `ECS_TASK_DEFINITION_API` | ECS タスク定義ファイルのパス(JSON) | 後ほど作成するファイルのパス       |

**注意:**

- `ECS_CLUSTER`、`ECS_SERVICE`、`ECR_REPOSITORY` の値は、事前に作成した実際のリソース名に置き換えてください
- `ECS_TASK_DEFINITION_API` のファイルは後の手順で作成します

#### 11.3.5 パーミッション設定

`env` セクションと `jobs` セクションの間に、パーミッション設定を追加します:

```yaml
permissions:
  id-token: write
  contents: read
```

**設定の意図:**

- `id-token: write`: OpenID Connect(OIDC)認証で必要となる ID トークンの書き込み権限
- `contents: read`: リポジトリのコンテンツ(コード)を読み取る権限
- この設定により、GitHub Actions から AWS へ安全に認証できるようになります

### 11.4 ジョブの定義

#### 11.4.1 概要

CI/CD パイプラインの基本的な流れは以下の通りです:

```
テスト → ビルド → デプロイ
```

今回は Spring Boot の特性を活かして、**テストとビルドを同時に実行**します。

#### 11.4.2 test-and-build ジョブの作成

##### ジョブの基本設定

```yaml
jobs:
  test-and-build:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: cicd-section/api
```

**設定の説明:**

| 項目                             | 説明                                           |
| -------------------------------- | ---------------------------------------------- |
| `test-and-build`                 | ジョブ名(テストとビルドを同時実行)             |
| `runs-on: ubuntu-latest`         | Ubuntu の最新版を使用                          |
| `defaults.run.working-directory` | 以降のステップで使用する作業ディレクトリを指定 |

**working-directory の利点:**

- Spring Boot のルートディレクトリを指定することで、以降のコマンドで毎回 `cd` する必要がなくなる
- コードの見通しが良くなる

##### ステップ 1: コードのチェックアウト

```yaml
   steps:
   - name: Checkout
      uses: actions/checkout@v4
```

**設定の説明:**

- `uses`: GitHub Marketplace の公開アクションを使用
- `actions/checkout@v4`: GitHub 公式が提供するチェックアウトアクション
- リポジトリのコードをワークフローの実行環境にダウンロード

**参考:**

- GitHub Marketplace で "checkout" を検索すると詳細を確認可能
- https://github.com/marketplace/actions/checkout

##### ステップ 2: テストとビルドイメージの作成

```yaml
- name: Run Test and Build an Image
  run: docker image build -t temp_api_image:latest .
```

**設定の説明:**

- Docker イメージをビルドし、`temp_api_image:latest` という一時的な名前を付与

2. `Dockerfile` 内で `gradle build` が実行される
3. `gradle build` の過程でテストコード(`src/test`)が自動実行される
4. テストが失敗した場合、ビルドが失敗してパイプラインが停止

**一時的な名前を付ける理由:**

- この時点では ECR のレジストリ情報が確定していない
- 後のステップで認証後に正式な名前に変更する
- イメージ名(`my-app-api-image`)は ECR リポジトリ名に合わせて命名

##### ステップ 3: AWS 認証の設定

```yaml
   - name: Configure AWS Credentials
   uses: aws-actions/configure-aws-credentials@v4
   with:
      aws-region: ${{ env.AWS_REGION }}
      role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
```

**設定の説明:**

| 項目             | 説明                                                               |
| ---------------- | ------------------------------------------------------------------ |
| `uses`           | AWS 公式が提供する認証設定アクション                               |
| `with`           | アクションに渡すパラメータ                                         |
| `aws-region`     | `${{ env.AWS_REGION }}`: 環境変数から取得                          |
| `role-to-assume` | `${{ secrets.AWS_ROLE_TO_ASSUME }}`: GitHub Secrets から安全に取得 |

**変数の使い分け:**

| 使用箇所  | 用途                      | セキュリティレベル |
| --------- | ------------------------- | ------------------ |
| `env`     | リージョンなどの公開情報  | 低                 |
| `secrets` | ロール ARN などの秘匿情報 | 高                 |

**secrets の利点:**

- YAML ファイルには記載されない
- GitHub Actions のログにも表示されない(マスキングされる)
- 限られた人のみが値を確認可能

**OpenID Connect(OIDC)認証:**

- この設定により、OIDC を使用した AWS への認証が行われます
- 詳細は後のセクションで解説します

**注意:**

- `AWS_ROLE` の値は後ほど GitHub の Settings で設定します

##### ステップ 4: Amazon ECR へのログイン

```yaml
- name: Login to Amazon ECR
  id: login-ecr
  uses: aws-actions/amazon-ecr-login@v2
```

**設定の説明:**

- `id: login-ecr`: このステップに ID を付与(後続ステップで参照するため)
- AWS 公式が提供する ECR ログインアクション
- Docker Hub にログインするのと同様、ECR にイメージをプッシュするための事前認証

**実行される内容:**

- `docker login` コマンド相当の処理
- 以前手動で実行した「プッシュコマンドを表示」の手順を自動化
  　(ECR > プライベートリポジトリ > リポジトリ > my-app-api に「**プッシュコマンドを表示**」ボタンがあった)

##### ステップ 5: イメージのタグ付けと ECR へのプッシュ

````yaml
- name: Push the Image to ECR
  env:
    ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
  run: |
          docker image tag temp_api_image:latest $ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
**環境変数の定義:**

```yaml
env:
  ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
````

- このステップ内で使用する環境変数を定義
- `steps.login-ecr.outputs.registry`: 前のステップ(id: login-ecr)の出力結果を参照
- ECR のレジストリ URL(例: `123456789012.dkr.ecr.us-west-2.amazonaws.com`)が格納される

**タグの変更:**

```bash
docker image tag temp_api_image:latest $ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
```

元の名前: `temp_api_image:latest`
↓
新しい名前: `<ECRレジストリURL>/<リポジトリ名>:<コミットハッシュ>`

**例:**

```
123456789012.dkr.ecr.us-west-2.amazonaws.com/my-app-api:a1b2c3d4e5f6...
```

**各部分の説明:**

| 部分                        | 値の例                                         | 説明                 |
| --------------------------- | ---------------------------------------------- | -------------------- |
| `$ECR_REGISTRY`             | `123456789012.dkr.ecr.us-west-2.amazonaws.com` | ECR のレジストリ URL |
| `${{ env.ECR_REPOSITORY }}` | `my-app-api`                                   | リポジトリ名         |
| `${{ github.sha }}`         | `a1b2c3d4e5f6...`                              | Git コミットハッシュ |

**コミットハッシュをタグに使用する理由:**

1. **一意性の保証**: コミットごとに必ず異なるハッシュが生成される
2. **トレーサビリティ**: どのコミットからビルドされたイメージか特定可能
3. **重複防止**: 同じタグ名でイメージが上書きされることを防ぐ

**プッシュの実行:**

```bash
docker image push $ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
```

- タグ付けしたイメージを ECR にプッシュ

**ECR_REGISTRY を環境変数から参照する理由:**

- AWS アカウント ID が含まれており、秘匿性が高い情報
- メールアドレスのように、漏洩しても直接攻撃されるわけではないが、公開すべきでない情報
- ログイン後の出力結果から動的に取得することで、YAML ファイルに直接記載しない

### 11.5 ワークフロー全体の確認

ここまでで作成した `.github/workflows/cicd.yml` の全体像:

```yaml
name: Web API デプロイパイプライン

on:
  push:
    paths:
      - "cicd-section/api/**"

env:
  AWS_REGION: us-west-2
  ECS_CLUSTER: my-app-cluster
  ECS_SERVICE: my-app-api-service
  ECR_REPOSITORY: my-app-api
  ECS_TASK_DEFINITION_API: cicd-section/.aws/task-definition.json

permissions:
  id-token: write
  contents: read

jobs:
  test-and-build:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: cicd-section/api

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Test and Build Image
        run: |
          docker image build -t temp_api_image:latest .

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: ${{ env.AWS_REGION }}
          role-to-assume: ${{ secrets.AWS_ROLE }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Push Image to ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        run: |
          docker image tag my-app-api-image:temp $ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
          docker image push $ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
```

---

## 12. AWS 側の GitHub Actions 連携設定

### 12.1 概要

GitHub Actions から AWS のリソース(ECR、ECS など)にアクセスできるように、AWS 側の設定を行います。

前回の講義で説明した **OpenID Connect(OIDC)** を使用して、AWS IAM 側に特定のリポジトリ、特定のブランチなど、特定の条件を満たしたリクエストだけに対してアクセストークンを与えるという設定をします。

### 12.2 OpenID Connect(OIDC)とは

#### 12.2.1 従来の認証方法の問題点

**従来の方法: アクセスキーとシークレットキーを使用**

- AWS のアクセスキーとシークレットキーを GitHub Secrets に保存
- GitHub Actions から AWS へアクセスする際に使用
- **問題点:**
  - キーの有効期限がない(漏洩した場合のリスクが高い)
  - キーのローテーション管理が必要
  - 漏洩した場合の影響範囲が大きい

#### 12.2.2 OIDC 認証のメリット

**OIDC を使用した認証:**

- 一時的なアクセストークンを発行
- トークンの有効期限が短い(漏洩リスクの軽減)
- 特定の条件(リポジトリ、ブランチなど)を満たした場合のみアクセス許可
- アクセスキーを GitHub に保存する必要がない

**認証フロー:**

```
GitHub Actions → AWS STS(Security Token Service)
              ↓
         IDトークンを送信
              ↓
    AWS IAMで条件を検証(リポジトリ、ブランチなど)
              ↓
  条件を満たす場合のみ一時的なアクセストークンを発行
              ↓
    GitHub Actionsが一時トークンでAWSリソースにアクセス
```

### 12.3 IAM ID プロバイダーの作成

#### 12.3.1 IAM コンソールへアクセス

1. AWS マネジメントコンソールで「**IAM**」を検索します
2. 「IAM」をクリックして開きます

#### 12.3.2 ID プロバイダーの追加

1. 左側のメニューで「**ID プロバイダー**」をクリックします
2. 「**プロバイダーを追加**」ボタンをクリックします

#### 12.3.3 プロバイダーの設定

**プロバイダーのタイプ:**

- 「**OpenID Connect**」を選択します
  - SAML ではなく、OpenID Connect を選択

**プロバイダーの URL:**

```
https://token.actions.githubusercontent.com
```

- この値は GitHub 公式ドキュメントで指定されているものです
- 参考: [GitHub Actions で OpenID Connect を使用した AWS の設定](https://docs.github.com/ja/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)

**対象者(Audience):**

```
sts.amazonaws.com
```

- こちらも GitHub 公式ドキュメントに記載されている値です

**タグ:**

- 設定不要(デフォルトのまま)

#### 12.3.4 サムプリントの取得

1. 「**サムプリントを取得**」ボタンをクリックします
   - サムプリントが自動的に取得されます

#### 12.3.5 プロバイダーの作成

1. 「**プロバイダーを追加**」ボタンをクリックします
2. ID プロバイダーが作成されます

**作成完了メッセージ:**

> このプロバイダーの使用を開始するには、IAM ロールを割り当てる必要があります。

次のセクションでロールの割り当てを行います。

---

### 12.4 IAM ロールの作成と割り当て

#### 12.4.1 ロールの割り当て開始

1. ID プロバイダーの作成完了画面で「**ロールの割り当て**」ボタンをクリックします
2. 「**新しいロールを作成**」を選択した状態で「**次へ**」をクリックします

#### 12.4.2 信頼されたエンティティの設定

**信頼されたエンティティタイプ:**

- 「**Web アイデンティティ**」が選択された状態を確認します

**アイデンティティプロバイダー:**

- 先ほど作成した「**token.actions.githubusercontent.com**」が選択されていることを確認します

**Audience:**

- 「**sts.amazonaws.com**」が選択されていることを確認します

#### 12.4.3 GitHub 組織とリポジトリの設定

画面を下にスクロールして、以下の項目を設定します:

**GitHub 組織:**

```
<あなたのGitHubアカウント名またはOrganization名>
```

- 個人アカウントの場合: GitHub のユーザー名を入力
- Organization の場合: Organization 名を入力
- 例: リポジトリ URL が `https://github.com/yourname/ecs-learning-course` の場合、`yourname` を入力

**GitHub リポジトリ:**

```
ecs-learning-course
```

- 今回使用するリポジトリの名前を入力します

**GitHub ブランチ:**

```
main
```

- `main` ブランチからのプッシュに対してのみ許可するように設定します
- 本番環境では、特定のブランチのみを許可することでセキュリティを向上できます

#### 12.4.4 次へ

1. 設定が完了したら「**次へ**」ボタンをクリックします

---

### 12.5 許可ポリシーの設定(後で設定)

#### 12.5.1 許可ポリシーの概要

このロールに割り当てるポリシーによって、発行されるアクセストークンでできることが決まります。

今回必要な権限:

- ECR へのイメージのアップロード
- ECS タスク定義の更新
- ECS サービスの更新

#### 12.5.2 一旦スキップ

1. この画面では**何も選択せずに**「**次へ**」をクリックします
   - 後ほどインラインポリシーとして設定します

---

### 12.6 ロール名と信頼ポリシーの確認

#### 12.6.1 ロール名の設定

**ロール名:**

```
GitHubActionsEcsLearningCourse
```

- わかりやすい名前を付けます
- 例: `GitHubActions` + `ECSLearningCourse` リポジトリ用

**説明(任意):**

```
Role for GitHub Actions ecs-learning-course-repository
```

#### 12.6.2 信頼されたエンティティの確認

画面を下にスクロールして、「**信頼されたエンティティ**」セクションを確認します。

**信頼ポリシーの例:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:yourname/ecs-learning-course:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

**重要な部分:**

```json
"token.actions.githubusercontent.com:sub": "repo:yourname/ecs-learning-course:ref:refs/heads/main"
```

この設定により:

- 指定した組織(`yourname`)
- 指定したリポジトリ(`ecs-learning-course`)
- 指定したブランチ(`main`)

からのリクエストに対してのみアクセストークンを発行します。

#### 12.6.3 ロールの作成

1. 内容を確認したら「**ロールを作成**」ボタンをクリックします
2. ロールが作成されます

---

### 12.7 インラインポリシーの追加

#### 12.7.1 作成したロールを開く

1. IAM コンソールで左側のメニューから「**ロール**」をクリックします
2. ロール一覧から「**GitHubActionsEcsLearningCourse**」を探してクリックします

#### 12.7.2 許可ポリシーの追加開始

1. 「**許可を追加**」ボタンをクリックします
2. 「**インラインポリシーを作成**」を選択します

#### 12.7.3 ポリシーエディターの切り替え

1. デフォルトでは「**ビジュアル**」エディターが表示されています
2. 「**JSON**」タブをクリックして JSON エディターに切り替えます

#### 12.7.4 ECR 用のポリシーを流用

**以前作成したローカルユーザーのポリシーを確認:**

> **注意:** 以前のセクションで「ECR へのプッシュ・プル用のローカルユーザー」を作成していた場合、そのユーザーにアタッチされているポリシーを流用できます。削除してしまった場合は、次のステップで記載する JSON をそのまま使用してください。

1. 新しいタブで IAM コンソールを開きます(右クリック → 新しいタブで開く)
2. 左側のメニューで「**ユーザー**」をクリックします
3. ローカルユーザー(例: `local-user`)をクリックします
4. 「**許可**」タブで、アタッチされているポリシー(例: `ECRPushPushPullImage`)をクリックします
5. 「**JSON**」タブで内容を確認します

**ポリシーの例:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid":"Statement1",
      "Effect": "Allow",
      "Action": [
        "ecr:GetDownloadUrlForLayer",
        "ecr: BatchGetImage",
        "ecr:BatchCheckLayerAvailability",
        "ecr: PutImage",
        "ecr:InitiateLayerUpload",
        "ecr: UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    }
  ]
}
```

6. このポリシーの `Statement` 配列内の波括弧 `{ ... }` 部分をコピーします

#### 12.7.5 ECS タスク定義更新用の権限を追加

1. ポリシー作成画面に戻ります
2. JSON エディターに以下の内容を貼り付けます:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid":"Statement1",
      "Effect": "Allow",
      "Action": [
        "ecr:GetDownloadUrlForLayer",
        "ecr: BatchGetImage",
        "ecr:BatchCheckLayerAvailability",
        "ecr: PutImage",
        "ecr:InitiateLayerUpload",
        "ecr: UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecs:UpdateService",
        "ecs:RegisterTaskDefinition",
        "ecs:ListTaskDefinitions",
        "ecs:DescribeServices"
      ],
      "Resource": "*"
    }
  ]
}
```

**ポリシーの内容説明:**

| アクション                             | 説明                                        |
| -------------------------------------- | ------------------------------------------- |
| `ecr:GetDownloadUrlForLayer`           | イメージレイヤーのダウンロード URL 取得     |
| `ecr:BatchGetImage`                    | イメージの取得                              |
| `ecr:BatchCheckLayerAvailability`      | イメージレイヤーの存在確認                  |
| `ecr:PutImage`                         | イメージのプッシュ                          |
| `ecr:InitiateLayerUpload`              | レイヤーアップロードの開始                  |
| `ecr:UploadLayerPart`                  | レイヤーの部分アップロード                  |
| `ecr:CompleteLayerUpload`              | レイヤーアップロードの完了                  |
| `ecr:GetAuthorizationToken`            | ECR へのログイン認証トークンを取得          |
| `ecs:UpdateService`                    | ECS サービスの更新                          |
| `ecs:RegisterTaskDefinition`           | 新しいタスク定義の登録                      |
| `ecs:ListTaskDefinitions`              | タスク定義の一覧取得                        |
| `ecs:DescribeServices`                 | ECS サービスの情報取得                      |

> **詳細を確認したい場合:**
> 各アクションの詳細は AWS 公式ドキュメントを参照してください。
>
> - [ECR のアクション](https://docs.aws.amazon.com/ja_jp/AmazonECR/latest/APIReference/API_Operations.html)
> - [ECS のアクション](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/APIReference/API_Operations.html)

#### 12.7.6 ビジュアルエディターでの確認

1. 「**ビジュアル**」タブをクリックして切り替えます
2. 以下のように表示されることを確認します:
   - **許可 1**: ECR 関連のアクション(8 個)
   - **許可 2**: ECS 関連のアクション(4 個)

#### 12.7.7 ポリシー名の設定

1. 「**次へ**」ボタンをクリックします
2. **ポリシー名**を入力します:

```
UpdateECSTask
```

#### 12.7.8 ポリシーの作成

1. 「**ポリシーの作成**」ボタンをクリックします
2. ポリシーがロールにアタッチされます

#### 12.7.9 ポリシーの確認

1. ロールの詳細ページに戻ります
2. 「**許可**」タブで、以下が表示されることを確認します:
   - インラインポリシー: `UpdateECSTask`
3. ポリシー名の左側の **`▶`(展開アイコン)** をクリックします
4. 先ほど記述した JSON の内容が表示されることを確認します

---

### 12.8 GitHub Secrets への AWS ロール ARN の設定

#### 12.8.1 ロール ARN のコピー

1. IAM コンソールのロール詳細ページで、「**概要**」セクションを確認します
2. **ARN(Amazon Resource Name)** が表示されています

   **例:**

   ```
   arn:aws:iam::123456789012:role/GitHubActionsEcsLearningCourse
   ```

3. ARN の右側にある **コピーアイコン** をクリックして ARN をコピーします

#### 12.8.2 GitHub リポジトリの Settings へアクセス

1. GitHub のリポジトリページ(`https://github.com/yourname/ecs-learning-course`)を開きます
2. ページ上部のタブメニューから「**Settings**」をクリックします

> **注意:** リポジトリのオーナーまたは管理者権限が必要です。Settings タブが表示されない場合は、権限を確認してください。

#### 12.8.3 Secrets and variables の設定

1. 左側のメニューで「**Secrets and variables**」をクリックして展開します
2. 「**Actions**」をクリックします

#### 12.8.4 新しい Secret の追加

1. 「**Secrets**」タブが選択されていることを確認します
2. 「**New repository secret**」ボタンをクリックします

#### 12.8.5 Secret の設定

**Name(変数名):**

```
AWS_ROLE_TO_ASSUME
```

- この変数名は、GitHub Actions のワークフローファイル(`cicd.yml`)で使用しています
- `${{ secrets.AWS_ROLE_TO_ASSUME }}` として参照されます

**Secret(値):**

- 先ほどコピーした **ロールの ARN** を貼り付けます

  **例:**

  ```
  arn:aws:iam::123456789012:role/GitHubActionsEcsLearningCourse
  ```

#### 12.8.6 Secret の追加

1. 「**Add secret**」ボタンをクリックします
2. Secret が追加されます

**確認:**

- 「Repository secrets」セクションに `AWS_ROLE_TO_ASSUME` が表示されます
- 値は `***` でマスキングされて表示されます(セキュリティのため)

---

### 12.9 設定完了の確認

#### 12.9.1 設定内容のまとめ

これまでの設定により、以下が完了しました:

| 設定項目             | 内容                                                           |
| -------------------- | -------------------------------------------------------------- |
| **ID プロバイダー**  | GitHub Actions からの OIDC 認証を受け入れる設定               |
| **IAM ロール**       | 特定のリポジトリ・ブランチからのみアクセス許可                 |
| **許可ポリシー**     | ECR へのプッシュ、ECS タスク定義の更新権限を付与               |
| **GitHub Secrets**   | ワークフローからロール ARN を安全に参照できるように設定        |

#### 12.9.2 GitHub Actions から AWS へのアクセスフロー

```
┌─────────────────────────────────────────────┐
│ GitHub Actions (cicd.yml)                  │
│ - main ブランチへのプッシュをトリガー       │
└────────────────┬────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────┐
│ AWS STS (Security Token Service)            │
│ - IDトークンを検証                          │
│ - リポジトリ、ブランチなどの条件を確認      │
└────────────────┬────────────────────────────┘
                 │ 条件を満たす場合
                 ▼
┌─────────────────────────────────────────────┐
│ 一時的なアクセストークンを発行              │
│ - 有効期限: 短時間(例: 1時間)               │
│ - 権限: UpdateECSTask の範囲内        │
└────────────────┬────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────┐
│ AWS リソースへのアクセス                    │
│ - ECR: イメージのプッシュ                   │
│ - ECS: タスク定義の更新、サービスの更新     │
└─────────────────────────────────────────────┘
```

#### 12.9.3 次のステップ

これで、GitHub Actions から AWS にアクセスするための設定がすべて完了しました。

次のセクションでは:

- ワークフローファイルの最終調整
- GitHub へのプッシュ
- パイプラインの実行確認
- ECR へのイメージプッシュの確認

を行います。

---

## 13. ワークフローファイルの最終調整とパイプライン実行

### 13.1 概要

ワークフローファイル(`cicd.yml`)を最終調整し、GitHub にプッシュしてパイプラインを実行します。

### 13.2 不要なジョブの削除

#### 13.2.1 ファイルを開く

1. VS Code で `.github/workflows/cicd.yml` を開きます

#### 13.2.2 不要なジョブを削除

以前のテスト用に作成した以下のジョブが残っている場合は削除します:

- `hello-job`
- `goodbye-job`

**削除対象の例:**

```yaml
hello-job:
  runs-on: ubuntu-latest
  steps:
    - run: echo Hello
    - run: echo "GitHub Actions!"

goodbye-job:
  runs-on: ubuntu-latest
  steps:
    - run: echo "Goodbye GitHub Actions!"
```

これらのジョブ定義を削除してください。

> **注意:** `deploy` ジョブに関するコメントが残っている場合は、そのまま残しておいてください。後ほどデプロイ機能を追加します。

#### 13.2.3 ファイルを保存

1. `Ctrl + S` でファイルを保存します

---

### 13.3 トリガー条件の修正

#### 13.3.1 問題点の確認

現在の `on` セクションは以下のようになっています:

```yaml
on:
  push:
    paths:
      - "cicd-section/api/**"
```

**この設定の問題点:**

- `cicd-section/api` ディレクトリ配下のファイルが変更された場合のみワークフローが実行される
- しかし、`cicd.yml` ファイル自体は `.github/workflows/` ディレクトリに存在する
- **結果:** `cicd.yml` ファイルを変更してプッシュしても、ワークフローが実行されない

#### 13.3.2 解決方法

`cicd.yml` ファイル自体の変更もトリガーとして認識されるよう、パスの条件を追加します。

#### 13.3.3 修正内容

`on` セクションを以下のように修正します:

**修正前:**

```yaml
on:
  push:
    paths:
      - "cicd-section/api/**"
```

**修正後:**

```yaml
on:
  push:
    paths:
      - "cicd-section/api/**"
      - ".github/workflows/**"
```

**追加した条件の説明:**

- `.github/workflows/**`: `.github/workflows` ディレクトリ配下のすべてのファイル
- `**` はワイルドカードで、サブディレクトリを含むすべてのファイルを対象
- この条件により、`cicd.yml` ファイル自体の変更もトリガーとして認識される

#### 13.3.4 ファイルを保存

1. `Ctrl + S` でファイルを保存します

---

### 13.4 GitHub へのプッシュ

#### 13.4.1 ターミナルを開く

1. VS Code でターミナルを開きます
   - メニュー: **Terminal → New Terminal**
   - または `Ctrl + `(バッククォート)`

#### 13.4.2 リポジトリのルートディレクトリに移動

```bash
cd /path/to/ecs-learning-course
```

※ご自身の環境に合わせてパスを変更してください

#### 13.4.3 変更をステージング

```bash
git add .
```

- すべての変更ファイルをステージングエリアに追加します

#### 13.4.4 コミットの作成

```bash
git commit -m "Update cicd.yml"
```

**出力例:**

```
[main xxxxxxxx] Update cicd.yml
 1 file changed, 5 insertions(+), 20 deletions(-)
```

#### 13.4.5 リモートリポジトリへプッシュ

```bash
git push origin main
```

**出力例:**

```
Enumerating objects: 7, done.
Counting objects: 100% (7/7), done.
Delta compression using up to 8 threads
Compressing objects: 100% (4/4), done.
Writing objects: 100% (4/4), 456 bytes | 456.00 KiB/s, done.
Total 4 (delta 3), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (3/3), completed with 3 local objects.
To https://github.com/yourname/ecs-learning-course.git
   xxxxxxxx..yyyyyyyy  main -> main
```

プッシュが完了すると、GitHub Actions が自動的にワークフローを実行開始します。

---

### 13.5 GitHub Actions の実行確認

#### 13.5.1 Actions ページへアクセス

1. GitHub のリポジトリページにアクセスします
   ```
   https://github.com/<yourname>/ecs-learning-course
   ```

2. ページ上部のタブメニューから「**Actions**」をクリックします

#### 13.5.2 ワークフローの実行状態を確認

**ワークフロー一覧:**

- 最新のワークフロー実行として「**Update cicd.yml**」(先ほどのコミットメッセージ)が表示されます

**ステータスの種類:**

| アイコン | 色   | 状態             | 説明                         |
| -------- | ---- | ---------------- | ---------------------------- |
| ⏳       | 黄色 | 実行中(In progress) | ワークフローが現在実行中     |
| ✓        | 緑色 | 成功(Success)    | すべてのジョブが正常に完了   |
| ✗        | 赤色 | 失敗(Failure)    | いずれかのジョブが失敗       |

#### 13.5.3 ワークフローの詳細を確認

1. 「**Update cicd.yml**」のワークフロー実行をクリックします

2. ワークフローの詳細ページが表示されます

**ページ構成:**

- **左側:** ジョブ一覧
  - `test-and-build` ジョブが表示されます
- **右側:** ログ表示領域

#### 13.5.4 ジョブの実行ログを確認

1. 左側のジョブ一覧から「**test-and-build**」をクリックします

2. ジョブ内のステップが展開されて表示されます

**表示されるステップ:**

```
✓ Set up job
✓ Checkout code
✓ Test and Build Image
✓ Configure AWS Credentials
✓ Login to Amazon ECR
✓ Push Image to ECR
✓ Complete job
```

3. 各ステップをクリックすると、詳細なログを確認できます

**確認ポイント:**

- すべてのステップが緑色のチェックマーク(✓)になっていることを確認
- 特に「**Push Image to ECR**」ステップが成功していることを確認

#### 13.5.5 エラーが発生した場合

**赤色の✗マークが表示された場合:**

1. 失敗したステップをクリックして詳細ログを確認
2. エラーメッセージを読んで原因を特定
3. 設定の見直しや修正を行う

**よくあるエラー:**

| エラー内容                     | 原因                                       | 対処法                                |
| ------------------------------ | ------------------------------------------ | ------------------------------------- |
| `Error: Could not assume role` | IAM ロールの ARN が正しく設定されていない  | GitHub Secrets の `AWS_ROLE_TO_ASSUME` を確認 |
| `Error: ECR login failed`      | ECR へのログイン権限がない                 | IAM ポリシーで ECR 権限を確認         |
| `Error: Cannot push to ECR`    | ECR リポジトリが存在しない、または権限不足 | ECR リポジトリの存在と権限を確認      |

---

### 13.6 ECR でイメージの確認

#### 13.6.1 ECR コンソールへアクセス

1. AWS マネジメントコンソールで「**ECR**」を検索します
2. 「Elastic Container Registry」を選択します

#### 13.6.2 プライベートリポジトリを開く

1. 左側のメニューで「**プライベートリポジトリ**」をクリックします
2. リポジトリ一覧から「**my-app-api**」をクリックします

#### 13.6.3 イメージの確認

**イメージ一覧:**

プライベートリポジトリの詳細ページで、「**イメージ**」タブを開くと、以下のような情報が表示されます:

| イメージタグ                     | プッシュ日時             | イメージサイズ |
| -------------------------------- | ------------------------ | -------------- |
| `5364a8c...`(長いハッシュ値) | 2025-12-13 10:30:45 JST | 524 MB         |

**確認ポイント:**

- 新しいイメージが追加されていることを確認
- 以前は空だったリポジトリにイメージが存在する
- これが GitHub Actions パイプラインによって自動的にプッシュされたイメージです

#### 13.6.4 イメージタグの意味

**イメージタグの値:**

```
5364a8c1b2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7
```

この値は **Git のコミットハッシュ(SHA)** です。

**タグに SHA を使用するメリット:**

| メリット               | 説明                                                    |
| ---------------------- | ------------------------------------------------------- |
| **一意性の保証**       | 各コミットで必ず異なるハッシュが生成される              |
| **トレーサビリティ**   | どのコミットからビルドされたイメージか特定可能          |
| **デプロイの追跡**     | 本番環境で動作しているコードのバージョンを正確に把握    |
| **ロールバックの容易性** | 特定のコミット時点のイメージに簡単に戻すことができる    |

---

### 13.7 イメージタグと Git SHA の照合

#### 13.7.1 GitHub のコミット履歴を確認

1. GitHub のリポジトリページに戻ります
2. ページ上部のタブメニューから「**Code**」をクリックします
3. コミット数が表示されている部分をクリックします
   - 例: 「**5 commits**」のようなリンク

#### 13.7.2 コミット履歴ページ

コミット履歴ページが表示されます。

**表示内容:**

```
Update cicd.yml
5364a8c · 10 minutes ago · yourname
─────────────────────────────────────

Add GitHub Actions CI/CD pipeline
a1b2c3d · 2 hours ago · yourname
─────────────────────────────────────

Initial commit
e1f2g3h · 1 day ago · yourname
```

#### 13.7.3 コミット SHA の確認

最新のコミット「**Update cicd.yml**」の右側を見ると:

```
5364a8c
```

このように短縮された SHA が表示されています。

#### 13.7.4 SHA の照合

**ECR のイメージタグ(フル):**

```
5364a8c1b2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7
```

**GitHub の SHA(短縮版):**

```
5364a8c
```

**照合結果:**

- ECR のイメージタグの **先頭部分** が GitHub の SHA と一致しています
- ECR には **フル SHA**(40 文字)が使用されています
- GitHub では **短縮 SHA**(7 文字)が表示されています

> **補足:**
>
> - Git の SHA ハッシュは本来 40 文字の 16 進数文字列です
> - GitHub の UI では、見やすさのために最初の 7 文字のみを表示します
> - しかし、どちらも同じコミットを指しています

#### 13.7.5 フル SHA の確認方法

**GitHub でフル SHA を確認する方法:**

1. コミット履歴ページで、コミットメッセージをクリックします
2. コミット詳細ページが開きます
3. URL を確認します:

   ```
   https://github.com/yourname/ecs-learning-course/commit/5364a8c1b2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7
   ```

4. URL の最後の部分がフル SHA です

**または、ページ内の表示を確認:**

- コミット詳細ページの右側に、フル SHA がコピー可能な形式で表示されています

---

### 13.8 動作確認のまとめ

#### 13.8.1 確認できたこと

| 項目                           | 確認内容                                       | 結果 |
| ------------------------------ | ---------------------------------------------- | ---- |
| **GitHub Actions の実行**      | ワークフローが正常に完了                       | ✓    |
| **AWS 認証**                   | OIDC を使用した AWS への認証が成功             | ✓    |
| **Docker イメージのビルド**    | Spring Boot アプリケーションのイメージが作成   | ✓    |
| **ECR へのプッシュ**           | イメージが ECR にアップロードされた            | ✓    |
| **イメージタグの設定**         | Git の SHA がタグとして正しく設定された        | ✓    |

#### 13.8.2 パイプラインの全体フロー

```
┌──────────────────────────────────────┐
│ 1. 開発者がコードをプッシュ          │
│    (git push origin main)            │
└────────────────┬─────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────┐
│ 2. GitHub Actions がトリガー         │
│    (main ブランチへのプッシュ検知)   │
└────────────────┬─────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────┐
│ 3. ソースコードのチェックアウト      │
└────────────────┬─────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────┐
│ 4. Docker イメージのビルド           │
│    (テストも同時に実行)              │
└────────────────┬─────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────┐
│ 5. AWS への認証(OIDC)                │
└────────────────┬─────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────┐
│ 6. ECR へのログイン                  │
└────────────────┬─────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────┐
│ 7. イメージのタグ付け                │
│    (タグ = Git SHA)                  │
└────────────────┬─────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────┐
│ 8. ECR へイメージをプッシュ          │
└──────────────────────────────────────┘
```

#### 13.8.3 現在の状態

**完了したこと:**

- ✓ CI(継続的インテグレーション)パイプラインの構築
- ✓ テストとビルドの自動化
- ✓ ECR へのイメージプッシュの自動化

**次のステップ:**

- CD(継続的デプロイメント)パイプラインの構築
- ECS サービスへの自動デプロイ
- タスク定義の自動更新

次のセクションでは、ECR にプッシュされたイメージを使用して、ECS サービスに自動的にデプロイする仕組みを構築します。

---

## 14. ECS タスク定義の更新とデプロイパイプラインの構築

### 14.1 概要

ECR にプッシュされたイメージを使用して、ECS タスク定義を更新し、サービスにデプロイする CD(継続的デプロイメント)パイプラインを構築します。

### 14.2 ECS タスク更新の仕組み

#### 14.2.1 現在の状態

**現在動作しているコンテナ:**

- ECS サービス: `my-app-api-service`
- タスク定義: `my-app-api` (リビジョン 1)
- コンテナ: NGINX(ダミーコンテナ)

#### 14.2.2 目標とする状態

**デプロイ後の状態:**

- ECS サービス: `my-app-api-service`
- タスク定義: `my-app-api` (リビジョン 2)
- コンテナ: Spring Boot WEB API

#### 14.2.3 タスク定義の更新プロセス

ECS のアプリケーション更新は、**タスク定義を更新してデプロイする**ことを意味します。

**更新の流れ:**

```
┌────────────────────────────────────┐
│ 1. 現在のタスク定義               │
│    my-app-api:1 (NGINX)            │
└────────────────┬───────────────────┘
                 │
                 ▼
┌────────────────────────────────────┐
│ 2. タスク定義を更新                │
│    - JSONファイルを編集            │
│    - イメージURIを書き換え         │
└────────────────┬───────────────────┘
                 │
                 ▼
┌────────────────────────────────────┐
│ 3. 新しいタスク定義を登録          │
│    my-app-api:2 (Spring Boot)      │
└────────────────┬───────────────────┘
                 │
                 ▼
┌────────────────────────────────────┐
│ 4. ECSサービスをデプロイ           │
│    新しいタスク定義を使用          │
└────────────────────────────────────┘
```

#### 14.2.4 タスク定義 JSON の構造

タスク定義は JSON 形式で記述されます:

```json
{
  "family": "my-app-api",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "123456789012.dkr.ecr.us-west-2.amazonaws.com/my-app-api:abc123...",
      "cpu": 256,
      "memory": 512,
      ...
    }
  ],
  ...
}
```

**重要なポイント:**

- タスク定義のほとんどの属性は変更不要
- **変更が必要な部分:** `image` フィールドのみ
  - このフィールドに ECR のイメージ URI を指定
  - コミットごとに異なるイメージタグ(SHA)を使用

**デプロイの戦略:**

1. タスク定義の**テンプレート JSON ファイル**を用意
2. GitHub Actions 内で `image` フィールドだけを動的に書き換え
3. 新しいタスク定義として登録
4. ECS サービスにデプロイ

---

### 14.3 deploy ジョブの作成

#### 14.3.1 ワークフローファイルを開く

1. VS Code で `.github/workflows/ci-cd.yml` を開きます

#### 14.3.2 deploy ジョブの追加

`test-and-build` ジョブの後に、`deploy` ジョブを追加します。

**追加する内容:**

```yaml
  deploy:
    runs-on: ubuntu-latest
    needs: [test-and-build]

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: ${{ env.AWS_REGION }}
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
```

#### 14.3.3 設定内容の説明

**ジョブの基本設定:**

```yaml
deploy:
  runs-on: ubuntu-latest
  needs: [test-and-build]
```

| 項目                | 説明                                                   |
| ------------------- | ------------------------------------------------------ |
| `deploy`            | ジョブ名                                               |
| `runs-on`           | Ubuntu の最新版を使用                                  |
| `needs`             | `test-and-build` ジョブの完了を待つ                    |

**needs フィールドの役割:**

- ジョブはデフォルトで並列実行される
- `needs` を指定することで、ジョブの実行順序を制御できる
- `test-and-build` が完了してから `deploy` を実行

**複数のジョブに依存させる例:**

```yaml
needs:
  - test-and-build
  - security-scan
```

このように記述すると、すべてのジョブが完了してから実行されます。

**ステップの説明:**

1. **Checkout code**: ソースコードをチェックアウト
2. **Configure AWS Credentials**: AWS への認証
   - デプロイジョブでも AWS リソース(ECS、タスク定義)にアクセスするため必要
   - 環境変数とシークレットは既に定義済みなのでそのまま使用可能

---

### 14.4 ジョブ間でのデータ共有(アーティファクト)

#### 14.4.1 問題の確認

**課題:**

- `deploy` ジョブでは、ECR にプッシュしたイメージの URI が必要
- イメージ URI の形式: `<ECRレジストリ>/<リポジトリ名>:<コミットSHA>`
- しかし、`ECR_REGISTRY` の値は `test-and-build` ジョブ内でのみ定義されている

**ジョブの独立性:**

- 各ジョブは異なるランナー(仮想マシン)で実行される
- ジョブ間でデータや変数を直接共有することはできない

#### 14.4.2 解決方法: アーティファクトの使用

**アーティファクト(Artifact)とは:**

- GitHub Actions が提供する特別なストレージ領域
- ジョブの実行結果(ファイル)を保存・共有できる
- ジョブをまたいでデータをやり取りする仕組み

**データ共有の流れ:**

```
┌──────────────────────────┐
│ test-and-build ジョブ    │
│                          │
│ 1. イメージURIを取得     │
│ 2. テキストファイルに保存│
│ 3. アーティファクトに     │
│    アップロード          │
└────────────┬─────────────┘
             │
             ▼
      ┌─────────────┐
      │ Artifact    │
      │ (Storage)   │
      └─────────────┘
             │
             ▼
┌──────────────────────────┐
│ deploy ジョブ            │
│                          │
│ 1. アーティファクトから  │
│    ダウンロード          │
│ 2. テキストファイルを読取│
│ 3. 環境変数に設定        │
└──────────────────────────┘
```

---

### 14.5 イメージ URI のアーティファクトへのアップロード

#### 14.5.1 test-and-build ジョブにステップを追加

`test-and-build` ジョブの最後(「Push Image to ECR」ステップの後)に、以下のステップを追加します:

**ステップ1: イメージ URI をファイルに保存**

```yaml
      - name: Push the image to Amazon ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        run: |
          docker image tag temp_api_image:latest $ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
          docker image push $ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
          echo $ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:${{ github.sha }} > api-image-uri.txt

      - name: Upload the image uri file as an artifact
        uses: actions/upload-artifact@v2
        with:
          name: api-image-uri
          path: cicd-section/api/api-image-uri.txt
```

#### 14.5.2 詳細説明

**イメージ URI をファイルに保存:**

```bash
echo $ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:${{ github.sha }} > api-image-uri.txt
```

**コマンドの説明:**

| 部分                  | 説明                                                 |
| --------------------- | ---------------------------------------------------- |
| `echo`                | 指定した文字列を出力するコマンド                     |
| `$ECR_REGISTRY/...`   | 出力する文字列(イメージの完全な URI)                 |
| `>`                   | リダイレクト演算子(出力をファイルに書き込む)         |
| `api-image-uri.txt`   | 作成するテキストファイルの名前                       |

**例:**

実行結果として、`api-image-uri.txt` ファイルに以下の内容が保存されます:

```
123456789012.dkr.ecr.us-west-2.amazonaws.com/my-app-api:5364a8c1b2e3...
```

> **Linux コマンドの補足:**
>
> - `echo $PATH > path.txt`: PATH 環境変数の内容を path.txt に保存
> - `>` 演算子は、コマンドの出力をファイルにリダイレクト(書き込み)する

**アーティファクトへのアップロード:**

```yaml
- name: Upload the image uri file as an artifact
  uses: actions/upload-artifact@v2
  with:
    name: api-image-uri
    path: cicd-section/api/api-image-uri.txt
```

**パラメータの説明:**

| パラメータ | 値                              | 説明                                       |
| ---------- | ------------------------------- | ------------------------------------------ |
| `uses`     | `actions/upload-artifact@v2`    | GitHub 公式のアーティファクトアップロード  |
| `name`     | `api-image-uri`                 | アーティファクトの識別名(任意に命名可能)   |
| `path`     | `cicd-section/api/api-image-uri.txt` | アップロードするファイルのパス        |

> **重要な注意事項:**
>
> `path` には**絶対パス**または**リポジトリルートからの相対パス**を指定する必要があります。
>
> - `defaults.run.working-directory` の設定は `uses` アクションには適用されません
> - `run` コマンドでは `working-directory` が有効
> - `uses` アクションでは無効
>
> したがって、`cicd-section/api/` を明示的に含める必要があります。

---

### 14.6 イメージ URI のアーティファクトからのダウンロード

#### 14.6.1 deploy ジョブにステップを追加

AWS 認証の後に、以下のステップを追加します:

```yaml
      - name: Download the artifact
        uses: actions/download-artifact@v2
        with:
          name: api-image-uri
          path: artifacts

      - name: Define the image URI
        run: |
          echo "API_IMAGE_URI=$(cat artifacts/api-image-uri.txt)" >> $GITHUB_ENV
```

#### 14.6.2 詳細説明

**アーティファクトのダウンロード:**

```yaml
- name: Download the artifact
  uses: actions/download-artifact@v2
  with:
    name: api-image-uri
    path: artifacts
```

**パラメータの説明:**

| パラメータ | 値                           | 説明                                     |
| ---------- | ---------------------------- | ---------------------------------------- |
| `uses`     | `actions/download-artifact@v2` | GitHub 公式のダウンロードアクション      |
| `name`     | `api-image-uri`              | アップロード時に指定した識別名と同じ     |
| `path`     | `artifacts`                  | ダウンロード先のディレクトリ             |

**結果:**

- `artifacts/api-image-uri.txt` にイメージ URI が保存されます

**環境変数への設定:**

```bash
echo "API_IMAGE_URI=$(cat artifacts/api-image-uri.txt)" >> $GITHUB_ENV
```

**コマンドの分解:**

1. **`cat artifacts/api-image-uri.txt`**
   - `cat` コマンドでファイルの内容を読み取り
   - ファイルの中身: `123456789012.dkr.ecr.us-west-2.amazonaws.com/my-app-api:5364a8c...`

2. **`$(...)` - コマンド置換**
   - `$()` 内のコマンドを実行し、その出力を文字列として展開
   - 結果: ファイルの中身が文字列として使用される

3. **`echo "API_IMAGE_URI=..."`**
   - 以下の形式の文字列を出力:
     ```
     API_IMAGE_URI=123456789012.dkr.ecr.us-west-2.amazonaws.com/my-app-api:5364a8c...
     ```

4. **`>> $GITHUB_ENV`**
   - `$GITHUB_ENV` は GitHub Actions の特殊な環境変数
   - この環境変数にリダイレクトすると、後続のステップで環境変数として使用可能になる

**結果:**

以降のステップで `${{ env.API_IMAGE_URI }}` としてイメージ URI にアクセスできるようになります。

---

### 14.7 タスク定義の更新

#### 14.7.1 タスク定義のレンダリング

イメージ URI を取得したので、タスク定義 JSON の `image` フィールドを更新します。

**deploy ジョブに以下のステップを追加:**

```yaml
      - name: Fill in the new image URI in the amazon ECS task definition
        id: render-task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: ${{ env.ECS_TASK_DEFINITION_API }}
          container-name: api
          image: ${{ env.API_IMAGE_URI }}
```

#### 14.7.2 詳細説明

**パラメータの説明:**

| パラメータ        | 値                                   | 説明                                     |
| ----------------- | ------------------------------------ | ---------------------------------------- |
| `id`              | `render-task-def`                    | このステップの ID(後続ステップで参照)    |
| `uses`            | `aws-actions/amazon-ecs-render-task-definition@v1` | AWS 公式のタスク定義レンダリングアクション |
| `task-definition` | `${{ env.ECS_TASK_DEFINITION_API }}` | タスク定義テンプレート JSON のパス       |
| `container-name`  | `api`                                | 更新対象のコンテナ名                     |
| `image`           | `${{ env.API_IMAGE_URI }}`           | 新しいイメージ URI                       |

**環境変数の確認:**

`ECS_TASK_DEFINITION_API` は、ワークフローファイルの冒頭で定義済み:

```yaml
env:
  ECS_TASK_DEFINITION_API: cicd-section/.aws/task-definition.json
```

**このステップの動作:**

1. `task-definition` で指定された JSON ファイルを読み込む
2. `container-name` で指定されたコンテナの `image` フィールドを検索
3. `image` パラメータで指定された URI に書き換え
4. 新しいタスク定義 JSON を出力(このステップの outputs に保存)

**id の使用目的:**

- 次のステップでこのステップの出力(更新されたタスク定義 JSON)を参照するため
- `steps.render-task-def.outputs.task-definition` として参照可能

---

### 14.8 ECS サービスへのデプロイ

#### 14.8.1 デプロイステップの追加

最後に、更新したタスク定義を使用して ECS サービスにデプロイします。

**deploy ジョブに以下のステップを追加:**

```yaml
      - name: Deploy ECS task
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.render-task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
```

#### 14.8.2 詳細説明

**パラメータの説明:**

| パラメータ                   | 値                                            | 説明                                         |
| ---------------------------- | --------------------------------------------- | -------------------------------------------- |
| `uses`                       | `aws-actions/amazon-ecs-deploy-task-definition@v1` | AWS 公式の ECS デプロイアクション            |
| `task-definition`            | `${{ steps.render-task-def.outputs.task-definition }}` | 前ステップで作成した新しいタスク定義         |
| `service`                    | `${{ env.ECS_SERVICE }}`                      | デプロイ対象の ECS サービス名                |
| `cluster`                    | `${{ env.ECS_CLUSTER }}`                      | ECS クラスター名                             |
| `wait-for-service-stability` | `true`                                        | デプロイ完了まで待機するかどうか             |

**前ステップの出力の参照:**

```yaml
task-definition: ${{ steps.render-task-def.outputs.task-definition }}
```

- `steps`: 現在のジョブ内のステップを参照
- `render-task-def`: 前ステップの ID
- `outputs`: ステップの出力データ
- `task-definition`: 出力データのキー(更新されたタスク定義 JSON)

> **outputs の詳細:**
>
> 各アクションがどのような outputs を提供しているかは、そのアクションの公式ドキュメントで確認できます。
>
> - GitHub Marketplace でアクションを検索
> - README の「Outputs」セクションを確認

**wait-for-service-stability の役割:**

| 値      | 動作                                                   |
| ------- | ------------------------------------------------------ |
| `true`  | デプロイが完了し、サービスが安定するまで待機           |
| `false` | デプロイをトリガーしてすぐに次のステップへ進む(非推奨) |

**推奨設定:** `true`

- デプロイが成功したことを確認してからパイプラインを完了
- エラーが発生した場合、パイプラインが失敗として報告される

---

### 14.9 ワークフローファイルの完成形

#### 14.9.1 deploy ジョブ全体

これまでの内容をまとめた `deploy` ジョブの完成形:

```yaml
  deploy:
    runs-on: ubuntu-latest
    needs: [test-and-build]

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: ${{ env.AWS_REGION }}
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}

      - name: Download the artifact
        uses: actions/download-artifact@v2
        with:
          name: api-image-uri
          path: artifacts

      - name: Define the image URI
        run: |
          echo "API_IMAGE_URI=$(cat artifacts/api-image-uri.txt)" >> $GITHUB_ENV

      - name: Fill in the new image URI in the amazon ECS task definition
        id: render-task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: ${{ env.ECS_TASK_DEFINITION_API }}
          container-name: api
          image: ${{ env.API_IMAGE_URI }}

      - name: Deploy ECS task
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.render-task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
```

#### 14.9.2 ワークフロー全体の流れ

```
┌────────────────────────────────────┐
│ test-and-build ジョブ              │
├────────────────────────────────────┤
│ 1. コードチェックアウト            │
│ 2. テスト & イメージビルド         │
│ 3. AWS 認証                        │
│ 4. ECR ログイン                    │
│ 5. イメージをECRにプッシュ         │
│ 6. イメージURIをファイルに保存     │
│ 7. アーティファクトにアップロード  │
└────────────────┬───────────────────┘
                 │ needs
                 ▼
┌────────────────────────────────────┐
│ deploy ジョブ                      │
├────────────────────────────────────┤
│ 1. コードチェックアウト            │
│ 2. AWS 認証                        │
│ 3. アーティファクトをダウンロード  │
│ 4. イメージURIを環境変数に設定     │
│ 5. タスク定義のレンダリング        │
│ 6. ECS サービスにデプロイ          │
└────────────────────────────────────┘
```

#### 14.9.3 ファイルの保存

1. `ci-cd.yml` ファイルを保存します(`Ctrl + S`)

---

### 14.10 次のステップ

これで CD パイプラインのワークフロー定義が完了しました。

**次に必要な作業:**

- タスク定義テンプレート JSON ファイルの作成
- ワークフローで参照されている `cicd-section/.aws/task-def-api.json` を作成する必要があります

次のセクションでは、タスク定義テンプレートファイルを作成します。

---

## 15. タスク定義テンプレート JSON ファイルの作成

### 15.1 概要

GitHub Actions ワークフローで参照するタスク定義のテンプレート JSON ファイルを作成します。このファイルは、デプロイ時にイメージ URI が動的に書き換えられるベースとなります。

### 15.2 ディレクトリとファイルの作成

#### 15.2.1 ターミナルを開く

1. VS Code でターミナルを開きます
2. リポジトリのルートディレクトリ(`ecs-learning-course`)にいることを確認します

```bash
pwd
```

**出力例:**

```
/Users/yourname/Desktop/ecs-learning-course
```

#### 15.2.2 ディレクトリの作成

`.aws` ディレクトリを作成します:

```bash
mkdir -p cicd-section/.aws
```

**コマンドの説明:**

| 部分                   | 説明                                          |
| ---------------------- | --------------------------------------------- |
| `mkdir`                | ディレクトリを作成するコマンド                |
| `-p`                   | 親ディレクトリも含めて作成(存在しない場合)    |
| `cicd-section/.aws`    | 作成するディレクトリのパス                    |

> **補足:**
>
> - ディレクトリ名やネーミングルールに特別な制限はありません
> - ワークフローファイルで指定したパスと一致していれば問題ありません

#### 15.2.3 JSON ファイルの作成

空の JSON ファイルを作成します:

```bash
touch cicd-section/.aws/task-def-api.json
```

**確認:**

VS Code のエクスプローラーで、以下のディレクトリ構造が作成されていることを確認します:

```
ecs-learning-course/
├── cicd-section/
│   ├── .aws/
│   │   └── task-def-api.json  ← 新規作成
│   └── api/
│       └── ...
└── .github/
    └── workflows/
        └── cicd.yml
```

---

### 15.3 AWS コンソールからタスク定義のエクスポート

#### 15.3.1 ECS コンソールへアクセス

1. AWS マネジメントコンソールで「**ECS**」を検索します
2. 左側のメニューから「**タスク定義**」をクリックします

#### 15.3.2 タスク定義の選択

1. タスク定義ファミリー一覧から「**my-app-api**」をクリックします

2. リビジョン一覧が表示されます
   - 現時点では「**my-app-api:1**」のみが存在するはずです
   - リビジョン 1 をクリックします

#### 15.3.3 JSON のコピー

1. タスク定義の詳細ページで「**JSON**」タブをクリックします

2. JSON 内容が表示されます

3. 「**クリップボードにコピー**」ボタンをクリックします

---

### 15.4 JSON ファイルの編集

#### 15.4.1 VS Code でファイルを開く

1. VS Code で `cicd-section/.aws/task-def-api.json` を開きます

2. コピーした JSON を貼り付けます(`Ctrl + V`)

#### 15.4.2 不要なフィールドの削除と値の調整

AWS からエクスポートした JSON には、デプロイ時に不要なフィールドが含まれています。以下の手順で編集します。

**1. taskDefinitionArn を削除**

```json
"taskDefinitionArn": "arn:aws:ecs:us-west-2:123456789012:task-definition/my-app-api:1",
```

この行を削除します。

**理由:** 新しいタスク定義を登録する際、ARN は自動的に生成されるため不要

---

**2. コンテナ名を変更**

`containerDefinitions` 配列内の `name` フィールドを確認します:

```json
"containerDefinitions": [
  {
    "name": "dummy",  ← これを変更
    ...
  }
]
```

**変更前:**

```json
"name": "dummy",
```

**変更後:**

```json
"name": "api",
```

**理由:** ワークフローで `container-name: api` と指定しているため一致させる必要がある

---

**3. イメージ URI をプレースホルダーに変更**

`image` フィールドの値を一時的なプレースホルダーに変更します:

```json
"image": "public.ecr.aws/nginx/nginx:stable-perl",
```

**変更後:**

```json
"image": "<IMAGE_URI>",
```

**理由:** このフィールドは GitHub Actions で動的に書き換えられるため、一時的な値で問題ありません

---

**4. ポートマッピングの設定**

`portMappings` セクションを確認・修正します:

**変更前:**

```json
"portMappings": [
  {
    "name": "dummy-80-tcp",
    "containerPort": 80,
    "hostPort": 80,
    "protocol": "tcp",
    "appProtocol": "http"
  }
]
```

**変更後:**

```json
"portMappings": [
  {
    "name": "api-8080-tcp",
    "containerPort": 8080,
    "hostPort": 8080,
    "protocol": "tcp",
    "appProtocol": "http"
  }
]
```

**変更内容:**

| フィールド      | 変更前      | 変更後         | 理由                                   |
| --------------- | ----------- | -------------- | -------------------------------------- |
| `name`          | dummy-80-tcp | api-8080-tcp   | コンテナ名に合わせて変更               |
| `containerPort` | 80          | 8080           | Spring Boot はデフォルトで 8080 を使用 |
| `hostPort`      | 80          | 8080           | コンテナポートと一致させる             |

---

**5. taskRoleArn の削除(オプション)**

35行目付近の `taskRoleArn` フィールドを削除します:

```json
"taskRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
```

**理由:**
- AWS アカウント ID が含まれているため、GitHub に公開したくない
- タスク実行ロールは `executionRoleArn` で指定されているため、こちらは不要

---

**6. revision フィールドの削除**

```json
"revision": 1,
```

この行を削除します。

**理由:** 新しいタスク定義を登録する際、リビジョン番号は自動的に採番されるため不要

---

**7. requiresAttributes の削除**

`requiresAttributes` 配列全体を削除します:

```json
"requiresAttributes": [
  {
    "name": "com.amazonaws.ecs.capability.logging-driver.awslogs"
  },
  ...
],
```

**理由:** デプロイ時に不要な情報

---

**8. status フィールドの削除**

```json
"status": "ACTIVE",
```

この行を削除します。

---

**9. compatibilities の削除**

```json
"compatibilities": [
  "EC2",
  "FARGATE"
],
```

この配列を削除します。

---

**10. CPU とメモリの確認**

`cpu` と `memory` フィールドの値を確認します:

```json
"cpu": "512",
"memory": "1024",
```

**これらの値はそのままでも OK です。必要に応じて変更してください:**

| リソース | 値     | 説明                              |
| -------- | ------ | --------------------------------- |
| `cpu`    | "512"  | 0.5 vCPU(1024 = 1 vCPU)           |
| `memory` | "1024" | 1 GB                              |

> **変更例:**
>
> - より多くのリソースが必要な場合: `"cpu": "1024"`, `"memory": "2048"`
> - リソースを削減したい場合: `"cpu": "256"`, `"memory": "512"`

---

**11. runtimePlatform の確認**

```json
"runtimePlatform": {
  "cpuArchitecture": "X86_64",
  "operatingSystemFamily": "LINUX"
},
```

**このままでも OK です。ARM を使用する場合:**

```json
"cpuArchitecture": "ARM64"
```

に変更してください。

---

**12. registeredAt と registeredBy の削除**

```json
"registeredAt": "2025-12-13T01:23:45.678Z",
"registeredBy": "arn:aws:iam::123456789012:user/admin",
```

これらの行を削除します。

---

**13. tags の削除**

`tags` 配列がある場合は削除します:

```json
"tags": []
```

---

**14. 最終行のカンマを削除**

JSON の最終行に不要なカンマ(`,`)がないことを確認します:

```json
  "memory": "1024"
}  ← カンマがないことを確認
```

---

#### 15.4.3 編集後の JSON の全体像

編集後、JSON は以下のような構造になるはずです:

```json
{
  "family": "my-app-api",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "<IMAGE_URI>",
      "cpu": 0,
      "portMappings": [
        {
          "name": "api-8080-tcp",
          "containerPort": 8080,
          "hostPort": 8080,
          "protocol": "tcp",
          "appProtocol": "http"
        }
      ],
      "essential": true,
      "environment": [],
      "environmentFiles": [],
      "mountPoints": [],
      "volumesFrom": [],
      "ulimits": [],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-app-api",
          "mode": "non-blocking",
          "awslogs-create-group": "true",
          "max-buffer-size": "25m",
          "awslogs-region": "us-west-2",
          "awslogs-stream-prefix": "ecs"
        },
        "secretOptions": []
      },
      "systemControls": []
    }
  ],
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "networkMode": "awsvpc",
  "placementConstraints": [],
  "cpu": "512",
  "memory": "1024",
  "runtimePlatform": {
    "cpuArchitecture": "X86_64",
    "operatingSystemFamily": "LINUX"
  }
}
```

**行数の確認:**

- 全体で約 48 行程度になるはずです
- 行数が大きく異なる場合、削除漏れがないか確認してください

#### 15.4.4 ファイルの保存

1. `Ctrl + S` でファイルを保存します

---

### 15.5 GitHub へのプッシュとパイプラインの実行

#### 15.5.1 変更内容の確認

```bash
git status
```

**出力例:**

```
On branch main
Changes not staged for commit:
  modified:   .github/workflows/cicd.yml

Untracked files:
  cicd-section/.aws/
```

- `cicd.yml`: 編集済み
- `cicd-section/.aws/`: 新規作成(未追跡)

#### 15.5.2 変更をステージング

```bash
git add .
```

すべての変更をステージングエリアに追加します。

#### 15.5.3 ステージング状態の確認

```bash
git status
```

**出力例:**

```
On branch main
Changes to be committed:
  modified:   .github/workflows/cicd.yml
  new file:   cicd-section/.aws/task-def-api.json
```

#### 15.5.4 コミットの作成

```bash
git commit -m "Add ECS deploy workflow and task definition template"
```

**出力例:**

```
[main xxxxxxxx] Add ECS deploy workflow and task definition template
 2 files changed, 150 insertions(+)
 create mode 100644 cicd-section/.aws/task-def-api.json
```

#### 15.5.5 リモートリポジトリへプッシュ

```bash
git push origin main
```

**出力例:**

```
Enumerating objects: 10, done.
Counting objects: 100% (10/10), done.
Delta compression using up to 8 threads
Compressing objects: 100% (6/6), done.
Writing objects: 100% (7/7), 1.23 KiB | 1.23 MiB/s, done.
Total 7 (delta 2), reused 0 (delta 0), pack-reused 0
To https://github.com/yourname/ecs-learning-course.git
   xxxxxxxx..yyyyyyyy  main -> main
```

プッシュが完了すると、GitHub Actions が自動的にワークフローを実行開始します。

---

### 15.6 パイプラインの実行確認

#### 15.6.1 GitHub Actions ページへアクセス

1. GitHub のリポジトリページにアクセスします
2. 「**Actions**」タブをクリックします

#### 15.6.2 ワークフロー実行の確認

最新のワークフロー実行が表示されます:

- コミットメッセージ: 「Add ECS deploy workflow and task definition template」
- ステータス: 実行中(黄色)または完了(緑色/赤色)

#### 15.6.3 予想されるエラー

> **重要な注意事項:**
>
> この時点では、パイプラインは**deploy ジョブで失敗する**ことが予想されます。
>
> **失敗の理由:**
>
> - IAM ロールに必要な権限が不足している
> - 具体的には、`iam:PassRole` 権限が不足

---

## 16. デプロイエラーの確認と原因分析

### 16.1 概要

前のセクションでパイプラインを実行したところ、テストビルドは成功しますが、デプロイジョブで失敗します。このセクションでは、エラーの詳細を確認し、原因を特定します。

### 16.2 エラーの確認手順

#### 16.2.1 ワークフロー実行結果の確認

1. GitHub リポジトリの「**Actions**」タブにアクセスします
2. 最新のワークフロー実行をクリックします
3. 実行結果を確認します:
   - **test-build** ジョブ: ✅ 成功(緑色のチェックマーク)
   - **deploy** ジョブ: ❌ 失敗(赤色のバツマーク)

#### 16.2.2 失敗したジョブの詳細確認

1. **deploy** ジョブをクリックして詳細を表示します
2. ジョブの各ステップが表示されます
3. 失敗したステップを確認します:
   - **Deploy to ECS task** というステップで失敗しています

#### 16.2.3 エラーメッセージの確認

失敗したステップを展開すると、以下のようなエラーメッセージが表示されます:

```
タスク定義の登録に失敗しました
```

エラーの詳細を確認すると:

```
User: arn:aws:sts::xxxxxxxxxxxx:assumed-role/GitHubActionsEcsLearningCourse/GitHubActions 
is not authorized to perform: iam:PassRole on resource: 
arn:aws:iam::xxxxxxxxxxxx:role/ecsTaskExecutionRole
```

**エラーの意味:**

- `GitHubActionsEcsLearningCourse` ロール(OIDC 用に作成したロール)に、`iam:PassRole` アクションを実行する権限がありません
- `ecsTaskExecutionRole` に対して PassRole を実行できないため、タスク定義の登録が失敗しています

---

## 17. iam:PassRole 権限の理解

### 17.1 PassRole とは

#### 17.1.1 PassRole の役割

**PassRole** とは、AWS のリソース(ここでは ECS タスク)に対して、別の IAM ロールを渡す(割り当てる)権限のことです。

#### 17.1.2 なぜ PassRole が必要なのか

**タスク定義における executionRoleArn の確認:**

以前作成したタスク定義ファイル `task-def-api.json` の 35 行目付近を確認すると:

```json
{
  "executionRoleArn": "arn:aws:iam::xxxxxxxxxxxx:role/ecsTaskExecutionRole",
  ...
}
```

このように、**executionRoleArn** で `ecsTaskExecutionRole` を指定しています。

**executionRoleArn の意味:**

- ECS タスクが実行されるときに、そのタスクに与えられる IAM ロールを指定しています
- つまり、ECS タスクに対して `ecsTaskExecutionRole` という権限を**渡している**ことになります

**PassRole が重要な理由:**

権限を他のリソースに渡す行為自体が、非常に強力で重要な操作です。そのため、AWS では以下のようなセキュリティポリシーが適用されています:

- デフォルトでは、他のリソースに対してロールを渡すことは**禁止**されています
- 明示的に `iam:PassRole` 権限を付与することで、初めてロールを渡すことが許可されます

**今回のケース:**

- GitHub Actions が使用する `GitHubActionsEcsLearningCourse` ロールは、ECS タスクに対して `ecsTaskExecutionRole` を渡そうとしています
- しかし、`GitHubActionsEcsLearningCourse` ロールに `iam:PassRole` 権限が付与されていないため、エラーが発生しています

### 17.2 必要な対応

`GitHubActionsEcsLearningCourse` ロールに対して、以下の権限を追加する必要があります:

- **アクション**: `iam:PassRole`
- **リソース**: `ecsTaskExecutionRole`(渡すことを許可するロール)

---

## 18. IAM ロールへの PassRole 権限の追加

### 18.1 概要

`GitHubActionsEcsLearningCourse` ロールに `iam:PassRole` 権限を追加し、ECS タスクに対して `ecsTaskExecutionRole` を渡せるようにします。

### 18.2 手順

#### 18.2.1 IAM コンソールへアクセス

1. AWS マネジメントコンソールで「**IAM**」を検索し、選択します

#### 18.2.2 ロールの選択

1. 左側のメニューから「**ロール**」をクリックします
2. ロール一覧から `GitHubActionsEcsLearningCourse` を検索し、クリックします

#### 18.2.3 既存のポリシーの確認

**許可ポリシー**セクションで、以下のポリシーがアタッチされていることを確認します:

- カスタマーインライン形式のポリシーが 1 つ存在します
  - これは以前のセクションで追加したポリシーです
  - タスク定義の書き換え、ECR へのファイルアップロードなどの権限を含んでいます

#### 18.2.4 ポリシーの編集

1. カスタマーインラインポリシーの名前をクリックします
2. 「**編集**」ボタンをクリックします

#### 18.2.5 編集モードの選択

**JSON** タブで編集することもできますが、今回は**ビジュアルエディタ**を使用します:

1. 「**ビジュアル**」タブをクリックします

#### 18.2.6 新しい許可の追加

1. 「**許可を追加**」ボタンをクリックします
2. 「**サービスを選択**」で以下を設定します:

**サービス:**

- 検索ボックスに「**IAM**」と入力します
- 「**IAM**」を選択します

**アクション:**

1. 「**アクション**」セクションで、検索ボックスに「**PassRole**」と入力します
2. 「**PassRole**」にチェックを入れます

#### 18.2.7 リソースの指定

「**リソース**」セクションで、PassRole を許可するロールを指定します:

**リソースタイプの選択:**

- 「**特定**」を選択します
  - すべてのロールを渡せるようにするのはセキュリティ上好ましくないため、特定のロールのみに制限します

**ARN の追加:**

1. 「**ARN を追加**」をクリックします
2. 「**リソース ARN を指定**」ダイアログが表示されます
3. 「**ロール名**」フィールドに以下を入力します:

```
ecsTaskExecutionRole
```

4. ARN が自動的に表示されることを確認します:

```
arn:aws:iam::xxxxxxxxxxxx:role/ecsTaskExecutionRole
```

5. 「**ARN を追加**」ボタンをクリックします

#### 18.2.8 設定の確認

リソースセクションに以下が追加されたことを確認します:

```
arn:aws:iam::xxxxxxxxxxxx:role/ecsTaskExecutionRole
```

#### 18.2.9 ポリシーの保存

1. 「**次へ**」ボタンをクリックします
2. ポリシーの内容を確認します
3. 「**変更を保存**」ボタンをクリックします

#### 18.2.10 更新後のポリシーの確認

**許可ポリシー**セクションで、カスタマーインラインポリシーをクリックして内容を確認します:

**確認事項:**

- ビジュアルエディタで編集したため、JSON のフォーマットが若干変わっている可能性があります
- しかし、**内容自体は変わっていません**(既存の権限はそのまま残っています)
- 重要なのは、以下の新しい権限が追加されていることです:

```json
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "arn:aws:iam::xxxxxxxxxxxx:role/ecsTaskExecutionRole"
}
```

---

## 19. パイプラインの再実行

### 19.1 概要

IAM ロールに PassRole 権限を追加したので、再度パイプラインを実行します。今回は JSON や YAML ファイルを変更していないため、コミットやプッシュは不要です。GitHub Actions の UI から直接再実行します。

### 19.2 手順

#### 19.2.1 ワークフローの再実行

1. GitHub リポジトリの「**Actions**」タブにアクセスします
2. 失敗したワークフロー実行をクリックします
3. 画面右上の「**Re-run jobs**」(再実行)ボタンをクリックします
4. 「**Re-run failed jobs**」(失敗したジョブを再実行)を選択します

> **補足:**
>
> - 「Re-run all jobs」を選択すると、すべてのジョブが再実行されます
> - 今回は **test-build** ジョブは既に成功しているため、「Re-run failed jobs」を選択することで、**deploy** ジョブのみが再実行されます

#### 19.2.2 確認ダイアログ

確認ダイアログが表示されたら、「**Re-run jobs**」ボタンをクリックします。

#### 19.2.3 実行の確認

- **deploy** ジョブのみが再実行されます
- **test-build** ジョブは成功のまま(緑色のチェックマーク)で残ります

#### 19.2.4 実行完了の待機

デプロイジョブの実行には約 3 分程度かかります。しばらく待ちます。

### 19.3 実行結果の確認

#### 19.3.1 成功の確認

約 3 分後、GitHub Actions ページを確認します:

- **test-build** ジョブ: ✅ 成功(緑色のチェックマーク)
- **deploy** ジョブ: ✅ 成功(緑色のチェックマーク)

両方のジョブが成功していれば、ECS へのデプロイが完了しています。

#### 19.3.2 デプロイの意味

デプロイが完了したということは:

- Docker イメージが ECR にプッシュされました
- タスク定義が新しいリビジョンで更新されました
- ECS サービスが新しいタスク定義を使用して、新しいタスク(コンテナ)を起動しました

---

## 21. デプロイ状態の確認

### 21.1 概要

パイプラインが成功したので、ECS 上でタスク定義とサービスが正しく更新されているか確認します。

### 21.2 タスク定義の確認

#### 21.2.1 タスク定義一覧へアクセス

1. AWS マネジメントコンソールで「**ECS**」を開きます
2. 左側のメニューから「**タスク定義**」をクリックします

#### 21.2.2 タスク定義のリビジョン確認

1. タスク定義ファミリー一覧から `my-app-api` をクリックします
2. リビジョン一覧が表示されます
3. **リビジョン番号が増えている**ことを確認します
   - 例: リビジョン 1、リビジョン 2、リビジョン 3...
   - CI/CD パイプラインが実行されるたびに、新しいリビジョンが作成されます

#### 21.2.3 最新リビジョンの内容確認

1. **最新のリビジョン**(番号が最も大きいもの)をクリックします
2. 画面右上の「**JSON**」タブをクリックします

#### 21.2.4 JSON の確認

JSON の内容を確認し、以下のポイントをチェックします:

**1. コンテナ名の確認:**

`containerDefinitions` セクションの `name` フィールドを確認します:

```json
{
  "containerDefinitions": [
    {
      "name": "api",
      ...
    }
  ]
}
```

- コンテナ名が `api` になっていることを確認します
- 以前のダミーコンテナ名 `dummy` から更新されています

**2. イメージ URI の確認:**

`containerDefinitions` セクションの `image` フィールドを確認します:

```json
{
  "containerDefinitions": [
    {
      "image": "xxxxxxxxxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/my-app-api:<COMMIT_HASH>",
      ...
    }
  ]
}
```

**確認事項:**

- ECR のドメイン(`xxxxxxxxxxxx.dkr.ecr.ap-northeast-1.amazonaws.com`)が含まれています
- リポジトリ名 `my-app-api` が含まれています
- タグ部分には **GitHub のコミットハッシュ値**が使用されています
  - 例: `a1b2c3d4e5f6...`
  - これにより、どのコミットからビルドされたイメージかを追跡できます

> **補足:**
>
> - 以前は `public.ecr.aws/nginx/nginx:stable-perl` という NGINX のイメージを使用していました
> - CI/CD パイプラインにより、ECR の Spring Boot イメージに正しく更新されています

### 21.3 ECS サービスとタスクの確認

#### 21.3.1 クラスターへアクセス

1. ECS コンソールの左側メニューから「**クラスター**」をクリックします
2. クラスター一覧から `my-app-cluster` をクリックします

#### 21.3.2 サービスの確認

1. 「**サービス**」タブをクリックします
2. サービス一覧から `my-app-api-service` をクリックします

#### 21.3.3 実行中のタスクの確認

1. 「**タスク**」タブをクリックします
2. タスク一覧を確認します:
   - タスクが **1 つ実行中(RUNNING)**になっていることを確認します

#### 21.3.4 タスク詳細の確認

1. 実行中のタスクの **タスク ID**(リンク)をクリックします
2. タスク詳細ページが表示されます

**確認事項:**

**ステータス:**

- **ラストステータス**: `RUNNING`(実行中)

**ヘルスステータス:**

- **ヘルスステータス**: `UNKNOWN`(不明)

> **ヘルスステータスが「不明」である理由:**
>
> これは**問題ありません**。以下の理由により「不明」と表示されます:
>
> - ヘルスステータスは、タスク定義ファイルに**ヘルスチェックコマンド**を定義した場合にのみ有効になります
> - 今回のタスク定義では、ヘルスチェックコマンドを定義していないため、「不明」が表示されます
> - これは期待通りの動作です
>
> **ヘルスチェックとの違い:**
>
> - この「ヘルスステータス」は、以前のブルー/グリーンデプロイで使用した **ターゲットグループのヘルスチェック**とは別物です
> - タスク定義ファイルに `healthCheck` セクションを追加することで、独自のヘルスチェックを定義できます
> - 例: `curl` コマンドで API エンドポイントにアクセスするシェルスクリプトを実行
>
> **参考(ヘルスチェックの定義例):**
>
> ```json
> {
>   "healthCheck": {
>     "command": ["CMD-SHELL", "curl -f http://localhost:8080/api/hello || exit 1"],
>     "interval": 30,
>     "timeout": 5,
>     "retries": 3,
>     "startPeriod": 60
>   }
> }
> ```

---

## 22. アプリケーションへのアクセス(初回失敗)

### 22.1 パブリック IP の取得

#### 22.1.1 パブリック IP のコピー

タスク詳細ページの「**ネットワーキング**」セクションで、以下を確認します:

- **パブリック IP**: タスクに割り当てられたパブリック IP アドレスが表示されています
  - 例: `54.123.45.67`

パブリック IP をコピーします。

### 22.2 ブラウザでのアクセス

#### 22.2.1 URL の構築

ブラウザで新しいタブを開き、以下の形式で URL を入力します:

```
http://<パブリックIP>:8080/api/hello
```

**URL の構成要素:**

- `http://`: プロトコル
- `<パブリックIP>`: コピーしたパブリック IP アドレス
- `:8080`: Spring Boot アプリケーションが起動しているポート番号
- `/api/hello`: API エンドポイントのパス

**例:**

```
http://54.123.45.67:8080/api/hello
```

#### 22.2.2 アクセス結果

Enter キーを押してアクセスすると:

- **ページが読み込み中のまま**(グルグルマーク)になります
- タイムアウトまで待っても、レスポンスが返ってきません

### 22.3 問題の原因

**セキュリティグループの設定不足:**

現在、セキュリティグループは **80 番ポートのみ**を許可しており、**8080 番ポートへのアクセスは許可されていません**。

そのため、セキュリティグループのファイアウォールによってアクセスがブロックされています。

**なぜ 80 番ポートだけが許可されているのか:**

- ECS サービス作成時に、セキュリティグループで HTTP(80 番ポート)のインバウンドルールを設定しました
- これは、以前のダミー NGINX コンテナが 80 番ポートで動作していたためです
- しかし、Spring Boot アプリケーションは **8080 番ポート**で動作するため、8080 番ポートへのアクセスを許可する必要があります

---

## 23. セキュリティグループの修正

### 23.1 概要

セキュリティグループのインバウンドルールを編集し、8080 番ポートへのアクセスを許可します。同時に、不要になった 80 番ポートのルールを削除します。

### 23.2 手順

#### 23.2.1 セキュリティグループへのアクセス

**方法 1: ECS サービスから直接アクセス**

1. ECS コンソールで「クラスター」→「my-app-cluster」→「サービス」→「my-app-api-service」をクリックします
2. 「**設定とネットワーキング**」タブをクリックします
3. 「**ネットワーキング**」セクションで、「**セキュリティグループ**」のリンク(青色のリンク)をクリックします
   - セキュリティグループ ID(例: `sg-xxxxxxxxxxxx`)が表示されています
4. 新しいタブ/ウィンドウで、セキュリティグループの詳細ページが開きます

**方法 2: EC2 コンソールから検索**

1. AWS マネジメントコンソールで「**EC2**」を開きます
2. 左側のメニューから「**セキュリティグループ**」をクリックします
3. セキュリティグループ一覧から `my-app-api-sg` を検索し、クリックします

#### 23.2.2 現在のインバウンドルールの確認

セキュリティグループ詳細ページの「**インバウンドルール**」タブをクリックします。

**現在のルール:**

| タイプ | プロトコル | ポート範囲 | ソース      | 説明               |
| ------ | ---------- | ---------- | ----------- | ------------------ |
| HTTP   | TCP        | 80         | 0.0.0.0/0   | HTTP アクセス許可  |
| HTTP   | TCP        | 80         | ::/0        | HTTP アクセス許可  |

> **補足:**
>
> - `0.0.0.0/0`: すべての IPv4 アドレスからのアクセスを許可
> - `::/0`: すべての IPv6 アドレスからのアクセスを許可
> - 現在は 80 番ポートのみが許可されています

#### 23.2.3 インバウンドルールの編集

1. 「**インバウンドルールを編集**」ボタンをクリックします

#### 23.2.4 新しいルールの追加(8080 番ポート)

1. 「**ルールを追加**」ボタンをクリックします
2. 新しいルールに以下を設定します:

**タイプ:**

- 「**カスタム TCP**」を選択します

**ポート範囲:**

```
8080
```

**ソース:**

- 「**Anywhere-IPv4**」を選択します
  - 自動的に `0.0.0.0/0` が設定されます

> **セキュリティに関する注意:**
>
> - より厳格なセキュリティ設定が必要な場合は、「**カスタム**」を選択し、自分の IP アドレスのみを許可することを推奨します
> - 例: `123.456.78.90/32`(自分の固定 IP アドレス)
> - このコースでは設定を簡略化するため、すべての IP アドレスからのアクセスを許可しています

**説明(オプション):**

```
Allow Spring Boot API access on port 8080
```

#### 23.2.5 不要なルールの削除(80 番ポート)

80 番ポートのルールは、NGINX コンテナ用に設定したものであり、現在は不要です:

1. 80 番ポートのルール(IPv4 と IPv6 の両方)の右端にある「**削除**」ボタンをクリックします
2. 両方の 80 番ポートルールを削除します

> **セキュリティのベストプラクティス:**
>
> - 不要なポートは開放しないことが、セキュリティ上重要です
> - 使用しないポートを閉じることで、攻撃対象領域を減らすことができます

#### 23.2.6 ルールの保存

1. 「**ルールを保存**」ボタンをクリックします
2. インバウンドルールが更新されます

#### 23.2.7 更新後のルールの確認

**更新後のルール:**

| タイプ       | プロトコル | ポート範囲 | ソース    | 説明                               |
| ------------ | ---------- | ---------- | --------- | ---------------------------------- |
| カスタム TCP | TCP        | 8080       | 0.0.0.0/0 | Allow Spring Boot API access on... |

> **即座に反映:**
>
> - セキュリティグループの変更は**リアルタイムで反映**されます
> - ECS サービスやタスクを再起動する必要はありません

---

## 24. アプリケーションへのアクセス(成功)

### 24.1 再度アクセス

#### 24.1.1 ブラウザで URL を開く

セキュリティグループの設定が完了したので、再度ブラウザで以下の URL にアクセスします:

```
http://<パブリックIP>:8080/api/hello
```

例:

```
http://54.123.45.67:8080/api/hello
```

#### 24.1.2 成功の確認

ブラウザに以下のようなレスポンスが表示されます:

```
Hello
```

または、Spring Boot アプリケーションで設定した文字列が表示されます。

> **成功!**
>
> これで、GitHub から ECS へのデプロイが完全に成功し、アプリケーションにアクセスできるようになりました。

---

## 25. CI/CD パイプライン完成

### 25.1 達成したこと

このセクションを通じて、以下を達成しました:

#### 25.1.1 構築したリソース

1. **GitHub リポジトリ**: `ecs-learning-course`
2. **ECR リポジトリ**: `my-app-api`(Docker イメージを保存)
3. **ECS タスク定義**: `my-app-api`(コンテナの設定を定義)
4. **ECS サービス**: `my-app-api-service`(タスクを管理)
5. **セキュリティグループ**: `my-app-api-sg`(8080 番ポートを許可)
6. **IAM ロール**: `GitHubActionsEcsLearningCourse`(OIDC 用、PassRole 権限を含む)

#### 25.1.2 構築したパイプライン

**GitHub Actions ワークフロー**により、以下の自動化を実現しました:

1. **テストビルド**:
   - コードを GitHub にプッシュ
   - Spring Boot アプリケーションをビルド
   - Gradle テストを実行
2. **デプロイ**:
   - Docker イメージをビルド
   - ECR にイメージをプッシュ
   - タスク定義を更新(新しいリビジョンを作成)
   - ECS サービスを更新(新しいタスクをデプロイ)

### 25.2 CI/CD の流れ

```
┌─────────────────┐
│  開発者         │
│  コードを変更   │
└────────┬────────┘
         │
         │ git push
         ▼
┌─────────────────────────┐
│  GitHub リポジトリ      │
│  (ecs-learning-course)  │
└────────┬────────────────┘
         │
         │ トリガー
         ▼
┌─────────────────────────┐
│  GitHub Actions         │
│  ワークフロー実行       │
├─────────────────────────┤
│  1. テストビルド        │
│     - Gradle ビルド     │
│     - テスト実行        │
│  2. デプロイ            │
│     - Docker ビルド     │
│     - ECR プッシュ      │
│     - タスク定義更新    │
│     - ECS デプロイ      │
└────────┬────────────────┘
         │
         │ デプロイ完了
         ▼
┌─────────────────────────┐
│  AWS ECS                │
│  (my-app-api-service)   │
├─────────────────────────┤
│  Spring Boot API        │
│  http://IP:8080/api/... │
└─────────────────────────┘
```

### 25.3 今後の開発フロー

これからは、以下のような開発フローで作業できます:

1. **ローカルで開発**:
   - コードを修正
   - ローカルでテスト
2. **GitHub にプッシュ**:
   ```bash
   git add .
   git commit -m "新機能を追加"
   git push origin main
   ```
3. **自動デプロイ**:
   - GitHub Actions が自動的に実行されます
   - テスト → ビルド → デプロイが自動化されています
4. **確認**:
   - GitHub Actions で実行結果を確認
   - ECS でデプロイされたアプリケーションを確認

### 25.4 所要時間

**デプロイ時間:**

- パイプライン全体: 約 3〜5 分
  - テストビルド: 約 1〜2 分
  - デプロイ: 約 2〜3 分

### 25.5 まとめ

おつかれさまでした!

非常に長い手順でしたが、これで **GitHub から ECS へのタスクデプロイ**を実現する CI/CD パイプラインが完成しました。

**重要なポイント:**

- **自動化**: コードをプッシュするだけで、自動的にデプロイされます
- **セキュリティ**: OIDC を使用し、長期的なアクセスキーを使用しません
- **再現性**: タスク定義ファイルでインフラをコード管理できます
- **トレーサビリティ**: Docker イメージにコミットハッシュをタグ付けし、どのコードからビルドされたかを追跡できます

---

## 26. リソースのクリーンアップ

### 26.1 概要

今回の講義で使用したタスクは不要となりましたので、削除を行います。起動したままにしておくとコストが発生し続けるため、使用しないリソースは削除することをおすすめします。

> **注意**: ECS クラスター`my-app-cluster`は今後の講義でも使用しますので、削除しないでください。

### 26.2 削除するリソース

| リソース         | 名前                 | 削除 | 理由                                 |
| ---------------- | -------------------- | ---- | ------------------------------------ |
| **ECS クラスター** | `my-app-cluster`     | ✗    | 今後の講義でも使用するため残す       |
| **ECS サービス**   | `my-app-api-service` | ✓    | 今回の講義で使用したため削除         |
| **ECR イメージ**   | `my-app-api`         | △    | オプション(イメージサイズに応じて課金) |

---

## 27. ECS サービスの削除

### 27.1 概要

ECS サービス`my-app-api-service`を CloudFormation から削除します。

> **重要**: ECS コンソールから直接削除することもできますが、CloudFormation から削除することで、関連リソースをまとめて削除し、ゴミが残らないようにします。

### 27.2 手順

#### 27.2.1 CloudFormation コンソールへアクセス

1. AWS マネジメントコンソールで「CloudFormation」を検索します
2. 「CloudFormation」を選択します

#### 27.2.2 サービス用のスタックを確認

1. スタックの一覧から、`my-app-api-service`を作成した際に自動生成された CloudFormation スタックを探します
   - スタック名に`my-app-api-service`や`ECS-Console-V2`などの文字列が含まれているはずです
   - このスタックは、ECS コンソールからサービスを作成した際に自動的に AWS が作成したものです

#### 27.2.3 スタックの削除

1. 削除したいスタックにチェックを入れます
2. 「削除」ボタンをクリックします
3. 確認画面が表示されたら、「削除」をクリックします

#### 27.2.4 削除の確認

1. 「更新」ボタンをクリックして、ステータスを確認します
2. ステータスが`DELETE_IN_PROGRESS`(削除中)になっていることを確認します
3. しばらく待つと、スタックが一覧から消えます(削除完了)

> **所要時間**: 削除には数分かかる場合があります。

#### 27.2.5 ECS サービスが削除されたことを確認

1. ECS コンソールに戻ります
2. クラスター`my-app-cluster`を開きます
3. 「更新」ボタンをクリックします
4. サービス`my-app-api-service`が一覧から消えていることを確認します

---

## 28. ECR イメージの削除(オプション)

### 28.1 概要

ECR(Elastic Container Registry)に保存されているイメージは、イメージのサイズに応じて課金されます。不要なイメージを削除することでコストを削減できます。

> **注意**: このステップはオプションです。今後も使用する可能性がある場合は、削除しなくても問題ありません。

### 28.2 課金について

**ECR の課金ポイント:**

- **保存容量**: イメージのサイズに応じて課金
- **データ転送**: イメージの pull/push 時のデータ転送量に応じて課金

**無料枠:**

- 毎月 500MB のストレージ(12 ヶ月間)

### 28.3 ECR イメージの削除手順

#### 28.3.1 ECR コンソールへアクセス

1. AWS マネジメントコンソールで「ECR」を検索します
2. 「Elastic Container Registry」を選択します

#### 28.3.2 リポジトリの確認

1. 左側のメニューから「リポジトリ」をクリックします
2. リポジトリ`my-app-api`をクリックして、イメージ一覧を表示します

#### 28.3.3 イメージの削除

1. 削除したいイメージにチェックを入れます
   - 今までにプッシュしたイメージが複数存在している可能性があります
   - 不要なイメージを選択してください
2. 「削除」ボタンをクリックします
3. 確認画面が表示されたら、`delete`と入力して「削除」をクリックします

> **注意**: イメージを削除すると、そのイメージを使用しているタスクは起動できなくなります。必要なイメージまで削除しないよう注意してください。

#### 28.3.4 リポジトリ全体の削除(オプション)

リポジトリ自体を削除する場合は、以下の手順を実行します:

1. リポジトリ一覧に戻ります
2. 削除したいリポジトリ`my-app-api`にチェックを入れます
3. 「削除」ボタンをクリックします
4. 確認画面が表示されたら、リポジトリ名を入力して「削除」をクリックします

> **注意**: リポジトリを削除すると、その中のすべてのイメージも削除されます。

---

## 29. クリーンアップ完了

### 29.1 削除されたリソースの確認

以下のリソースが削除されました:

- [ ] ECS サービス`my-app-api-service`が削除された
- [ ] CloudFormation スタックが削除された
- [ ] (オプション)ECR イメージが削除された

### 29.2 残っているリソース

以下のリソースは今後の講義で使用するため、残しておきます:

- ✓ ECS クラスター`my-app-cluster`
- ✓ VPC`my-workspace-vpc`
- ✓ サブネット(パブリック・プライベート)
- ✓ タスク実行ロール`ecsTaskExecutionRole`
- ✓ GitHub リポジトリ
- ✓ IAM ロール(GitHub Actions 用)
- ✓ (オプション)ECR リポジトリ

### 29.3 コスト管理のベストプラクティス

**使用しないリソースは削除する:**

- ECS サービスやタスクは、使用していなくても課金されます
- 不要になったらすぐに削除しましょう

**定期的にリソースを確認する:**

- 定期的に AWS コンソールでリソースを確認し、不要なものがないかチェックしましょう
- CloudWatch や Cost Explorer を使用してコストを監視しましょう

**タグを活用する:**

- リソースにタグを付けることで、どのプロジェクトで使用しているか管理しやすくなります

---





