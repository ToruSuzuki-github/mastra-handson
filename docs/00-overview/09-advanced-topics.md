# 9. ハンズオンでは扱わない発展機能（3分）

Mastra v1 の守備範囲は広く、ハンズオン20.5時間で扱えるのはその一部です。
**「そういう機能が存在する」ことだけ知っておけば十分**です。必要になったときに公式を読めば追いつけます。

| 機能 | 一行説明 | 公式 |
|---|---|---|
| RAG / ベクトル検索 | 社内文書を埋め込み化して検索し、その結果を根拠に回答させる | https://mastra.ai/reference/rag/overview |
| MCP | Model Context Protocol 対応の外部ツールを接続する | https://mastra.ai/docs/connections/mcp |
| Subagents & Skills | エージェントに専門エージェントや定型スキルを持たせて分担させる | https://mastra.ai/docs/subagents / https://mastra.ai/docs/skills |
| Harness | 何日も動き続けるエージェントの基盤。永続エージェント・バックグラウンドタスク・目標・スケジュール・シグナル | https://mastra.ai/docs/harness/agent-controller |
| Channels | Slack / Teams / Discord / Telegram などのチャットからエージェントを使う | https://mastra.ai/docs/channels |
| Browser | エージェントにブラウザ操作をさせる（操作の録画・閲覧機能つき） | https://mastra.ai/docs |
| Sandbox | 隔離環境でのコード実行・ファイルシステム操作 | https://mastra.ai/docs |
| A2A / ACP | エージェント同士を接続するためのプロトコル対応 | https://mastra.ai/docs |
| Guardrails / Processors | 入出力に検閲・整形・安全確認の処理を挟む | https://mastra.ai/docs |
| Auth | Auth0 / Clerk / Firebase / Okta / Supabase などによる認証 | https://mastra.ai/docs |
| Scheduled Workflows | ワークフローを定期実行する | https://mastra.ai/docs |

> 💡 ドキュメントは **Build（Agents / Workflows / Harness / Memory）→ Extend（Subagents / Sandbox / Channels / Browser / Connections）→ Deploy（Storage / Server / Studio / Auth）→ Observe（Traces / Evals / Datasets）** という4分類で並んでいます。この地図を頭に入れておくと目的のページに辿り着きやすくなります。
