# Mastra ハンズオン ドキュメント一式

Mastra を学ぶための、座学・環境構築からハンズオンまでの手順書セットです。

- **想定環境**: Windows 11 Home + VS Code + Dev Containers
- **想定 LLM**: OpenAI API（標準は `openai/gpt-5.6-luna`）。[02-templates/](./02-templates/) の公式テンプレートも OpenAI 前提なので、本編とテンプレートで同じ `OPENAI_API_KEY` を使います。Gemini は代替案です（[01-setup/02-llm.md](./01-setup/02-llm.md)）
- **ハンズオン所要時間**: 約 20.5 時間（環境構築・インストール時間を除く。うち Day 3 の3時間は任意）
- **調査日**: 2026-08-09（各社の公式ドキュメントに基づく）

---

## フォルダ構成

**番号付きフォルダは実施推奨順です。上から順に進めてください。**
各フォルダの `README.md` がその入口（目次・ゴール・所要時間）になっています。
本ファイル（全体像・進め方）の通読目安は 10分です。

| フォルダ | 内容 | 所要目安 |
|---|---|---|
| [00-overview/](./00-overview/) | **座学（事前学習）**。Mastra とは何か・全体像・他フレームワークとの比較（全12章） | 67分 |
| [01-setup/](./01-setup/) | **事前準備**。DevContainer の構築（Windows 11 Home 向け）と、LLM の選定・API キー取得 | 1.5〜2.5時間 |
| [02-templates/](./02-templates/) | **公式テンプレート実行編（最短ルート）**。完成済みプロジェクトを動かして完成形を先に見る | 2〜3時間 |
| [03-handson/](./03-handson/) | **ハンズオン本体**。[day1/](./03-handson/day1/) 基礎編（Step 0〜6）+ [day2/](./03-handson/day2/) 応用編（Step 7〜12）+ [day3/](./03-handson/day3/) フロントエンド編（Step 13〜14・任意） | 20.5時間 |

**番号なしフォルダは随時参照**です。実施順を持ちません。

| フォルダ | 内容 |
|---|---|
| [troubleshooting/](./troubleshooting/) | トラブルシューティング & FAQ。詰まったらここ。[pre-check.md](./troubleshooting/pre-check.md) は最新版で手順が通るかの事前動作確認 |
| [reference/](./reference/) | 参照資料。Mastra Studio の画面操作、[参考リンク集](./reference/links.md)、テンプレート同梱の `.agents/`、`devcontainer.json` の正典 |

---

## 進め方（推奨スケジュール）

```
【事前学習】 00-overview/     座学 67分（Mastra の全体像を先に掴む）
【事前】     01-setup/        環境構築と API キー取得（別日にやっておく）
【Day 0】    02-templates/    公式テンプレート実行 2〜3時間（完成形を先に見る）
【Day 1】    03-handson/day1/ 基礎編 6時間：Agent / Tool / Studio / API
【Day 2】    03-handson/day2/ 応用編 11.5時間：Memory / Workflow（分岐・ループ・承認待ち）/ マルチエージェント / 観測 / 評価
【Day 3】    03-handson/day3/ フロントエンド編 3時間（任意）：型付きクライアント / Next.js 同居
【随時】     troubleshooting/ トラブルシューティング
             reference/       Studio の操作・参考リンク集・devcontainer.json の正典
```

**Day 3 まで終えればハンズオンは完了**です。

2日連続で取るのが難しければ、基礎編を「Step 1〜3」「Step 4〜6」の2回（各3時間）に分割しても成立します。応用編も同様です。

**Day 3 は独立性が高い**ため、後日に回しても省いても本編は成立します。ただし「Mastra は TypeScript
ネイティブでフロントエンドと相性がいい」を実感できるのはここだけです。

**Step ごとの時間配分**は [03-handson/day1/](./03-handson/day1/)・[03-handson/day2/](./03-handson/day2/)・
[03-handson/day3/](./03-handson/day3/) の各 README を、**何を作るのか**は
[03-handson/](./03-handson/) を参照してください。

### 時間が取れない場合

**[02-templates/04-chat-with-database.md](./02-templates/04-chat-with-database.md)（Chat with Database）だけなら 60分で完了**します。必要なのは OpenAI キー1つだけ（本編と同じキー。サンプルデータは初回起動時に自動生成）で、「自然言語で DB を検索する」という用途を想像しやすいデモがそのまま手に入ります。

まずここだけ実施して手応えを確かめてから [03-handson/](./03-handson/) に進む、という進め方も有効です。

---

## 重要な前提知識（先に読んでおく）

### Mastra とは

TypeScript で AI エージェントを構築するためのフレームワークです。主な構成要素は5つ。
以下は要点だけの早見表です。**全体像の解説は [00-overview/](./00-overview/)（座学 67分）にまとめてあります。ハンズオン前に読んでください。**

| 要素 | 役割 |
|---|---|
| **Agent** | LLM とツールを使って自律的にタスクを解く主体 |
| **Tool** | エージェントが呼び出せる関数（API 呼び出し、DB 検索など） |
| **Workflow** | 手順が決まっている処理をグラフとして明示的に組む仕組み |
| **Memory** | 会話履歴やユーザー固有情報を保持する仕組み |
| **Studio** | ローカルで動く GUI。エージェントとの対話・トレース確認ができる |

Agent と Workflow の使い分けが最初の理解ポイントです。公式は「**手順が事前に決まっていないタスクは Agent、決まっている多段処理は Workflow**」と説明しています。

### 必須要件

- **Node.js 22.13.0 以上**（Mastra v1 の要件）。ハンズオンでは現行 Active LTS の **Node 24** を使用（[Node.js のリリース一覧](https://nodejs.org/en/about/previous-releases)。Mastra の要件は下限のみで、上限はありません）
- モデルプロバイダの API キー

### 最初にハマりやすい3点

1. **`tsconfig.json` の `module` / `moduleResolution`** — 公式指定（`ES2022` / `bundler`）から外すと解決エラーになる
2. **ツールは必ず `createTool()` で定義する** — 素のオブジェクトで書くとエラーにならないまま実行されない
3. **ローカルファイルの import は拡張子付き** — `'../tools/weather-tool.ts'` のように書く

---

## 公式ドキュメント（起点）

- Get Started: https://mastra.ai/docs
- Reference: https://mastra.ai/reference
- GitHub: https://github.com/mastra-ai/mastra

> 公式は 2026-08 に **Guides セクションを廃止**し、Quickstart やチュートリアルを Docs 本体と
> ブログへ再編しました。旧 `mastra.ai/guides/...` の URL はすべて転送されるだけなので、
> 現在の掲載先は [reference/links.md](./reference/links.md) を参照してください。

詳細なリンク集は [reference/links.md](./reference/links.md) にまとめています。
