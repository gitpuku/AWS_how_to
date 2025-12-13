```json
{
  "name": "A2A Dockerfile",
  "build": {
    "context": "..",
    "dockerfile": "../Dockerfile"
  },
  "postCreateCommand": "npm config set prefix ~/.npm-global && npm install -g @anthropic-ai/claude-code && ln -sf ~/.npm-global/lib/node_modules/@anthropic-ai/claude-code/cli.js ~/.npm-global/bin/claude-code && mkdir -p ~/.claude && echo '{\"env\":{\"CLAUDE_CODE_USE_BEDROCK\":\"true\",\"AWS_REGION\":\"ap-southeast-1\",\"ANTHROPIC_MODEL\":\"apac.anthropic.claude-sonnet-4-20250514-v1:0\"}}' > ~/.claude/settings.json && echo 'export PATH=\"$PATH:$HOME/.npm-global/bin\"' >> ~/.bashrc",
  "mounts": [
    "source=${localEnv:HOME}/.aws,target=/opt/app-root/.aws,type=bind,consistency=cached"
  ],
  "remoteUser": "root",
  "containerUser": "root",
  "remoteEnv": {
    "PATH": "${containerEnv:PATH}:${containerEnv:HOME}/.npm-global/bin",
    "CLAUDE_CODE_USE_BEDROCK": "true",
    "AWS_REGION": "us-east-1",
    "ANTHROPIC_MODEL": "apac.anthropic.claude-sonnet-4-20250514-v1:0"
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "GitHub.copilot",
        "GitHub.copilot-chat",
        "donjayamanne.python-extension-pack",
        "ms-toolsai.jupyter"
      ]
    }
  }
}
```

## EC2 ユーザーデータスクリプト

```bash
#!/bin/bash

# ログ出力の設定
exec > >(tee /var/log/user-data.log|logger -t user-data -s 2>/dev/console) 2>&1

# システムアップデート
sudo yum update -y

# 基本パッケージのインストール
sudo yum install -y docker git nodejs npm

# Dockerサービスの設定
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user

# npm権限問題を解決とclaudeインストール
sudo -u ec2-user bash << 'USERSCRIPT'
cd /home/ec2-user

# npm設定
mkdir -p ~/.npm-global
npm config set prefix "~/.npm-global"

# .bashrcにPATH設定を追加(重複チェック付き)
if ! grep -q "npm-global" ~/.bashrc; then
  echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc
fi

# 現在のセッションでPATHを設定
export PATH="$HOME/.npm-global/bin:$PATH"

# claudeのインストール
npm install -g @anthropic-ai/claude-code
USERSCRIPT

# Claude設定ディレクトリと設定ファイルの作成
sudo -u ec2-user mkdir -p /home/ec2-user/.claude/
sudo -u ec2-user cat > /home/ec2-user/.claude/settings.json << 'EOF'
{
  "env": {
    "CLAUDE_CODE_USE_BEDROCK": "true",
    "AWS_REGION": "ap-southeast-1",
    "ANTHROPIC_MODEL": "apac.anthropic.claude-sonnet-4-20250514-v1:0"
  }
}
EOF

# 完了確認
echo "Setup completed at $(date)" > /home/ec2-user/setup-complete.txt
chown ec2-user:ec2-user /home/ec2-user/setup-complete.txt

# 動作確認(新しいシェルで実行)
sudo -u ec2-user bash -c 'source ~/.bashrc && claude --version' > /home/ec2-user/claude-version.txt 2>&1
chown ec2-user:ec2-user /home/ec2-user/claude-version.txt

echo "User data script completed successfully"
```
