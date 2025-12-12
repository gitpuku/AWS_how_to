# Day04：Amazon VPC を使って仮想ネットワーク環境を構築する(後編) - Next.js 版

## 学習目標

プライベートサブネットを作成し、MySQL データベースサーバーを構築。Next.js アプリケーションからデータベースに接続してブログ機能を実装する。

## 📝 PHP 版からの変更点

**主な変更点：**
- PHP + mysqli → Next.js + mysql2 パッケージ
- サーバーサイドでのデータベース接続
- Docker Compose で環境変数管理
- API Routes でのデータ取得・投稿

---

## 前提条件

- Day03 (Next.js版) を完了していること
- VPC とパブリックサブネットが構築済み
- Next.js アプリケーションが起動済み

---

## 1. プライベートサブネットの作成

### 1.1 プライベートサブネット 1a の作成

1. VPC コンソール → 「サブネット」→「サブネットを作成」

**設定内容：**
- **VPC**: `udemy-aws-14days-vpc`
- **サブネット名**: `udemy-aws-14days-private-subnet-1a`
- **アベイラビリティーゾーン**: `ap-northeast-1a`
- **IPv4 CIDR ブロック**: `10.0.101.0/24`

### 1.2 プライベートサブネット 1c の作成

同様に以下を作成：
- **サブネット名**: `udemy-aws-14days-private-subnet-1c`
- **アベイラビリティーゾーン**: `ap-northeast-1c`
- **IPv4 CIDR ブロック**: `10.0.102.0/24`

### 1.3 プライベート用ルートテーブルの作成

1. VPC コンソール → 「ルートテーブル」→「ルートテーブルを作成」
2. **名前**: `udemy-aws-14days-private-rtb`
3. **VPC**: `udemy-aws-14days-vpc`
4. サブネットの関連付け：
   - プライベートサブネット 1a
   - プライベートサブネット 1c

---

## 2. データベース用セキュリティグループの作成

### 2.1 セキュリティグループの作成

1. VPC コンソール → 「セキュリティグループ」→「セキュリティグループを作成」

**設定内容：**
- **セキュリティグループ名**: `udemy-aws-14days-db-sg`
- **説明**: Security group for MySQL database
- **VPC**: `udemy-aws-14days-vpc`

**インバウンドルール：**
| タイプ | プロトコル | ポート範囲 | ソース | 説明 |
|--------|-----------|-----------|--------|------|
| MySQL/Aurora | TCP | 3306 | udemy-aws-14days-web-sg | From web servers |

2. 「セキュリティグループを作成」をクリック

---

## 3. MySQL データベースサーバーの構築

### 3.1 EC2 インスタンスの起動

1. EC2 コンソール → 「インスタンスを起動」

**設定内容：**
- **名前**: `db-1a`
- **AMI**: Amazon Linux 2023
- **インスタンスタイプ**: `t2.micro`
- **キーペア**: `udemy-aws-14days`
- **ネットワーク設定**:
  - **VPC**: `udemy-aws-14days-vpc`
  - **サブネット**: `udemy-aws-14days-private-subnet-1a`
  - **パブリック IP の自動割り当て**: 無効
  - **セキュリティグループ**: `udemy-aws-14days-db-sg`
- **高度な詳細**:
  - **プライマリ IP**: `10.0.101.20`

### 3.2 踏み台経由での SSH 接続

```bash
# CloudShell から web-1a 経由で db-1a に接続
ssh -i udemy-aws-14days.pem ec2-user@[web-1a のパブリック IP]

# web-1a から db-1a に接続するためのキー転送（または事前にキーをコピー）
# 注意: 実運用では踏み台サーバーを別途用意するか、Systems Manager Session Manager を使用
```

### 3.3 MySQL のインストールと設定

```bash
# db-1a にログイン後

# MySQL リポジトリの追加
sudo dnf -y install https://dev.mysql.com/get/mysql84-community-release-el9-1.noarch.rpm

# MySQL サーバーのインストール
sudo dnf install -y mysql-community-server

# MySQL の起動と自動起動設定
sudo systemctl start mysqld
sudo systemctl enable mysqld

# 初期パスワードの確認
sudo grep 'temporary password' /var/log/mysqld.log
```

### 3.4 MySQL の初期設定

```bash
# セキュリティ設定
sudo mysql_secure_installation

# 表示される初期パスワードを入力
# 新しいパスワード: RootPass!1234
# パスワード検証プラグイン: y
# レベル: 1
# 匿名ユーザー削除: y
# リモートroot禁止: y
# testデータベース削除: y
# 権限テーブル再読み込み: y
```

### 3.5 データベースとユーザーの作成

```bash
# MySQL にログイン
mysql -u root -p
# パスワード: RootPass!1234
```

```sql
-- データベース作成
CREATE DATABASE simpleblog_db;

-- ユーザー作成とアクセス権限付与
CREATE USER 'simpleblog_user'@'%' IDENTIFIED BY 'User!1234';
GRANT ALL PRIVILEGES ON simpleblog_db.* TO 'simpleblog_user'@'%';
FLUSH PRIVILEGES;

-- データベースに切り替え
USE simpleblog_db;

-- テーブル作成
CREATE TABLE posts (
  id INT AUTO_INCREMENT PRIMARY KEY,
  title VARCHAR(255) NOT NULL,
  content TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- サンプルデータ挿入
INSERT INTO posts (title, content) VALUES
('データベース接続成功', 'Next.jsからMySQLに接続できました！'),
('AWS学習', 'VPCとEC2、MySQLの構成が完成しました');

-- 確認
SELECT * FROM posts;

-- 終了
EXIT;
```

---

## 4. Web サーバーから MySQL への接続確認

### 4.1 Web サーバーに MySQL クライアントをインストール

```bash
# web-1a にログイン
ssh -i udemy-aws-14days.pem ec2-user@[web-1a のパブリック IP]

# MySQL リポジトリの追加
sudo dnf -y install https://dev.mysql.com/get/mysql84-community-release-el9-1.noarch.rpm

# MySQL クライアントのインストール
sudo dnf install -y mysql
```

### 4.2 データベース接続テスト

```bash
# db-1a への接続テスト
mysql -h 10.0.101.20 -u simpleblog_user -p
# パスワード: User!1234

# 接続成功後
USE simpleblog_db;
SELECT * FROM posts;
EXIT;
```

---

## 5. Next.js アプリケーションの MySQL 対応

### 5.1 mysql2 パッケージの追加

```bash
cd ~/simple-blog

# package.jsonに mysql2 を追加
npm install mysql2
```

### 5.2 データベース接続ライブラリの作成

**lib/db.ts** を作成：

```typescript
import mysql from 'mysql2/promise';

const pool = mysql.createPool({
  host: process.env.DB_HOST || '10.0.101.20',
  user: process.env.DB_USER || 'simpleblog_user',
  password: process.env.DB_PASSWORD || 'User!1234',
  database: process.env.DB_NAME || 'simpleblog_db',
  waitForConnections: true,
  connectionLimit: 10,
  queueLimit: 0,
});

export default pool;
```

### 5.3 docker-compose.yml の更新

```bash
# ~/simple-blog/docker-compose.yml を編集
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - INSTANCE_ID=${INSTANCE_ID:-web-1a}
      - DB_HOST=${DB_HOST:-10.0.101.20}
      - DB_USER=${DB_USER:-simpleblog_user}
      - DB_PASSWORD=${DB_PASSWORD:-User!1234}
      - DB_NAME=${DB_NAME:-simpleblog_db}
    restart: unless-stopped
    container_name: simple-blog
EOF
```

### 5.4 .env ファイルの更新

```bash
cat > .env << 'EOF'
DB_HOST=10.0.101.20
DB_USER=simpleblog_user
DB_PASSWORD=User!1234
DB_NAME=simpleblog_db
INSTANCE_ID=web-1a
EOF
```

### 5.4 package.json の更新

```json
{
  "name": "aws-simple-blog-nextjs",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "mysql2": "^3.11.5",
    "next": "14.2.18",
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@types/node": "^20",
    "@types/react": "^18",
    "@types/react-dom": "^18",
    "autoprefixer": "^10.4.20",
    "postcss": "^8",
    "tailwindcss": "^3.4.1",
    "typescript": "^5"
  }
}
```

### 5.5 API ルートの更新

**app/api/posts/route.ts** を更新：

```typescript
import { NextRequest, NextResponse } from "next/server";
import pool from "@/lib/db";
import { RowDataPacket, ResultSetHeader } from "mysql2";

interface PostRow extends RowDataPacket {
  id: number;
  title: string;
  content: string;
  created_at: Date;
}

// 投稿一覧取得
export async function GET() {
  try {
    const [rows] = await pool.query<PostRow[]>(
      "SELECT * FROM posts ORDER BY created_at DESC"
    );

    return NextResponse.json(rows);
  } catch (error) {
    console.error("Database error:", error);
    return NextResponse.json(
      { error: "Failed to fetch posts" },
      { status: 500 }
    );
  }
}

// 新規投稿作成
export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const { title, content } = body;

    if (!title || !content) {
      return NextResponse.json(
        { error: "Title and content are required" },
        { status: 400 }
      );
    }

    const [result] = await pool.query<ResultSetHeader>(
      "INSERT INTO posts (title, content) VALUES (?, ?)",
      [title, content]
    );

    return NextResponse.json(
      { id: result.insertId, title, content },
      { status: 201 }
    );
  } catch (error) {
    console.error("Database error:", error);
    return NextResponse.json(
      { error: "Failed to create post" },
      { status: 500 }
    );
  }
}
```

### 5.6 フロントエンドコンポーネントの更新

**app/page.tsx** を更新：

```typescript
"use client";

import { useState, useEffect } from "react";
import { Post } from "@/lib/types";

export default function Home() {
  const [posts, setPosts] = useState<Post[]>([]);
  const [title, setTitle] = useState("");
  const [content, setContent] = useState("");
  const [loading, setLoading] = useState(true);
  const [submitting, setSubmitting] = useState(false);

  useEffect(() => {
    fetchPosts();
  }, []);

  const fetchPosts = async () => {
    try {
      const response = await fetch("/api/posts");
      const data = await response.json();
      setPosts(data);
    } catch (error) {
      console.error("Failed to fetch posts:", error);
    } finally {
      setLoading(false);
    }
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setSubmitting(true);

    try {
      const response = await fetch("/api/posts", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({ title, content }),
      });

      if (response.ok) {
        setTitle("");
        setContent("");
        fetchPosts();
      }
    } catch (error) {
      console.error("Failed to create post:", error);
    } finally {
      setSubmitting(false);
    }
  };

  if (loading) {
    return (
      <div className="min-h-screen flex items-center justify-center">
        <div className="text-xl text-gray-600">読み込み中...</div>
      </div>
    );
  }

  return (
    <div className="min-h-screen py-8 bg-gray-50">
      <div className="max-w-4xl mx-auto px-4">
        <header className="mb-8 text-center">
          <h1 className="text-4xl font-bold text-blue-600 mb-2">
            シンプルブログ
          </h1>
          <p className="text-gray-600">Next.js + MySQL on AWS</p>
        </header>

        {/* 投稿フォーム */}
        <div className="bg-white rounded-lg shadow-md p-6 mb-8">
          <h2 className="text-2xl font-semibold text-gray-800 mb-4">
            新規投稿
          </h2>
          <form onSubmit={handleSubmit} className="space-y-4">
            <div>
              <label
                htmlFor="title"
                className="block text-sm font-medium text-gray-700 mb-1"
              >
                タイトル
              </label>
              <input
                type="text"
                id="title"
                value={title}
                onChange={(e) => setTitle(e.target.value)}
                className="w-full px-4 py-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                required
              />
            </div>
            <div>
              <label
                htmlFor="content"
                className="block text-sm font-medium text-gray-700 mb-1"
              >
                内容
              </label>
              <textarea
                id="content"
                value={content}
                onChange={(e) => setContent(e.target.value)}
                rows={4}
                className="w-full px-4 py-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                required
              />
            </div>
            <button
              type="submit"
              disabled={submitting}
              className="w-full bg-blue-600 text-white py-2 px-4 rounded-md hover:bg-blue-700 disabled:bg-gray-400 transition-colors"
            >
              {submitting ? "投稿中..." : "投稿する"}
            </button>
          </form>
        </div>

        {/* 投稿一覧 */}
        <div className="space-y-4">
          <h2 className="text-2xl font-semibold text-gray-800 mb-4">
            投稿一覧
          </h2>
          {posts.length === 0 ? (
            <p className="text-gray-600 text-center py-8">
              まだ投稿がありません
            </p>
          ) : (
            posts.map((post) => (
              <article
                key={post.id}
                className="bg-white rounded-lg shadow-md p-6 hover:shadow-lg transition-shadow"
              >
                <h3 className="text-2xl font-semibold text-gray-800 mb-2">
                  {post.title}
                </h3>
                <p className="text-gray-600 mb-4 whitespace-pre-wrap">
                  {post.content}
                </p>
                <time className="text-sm text-gray-500">
                  {new Date(post.created_at).toLocaleString("ja-JP")}
                </time>
              </article>
            ))
          )}
        </div>

        <div className="mt-8 p-4 bg-blue-50 rounded-lg">
          <h3 className="font-semibold text-blue-900 mb-2">技術スタック</h3>
          <ul className="text-sm text-blue-800 space-y-1">
            <li>• Next.js 14 (App Router)</li>
            <li>• TypeScript</li>
            <li>• Tailwind CSS</li>
            <li>• MySQL (mysql2)</li>
            <li>• Amazon EC2</li>
            <li>• Amazon VPC (Public + Private Subnets)</li>
          </ul>
        </div>
      </div>
    </div>
  );
}
```

### 5.7 アプリケーションの再ビルドと再起動

```bash
cd ~/simple-blog

# Docker イメージを再ビルドしてコンテナを再起動
docker compose down
docker compose up -d --build

# ログ確認
docker compose logs -f
```

---

## 6. 動作確認

### 6.1 ブラウザでアクセス

`http://[web-1aのパブリックIP]:3000` にアクセス

### 6.2 確認項目

1. データベースから取得した投稿が表示される
2. 新規投稿フォームから投稿できる
3. 投稿後、自動的に一覧が更新される

---

## 📝 トラブルシューティング

### データベースに接続できない

```bash
# web-1a から db-1a への接続確認
mysql -h 10.0.101.20 -u simpleblog_user -p

# セキュリティグループの確認
# - db-sg がポート 3306 を web-sg から許可しているか
# - MySQL が外部接続を受け付けているか
```

### "Client does not support authentication protocol" エラー

```sql
-- db-1a の MySQL で実行
ALTER USER 'simpleblog_user'@'%' IDENTIFIED WITH mysql_native_password BY 'User!1234';
FLUSH PRIVILEGES;
```

### Docker コンテナがクラッシュする

```bash
# 詳細ログを確認
docker compose logs

# 環境変数が読み込まれているか確認
docker compose config

# コンテナを再構築
docker compose down
docker compose up -d --build
```

---

## まとめ

Day4 (Next.js版) では以下を実施しました：

### 学習したキーワード

- プライベートサブネット
- データベースサーバー
- MySQL
- **mysql2 パッケージ**
- **コネクションプール**
- **API Routes**
- **環境変数管理**

### 構築した環境

1. プライベートサブネット（2つの AZ）
2. MySQL データベースサーバー
3. Next.js からの MySQL 接続
4. CRUD 操作（作成・読み取り）

### PHP 版との主な違い

| 項目 | PHP 版 | Next.js 版 |
|------|--------|------------|
| DB ドライバ | mysqli | mysql2 |
| 接続方式 | 都度接続 | コネクションプール |
| SQL 実行 | mysqli_query | pool.query (Promise) |
| フォーム処理 | POST + header redirect | fetch API + React State |
| エラーハンドリング | PHP エラー | try-catch + JSON レスポンス |

**次回の Day05 では、RDS を使用したマネージドデータベースに移行します。**
