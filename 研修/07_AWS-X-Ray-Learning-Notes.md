# AWS X-Ray 学習ノート

ラボが使えなくても復習できる内容に厳選した学習ガイド

---

## 1. AWS X-Ray の役割（最重要）

X-Ray はアプリケーションのリクエストをトレースするサービス。

### できること

- Lambda、API Gateway のリクエストの流れを可視化
- 下流サービス（DynamoDB、外部 API など）への呼び出しも含めて観測
- パフォーマンス問題の特定（ボトルネックの特定）
- 例外・エラーの可視化
- 注釈（annotation）を使ってビジネスロジックのメタデータを追加可能

---

## 2. 今回のラボで学ぶべき X-Ray の実装ポイント

### ✅ ① requirements.txt に X-Ray SDK を追加

**対象：** delete-function のみ

```text
aws-xray-sdk==2.14.0
```

> **注意：** SAM build により Lambda パッケージに含まれます。

### ✅ ② Lambda で X-Ray のログ出力を有効化

**場所：** app.py の TODO 1 に追加

```python
import logging
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

logger = logging.getLogger()
logger.setLevel(logging.INFO)
patch_all()
```

**重要ポイント：**

- `patch_all()` で boto3、HTTP クライアントなどが自動でインストルメント化
- Lambda では root segment は Lambda が管理するので、自分で作れるのはサブセグメントのみ

### ✅ ③ X-Ray 注釈（annotation）を追加

**場所：** TODO 2 の答え

```python
xray_recorder.begin_subsegment('Delete a note')
xray_recorder.put_annotation("UserId", UserId)
xray_recorder.put_annotation("NoteId", NoteId)
xray_recorder.end_subsegment()
```

**重要ポイント：**

- `begin_subsegment()` → `end_subsegment()` で処理の範囲を区切る
- `put_annotation()` を使う（`add_annotation` は別用途）
- 注釈を付けることで、X-Ray コンソール上で UserId をフィルターに使える

---

## 3. SAM テンプレートでの X-Ray トレース設定

### ✅ ④ Globals セクションで一括有効化

SAM テンプレート (`template.yml`) に以下を追加することで、すべての Lambda と API にトレースが有効化される。

```yaml
Globals:
  Function:
    Tracing: Active
  Api:
    TracingEnabled: true
```

**重要ポイント：**

- Lambda は `Tracing: Active`
- API Gateway は `TracingEnabled: true`
- SAM の Globals に定義すると全関数に適用できる（超便利）

---

## 📌 まとめ：ここまで理解できれば X-Ray は 80% マスター

| 項目                                        | 必須度 | 理由                         |
| ------------------------------------------- | ------ | ---------------------------- |
| requirements.txt に X-Ray SDK を追加        | ★★★★★  | Lambda の準備で必須          |
| patch_all() で boto3 等をインストルメント化 | ★★★★★  | ほぼ最重要                   |
| 注釈 (put_annotation) の使い方              | ★★★★★  | 実務で役立つ                 |
| SAM Globals の Tracing 設定                 | ★★★★★  | 設定を全部書く必要が無くなる |
| トレース構造（セグメント・サブセグメント）  | ★★★★☆  | バグ調査に必須               |

---

## 🔧 ラボが使えない後に勉強を続ける方法（超重要）

明日からラボ環境がなくても続けられるように、手元に再現可能な学習方法をまとめました。

### 🔧 1. ローカルに SAM + Lambda 開発環境を構築する

**必要なもの：**

```bash
# AWS CLI のインストール
brew install awscli

# AWS SAM CLI のインストール
brew tap aws/tap
brew install aws-sam-cli
```

**追加要件：**

- Docker → Lambda のローカル実行に必須（Docker Desktop をインストール）

### 🔧 2. SAM テンプレートを使って実際にローカルで Lambda を動かす

**SAM プロジェクト作成：**

```bash
sam init
# → Hello World テンプレートを選択
# → Python を選択
```

**ローカル実行：**

```bash
sam build
sam local invoke
```

### 🔧 3. ローカルで X-Ray を体験する方法

Lambda + X-Ray をローカル環境で動かすには 2 つのステップが必要です。

**① X-Ray Daemon をローカルで起動**

```bash
docker run -d -p 2000:2000/udp amazon/aws-xray-daemon
```

AWS 提供の公式 docker イメージを使用します。

**② Lambda の環境変数に X-Ray のエンドポイントを設定**

SAM の `template.yml` に以下を追加：

```yaml
Environment:
  Variables:
    AWS_XRAY_DAEMON_ADDRESS: "127.0.0.1:2000"
```

これでローカルでも X-Ray トレースが生成されます。

> **注意：** AWS コンソールには送信されませんが、ログに出力された JSON を見ることは可能です。

### 🔧 4. AWS の無料枠で最小構成の X-Ray を学習する

ラボが無くても、以下の構成で無料（or ほぼ無料）で学習可能です。

**無料で使用できるサービス：**

- ✅ Lambda（無料枠で問題なし）
- ✅ API Gateway（少量なら無料）
- ✅ DynamoDB（オンデマンド課金、ほぼ無料）
- ✅ X-Ray（トレース数が少なければ無料）

**簡易構成（研修アプリを簡略化）：**

```
Lambda × 1
    ↓
DynamoDB テーブル × 1
    ↓
API Gateway × 1
    ↓
X-Ray でトレース確認
```

**SAM デプロイ：**

```bash
sam deploy --guided
```

---

## 📊 学習優先度チェックリスト

実装の優先順位をつけるなら以下の順序でマスターしましょう：

1. **requirements.txt に X-Ray SDK を追加** ← まずこれから
2. **patch_all() でインストルメント化**
3. **注釈を使ったメタデータ追加**
4. **SAM テンプレートでの設定**
5. **トレース構造の理解**
6. **ローカル開発環境での動作確認**

---

## 💡 よくある質問と回答

**Q: ローカルで X-Ray を動かしても AWS コンソールに送信されないの？**

A: はい。ローカルの X-Ray Daemon はスタンドアロン動作するため、AWS コンソールには送信されません。ただし、Lambda のログに JSON 形式のトレース情報が出力されるので、それで学習可能です。

**Q: 本番環境デプロイ時は何か追加で設定が必要？**

A: SAM テンプレートの `Tracing: Active` が指定されていれば、AWS が자동으로 X-Ray に送信してくれます。追加設定は不要です。

**Q: put_annotation と add_annotation の違いは？**

A: `put_annotation()` はフィルタリングに使えるメタデータを追加します。`add_annotation` は別の用途で使用されます。実装では `put_annotation()` を使ってください。
