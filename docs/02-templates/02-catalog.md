# 3. テンプレート一覧（2026-08-09 時点）

| テンプレート | ページ URL | 内容 | 追加で必要なもの |
|---|---|---|---|
| **Chat with Database** | https://mastra.ai/templates/text-to-sql | 自然言語で SQLite を問い合わせる。スキーマ自己調査 → SQL 生成 → 実行 | **OpenAI キーのみ**（DB は初回起動時に自動生成）／⚠️ `catalog:` 不具合あり。4-1 の回避策が必須 |
| **Chat with PDF** | https://mastra.ai/templates/chat-with-pdf | PDF に対して質問応答 | LLM キー（+ 埋め込みモデル） |
| **Flash Cards from PDF** | https://mastra.ai/templates/flash-cards-from-pdf | PDF から学習用フラッシュカードを生成 | LLM キー |
| **CSV to Questions Generator** | https://mastra.ai/templates/csv-to-questions | CSV をエージェントに分析させる | LLM キー |
| **Customer Feedback Summarization** | https://mastra.ai/templates/customer-feedback-summarization | 顧客フィードバックを分析して実行可能な要約を出す | LLM キー |
| **Deep Search** | https://mastra.ai/templates/deep-search | 自己評価する調査エージェント。ネストしたワークフロー、suspend/resume による人間の介入、複数の専門エージェント連携 | OpenAI キー + **Exa キー**（無料枠あり。4-2 参照） |
| **Chat with YouTube** | https://mastra.ai/templates/chat-with-youtube | URL を貼ると要約・チャプター・引用付き回答 | LLM キー（+ 外部サービス） |
| **Docs Chatbot** | https://mastra.ai/templates/docs-chatbot | MCP サーバーを作ってエージェントに接続する | LLM キー／⚠️ `catalog:` 不具合あり。4-1 の回避策が必須 |
| **Agent Harness** | https://mastra.ai/templates/agent-harness | 汎用エージェント。ローカルワークスペース、シェルツール、メモリ、タスク管理、Web アクセス、定期実行 | LLM キー |
| **Agent Builder** | https://mastra.ai/templates/agent-builder | 自然言語や GUI でエージェントを構築できるテンプレート | LLM キー |
| **Google Sheet Analysis** | https://mastra.ai/templates/google-sheets-analysis | Google Sheets の分析・編集 | Composio 連携 + Google OAuth |
| **Browser Agent** | https://mastra.ai/templates/browsing-agent | Stagehand を組み込んだブラウザ操作エージェント | ブラウザ自動化サービスのキー |
| **GitHub PR Code Review** | https://mastra.ai/templates/github-pr-code-review-agent | GitHub の PR を自動レビュー | GitHub トークン |
| **Slack Agent** | https://mastra.ai/templates/slack-agent | Slack で動くエージェント | Slack アプリ設定 |
| **Mastra Code** | https://github.com/mastra-ai/mastra/tree/main/mastracode | ターミナル型 AI コーディングエージェント | LLM キー |

> 「追加で必要なもの」は各テンプレートの `.env.example` と README が正です。上表は目安として使ってください。
