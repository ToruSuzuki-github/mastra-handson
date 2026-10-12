# Mastra 座学 — ハンズオン前の事前学習

**ゴール**: Mastra が「何をしてくれるフレームワークなのか」「どんな部品でできているのか」を、コードを書く前に把握する。
**対象**: Mastra を一度も使ったことがない人。TypeScript の基本（`import`、型注釈、`async/await`）が読めれば十分です。自信がなければ0章で簡単に補います。
**所要時間**: 約67分（通読。TypeScript の経験があれば0章を飛ばして64分）
**調査日**: 2026-08-11 時点（Mastra 公式ドキュメントおよび各フレームワークの公式リポジトリに基づく）。Strands Agents に関する記載のみ 2026-08-17 時点、Dify に関する記載のみ 2026-08-22 時点

この資料を読んでからハンズオン（[03-handson/](../03-handson/)）に入ると、
「今なぜこのコードを書いているのか」が分かった状態で手を動かせます。手順書は全体像を説明しないので、その穴を埋めるのが本資料の役割です。

> 💡 **急ぐ場合**: 1章・3章・10章の3つだけでも読んでおくと、ハンズオン中の迷いがかなり減ります（約21分）。

---

## 目次

| 章 | 内容 | 目安 |
|---|---|---|
| [0. 準備：この資料のコードを読むための TypeScript](./00-typescript-primer.md) | `import` / 型注釈 / `async`・`await` / 設定オブジェクト。TypeScript 経験者は飛ばす | 3分 |
| [1. なぜエージェントフレームワークが必要か](./01-why-framework.md) | ツール呼び出しのループ、自前実装で発生する作業、用語の整理 | 4分 |
| [2. Mastra とは](./02-what-is-mastra.md) | 基本情報、`ee/` ライセンス補足、採用事例、選ぶ理由 | 4分 |
| [3. 全体像：構成要素マップ](./03-architecture.md) | 部品の関係図、構成要素の一覧、ディレクトリ構成 | 8分 |
| [4. Agent — 次の行動を LLM が決める](./04-agent.md) | 定義のしかた、内部で起きていること、応答の受け取り方3通り | 5分 |
| [5. Tool — エージェントに持たせる道具](./05-tool.md) | `createTool()`、`description` の重要性、MCP | 4分 |
| [6. Workflow と、Agent との使い分け](./06-workflow.md) | ステップの連結、制御フロー（並列・分岐・ループ）、suspend / resume、判断表 | 12分 |
| [7. Memory と Storage](./07-memory-storage.md) | 記憶の4種類、thread と resource、Storage が必須な理由 | 5分 |
| [8. 開発・運用を支える仕組み](./08-devops.md) | Studio、Server、可観測性と品質評価、デプロイ | 5分 |
| [9. ハンズオンでは扱わない発展機能](./09-advanced-topics.md) | RAG / MCP / Harness / Channels など「あることだけ知っておく」一覧 | 3分 |
| [10. 他フレームワークとの比較](./10-comparison.md) | 8フレームワークの基本情報・機能マトリクス・向く場面 | 9分 |
| [11. ハンズオンとの対応 / 理解度チェック](./11-wrap-up.md) | 各章がハンズオンのどこに対応するかの表、設問10問と解答 | 5分 |

---

## 参考リンク

- Mastra 公式ドキュメント: https://mastra.ai/docs
- リファレンス: https://mastra.ai/reference
- ドキュメント全ページの索引: https://mastra.ai/llms.txt
- GitHub: https://github.com/mastra-ai/mastra

詳細なリンク集（公式チュートリアル、学習コース、書籍、Discord など）は [reference/links.md](../reference/links.md) にまとめています。

---

次は [01-setup/](../01-setup/) へ。
