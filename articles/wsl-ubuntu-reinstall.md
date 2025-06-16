---
title: "WSL Ubuntuの再インストール手順 with ClaudeCode"
emoji: "🐧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wsl", "ubuntu", "windows", "開発環境", "ClaudeCode"]
published: false
publication_name: "yaoko_tech_blog"
---

# はじめに

毎回調べているような気がするので、WSL Ubuntuの再インストール手順をまとめておきます。

## TL;DR

```bash
wsl --unregister Ubuntu
wsl --install -d Ubuntu
```


必要に応じて以下のコマンドをWSL上で実行


```bash
sudo apt update -y
sudo apt upgrade -y

# 最新のNode.js特有のコマンド
sudo apt install curl -y
sudo curl -sL https://deb.nodesource.com/setup | sudo bash -
sudo apt install npm -y

# ClaudeCodeのインストール
npm install -g @anthropic-ai/claude-code
```

はいそうです。ClaudeCodeをインストールするための手順でした。

