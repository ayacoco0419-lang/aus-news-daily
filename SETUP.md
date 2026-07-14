# 豪州ニュース デイリーダイジェスト — GitHub Actions セットアップ手順

## 概要

Claude Code を GitHub Actions 上で実行し、毎朝 06:00 AEST（日本時間 05:00）に豪州金融ニュースを収集・要約して Notion に保存する自動化ワークフローです。

---

## 前提条件

- GitHubアカウントとリポジトリ（パブリック／プライベートどちらでも可）
- Anthropic API キー（有料プラン推奨: Claude Sonnet 以上）
- Notion Integration Token（下記参照）

---

## Step 1: Notion Integration の作成

1. https://www.notion.so/my-integrations にアクセス
2. 「新しいインテグレーション」をクリック
3. 名前（例: `aus-news-bot`）を入力して作成
4. 表示された **Internal Integration Token** をコピー（後でGitHub Secretsに登録）
5. Notionで親ページ（ID: `354e36114e2381c5a8f0d5b8be95113f`）を開き、右上メニュー → 「接続を追加」→ 作成したインテグレーションを選択

---

## Step 2: GitHubリポジトリへのファイル配置

このリポジトリのファイル構成:

```
.
├── .github/
│   └── workflows/
│       └── aus-news-daily.yml   # GitHub Actions ワークフロー
├── .claude/
│   └── settings.local.json      # ローカル開発用（GitHubではsettings.jsonを動的生成）
├── prompts/
│   └── aus-news-daily.md        # Claude Codeへの指示プロンプト
└── SETUP.md                     # このファイル
```

---

## Step 3: GitHub Secrets の登録

リポジトリ → Settings → Secrets and variables → Actions → 「New repository secret」で以下を登録:

| Secret名 | 値 |
|---|---|
| `ANTHROPIC_API_KEY` | Anthropic APIキー（`sk-ant-...`） |
| `NOTION_TOKEN` | Notion Integration Token（`secret_...`） |

---

## Step 4: リポジトリへのpush

```bash
cd /home/user
git init
git add .github/ .claude/ prompts/ SETUP.md
git commit -m "Add aus-news-daily GitHub Actions workflow"
git remote add origin https://github.com/<your-username>/<your-repo>.git
git push -u origin main
```

---

## Step 5: 動作確認（手動実行）

1. GitHubリポジトリ → Actions タブ
2. 「豪州ニュース デイリーダイジェスト」ワークフローを選択
3. 「Run workflow」→「Run workflow」でテスト実行
4. ログで進捗確認（約10〜30分）
5. Notionに「【豪州ニュース】YYYY-MM-DD（合計X件）」ページが作成されているか確認

---

## スケジュール

- 自動実行: 毎日 **06:00 AEST**（UTC 20:00）
- 夏時間（AEDT, UTC+11）時は 07:00 AEDTに相当

cron設定を変更する場合は `.github/workflows/aus-news-daily.yml` の `cron:` 行を編集してください。

---

## トラブルシューティング

| 症状 | 対処 |
|---|---|
| `ANTHROPIC_API_KEY` エラー | GitHub Secretsの登録を確認 |
| Notion書き込み失敗 | Integration がページに接続されているか確認 |
| 記事0件 | WebSearch/WebFetchのアクセス制限を確認（GitHub Actionsは通常制限なし） |
| タイムアウト | `timeout-minutes` を増やす（現在60分） |
