# Step 11. 評価：出力を採点して、変更の前後を比べる（180分）

Step 9 でエージェントを分割し、Step 10 でトレースを読めるようになりました。ここで気になるのは
「**あの変更で、エージェントは良くなったのか、悪くなったのか**」です。目視で数件を読むだけでは、
LLM の出力が毎回変わるため判断できません（理由は [00-overview/08-devops.md](../../00-overview/08-devops.md) §8-3）。
この Step では、出力を**数値で採点**し、**同じ質問集で変更の前後を比べ**、最後に**コマンド1本で合否を返す**ところまで作ります。

**前提**: Step 9 まで完了。`travelAgent` が `packingAgent` をサブエージェントに持つ構成で、`npm run dev` が動くこと。

> ⚠️ **OpenAI の API 料金が少額かかります。** 採点の一部は LLM（judge）に判定させるので、エージェント本体の
> 呼び出しに加えて、採点のたびにもう1回 LLM が動きます。この Step 全体で数十円程度の目安ですが、
> 気になる場合は [01-setup/02-llm.md](../../01-setup/02-llm.md) の料金表と、11-8 のトレースで確かめてください。

---

## 11-1. 評価の部品を整理する（10分）

| 用語           | 意味                                                                   | この Step での実体                 |
| -------------- | ---------------------------------------------------------------------- | ---------------------------------- |
| **Scorer**     | 1件の出力を **0〜1** で採点する部品                                    | 自作2つ（ルールベース / LLM 採点） |
| **Score**      | Scorer が返す点数と**理由**                                            | Studio の Scorers 画面に溜まる     |
| **Dataset**    | 評価用の質問集。**変更しても同じ質問で比べるための固定物**             | 旅行相談 5件                       |
| **Item**       | Dataset の1件。`input`（質問）と、任意で `groundTruth`（期待する答え） | 「札幌に行きます」など             |
| **Experiment** | Dataset の全件をエージェントに流し、Scorer で採点する**1回の実験**     | `baseline` と `improved`           |

Scorer には2種類あります。**使い分けが設計の要点**です。

|          | ルールベース                                     | LLM 採点（LLM-as-a-judge）                     |
| -------- | ------------------------------------------------ | ---------------------------------------------- |
| 例       | 「`get-weather` を呼んだか」                     | 「天気に触れ、理由を添えているか」             |
| コスト   | 無料                                             | 採点のたびに LLM 呼び出し（課金）              |
| 再現性   | 毎回同じ結果                                     | **採点自体も揺れる**                           |
| 向く対象 | 呼んだ・含む・形式が合う、など機械的に決まること | 「分かりやすいか」など、コードでは書けない品質 |

> 💡 **まず書けるものはルールベースで書く**。LLM 採点は、ルールで書けない品質だけに使います。
> Mastra には [Quick Checks](https://mastra.ai/docs/evals/quick-checks)（`checks.calledTool()` 等、LLM を使わない採点部品）や
> 組み込み Scorer（回答の関連度・毒性・ハルシネーション等）もあります。**ここでは仕組みを理解するため、
> あえて自作します**（11-9 では Quick Checks も使います）。

---

## 11-2. 準備（5分）

```bash
npm install @mastra/evals
mkdir -p src/mastra/scorers
```

`@mastra/evals` は組み込み Scorer と、採点用の補助関数（`extractToolCalls` など）のパッケージです。
`createScorer` 自体は `@mastra/core/evals` にあります。

---

## 11-3. ルールベースの Scorer を作る（25分）

「回答する前に `get-weather` を呼んだか」を、呼ばれたツール名の一覧から 1 / 0 で判定します。

```ts
// src/mastra/scorers/weather-tool-scorer.ts
import { createScorer } from "@mastra/core/evals";
import { extractToolCalls } from "@mastra/evals/scorers/utils";

const TOOL_NAME: string = "get-weather";

export const weatherToolScorer = createScorer({
    id: "weather-tool-called",
    description: "回答する前に get-weather ツールを呼んだかを判定する",
    type: "agent",
})
    // 1) 前処理: 出力から「呼ばれたツール名の一覧」を取り出す
    .preprocess(({ run }): { tools: string[] } => {
        const { tools } = extractToolCalls(run.output);
        return { tools };
    })
    // 2) 採点: 呼んでいれば 1、呼んでいなければ 0
    .generateScore(({ results }): number => {
        const tools: string[] = results.preprocessStepResult?.tools ?? [];
        return tools.includes(TOOL_NAME) ? 1 : 0;
    })
    // 3) 理由: 人が読む説明文（Studio のスコア詳細に出る）
    .generateReason(({ results, score }): string => {
        const tools: string[] = results.preprocessStepResult?.tools ?? [];
        const used: string = tools.length > 0 ? tools.join(", ") : "（なし）";
        return score === 1
            ? `${TOOL_NAME} を呼んでいる。呼ばれたツール: ${used}`
            : `${TOOL_NAME} を呼んでいない。呼ばれたツール: ${used}`;
    });
```

### 読むときの要点

Scorer は**決まった順序のパイプライン**で、必須なのは `generateScore` だけです。

| 段階             | 役割                                                       | 必須     |
| ---------------- | ---------------------------------------------------------- | -------- |
| `preprocess`     | 出力から必要な部分を取り出す・整形する                     | 任意     |
| `analyze`        | 採点の材料になる分析をする（LLM に判定させるのは主にここ） | 任意     |
| `generateScore`  | **数値を返す**                                             | **必須** |
| `generateReason` | 理由の文章を返す                                           | 任意     |

各段階の戻り値は、次の段階に `results.preprocessStepResult` / `results.analyzeStepResult` として渡ります。

| 書き方                                      | 意味                                                                                                            |
| ------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `type: 'agent'`                             | エージェントの入出力用の型が付く。**ライブ採点・過去トレースの採点・Experiment のどれでも同じ Scorer を使える** |
| `run.output`                                | エージェントの応答（メッセージの配列）。ツール呼び出しもここに含まれる                                          |
| `results.preprocessStepResult?.tools ?? []` | 前段の結果。型上は `undefined` を含むので `?.` と `?? []` で既定値を置く                                        |

<details>
<summary>📖 解説: <code>?.</code> と <code>??</code> って何？</summary>

```ts
const tools: string[] = results.preprocessStepResult?.tools ?? [];
```

| 記号 | 名前                 | 働き                                                                    |
| ---- | -------------------- | ----------------------------------------------------------------------- |
| `?.` | オプショナルチェーン | 左が `undefined` / `null` なら、エラーにせず式全体を `undefined` にする |
| `??` | Null 合体演算子      | 左が `undefined` / `null` のときだけ、右の値（ここでは `[]`）を使う     |

| `preprocessStepResult`       | `?.tools`         | `?? []` 後                     |
| ---------------------------- | ----------------- | ------------------------------ |
| `{ tools: ['get-weather'] }` | `['get-weather']` | `['get-weather']`              |
| `{ tools: [] }`              | `[]`              | `[]`（空配列は置き換わらない） |
| `undefined`                  | `undefined`       | `[]`                           |

- 結果として `tools` は常に `string[]` になり、次の `tools.includes(...)` が安全に呼べます。
- `?.` が無いと、`undefined` のときに `Cannot read properties of undefined` で落ちます。
- `|| []` との違い: `||` は `0` / `''` / `false` も置き換えますが、`??` は「値が無いとき」だけ置き換えます。
- 本手順書は `!` や `as` を使わない方針なので、この書き方で型を満たしています。

</details>

> 💡 `get-weather` は、`tools: { [weatherTool.id]: weatherTool }` の**キー名**（Step 4）がそのまま入ります。
> サブエージェント経由なら `agent-packingAgent`、ワークフロー経由なら `workflow-packingBranch` のように
> 接頭辞付きの名前で現れます（実際に呼ばれたツール名は、理由の欄に出るので確認できます）。

---

## 11-4. LLM に採点させる Scorer を作る（25分）

「持ち物の提案として良いか」を、3つの観点（**天気に触れているか / 各項目に理由があるか / 日本語か**）で
LLM に判定させます。LLM には**点数を付けさせず、観点ごとの true / false だけ**を返させ、
点数はコードで計算します（LLM に直接「0〜1 で採点して」と頼むより、結果が安定します）。

```ts
// src/mastra/scorers/packing-advice-scorer.ts
import { createScorer } from "@mastra/core/evals";
import { getAssistantMessageFromRunOutput } from "@mastra/evals/scorers/utils";
import { z } from "zod";

const analyzeSchema = z.object({
    mentionsWeather: z.boolean().describe("天気や気温に具体的に触れているか"),
    hasReasons: z.boolean().describe("持ち物の各項目に理由が添えられているか"),
    isJapanese: z.boolean().describe("日本語で書かれているか"),
    comment: z.string().describe("判定の根拠を一文で"),
});
type AnalyzeResult = z.infer<typeof analyzeSchema>;

export const packingAdviceScorer = createScorer({
    id: "packing-advice-quality",
    description:
        "持ち物提案が「天気に触れる・理由を添える・日本語」を満たすかを LLM が採点する",
    type: "agent",
    judge: {
        model: "openai/gpt-5.6-luna",
        instructions:
            "あなたは旅行アシスタントの回答を採点する、厳格で公平な評価者です。",
    },
})
    // 1) 分析: judge の LLM に3つの観点を true / false で判定させる
    .analyze({
        description: "回答が3つの観点を満たすかを判定する",
        outputSchema: analyzeSchema,
        createPrompt: ({ run }): string => {
            const answer: string =
                getAssistantMessageFromRunOutput(run.output) ?? "";
            return `次の旅行アシスタントの回答を評価してください。\n\n---\n${answer}\n---`;
        },
    })
    // 2) 採点: true の数 ÷ 3（0 / 0.33 / 0.67 / 1 の4段階）
    .generateScore(({ results }): number => {
        const r: AnalyzeResult = results.analyzeStepResult;
        const passed: number = [
            r.mentionsWeather,
            r.hasReasons,
            r.isJapanese,
        ].filter(Boolean).length;
        return passed / 3;
    })
    // 3) 理由: 分析結果をそのまま文章にする（ここでは LLM を使わない）
    .generateReason(({ results, score }): string => {
        const r: AnalyzeResult = results.analyzeStepResult;
        return `score=${score.toFixed(2)} 天気:${r.mentionsWeather} 理由:${r.hasReasons} 日本語:${r.isJapanese} / ${r.comment}`;
    });
```

11-3 との違いは2点だけです。

| 違い                                                    | 内容                                                                                                                                         |
| ------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `judge`                                                 | 採点役の LLM（モデルと役割）。**プロンプトオブジェクトを使う段階がある Scorer にだけ必要**                                                   |
| `.analyze({ description, outputSchema, createPrompt })` | 関数ではなく**オブジェクト**を渡すと、Mastra が judge を呼び、`outputSchema`（zod）に沿った構造化出力を `results.analyzeStepResult` に入れる |

> 💡 **judge のモデルは、評価対象と同じでも別でも構いません。** 自分の出力を自分で採点すると甘くなる
> 傾向があるため、本番では別のモデルや、より上位のモデルを judge に選ぶ運用もあります。
> ここでは料金を抑えるため同じモデルにしています。

---

## 11-5. エージェントに付けてライブ採点する（20分）

Scorer を**エージェントに付ける**と、会話のたびに自動で採点されます（ライブ採点）。
あわせて、Studio から Scorer を選べるよう **`Mastra` にも登録**します。

```ts
// src/mastra/agents/travel-agent.ts（追記部分）
import { weatherToolScorer } from "../scorers/weather-tool-scorer.ts";
import { packingAdviceScorer } from "../scorers/packing-advice-scorer.ts";

export const travelAgent = new Agent({
    // id / name / instructions / model / tools / agents / memory は Step 9 のまま
    scorers: {
        weatherTool: {
            scorer: weatherToolScorer,
            sampling: { type: "ratio", rate: 1 },
        },
        packingAdvice: {
            scorer: packingAdviceScorer,
            sampling: { type: "ratio", rate: 1 },
        },
    },
});
```

```ts
// src/mastra/index.ts（追記部分）
import { weatherToolScorer } from "./scorers/weather-tool-scorer.ts";
import { packingAdviceScorer } from "./scorers/packing-advice-scorer.ts";

export const mastra = new Mastra({
    // agents / workflows / storage / observability は Step 9 のまま
    scorers: { weatherToolScorer, packingAdviceScorer },
});
```

| 設定                                   | 意味                                                                                                                           |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `sampling: { type: 'ratio', rate: 1 }` | 全実行の **100%** を採点する。`0.1` なら10%。**採点のたびに LLM 代がかかるので、本番では下げるのが普通**（ここでは学習用に 1） |
| `new Mastra({ scorers })`              | Studio が Scorer を探す登録簿。**`storage` も必要**（点数の保存先）。登録が無いと保存に失敗する                                |

> ⚠️ **ストレージの設定は Step 7（Step 10 で DuckDB を足した複合ストレージでも同じ）のものを使います。** `storage` が無いと点数が残りません。

### 動かして確かめる

1. `npm run dev` で起動し、Studio（`http://localhost:4111`）の Agents で `travelAgent` と数往復する
   （例: 「来週、福岡に行きます。持ち物を教えて」）
2. 左ナビの **Evaluation → Scorers** を開き、`weather-tool-called` と `packing-advice-quality` を選ぶ

**点数は数十秒遅れて現れます。** 採点は応答を返したあとに**裏で**走るため（ライブ採点は応答を遅くしません）、
特に LLM 採点は judge の呼び出しが終わるまで出ません。すぐに出なくても慌てず、少し待ってから再読み込みします。

見るもの: 点数、**理由**（11-3・11-4 で書いた `generateReason` の文章）、入力と出力、judge に送られたプロンプト。
画面と仕様の正典は [reference/mastra-studio.md](../../reference/mastra-studio.md) §8 です（左ナビの構成は §2）。

> 💡 **Traces 画面に `scorer run: 'packing-advice-quality'` のようなトレースが混ざり始めます。** judge の
> LLM 呼び出しも1回の実行として記録されるためです。エージェント本体のトレースだけ見たいときは、
> 名前が `agent run:` のものを選びます。

---

## 11-6. 過去のトレースを後から採点する（15分）

ライブ採点を付ける**前**の会話や、採点対象にしなかった会話も、記録が残っていれば**あとから採点**できます。

1. Studio の **Traces** で、`agent run: 'travel-agent'` のトレースを開く
2. トレースの詳細画面にある **Evaluate Trace** から Scorer（`weather-tool-called` など）を選んで実行する
3. しばらくして Scorers 画面に点数が増える

また、同じ画面の **Save as Dataset Item** で、そのトレースの質問を**評価用データセットの1件**として保存できます
（次の 11-7 で作るデータセットに、本番で見つけた「うまくいかなかった質問」を足していく使い方です）。

> 💡 **実運用の流れ**: 本番のトレースで失敗例を見つける → Dataset に足す → 直す → 11-8 の実験で、直した結果が
> 他の質問を悪化させていないか確かめる。**失敗例が蓄積されるほど評価が賢くなる**のがこの仕組みの狙いです。

---

## 11-7. 評価用データセットを作る（20分）

Studio の左ナビ **Evaluation → Datasets** で **Create Dataset** を選び、名前を `travel-packing-cases` にして作ります
（説明は任意、スキーマは空のままで構いません）。

作成後の画面で **Add Item** から、次の5件を追加します。`input` は JSON 欄なので、**文字列は `"` で囲みます**。

| #   | `input`                                        | 狙い                                                 |
| --- | ---------------------------------------------- | ---------------------------------------------------- |
| 1   | `"来週、札幌に行きます。持ち物を教えて"`       | 基本形。寒い地域                                     |
| 2   | `"沖縄で海に行きたい。何を持っていけばいい？"` | 基本形。暑い地域。地名が曖昧                         |
| 3   | `"東京へ日帰り出張です。荷物は最小限にしたい"` | 制約付き                                             |
| 4   | `"天気だけ教えて。東京です"`                   | 持ち物を求めていない（理由欄の観点が当てはまらない） |
| 5   | `"旅行に行きます。何を持てばいい？"`           | **行き先が無い**。聞き返すのが正解                   |

- **`groundTruth`（期待する答え）は今回は空のまま**です。今回の Scorer は「正解との一致」ではなく「性質を満たすか」を
  見るため不要です。正解がある課題（翻訳・分類など）では、ここに入れて Scorer から比較します。
- 項目を追加するたびに、**Dataset のバージョンが1つ上がります**。Items 画面の **Versions** で履歴を見たり、
  2つのバージョンの差分（**Compare Versions**）を確認できます。「どのバージョンに対して実験したか」が
  記録されるので、**質問集を変えたせいで点数が変わった**のか**エージェントを変えたせいで変わった**のかを
  切り分けられます。
- 大量にあるときは、Items 画面から CSV / JSON を**インポート**することもできます。

> ⚠️ **スクリプトで作った Dataset は Studio に出ません。** `mastra.datasets.create()` を `node ...` で動かすと、
> DB はコマンドを打ったディレクトリ側にできます。`npm run dev` が読む DB とは**別のファイル**なので、
> Studio からは空に見えます（[reference/mastra-studio.md](../../reference/mastra-studio.md) §7-2）。
> この Step の Dataset は Studio で作ります。

> 💡 **5件目は「罠」です。** 行き先が無いので、エージェントは天気を調べず聞き返すのが正解です。
> ところが `weather-tool-called` は 0 点を付けます。**0 点が出たとき、直すのはエージェントとは限りません**。
> Scorer の前提が合っていない場合もあります（11-10 で考えます）。

<details>
<summary>📖 質問が多くて手作業がつらいときの回避策（スクリプトで作った Dataset を Studio で扱う）</summary>

| #   | 方法                        | 概要                                                                                                                                                                              |
| --- | --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ①   | CSV / JSON インポート       | 質問集を JSON / CSV にして、Items 画面の **Import JSON / Import CSV** で読み込む。最も手軽                                                                                        |
| ②   | dev サーバーの HTTP API     | `npm run dev` で動いているサーバーへ POST して作る。書き込み先が Studio と同じ DB になる                                                                                          |
| ③   | DB の場所を絶対パスに揃える | `LibSQLStore` の `url` を絶対パスにし、`node` で動かすスクリプトも `mastra dev` も同じ DB を読み書きさせる（[reference/mastra-studio.md](../../reference/mastra-studio.md) §7-2） |

大量投入なら①、Git で質問集を版管理したい・CI でも使いたいなら③が向きます。

**① JSON の形式**（`.json`、4 MB まで）: 項目の配列で、`input` だけ必須です。

```json
[
    { "input": "来週、札幌に行きます。持ち物を教えて" },
    {
        "input": "天気だけ教えて。東京です",
        "groundTruth": "（任意）",
        "metadata": { "topic": "weather" }
    }
]
```

**② API の例**（Dataset を作り、項目をまとめて追加。`4111` は `npm run dev` のポート）:

```bash
curl -X POST http://localhost:4111/api/datasets \
  -H 'content-type: application/json' -d '{"name":"api-created"}'
# 返ってきた id を使う
curl -X POST http://localhost:4111/api/datasets/<id>/items/batch \
  -H 'content-type: application/json' \
  -d '{"items":[{"input":"来週、札幌に行きます。持ち物を教えて"}]}'
```

**③ の注意**: `url` を絶対パスにするのは `LibSQLStore` だけです。DuckDB（Step 10）はトレース用で Dataset とは無関係なので、
そのままで構いません。`mastra.datasets.create()` の後に `addItems({ items: [...] })` で項目を足します。

> 💡 動作確認: ②と③は、別ポートの dev サーバーを使って「作った Dataset が API で読み出せること」まで確かめました
> （`mastra` CLI 1.30.0 / `@mastra/core` 1.67.0）。①は Studio の画面ソースで形式を確認しただけで、
> 画面でのインポート操作そのものは実機で試していません。

</details>

---

## 11-8. 実験：プロンプトを直して、前後を比べる（35分）

### ① ベースラインを測る

作った Dataset を開き、**Run Experiment** を選んで次のように設定して実行します。

| 設定        | 値                                                |
| ----------- | ------------------------------------------------- |
| 名前        | `baseline`                                        |
| ターゲット  | Agent の `travel-agent`                           |
| Scorers     | `weather-tool-called` と `packing-advice-quality` |
| リソース ID | **空のまま**                                      |

> 💡 リソース ID を空にすると、各質問は**記憶を持たない状態**で実行されます（`travelAgent` は Memory を持っていますが、
> リソース ID が無いと使われません）。**質問ごとに独立して評価したいので、空が正解**です。入れると、
> そのリソースに溜まった過去の会話や作業メモリが実験に混ざり、点数が揺れる原因になります
> （[公式: Memory-enabled agents](https://mastra.ai/docs/evals/experiments)）。
> 設定欄の名前や配置が違うときは、「リソース ID に相当する項目を入れない」と読み替えてください。

**Experiments** タブで、完了した実験を開くと、質問ごとの出力・各 Scorer の点数と理由・トレースが見えます。
Step 9 の構成のままだと、**点数はあまり高くありません**（筆者の環境では、平均で `weather-tool-called` が 0.4、
`packing-advice-quality` が 0.5 前後でした）。理由欄を読むと、`get-weather` を呼ばずに
`agent-packingAgent` だけに任せている質問や、持ち物に理由が付いていない質問が見つかります。

### ② プロンプトを直す

Step 9 の `instructions` は短く、守ってほしいことが書かれていません。ルールを足します。

```ts
// src/mastra/agents/travel-agent.ts の instructions を差し替える
instructions: `
あなたは旅行相談の窓口です。

- 行き先が分かったら、必ず get-weather ツールで実際の天気を確認する（推測で気候を語らない）
- 持ち物の提案は packingAgent に任せる。その際、取得した天気（気温・降水確率）を伝える
- 行き先が分からないときは、天気を調べる前に行き先を質問する
- 最終回答は日本語で、最初に天気の要点を1〜2行、続けて持ち物を箇条書きにし、各項目に理由を一言添える
`,
```

dev サーバーがコードの変更を取り込んだら、**同じ Dataset・同じ Scorer で、名前だけ `improved` にして**もう一度実行します。

### ③ 比べる

**Experiments** タブで `baseline` と `improved` を選び、**Compare** を押します。2つの実験の点数が
質問ごとに横並びで見えます。

| 見るところ   | 読み取れること                                                             |
| ------------ | -------------------------------------------------------------------------- |
| 平均の差     | 全体として良くなったか（筆者の環境では 0.5 前後 → 0.7 前後）               |
| 質問ごとの差 | **どの質問が良くなり、どれが変わらない／悪化したか**。平均だけでは見えない |
| 理由欄       | 点数が動いた原因（天気を調べるようになった、理由が付いた、など）           |

**数字は環境や実行のたびに変わります。** 次の点を覚えておいてください。

- **5件では断定できません。** 実行のたびに1〜2件は上下します。「差が出たか」は**傾向**として見て、
  怪しいときは同じ条件でもう一度回します。本番の評価では、質問を数十件〜に増やします
- **`improved` でも `沖縄で…` が 0 点のままのことがあります。** 「沖縄」だけだと地名が曖昧で、天気を調べずに
  答えることがあるためです。**プロンプトを直しても残る失敗**を見つけるのも評価の効用です
  （ここから先は、ツールの説明文や質問文を直す、という次の一手が考えられます）
- **天気は実データなので、日によって回答が変わります。** 天気のように外部に依存する部分を固定したい場合は、
  Dataset の Item に**ツールのモック**（ツールの戻り値を固定する機能）を付けられます
  （[公式: Tool mocks](https://mastra.ai/docs/evals/experiments)）

### コストの確認

Experiment の各質問は、エージェント本体の実行に加えて、`packing-advice-quality` の judge 呼び出しが1回走ります。
Step 10 のトレース画面で **`scorer run:` のトレースのトークン数**を見て、「評価を1回回すといくらか」を概算してください。
Step 10 で Metrics を入れていれば、Metrics 画面のモデルコストにも judge の分が合算されます。
これが 11-5 の `sampling` を絞る理由であり、CI で全件回すときの予算になります。

---

## 11-9. 合否をコマンド1本で返す — CI 用スクリプト（20分）

Studio での実験は手動です。**変更のたびに自動で回したい**ときは、コードから `runEvals` を呼びます。
結果を **`passed` / `scored` / `failed`** の判定で受け取り、`failed` なら終了コードを 1 にすれば、
CI（GitHub Actions など）をそのまま落とせます。

```ts
// src/evals/run-evals.ts
import { runEvals } from "@mastra/core/evals";
import { checks } from "@mastra/evals/checks";
import { mastra } from "../mastra/index.ts";
import { packingAdviceScorer } from "../mastra/scorers/packing-advice-scorer.ts";

async function main(): Promise<void> {
    const agent = mastra.getAgentById("travel-agent");

    const result = await runEvals({
        target: agent,
        // 行き先が分かっている質問だけ（「行き先が無い」質問は天気を調べないのが正解なので、ゲートに入れない）
        data: [
            { inputs: ["来週、札幌に行きます。持ち物を教えて"] },
            { inputs: ["沖縄で海に行きたい。何を持っていけばいい？"] },
            { inputs: ["東京へ日帰り出張です。荷物は最小限にしたい"] },
            { inputs: ["大阪に2泊します。持ち物のおすすめは？"] },
        ],
        // ゲート: 全件で 1.0 でなければ「failed」。絶対に守らせたい条件
        gates: [checks.calledTool("get-weather")],
        // スコアラー: 平均が threshold 未満なら「scored」。品質の目標値
        scorers: [{ scorer: packingAdviceScorer, threshold: 0.7 }],
        concurrency: 2,
    });

    console.log("verdict:", result.verdict);
    console.log("scores:", result.scores);
    console.log("gateResults:", result.gateResults);
    console.log("thresholdResults:", result.thresholdResults);

    if (result.verdict === "failed") {
        process.exitCode = 1;
    }
}

main().catch((error: unknown): void => {
    console.error(error);
    process.exitCode = 1;
});
```

```bash
node --env-file=.env src/evals/run-evals.ts
echo "終了コード: $?"
```

> 💡 **Studio（`npm run dev`）を起動していなくても動きます。** エージェントをこのプロセスの中で直接呼ぶためです。
> `.env` のある `travel-agent/` 直下で実行してください。

| 書き方                   | 意味                                                                                                                                                                           |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `gates`                  | **必ず満たす条件**。全件の平均が 1.0 に届かなければ `failed`。`checks.calledTool()` は LLM を使わない Quick Check（呼ばなければ 0、呼べば 1）                                  |
| `scorers` の `threshold` | **品質の目標値**。平均がこれ未満なら `scored`（ゲートは通ったが目標未達）。数値だけ渡すと「以上」、`{ max: 0.3 }` で「以下」（ハルシネーションなど、**高いほど悪い**もの向け） |
| `verdict`                | `passed`（全部クリア）/ `scored`（ゲートは通ったが目標未達）/ `failed`（ゲートが落ちた）                                                                                       |
| `inputs: ['…']`          | 1件を**会話1本**として流す書き方。質問ごとに別のスレッドが自動で作られる                                                                                                       |
| `concurrency: 2`         | 質問を**同時に何件まで並列で流すか**。省略すると 1 件ずつ。上げると速いが、上げすぎると OpenAI のレート制限（429）に当たりやすい |
| `process.exitCode = 1`   | シェルや CI に「失敗」を伝える。`process.exit(1)` と違い、出力を出し切ってから終わる                                                                                           |

> ⚠️ **`input` ではなく `inputs: ['…']` と書く理由**: `{ input: '…' }` でも動きますが、Step 9 のようにサブエージェントを持つ
> Memory 付きのエージェントでは、**`Thread ID is required` というエラーログが何件も出ます**（判定には影響しません）。
> `inputs` にすると、質問ごとにスレッドが用意されて消えます。

実行して `passed` が出たら、**わざと落としてみます**。ゲートを `checks.didNotCall('get-weather')`
（「天気を呼ばないこと」）に書き換えて再実行すると、`verdict: failed` になり、`終了コード: 1` が表示されます。確認したら戻します。

> ⚠️ **このスクリプトの結果は Studio には出ません。** `node` で動かすと DB が別になるため
> （11-7 の注意と同じ理由）、スコアもトレースもプロジェクト直下の `mastra.db` に入ります。
> **CI は合否が目的**なので問題ありませんが、画面で詳しく見たいときは Studio の Experiment を使います。

> 💡 **CI への載せ方**: GitHub Actions なら、`npm ci` のあと上のコマンドを1行実行するだけです
> （`OPENAI_API_KEY` はリポジトリの Secrets から渡します）。全件・LLM 採点で毎回回すと費用が増えるので、
> **プロンプトやツールを変えた PR のときだけ回す**、件数を絞る、といった運用が現実的です。
> 詳細は公式の [Running scorers in CI](https://mastra.ai/docs/evals/running-in-ci) を参照してください。

---

## 11-10. 考える（5分）

次の問いを、自分の言葉で書き出してください。

1. **5件目（行き先が無い質問）が 0 点になったとき、直すのは何か。** エージェントか、Scorer か、Dataset か。
   （ヒント: Item ごとに使う Scorer を変えられます。公式の「Select scorers per item」を参照）
2. 社内で使うエージェントに、**ルールベースで採点できる条件**と、**LLM に判定させるしかない品質**を、
   それぞれ3つずつ挙げてください。
3. judge の LLM は、モデルの更新で**判定の癖が変わる**ことがあります。それをどう検出しますか。
   （ヒント: 正解ラベル付きの Dataset を作り、Scorer 自体を Experiment のターゲットにする方法があります）
4. 本番で `sampling` を何％にしますか。その根拠を、11-8 で見た judge のトークン数で説明してください。

---

## チェックポイント

- [ ] ルールベースと LLM 採点の違い（コスト・再現性・向く対象）を説明できる
- [ ] `createScorer` の4段階（`preprocess` / `analyze` / `generateScore` / `generateReason`）のうち、必須が `generateScore` だけだと説明できる
- [ ] Scorer を `Mastra` に登録しないと Studio から選べない、`storage` が無いと点数が残らないことを理解した
- [ ] ライブ採点の点数が Studio の Scorers に出ること、採点が応答のあとに裏で走ることを確かめた
- [ ] Studio で Dataset を作り、Item を5件追加した
- [ ] `baseline` と `improved` の2つの Experiment を回し、Compare で比べた
- [ ] 「5件では断定できない」「質問ごとの差を見る」という読み方を理解した
- [ ] `runEvals` で `verdict` を受け取り、`failed` のとき終了コードが 1 になることを確かめた
- [ ] `gates`（必ず守る）と `threshold`（目標値）の違いを説明できる
