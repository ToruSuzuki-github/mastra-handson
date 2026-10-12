# 02. 公式テンプレート実行編（約2〜3時間）

「まず動くものを見たい」「短時間で使ってみたを成立させたい」場合の最短ルートです。
ハンズオン本体（[03-handson/](../03-handson/)）の前にやっても、代わりにやっても構いません。

**公式テンプレート一覧: https://mastra.ai/templates**

---

## 1. Mastra テンプレートとは

公式が用意している「完成済みの Mastra プロジェクト」です。クローンして API キーを入れるだけで動きます。

- エージェント・ワークフロー・MCP サーバーの実例が一式入っている
- Mastra モノレポ内で管理され、クローンしやすいよう個別リポジトリにも自動同期されている
- 各テンプレートに README（クイックスタート手順）が付属

**なぜ最初にこれをやると良いか**

1. **10〜20分で「動くエージェント」が手に入る** — Mastra で何ができるかを最短で体感できる
2. **実務レベルのコード構成が読める** — ゼロから書く前に「完成形」を見ておくと理解が速い
3. **ユースケースが具体的** — 「自分ならどの業務に使えるか」を考え始めやすい

---

## 目次

| ファイル | 内容 | 所要目安 |
|---|---|---|
| [01-basics.md](./01-basics.md) | §2 基本の使い方。プロジェクトが作られる場所、スラッグの指定、`--llm` が効かない理由、`.agents/` の正体 | 10分 |
| [02-catalog.md](./02-catalog.md) | §3 テンプレート一覧（15件）と、それぞれに追加で必要なもの | 5分 |
| [03-model-switch.md](./03-model-switch.md) | §4-0 モデル ID を GPT-5.6 系へ差し替える（**全テンプレート共通の前作業**） | 5分 |
| [04-chat-with-database.md](./04-chat-with-database.md) | §4-1 Chat with Database ★最推奨 | 60分 |
| [05-deep-search.md](./05-deep-search.md) | §4-2 Deep Search（ネストしたワークフロー・suspend/resume・4体のエージェント協調） | 60分 |
| [06-alternatives.md](./06-alternatives.md) | §4-3 代替枠（Agent Harness / Customer Feedback Summarization）と §4-4 振り返り | 45分 |
| [07-provider-switch.md](./07-provider-switch.md) | §5（任意）モデルを他プロバイダ（Gemini など）に差し替える手順 | 任意 |
| [08-notes.md](./08-notes.md) | §6 テンプレート実行時の注意点8項目 | 随時 |
| [09-cleanup.md](./09-cleanup.md) | §8 使い終わったテンプレートの片付け（**1つ約 337MB。必ず読む**） | 10分 |

---

## 4. 推奨コース（2〜3時間）

Mastra の特徴がよく分かるものを3つ選びました。**この章の方針は次の3点**です。

| 論点 | 本章の方針 |
|---|---|
| **プロバイダ** | **OpenAI のまま**。テンプレートが OpenAI 前提で書かれているため。`.env` には `OPENAI_API_KEY` を設定する |
| **モデル ID** | **GPT-5.6 系へ差し替える**。初期値は1世代古く、割高なうえ停止期限がある（手順は [4-0](./03-model-switch.md)） |
| **ハンズオン本体** | [03-handson/](../03-handson/) の `travel-agent/` も **OpenAI（`openai/gpt-5.6-luna`）**。本編とテンプレートで同じ `OPENAI_API_KEY` を使い回せる |

> **プロバイダを替える必要は基本的にありません**: 本編もテンプレートも OpenAI で揃えてあり、
> 差し替えに伴う不具合の切り分けに時間を取られるためです。この規模なら費用は数十円程度で済みます。
> Gemini など他プロバイダに寄せたい場合のみ [5章](./07-provider-switch.md)へ。

---

## 7. どのコースを選ぶか

| やりたいこと | 推奨 |
|---|---|
| とにかく短時間で「使ってみた」を成立させたい | **本フォルダの [4-0](./03-model-switch.md) + [4-1](./04-chat-with-database.md) だけ**（65分） |
| デモ素材を早く揃えたい | [4-0](./03-model-switch.md) + [4-1](./04-chat-with-database.md) + [4-2](./05-deep-search.md)（2時間） |
| Mastra を理解して自分で作れるようになりたい | [03-handson/day1/](../03-handson/day1/) → [day2/](../03-handson/day2/)（17.5時間） |
| 両方 | 本フォルダ（2〜3時間）→ Day1 → Day2（計 19.5〜20.5時間） |

**おすすめの順序は「テンプレート → Day1 → Day2」です。** 完成形を先に見てからゼロから作ると、各パーツの役割が理解しやすくなります。

---

## 参考リンク

- テンプレート一覧: https://mastra.ai/templates
- create-mastra CLI リファレンス（`--template` の仕様）: https://mastra.ai/reference/cli/create-mastra
- テンプレート リファレンス（テンプレートの構造・自作方法）: https://mastra.ai/reference/templates/overview
- Mastra モノレポ（テンプレートの実体）: https://github.com/mastra-ai/mastra
- Chat with Database リポジトリ: https://github.com/mastra-ai/template-text-to-sql

---

次は [03-handson/](../03-handson/) へ。
