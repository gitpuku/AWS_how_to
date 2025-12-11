# 📘 AWS Lab1（Python）重要ポイントノート

**テーマ：開発環境の準備・AWS CLI・IAM の基本理解**

---

## ■ 1. 開発環境で押さえるべき点

### ● Python が正しくインストールされているか確認

```bash
python3 --version
```

→ AWS Lambda やスクリプトを書く際に必須。

### ● AWS CLI v2 がインストールされているか確認

```bash
aws --version
```

→ AWS 操作をコマンドラインで行うための基本ツール。

### ● aws configure の設定ポイント

AWS Access Key / Secret Key は EC2 のロールを使う場合は空で OK

設定が必要なのは以下の 2 つ：

- **Default region name**
  → 例：ap-northeast-1 など

- **Default output format**
  → yaml（任意だが可読性が良い）

ロールを使う開発環境では、キー管理が不要で安全。

### ● get-caller-identity を使って権限確認

AWS CLI が どの IAM ロールとして実行されているか を確認するコマンド。

```bash
aws sts get-caller-identity
```

**出力例：**

```
Arn: arn:aws:sts::<AccountID>:assumed-role/notes-application-role/i-xxxx
```

→ EC2 にアタッチされた IAM ロールで CLI が動いていることを確認できる。

---

## ■ 2. VS Code AWS 拡張機能で理解すべきこと

### ● AWS Explorer が便利

AWS Explorer では、GUI で以下ができる：

- S3、Lambda、DynamoDB などのリソース確認
- 関数のデプロイやファイルアップロード
- プロファイルやリージョンごとに操作

CLI が苦手な時の補助ツールにもなる。

---

## ■ 3. IAM とアクセス許可の理解（このラボの核心）

### ● S3 バケット一覧を確認

```bash
aws s3 ls
```

### ● バケット削除を試す（失敗するのがポイント）

```bash
aws s3 rb s3://<bucket-name>
```

→ アクセス許可不足で削除できない

**これを通じて学ぶのは：**

- ✔ "IAM ロールにどの権限が付与されているかでできる操作が決まる"
- ✔ "CLI の操作は IAM により明確に制御される"

### ● 失敗理由を調べるには --debug

```bash
aws s3 rb s3://<bucket> --debug
```

→ どのポリシーで拒否されているか確認できる。

---

## ■ 4. このラボで得るべき本質

- **EC2 に割り当てられた IAM ロールが CLI の認証に使われる**
  → 開発環境用 EC2 では、**アクセスキーを使わない認証（ロールの一時クレデンシャル）**が主流。

- **AWS CLI は IAM の制御下で動く**
  → 操作失敗の多くは IAM の権限不足。

- **S3 削除のような "結果を変えてしまう操作" は特に IAM で厳しく制御**
  → 本番環境で事故を防ぐため。

---

## 📌 このラボで学べるコアスキルまとめ

- ✔ Python / CLI の基本チェック
- ✔ AWS CLI のセットアップ（region, output-format）
- ✔ IAM ロールで CLI 認証が行われる仕組み
- ✔ get-caller-identity によるアクセス権の確認
- ✔ AWS Explorer の使い方
- ✔ S3 操作と IAM パーミッションの関係
- ✔ --debug の活用方法
