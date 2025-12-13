# AWS SSM 経由で EC2 に SSH 接続する方法

## 概要

AWS Systems Manager (SSM) Session Manager を使用することで、セキュリティグループで SSH ポート(22)を開放せずに、EC2 インスタンスに安全に接続できます。

## メリット

- ✅ セキュリティグループでインバウンド SSH ポートを開放不要
- ✅ 踏み台サーバー不要
- ✅ 接続ログが CloudTrail に記録される
- ✅ パブリック IP アドレス不要（プライベートサブネットのインスタンスにも接続可能）

## 前提条件

### 1. EC2 インスタンス側の設定

- SSM Agent がインストールされている（Amazon Linux 2/Ubuntu 20.04 以降はデフォルトでインストール済み）
- EC2 インスタンスに `AmazonSSMManagedInstanceCore` ポリシーがアタッチされた IAM ロールが設定されている

### 2. ローカル環境の準備

- AWS CLI がインストールされている
- Session Manager plugin がインストールされている
  ```powershell
  # インストール確認
  session-manager-plugin --version
  ```
- SSH 秘密鍵ファイル（.pem）がローカルに保存されている

### 3. IAM ユーザー/ロールの権限

以下の権限が必要：

- `ssm:StartSession` （ドキュメント: AWS-StartSSHSession）
- `ssm:TerminateSession`

## SSH Config 設定

設定ファイルのパス: `C:/Users/nt12979/.ssh/config`

```ssh-config
Host my-ec2
    HostName i-0xxxxxxxxxxxx
    User ec2-user
    Port 22
    ProxyCommand powershell.exe -Command "$env:HTTPS_PROXY = 'http://your-proxy-server:port'; $env:AWS_PROFILE = 'your-profile'; aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p' --region us-east-1"
    IdentityFile "C:\Users\YourUserName\.ssh\your-key.pem"
```

## 各パラメータの説明

| パラメータ     | 説明                             | 備考                                         |
| -------------- | -------------------------------- | -------------------------------------------- |
| `Host`         | SSH 接続時に使用するエイリアス名 | 任意の名前を設定可能                         |
| `HostName`     | EC2 インスタンス ID              | `i-` から始まる ID                           |
| `User`         | EC2 にログインするユーザー名     | Amazon Linux: `ec2-user`<br>Ubuntu: `ubuntu` |
| `Port`         | SSH ポート番号                   | 通常は `22`                                  |
| `ProxyCommand` | SSM 経由で接続するためのコマンド | `%h` は HostName、`%p` は Port に置換される  |
| `IdentityFile` | SSH 秘密鍵のパス                 | `.pem` ファイルのフルパス                    |

### ProxyCommand の詳細

```powershell
powershell.exe -Command "
  $env:HTTPS_PROXY = 'http://your-proxy-server:port';  # 企業プロキシ設定（不要な場合は削除）
  $env:AWS_PROFILE = 'your-profile';                    # AWSプロファイル指定
  aws ssm start-session
    --target %h                                          # インスタンスID
    --document-name AWS-StartSSHSession                  # SSMドキュメント
    --parameters 'portNumber=%p'                         # ポート番号
    --region us-east-1                                   # AWSリージョン
"
```

## 接続方法

```bash
# SSH接続
ssh my-ec2

# SCPでファイル転送
scp local-file.txt my-ec2:/home/ec2-user/

# ポートフォワーディング
ssh -L 8080:localhost:80 my-ec2
```

## トラブルシューティング

### エラー: "An error occurred (TargetNotConnected)"

**原因**: EC2 インスタンスが SSM Agent に登録されていない

**解決方法**:

1. EC2 インスタンスに SSM 用の IAM ロールがアタッチされているか確認
2. SSM Agent が起動しているか確認
   ```bash
   sudo systemctl status amazon-ssm-agent
   ```

### エラー: "SessionManagerPlugin is not found"

**原因**: Session Manager plugin がインストールされていない

**解決方法**:

```powershell
# Windowsの場合
# https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html
# からインストーラーをダウンロードしてインストール
```

### エラー: Permission denied (publickey)

**原因**: SSH 鍵のパスが間違っているか、権限が正しくない

**解決方法**:

```powershell
# 鍵ファイルの権限を確認（Windowsの場合、ファイルプロパティで確認）
icacls "C:\Users\nt12979\.ssh\hayata-us-east-1-key.pem"
```

## 注意点

⚠️ **修正ポイント**:
元の設定の `ProxyCommand` 内で `--` の代わりに `-` が使われている箇所がありますが、正しくは以下のように統一すべきです：

```bash
# ❌ 間違い
aws ssm start-session -- target %h -- document-name ...

# ✅ 正しい
aws ssm start-session --target %h --document-name ...
```

## 参考リンク

- [AWS Systems Manager Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
- [Session Manager を使用して SSH 接続を有効にする](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-getting-started-enable-ssh-connections.html)
