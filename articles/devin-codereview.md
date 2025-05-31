---
title: "DevinによるLabelドリブンなコードレビューの実装"
emoji: "🤖"
type: "idea"
topics: ["ai", "生成ai", "技術選定", "開発効率化", "llm"]
published: false
publication_name: "yaoko_tech_blog"
---

## はじめに

コードレビューは開発プロセスにおいて重要な工程ですが、レビュアーの負荷や時間的制約などの課題があります。本記事では、AI エンジニア「Devin」を活用して、GitHub のラベル機能をトリガーとした自動コードレビューシステムの実装方法を紹介します。

## システム概要

この実装では以下の機能を提供します：

- 🏷️ **ラベルトリガー**: PRに `devin-review` ラベルを付けることでレビューを開始
- 🔄 **差分レビュー**: 新しいコミットが追加された際の自動再レビュー
- 🔁 **セッション継続**: 既存のDevinセッションを再利用してコンテキストを保持
- 🛡️ **無限ループ防止**: Devin自身が作成したPRはレビュー対象外

## 前提条件

- Devin APIキーの取得
- GitHub Actionsの利用権限
- リポジトリへの適切なアクセス権限

## 実装

### GitHub Actionsワークフロー

`.github/workflows/devin-review.yml` として以下のワークフローを作成します：

```yaml
name: Devin Code Review

on:
  pull_request:
    types:
      - labeled
      - synchronize
  issue_comment:
    types: 
      - created
      - edited
  workflow_dispatch:

jobs:
  code-review:
    runs-on: ubuntu-latest
    if: github.event.label.name == 'devin-review'
    permissions:
      contents: read
      pull-requests: write
    env:
      DEVIN_API_KEY: ${{ secrets.DEVIN_API_KEY }}
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - name: Skip if PR is created by Devin
        run: |
          if [ "${{ github.event.pull_request.user.login }}" = "devin-ai-integration[bot]" ]; then
            echo "Skipping review as PR is created by Devin"
            exit 0
          fi
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          ref: ${{ github.event.pull_request.head.sha }}

      - name: Install jq
        run: |
          if ! command -v jq &>/dev/null; then
            sudo apt-get update && sudo apt-get install -y jq
          fi

      - name: Devin review combined
        run: |
          session_comment=$(gh pr view "${{ github.event.number }}" --json comments --jq '.comments[] | select(.body | contains("Devin Session ID:")) | .body' | head -n 1)

            create_session() {
              local prompt="$1"

              local response
              # JSONエスケープ処理
              escaped_prompt=$(echo "$prompt" | jq -Rs .)
              response=$(curl -s -w "\n%{http_code}" -X POST "https://api.devin.ai/v1/sessions" \
                          -H "Authorization: Bearer $DEVIN_API_KEY" \
                          -H "Content-Type: application/json" \
                          -d "{\"prompt\": $escaped_prompt}")

              local http_status
              http_status=$(echo "$response" | tail -n 1)
              echo "Request info: status code = $http_status"

              if [ "$http_status" -ne 200 ]; then
                exit 1
              fi

              local http_body
              http_body=$(echo "$response" | sed '$d')

              local session_id
              session_id=$(echo "$http_body" | jq -r '.session_id')
              if [ -z "$session_id" ] || [ "$session_id" = "null" ]; then
                echo "Failed to get session ID, body = $http_body"
                exit 1
              fi
              gh pr comment "${{ github.event.number }}" --body "Devin Session ID: $session_id"
            }

            update_session() {
              local session_id="$1"
              local message="$2"
              local response
              # JSONエスケープ処理
              escaped_message=$(echo "$message" | jq -Rs .)
              response=$(curl -s -w "\n%{http_code}" -X POST "https://api.devin.ai/v1/session/${session_id}/message" \
                        -H "Authorization: Bearer $DEVIN_API_KEY" \
                        -H "Content-Type: application/json" \
                        -d "{\"message\": $escaped_message}")

              local http_status
              http_status=$(echo "$response" | tail -n 1)
              echo "Request info: status code = $http_status"

              if [ "$http_status" -ne 200 ]; then
                exit 1
              fi
            }

            # 指示内容をマークダウンファイルから読み込む
            instructions=$(cat .github/devin_review_instructions.md | jq -Rs .)

            if [ "${{ github.run_attempt }}" -gt 1 ]; then
              # 手動での再試行
              base_prompt="${{ github.repositoryUrl }} の PR number ${{ github.event.number }}をもう一度、0からレビューしてください。"
              additional="レビュー範囲については、すべての差分を対象にしてください。"
              prompt="$base_prompt $instructions $additional"

              if [ -z "$session_comment" ]; then
                create_session "$prompt"
              else
                session_id=$(echo "$session_comment" | sed -E 's/.*Devin Session ID: (devin-[a-zA-Z0-9]+).*/\1/')
                update_session "$session_id" "$prompt"
              fi

            elif [ "${{ github.event.action }}" = "synchronize" ] && [ "${{ github.run_attempt }}" -eq 1 ]; then
              # 新規コミットなどが入った場合の自動レビュー
              if [ -z "$session_comment" ]; then
                exit 0
              fi
              session_id=$(echo "$session_comment" | sed -E 's/.*Devin Session ID: (devin-[a-zA-Z0-9]+).*/\1/')
              base_prompt="${{ github.event.pull_request.html_url }} に変化がありました。再度レビューしてください。"
              additional="レビュー範囲については、基本あなたが前回見たCommitから、最新のCommitまでの範囲で十分です。ただし gh pr diff ${{ github.event.number }} などを使い、PRで変更のあったファイルに限定しレビューしてください。（ただ単にmasterをPRにマージした = ブランチを最新化しただけである場合は、merge commitの差分をレビューしないようにしてください。）"
              prompt="$base_prompt $instructions $additional"
              update_session "$session_id" "$prompt"

            else
              if [ -z "$session_comment" ]; then
                base_prompt="${{ github.event.pull_request.html_url }} をレビューしてください。"
                additional=""
                prompt="$base_prompt $instructions $additional"
                create_session "$prompt"
              else
                exit 0
              fi
            fi
```

### レビュー指示ファイル

`.github/devin_review_instructions.md` にDevin向けの詳細な指示を記述します：

```markdown
# Devinレビュー指示

## 言語
- レビューにコメントする際は日本語を利用すること。

## Knowledgeの活用
- レビュー時は、最大限Knowledgeを利用すること。
- .gemini/styleguide.mdを参照しレビューすること。
- 更新されたファイルタイプに応じて `.cursor/rules/*.mdc` を参照してレビューすること。

## Plan作成とチェックリスト
- Plan を作るときに、ファイルごとにチェックリストを作り、既存の指摘も列挙する。
- その上で、他の観点で指摘できることがないかチェックする（すでに指摘されているものは指摘しない）。

## 完了時の対応
- レビューが完了したら、必ずその旨を GitHub 上でコメントする。
- 問題がなかった箇所は、手短に列挙する（追加の理由説明は不要）。

## 状況の確認
- コメントのやり取りを再確認し、最新の状況を把握する。

## 重複防止
- 既に指摘した内容は再度指摘しない（ghコマンドで既存の指摘を確認する）。

## GitHub連携
- devin.ai 上での返信だけでなく、必ず gh コマンドを利用して GitHub 上にコメントを投稿する。
- ただし、レビューが完了した旨のメッセージを重複投稿してはならない。
```

## 設定手順

### 1. Secretsの設定

GitHub リポジトリの Settings > Secrets and variables > Actions で以下を設定：

- `DEVIN_API_KEY`: Devin APIキー
- `GITHUB_TOKEN`: 自動で設定済み（追加設定不要）

### 2. 権限の設定

ワークフローファイルで以下の権限を設定：

```yaml
permissions:
  contents: read        # リポジトリ内容の読み取り
  pull-requests: write  # PRへのコメント投稿
```

### 3. ワークフローの動作パターン

#### パターン1: 初回レビュー
1. PRに `devin-review` ラベルを付与
2. 新しいDevinセッションを作成
3. PR全体をレビュー

#### パターン2: 追加コミット時の再レビュー
1. 既存のPRに新しいコミットをプッシュ
2. 既存のDevinセッションを継続
3. 差分のみをレビュー

#### パターン3: 手動再実行
1. GitHub Actionsから手動でワークフローを再実行
2. PRを0からレビュー

## 技術的なポイント

### セッション管理

```bash
# セッションIDをPRコメントに保存
gh pr comment "${{ github.event.number }}" --body "Devin Session ID: $session_id"

# 既存セッションの検索
session_comment=$(gh pr view "${{ github.event.number }}" --json comments --jq '.comments[] | select(.body | contains("Devin Session ID:")) | .body' | head -n 1)
```

### JSONエスケープ処理

```bash
# プロンプトの安全なエスケープ
escaped_prompt=$(echo "$prompt" | jq -Rs .)
```

### 無限ループ防止

```bash
# Devin自身が作成したPRはスキップ
if [ "${{ github.event.pull_request.user.login }}" = "devin-ai-integration[bot]" ]; then
  echo "Skipping review as PR is created by Devin"
  exit 0
fi
```

## 使用方法

1. **レビュー開始**: PRに `devin-review` ラベルを付与
2. **レビュー結果確認**: DevinがPRにコメントとしてレビュー結果を投稿
3. **追加レビュー**: 新しいコミットをプッシュすると自動的に差分レビューが実行

## メリット・効果

### ✅ メリット
- **24時間対応**: 時間を問わずレビューが可能
- **一貫性**: 同じ基準でのレビューを保証
- **学習効果**: チーム全体のコード品質向上
- **負荷軽減**: 人的リソースの有効活用
 

### ⚠️ 注意点
- **コスト**: Devin APIの利用料金
- **精度**: 完全にヒューマンレビューを代替するものではない
- **セキュリティ**: APIキーの適切な管理が必要

## まとめ

Devinを活用したラベルドリブンなコードレビューシステムにより、効率的で継続的なコード品質向上を実現できます。特に以下のような場面で威力を発揮します：

- 深夜や休日のPR対応
- 大量のPRが発生する場合のスクリーニング
- 新人エンジニアの学習サポート

今後はレビュー結果の分析やチーム向けのダッシュボード作成なども検討しており、より包括的な開発支援システムへの発展を目指しています。

## 参考資料

- [Devin API Documentation](https://api.devin.ai/docs)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub CLI Reference](https://cli.github.com/manual/)
- [Devinにコードレビューをさせ、コード品質と開発速度を同時に高める話](https://zenn.dev/globis/articles/28e47f8107c5b5)