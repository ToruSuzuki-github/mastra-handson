# ハンズオン 応用編（Day 2 / 約11.5時間）

Memory・Workflow（分岐・ループ・承認待ちを含む）・マルチエージェント・観測・評価という、実務投入に必要な要素を扱います。

**ゴール**: 単発の受け答えではなく、記憶を持ち・手順を守り・役割分担するエージェントシステムを組める状態。

**前提**: [day1/](../day1/) の `travel-agent` プロジェクトが動いていること。

---

## Step 一覧

| Step | 内容 | 時間 | 手順 |
|---|---|---|---|
| 7 | メモリ：会話を記憶させる（Storage / Memory） | 90分 | [07-memory.md](./07-memory.md) |
| 8 | ワークフロー：決まった手順を組む（直列・分岐・ループ・承認待ち・エージェント連携） | 240分 | [08-workflow.md](./08-workflow.md) |
| 9 | マルチエージェント：役割分担させる | 60分 | [09-multi-agent.md](./09-multi-agent.md) |
| 10 | 観測とトレース：エージェントの中身を見る（トレース・メトリクス・観測機能の使い分け） | 90分 | [10-observability.md](./10-observability.md) |
| 11 | 評価：出力を採点して、変更の前後を比べる（Scorer・Dataset・Experiment・CI 用スクリプト） | 180分 | [11-evals.md](./11-evals.md) |
| 12 | ビルドと成果物確認 | 30分 | [12-build.md](./12-build.md) |

---

## Day 2 完了チェックリスト

- [ ] メモリを設定し、複数ターンで文脈が引き継がれることを確認した
- [ ] ワークフローを作り、Studio のグラフビューで実行を追えた
- [ ] `.branch()` で処理を振り分け、条件を書き換えて「true の枝はすべて動く」ことを確かめた
- [ ] `.dountil()` でループを組み、条件が最初から成立していても本体が1回動く（後判定）ことを確かめた
- [ ] `suspend()` で止めたワークフローを Studio から再開し、承認と差し戻しの両方を試した
- [ ] エージェントにワークフローを持たせ、登録しただけでは呼ばれないことを確かめた
- [ ] Agent と Workflow の使い分けを説明できる
- [ ] サブエージェントによる役割分担を試した
- [ ] トレースからトークン消費とコストを概算できた
- [ ] DuckDB を足して Metrics 画面に数字を出し、1回あたりの概算コストを計算できた
- [ ] Traces / Metrics / Logs / Feedback / Scorers / Datasets・Experiments が答える問いの違いを説明できる
- [ ] 自作の Scorer（ルールベースと LLM 採点）を作り、ライブ採点の点数と理由を Studio で読めた
- [ ] Dataset を作り、プロンプト変更の前後を Experiment で比べて、質問ごとの差まで読めた
- [ ] `runEvals` で合否（`verdict`）を返すスクリプトを作り、失敗時に終了コードが 1 になることを確かめた
- [ ] ビルドが通ることを確認した

---

## さらに先へ（ハンズオン後の自習用）

興味のあるものを選んでください。

> 公式は 2026-08 に Guides セクションを廃止し、これらのチュートリアルを**ブログ記事と
> ドキュメント本体へ再編**しました。ブログ側は読み物であり、手順書としての更新は期待できません。

| テーマ | URL |
|---|---|
| RAG: Research Assistant | https://mastra.ai/blog/building-a-research-paper-assistant-with-rag |
| MCP Server: Notes MCP Server | https://mastra.ai/blog/building-a-notes-mcp-server |
| Workflows: AI Recruiter | https://mastra.ai/blog/building-an-ai-recruiter |
| Building a Coding Agent | https://mastra.ai/blog/building-a-coding-agent |
| Channels: Slack Assistant | https://mastra.ai/integrations/channels/slack |
| テンプレート集 | https://mastra.ai/templates |

---

次は [day3/](../day3/)（フロントエンド編 3時間）へ。ここまでで育てたエージェントに、
TypeScript で書いたフロントエンドを繋ぎます。**Day3 は任意なので、ここで終えても成立します。**
