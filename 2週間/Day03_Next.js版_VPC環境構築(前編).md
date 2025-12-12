# Day03：Amazon VPC を使って仮想ネットワーク環境を構築する(前編) - Next.js 版

## 学習目標

Amazon VPC を使用して仮想ネットワーク環境を構築し、Next.js アプリケーション（App Router + TypeScript + Tailwind CSS）を EC2 インスタンスにデプロイして運用する。

## 📝 PHP 版からの変更点

このドキュメントは、元の Day03 の PHP + Apache 環境を **Next.js 14 (App Router) + TypeScript + Tailwind CSS + Docker** に置き換えたバージョンです。

**主な変更点：**
- Apache + PHP → Next.js + Docker コンテナ
- PHP ファイル → Next.js App Router (TypeScript)
- Docker & Docker Compose をインストール
- ポート 3000 で動作（セキュリティグループで 3000 を許可）

## 前提条件

- Day01、Day02 を完了していること
- Next.js の基本的な知識があること
- Docker の基本的な知識があること
- Git の基本操作ができること

---

## VPC の構築（元の Day03 と同じ）

### Step 1: VPC の作成

1. AWS マネジメントコンソールで「VPC」を検索
2. 「VPC を作成」をクリック

**設定内容：**
- **作成するリソース**: VPC のみ
- **名前タグ**: `udemy-aws-14days-vpc`
- **IPv4 CIDR ブロック**: `10.0.0.0/16`
- **IPv6 CIDR ブロック**: なし
- **テナンシー**: デフォルト

3. 「VPC を作成」をクリック

### Step 2: サブネットの作成

#### パブリックサブネット 1a の作成

1. VPC コンソール → 「サブネット」→「サブネットを作成」

**設定内容：**
- **VPC ID**: `udemy-aws-14days-vpc`
- **サブネット名**: `udemy-aws-14days-public-subnet-1a`
- **アベイラビリティーゾーン**: `ap-northeast-1a`
- **IPv4 CIDR ブロック**: `10.0.1.0/24`

2. 「サブネットを作成」をクリック

#### パブリックサブネット 1c の作成

同様に以下を作成：
- **サブネット名**: `udemy-aws-14days-public-subnet-1c`
- **アベイラビリティーゾーン**: `ap-northeast-1c`
- **IPv4 CIDR ブロック**: `10.0.2.0/24`

### Step 3: インターネットゲートウェイの作成

1. VPC コンソール → 「インターネットゲートウェイ」→「インターネットゲートウェイの作成」
2. **名前タグ**: `udemy-aws-14days-igw`
3. 「インターネットゲートウェイの作成」をクリック
4. 作成後、「VPC にアタッチ」をクリック
5. VPC を選択してアタッチ

### Step 4: ルートテーブルの設定

1. VPC コンソール → 「ルートテーブル」
2. 作成した VPC のメインルートテーブルを選択
3. **名前**: `udemy-aws-14days-public-rtb`に変更
4. 「ルート」タブ → 「ルートを編集」
5. 以下のルートを追加：
   - **送信先**: `0.0.0.0/0`
   - **ターゲット**: インターネットゲートウェイを選択
6. 「変更を保存」

### Step 5: セキュリティグループの作成

1. VPC コンソール → 「セキュリティグループ」→「セキュリティグループを作成」

**設定内容：**
- **セキュリティグループ名**: `udemy-aws-14days-web-sg`
- **説明**: Security group for Next.js web server
- **VPC**: `udemy-aws-14days-vpc`

**インバウンドルール：**
| タイプ | プロトコル | ポート範囲 | ソース | 説明 |
|--------|-----------|-----------|--------|------|
| SSH | TCP | 22 | 0.0.0.0/0 | SSH access |
| カスタム TCP | TCP | 3000 | 0.0.0.0/0 | Next.js server |

2. 「セキュリティグループを作成」をクリック

---

## EC2 インスタンスの作成と Next.js セットアップ

### Step 6: EC2 インスタンスの起動

1. EC2 コンソール → 「インスタンスを起動」

**設定内容：**
- **名前**: `web-1a`
- **AMI**: Amazon Linux 2023
- **インスタンスタイプ**: `t2.micro`
- **キーペア**: `udemy-aws-14days`（Day01 で作成）
- **ネットワーク設定**:
  - **VPC**: `udemy-aws-14days-vpc`
  - **サブネット**: `udemy-aws-14days-public-subnet-1a`
  - **パブリック IP の自動割り当て**: 有効化
  - **セキュリティグループ**: `udemy-aws-14days-web-sg`
- **ストレージ**: 8 GiB gp3

2. 「インスタンスを起動」をクリック

### Step 7: EC2 インスタンスへの接続

#### CloudShell から SSH 接続

```bash
# キーペアのアップロード（初回のみ）
# CloudShell のアクション → ファイルのアップロード → udemy-aws-14days.pem

# キーペアの権限設定
chmod 400 udemy-aws-14days.pem

# SSH 接続
ssh -i udemy-aws-14days.pem ec2-user@[EC2のパブリックIP]
```

### Step 8: Docker と Next.js アプリケーションのセットアップ

#### 8.1 Docker のインストール

```bash
# Docker のインストール
sudo dnf install docker -y

# Docker サービスの起動と自動起動設定
sudo systemctl start docker
sudo systemctl enable docker

# ec2-user を docker グループに追加
sudo usermod -aG docker ec2-user

# グループ変更を反映（再ログインまたは以下を実行）
newgrp docker

# Docker バージョン確認
docker --version
```

#### 8.2 Docker Compose のインストール

```bash
# Docker Compose V2（Docker Plugin版）のインストール
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose

# バージョン確認
docker compose version
```

#### 8.3 Git のインストール

```bash
sudo dnf install git -y
```

#### 8.4 アプリケーションディレクトリの準備

```bash
# アプリケーション用ディレクトリを作成
mkdir -p ~/simple-blog
cd ~/simple-blog
```

### Step 9: Next.js アプリケーションのデプロイ

#### 9.1 GitHub リポジトリからクローン（または手動作成）

**オプション A: 既存のリポジトリをクローン**
```bash
# 自分の GitHub リポジトリをクローン
git clone https://github.com/[your-username]/aws-simple-blog-nextjs.git .
```

**オプション B: 新規作成**

このドキュメントの後半にある「Next.js アプリケーションコード」セクションを参照してファイルを作成します。

#### 9.2 Dockerfile の作成

```bash
cat > Dockerfile << 'EOF'
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
ENV PORT=3000
CMD ["node", "server.js"]
EOF
```

#### 9.3 docker-compose.yml の作成

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - INSTANCE_ID=${INSTANCE_ID:-unknown}
    restart: unless-stopped
    container_name: simple-blog
EOF
```

#### 9.4 環境変数ファイルの作成

```bash
cat > .env << 'EOF'
INSTANCE_ID=web-1a
EOF
```

#### 9.5 Docker でアプリケーションを起動

```bash
# イメージをビルドしてコンテナを起動
docker compose up -d --build

# ログ確認
docker compose logs -f

# コンテナの状態確認
docker compose ps
```

### Step 10: 動作確認

1. ブラウザで `http://[EC2のパブリックIP]:3000` にアクセス
2. シンプルブログが表示されることを確認
3. コンテナが正常に起動していることを確認

```bash
# コンテナの状態確認
docker compose ps

# ログ確認
docker compose logs
```

---

## Next.js アプリケーションコード

### プロジェクト構造

```
simple-blog/
├── package.json
├── next.config.ts
├── tsconfig.json
├── tailwind.config.ts
├── postcss.config.mjs
├── app/
│   ├── layout.tsx
│   ├── page.tsx
│   ├── globals.css
│   └── api/
│       └── posts/
│           └── route.ts
└── lib/
    └── types.ts
```

### 1. package.json

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

### 2. next.config.ts

```typescript
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  output: 'standalone',
};

export default nextConfig;
```

### 3. tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "paths": {
      "@/*": ["./*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### 4. tailwind.config.ts

```typescript
import type { Config } from "tailwindcss";

const config: Config = {
  content: [
    "./pages/**/*.{js,ts,jsx,tsx,mdx}",
    "./components/**/*.{js,ts,jsx,tsx,mdx}",
    "./app/**/*.{js,ts,jsx,tsx,mdx}",
  ],
  theme: {
    extend: {},
  },
  plugins: [],
};
export default config;
```

### 5. postcss.config.mjs

```javascript
/** @type {import('postcss-load-config').Config} */
const config = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};

export default config;
```

### 6. app/globals.css

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### 7. lib/types.ts

```typescript
export interface Post {
  id: number;
  title: string;
  content: string;
  created_at: string;
}
```

### 8. app/layout.tsx

```typescript
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: "シンプルブログ",
  description: "AWS 14日間チャレンジ - Next.js版",
};

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="ja">
      <body className="bg-gray-50">{children}</body>
    </html>
  );
}
```

### 9. app/page.tsx

```typescript
import { Post } from "@/lib/types";

async function getPosts(): Promise<Post[]> {
  // Day03時点ではモックデータ（Day04でDB接続を追加）
  return [
    {
      id: 1,
      title: "AWS学習開始",
      content: "Next.jsでAWSを学習します",
      created_at: new Date().toISOString(),
    },
    {
      id: 2,
      title: "VPC構築完了",
      content: "仮想ネットワーク環境が完成しました",
      created_at: new Date().toISOString(),
    },
  ];
}

export default async function Home() {
  const posts = await getPosts();
  const instanceId = process.env.INSTANCE_ID || "Unknown";

  return (
    <div className="min-h-screen py-8">
      <div className="max-w-4xl mx-auto px-4">
        <header className="mb-8 text-center">
          <h1 className="text-4xl font-bold text-blue-600 mb-2">
            シンプルブログ
          </h1>
          <p className="text-gray-600">
            Instance: {instanceId}
          </p>
        </header>

        <div className="space-y-4">
          {posts.map((post) => (
            <article
              key={post.id}
              className="bg-white rounded-lg shadow-md p-6 hover:shadow-lg transition-shadow"
            >
              <h2 className="text-2xl font-semibold text-gray-800 mb-2">
                {post.title}
              </h2>
              <p className="text-gray-600 mb-4">{post.content}</p>
              <time className="text-sm text-gray-500">
                {new Date(post.created_at).toLocaleString("ja-JP")}
              </time>
            </article>
          ))}
        </div>

        <div className="mt-8 p-4 bg-blue-50 rounded-lg">
          <h3 className="font-semibold text-blue-900 mb-2">技術スタック</h3>
          <ul className="text-sm text-blue-800 space-y-1">
            <li>• Next.js 14 (App Router)</li>
            <li>• TypeScript</li>
            <li>• Tailwind CSS</li>
            <li>• Amazon EC2</li>
            <li>• Amazon VPC</li>
          </ul>
        </div>
      </div>
    </div>
  );
}
```

### 10. app/api/posts/route.ts（API ルート - オプション）

```typescript
import { NextResponse } from "next/server";
import { Post } from "@/lib/types";

export async function GET() {
  // Day03時点ではモックデータ
  const posts: Post[] = [
    {
      id: 1,
      title: "AWS学習開始",
      content: "Next.jsでAWSを学習します",
      created_at: new Date().toISOString(),
    },
    {
      id: 2,
      title: "VPC構築完了",
      content: "仮想ネットワーク環境が完成しました",
      created_at: new Date().toISOString(),
    },
  ];

  return NextResponse.json(posts);
}
```

---

## 📝 トラブルシューティング

### ポート 3000 にアクセスできない

1. セキュリティグループでポート 3000 が開いているか確認
2. Docker コンテナが起動しているか確認: `docker compose ps`
3. ログを確認: `docker compose logs`

### ビルドエラーが発生する

```bash
# イメージを再ビルド
docker compose down
docker compose build --no-cache
docker compose up -d
```

### コンテナが起動しない

```bash
# ログを詳細に確認
docker compose logs -f

# コンテナを再起動
docker compose restart

# 完全にクリーンアップして再構築
docker compose down -v
docker compose up -d --build
```

### Docker コマンドが権限エラー

```bash
# ec2-userをdockerグループに追加（再度）
sudo usermod -aG docker ec2-user

# 再ログインまたは
exit
ssh -i udemy-aws-14days.pem ec2-user@[EC2のパブリックIP]
```

---

## まとめ

Day3 (Next.js版) では以下を実施しました：

### 学習したキーワード

- リージョン
- アベイラビリティーゾーン（AZ）
- VPC（Virtual Private Cloud）
- サブネット
- CIDR ブロック
- インターネットゲートウェイ
- ルートテーブル
- セキュリティグループ
- **Docker & Docker Compose**
- **コンテナ化**
- **マルチステージビルド**

### 構築した環境

1. カスタム VPC とサブネット（2つの AZ）
2. インターネットゲートウェイとルート設定
3. セキュリティグループ
4. EC2 インスタンスに Docker をインストール
5. Next.js アプリケーションをコンテナ化してデプロイ

### PHP 版との主な違い

| 項目 | PHP 版 | Next.js 版 (Docker) |
|------|--------|---------------------|
| Web サーバー | Apache | Next.js (Docker コンテナ) |
| 言語 | PHP | TypeScript |
| スタイリング | 素の CSS | Tailwind CSS |
| ポート | 80 | 3000 |
| プロセス管理 | systemd (httpd) | Docker Compose |
| ビルド | 不要 | `docker compose build` 必要 |
| デプロイ | ファイルコピー | コンテナイメージ |

**次回の Day04 では、MySQL データベースとの接続を追加します。**
