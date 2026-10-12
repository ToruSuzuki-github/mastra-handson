# 4-2. Deep Search（60分）

**なぜこれか**

- **Mastra の高度な機能が一度に見られる** — ネストしたワークフロー、suspend/resume、4体のエージェント協調が1本に詰まっている
- **4-1 とちょうど裏返し** — 4-1 は「エージェント1体・ワークフロー無し」、4-2 は「ワークフロー主役・ツール無し」。2つやると Mastra の2つの設計軸が対比で理解できます

**このテンプレートの構造**

単体のエージェントではありません。**ワークフローが主役で、4体のエージェントを手順の各所で呼び分ける**
構成です。ツール（`createTool`）は1つもありません。

```
deep-search（ワークフロー）
├ clarify-intent ──────────── Intent Clarifier        質問を3つ作って suspend
├ dountil（最大4周・十分と判断されるまで繰り返す）
│   └ search-pass（サブワークフロー）
│       ├ generate-queries ── Research Planner        検索クエリを3〜5本立案
│       ├ search ──────────── エージェント無し。Exa を直接呼ぶ
│       └ evaluate-results ── Search Result Evaluator 十分か・不足点を返す
└ finalize-answer ─────────── Answerer                最終回答を生成
```

設計上の要点は3つです。

| 要点 | 中身 |
|---|---|
| **データは戻り値でなく共有 state で渡す** | 各ステップの `outputSchema` はほぼ空（`z.object({})`）。代わりに `stateSchema`（`initialQuery` / `clarifiedIntent` / `expandedQueries` / `searchResults` / `gaps` / `answerIsSatisfactory`）を `setState()` で更新して次へ渡す |
| **ループの終了はエージェントの自己評価** | `evaluate-results` が返す `answerIsSatisfactory` と `iterationCount >= 4` の OR で抜ける。**最大4周** |
| **エージェントはチャット相手ではない** | 4体とも `mastra.getAgent()` でワークフローから呼ばれるだけ。Studio でチャットする対象ではない |

**必要なキーは2つ**です。着手前に Exa のキーを取得しておいてください。

| 環境変数 | 用途 | 取得先 |
|---|---|---|
| `OPENAI_API_KEY` | 4体のエージェントが使う LLM | https://platform.openai.com |
| `EXA_API_KEY` | Web 検索とページ本文の取得 | https://dashboard.exa.ai/api-keys |

> **Exa とは**: AI エージェント向けの Web 検索 API です。キーワード一致ではなく意味ベースで検索し、
> **ヒットしたページの要約まで一度に返す**ため、「検索 → 各ページをスクレイピング」を自前で書かずに済みます。

> **💡 料金は気にしなくて構いません。** カード登録不要で、登録時の無償クレジット $20 分だけで
> **約60〜100回**の調査ができます。内訳は下の折りたたみを参照。

<details>
<summary>💰 <b>補足：Exa の料金と消費量</b>（費用の目安を知りたい人向け）</summary>

**課金の仕組み**（2026-08-15 時点、[公式 Billing](https://exa.ai/docs/reference/billing) / [料金ページ](https://exa.ai/pricing)）

Deep Search が使う **Search API はサブスクリプション不要の従量課金**で、無料枠は有料プラン加入を
前提としません。

| 項目 | 内容 |
|---|---|
| 登録時の無償クレジット | **$20 分**（"$20 in free credits on signup"）。**支払うのではなく配られる残高** |
| 毎月の無償クレジット | Free Tier で**$10 分**が追加される |
| 支払い方法の登録 | **不要**（"No payment method is required"）。カード無しでキーを取得できる |
| 使い切ったとき | **課金されず API リクエストが止まる**。使い過ぎて請求が届く事故は起きない |

**このテンプレートでの消費量**

| 項目 | 値 | 根拠 |
|---|---|---|
| 1周あたりのクエリ数 | 3〜5 | `generate-queries` の出力スキーマが `.min(3).max(5)` |
| 最大の周回数 | 4 | `.dountil()` の `iterationCount >= 4` |
| 1回の調査のリクエスト数 | **12〜20** | 上記の積 |
| 1クエリの単価 | **約 $0.017** | 検索 $0.007 + 要約10件 $0.010（`numResults: 10`・要約付き） |
| **1回の調査の費用** | **約 $0.20〜$0.34** | |

登録時の $20 分で約60〜100回、毎月の $10 分で約30〜50回に相当します。ハンズオンで数回試す分には
**自己負担ゼロ**で収まります。

> **⚠️ Websets と混同しないこと。** Exa の別製品 **Websets**（リスト作成ツール）は Free / Core $49月 /
> Pro $449月 の**サブスクリプション制**で、「Exa は有料」という情報はたいていこちらを指しています。
> 本テンプレートが叩く `exa.search()`（Search API）とは料金体系が別勘定です。

> 料金体系は変更されることがあります。**着手前に必ず [exa.ai/pricing](https://exa.ai/pricing) と
> ダッシュボードのクレジット残高を確認**してください。

</details>

```bash
cd ~/work/mastra-handson   # 4-1 のプロジェクト内で実行しないこと（6章の注意点5）
npx create-mastra@latest template-deep-search --template deep-search
cd template-deep-search
cp .env.example .env
# .env に OPENAI_API_KEY と EXA_API_KEY の2つを記入

# モデル ID を GPT-5.6 系へ差し替える（4-0 参照。エージェント4体すべてが対象）
grep -rl 'openai/gpt-5\.2'   src/ | xargs -r sed -i 's#openai/gpt-5\.2#openai/gpt-5.6-terra#g'
grep -rl 'openai/gpt-5-mini' src/ | xargs -r sed -i 's#openai/gpt-5-mini#openai/gpt-5.6-luna#g'
grep -rn "openai/" src/   # 4箇所すべてが gpt-5.6-* になっていることを確認

npm run dev
```

**動かす手順**

1. Studio の左ナビ **Workflows** → **deep-search** を開く
2. 実行フォームの `initialQuery` に調査したい質問を入れて実行する
3. **すぐ `clarify-intent` ステップで止まり、意図を確認する質問が3つ返る**
4. 3つへの回答を `clarifiedIntent` の欄にまとめて書き、resume する
5. 検索と評価を最大4周繰り返し、最終回答が出る

<details>
<summary>🖥️ <b>このテンプレートを Studio でどう見るか</b>（4-1 で「Workflows タブが空」だった答え合わせ）</summary>

> **Studio 自体の使い方**（左ナビの構成、ワークフローの実行と resume、Traces 画面の読み方）は
> [reference/mastra-studio.md](../reference/mastra-studio.md) にまとめてあります。**ワークフロー関連は §5**。
> ここでは Deep Search 固有の見どころだけを扱います。

**4-1 とちょうど裏返しになります。** 同じ左ナビでも、埋まる画面と空になる画面が逆です。

| 画面 | Deep Search では | 4-1（Chat with Database）では |
|---|---|---|
| **Workflows** | `deep-search` が1本 ← **ここがメイン** | **空** |
| **Tools** | **空**（`createTool` が1つも無い） | 2つ |
| **Agents** | 4体。ただし**直接チャットする画面ではない** | 1体・メイン |
| **Traces** | **実行しても空**（理由は下記5） | 実行すると出る |

## 1. ワークフローを実行する（Workflows）

一覧に並ぶのは **`deep-search` の1本だけ**です。`search-pass` サブワークフローは一覧に出ず、
親のグラフの中にステップとして描かれます。

実行フォームはワークフローの `inputSchema` から自動生成されるため、**入力欄は `initialQuery` の1つだけ**です。

```ts
const deepSearch = createWorkflow({
  id: 'deep-search',
  inputSchema: z.object({ initialQuery: z.string() }),   // ← フォームはこの1欄になる
  ...
});
```

## 2. suspend で止まったところを見る ← 4-1 からの続き

4-1 の「Workflows タブが空なのは正常」で予告した画面がここです。実行するとすぐ `clarify-intent` で
止まります。これが human-in-the-loop の実物です。

画面に**出るもの**と**入れるもの**は、ステップの2つのスキーマで決まっています。

```ts
const clarifyIntent = createStep({
  id: 'clarify-intent',
  suspendSchema: z.object({ assistantMessage: z.string() }),  // ← 表示される（質問3つ）
  resumeSchema:  z.object({ clarifiedIntent:  z.string() }),  // ← 入力欄（テキスト1つ）
  ...
});
```

つまり **「3つの質問が1つの文章でまとめて出て、答えも1つのテキスト欄にまとめて書く」**という UI です。
質問ごとに入力欄が分かれるわけではありません。実際に返る `assistantMessage` はこういう形です。

```
To help you better, I have a few questions:

1. …の基本的な概要を知りたいですか、それとも他フレームワークとの比較を重視しますか？
2. 利用目的は何ですか？（例：チャットボット、社内業務自動化、RAG …）
3. 開発環境や前提条件はありますか？
```

> **💡 質問は投げた言語で返ります**（日本語で聞けば日本語）。ただし前置きの
> `To help you better, I have a few questions:` はコードに直書きされているため英語のままです。

**最初に投げた質問（`initialQuery`）を後から確認する**

止まっている run が「そもそも何を聞いたものか」は run のデータに残っています。ふつうならトレースを
見るところですが、**このテンプレートは Traces が使えない**（下記5）ので run を直接見ます。

| 確認先 | 中身 |
|---|---|
| `payload` | 実行時に入力した `initialQuery` |
| `initialState` | 同じ値（state の初期値として入る） |
| `steps["clarify-intent"].payload` | そのステップが受け取った入力 |

```bash
# run の一覧（status が suspended のものが対象）
curl -s http://localhost:4111/api/workflows/deepSearch/runs

# 個別の run（上の3箇所すべてが入っている）
curl -s http://localhost:4111/api/workflows/deepSearch/runs/<runId>
```

**回答して resume する**

Studio なら resume フォームの `clarifiedIntent` に3問への回答をまとめて書きます。API から叩くなら
次の形です。

```bash
curl -X POST "http://localhost:4111/api/workflows/deepSearch/resume-async?runId=<runId>" \
  -H 'Content-Type: application/json' \
  -d '{
    "step": "clarify-intent",
    "resumeData": { "clarifiedIntent": "1. 他フレームワークとの比較を重視。2. 社内業務自動化が目的。3. TypeScript を想定。" }
  }'
```

> **💡 なぜ `if (!resumeData)` という書き方が多いのか**: resume するとそのステップの `execute` は
> **先頭から再実行**されます。1回目は `resumeData` が無いので `suspend()` に入り、2回目は入っている
> ので反対側へ進みます（reference §5-2）。

## 3. ネストとループをグラフで読む

グラフは3段構えです。根拠は `src/mastra/workflows/deep-search-workflow.ts` の末尾。

```ts
.then(clarifyIntent)
.dountil(searchPass, async ({ iterationCount, state }) =>
  iterationCount >= 4 || !!state.answerIsSatisfactory)   // ← 十分か、4周するまで繰り返す
.then(finalizeAnswer)
```

`search-pass` の中身は `generate-queries` → `search` → `evaluate-results` の3ステップです。
ネストしたステップは **`search-pass.generate-queries`** のようなドット区切りの ID で扱われます。

## 4. Agents は4体いるが、直接チャットしない

一覧には4体並びますが、これらは**ワークフローが内部で `mastra.getAgent()` で呼ぶ**ものです。
4-1 の SQL Agent のようにチャットして使う画面ではありません。役割分担の確認と、4-0 の差し替え結果の
確認に使います。

| エージェント | 役割 | 差し替え後のモデル |
|---|---|---|
| Intent Clarifier Agent | 意図の明確化（質問を3つ作る） | `gpt-5.6-luna` |
| Research Planner Agent | 検索クエリの立案 | `gpt-5.6-terra` |
| Search Result Evaluator Agent | 結果が十分かの評価 | `gpt-5.6-terra` |
| Answerer Agent | 最終回答の生成 | `gpt-5.6-terra` |

4-0 のコマンドで「4箇所」と書いているのは、**`gpt-5-mini` が1体 + `gpt-5.2` が3体**というこの内訳です。

## 5. Traces は実行しても空のまま（このテンプレート固有の落とし穴）

4-1 ではトレースが出たのに、**このテンプレートは実行しても `/traces` が0件のまま**です。バグではなく
`src/mastra/index.ts` の書き方の差です。

```ts
export const mastra = new Mastra({
  workflows: { deepSearch },
  storage: new LibSQLStore({ id: 'mastra-storage', url: ':memory:' }),
  // ← observability: new Observability({...}) が無い（4-1 にはある）
});
```

トレースを残すには「保存先（`storage`）」と「書き出すエクスポーター（`observability`）」の**両方**が
要ります（reference §7-1）。Deep Search は `package.json` の依存に `@mastra/observability` を持って
いるのに `new Mastra({...})` へ渡していないため、スパンがどこにも記録されません。

一方、**ワークフローの実行履歴（Runs）は残ります**。Runs と Traces は別物です（reference §5-5）。

> **💡 やってみる価値あり**: これは壊れているのではなく**設定が足りないだけ**なので、自分で直せます。
> 直せると [03-handson/day2/10-observability.md](../03-handson/day2/10-observability.md) Step 10（観測とトレース）の予習になります。

**ヒント**

直す先は `src/mastra/index.ts` の**1ファイルだけ**です。やることは2つ。

| # | やること | 具体的に |
|---|---|---|
| 1 | **保存先をファイルにする** | `url: ':memory:'` を `url: 'file:./mastra.db'` に変える。`:memory:` のままだと dev サーバーを止めた時点で消えます |
| 2 | **エクスポーターを足す** | `observability:` を `new Mastra({...})` に追加する |

2 はゼロから書く必要はありません。**4-1 で作った `template-sql/src/mastra/index.ts` を開くと、
そのまま流用できる `observability` ブロックが入っています。** 丸ごとコピーして `serviceName` を
`'deep-search'` に変えるだけです。import 文も同じファイルの先頭にあります。

> **`npm install` は不要です。** `@mastra/observability` は Deep Search の `package.json` に
> 最初から依存として入っています（`new Mastra({...})` へ渡していないだけ）。

<details>
<summary>✅ <b>回答（完成形のコード）</b> — まず自分で試してから開いてください</summary>

`src/mastra/index.ts` をこれに置き換えます。**変更点は import 1行・`url`・`observability` ブロックの3箇所**だけです。

```ts
import { Mastra } from '@mastra/core/mastra';
import { PinoLogger } from '@mastra/loggers';
import { LibSQLStore } from '@mastra/libsql';
import { Observability, MastraStorageExporter } from '@mastra/observability';   // ← 追加

import { intentClarifierAgent } from './agents/intent-clarifier-agent';
import { researchPlannerAgent } from './agents/research-planner-agent';
import { searchResultEvaluatorAgent } from './agents/search-result-evaluator-agent';
import { answererAgent } from './agents/answerer-agent';
import { deepSearch } from './workflows/deep-search-workflow';

export const mastra = new Mastra({
  workflows: { deepSearch },
  agents: {
    intentClarifierAgent,
    researchPlannerAgent,
    searchResultEvaluatorAgent,
    answererAgent,
  },
  storage: new LibSQLStore({
    id: 'mastra-storage',
    url: 'file:./mastra.db',        // ← ':memory:' から変更。止めても消えなくなる
  }),
  logger: new PinoLogger({ name: 'Mastra', level: 'info' }),
  observability: new Observability({   // ← ここが丸ごと追加分
    configs: {
      default: {
        serviceName: 'deep-search',
        exporters: [new MastraStorageExporter()],
      },
    },
  }),
});
```

**4-1 の `index.ts` にはあるのに、ここで写していない2つ**

丸ごとコピーすると `MastraPlatformExporter()` と `SensitiveDataFilter()` も付いてきますが、
**どちらも書かなくて構いません**（書いても害はありません）。

| 4-1 にある記述 | ここで省略した理由 |
|---|---|
| `MastraPlatformExporter()` | 送り先が **Mastra Cloud** で、ローカル Studio の `/traces` とは無関係。`MASTRA_PLATFORM_ACCESS_TOKEN` が未設定だと自動で無効化される（ログレベルは `debug` なので警告も出ない）。テンプレートの `.env.example` にこのキーは無いため、**4-1 でも実際には動いていない** |
| `SensitiveDataFilter()` | **書かなくても既定で自動的に追加される**（`spanOutputProcessors` に無ければ Mastra が足す）。明示は自動適用と同じものを手で書いているだけ。ただし `configs` にインスタンス化済みの Observability を渡す書き方をした場合は自動適用されないため、そのときだけ明示が必要 |

**確認方法**

```bash
npm run dev   # 起動ログに MastraStorageExporter disabled が出ないこと
```

ワークフローを1回実行してから `/traces` を開きます。`workflow run: 'deep-search'` を頂点に、
各ステップ（`workflow_step`）とエージェント呼び出しがぶら下がったツリーが出れば成功です。
DB の実体は `src/mastra/public/mastra.db` に作られます（reference §7-2）。

> **⚠️ このコードは 4-1 の動作している `index.ts` と同じ構成に揃えたもので、Deep Search 上での
> 実行確認は未実施です。** うまくいかない場合は 4-1 の `index.ts` と読み比べてください。

</details>

</details>

**観察するポイント**

1. 上の構造図が、Studio のグラフで**どう描かれるか**（ループのブロック、その中のサブワークフロー）
2. **suspend/resume で処理が止まり、人間の入力を待つ**挙動（human-in-the-loop）。`clarify-intent` が該当
3. **2周目以降のクエリが1周目と変わること。** `evaluate-results` が返した `gaps` が `generate-queries` へ渡り、
   さらに「既に検索したクエリ」も渡されるため、**不足点を埋める方向にクエリが変化**します。ここが
   「自己評価して調べ直す」の実体です
4. 4体のモデルが役割に応じて Luna / Terra で使い分けられていること（4-0 の差し替え結果）

**チェックポイント**

- [ ] Workflows 一覧に `deep-search` が1本だけ出る（`search-pass` は出ない）
- [ ] 実行すると `clarify-intent` で止まり、質問が3つ返る
- [ ] resume して最終回答まで到達する
- [ ] Tools が空、Agents が4体であることを確認した
- [ ] Traces が空である理由を `index.ts` で説明できる

> Exa のキーが用意できない場合は 4-3 に振り替えます。
