# Step 8. ワークフロー：決まった手順を組む（210分）

エージェントは「手順が事前に決まっていない」タスク向けです。決まっている多段処理はワークフローで明示的に組みます。

前半（8-1〜8-3）で**直列**のワークフローを作り、後半（8-4〜8-6）で座学 [§6-2・§6-3](../../00-overview/06-workflow.md) の
**分岐・ループ・人間の承認待ち**を自分の手で書きます。

## 8-1. 概念を押さえる（15分）

Mastra のワークフローは3つの要素で構成されます。

| 要素 | 役割 |
|---|---|
| `createStep()` | 1ステップを定義。`inputSchema` / `outputSchema` / `execute` を持つ |
| `createWorkflow()` | ステップを `.then()` で繋いで `.commit()` で確定する |
| 実行 | 中断（suspend）・再開（resume）・ストリーミングをサポート |

各ステップが入出力スキーマを持つので、データの流れが型で保証されます。

## 8-2. 最小のワークフローを作る（30分）

### 何を作るのか

**「文字列を受け取り、大文字にして、飾りを付けて返す」だけのワークフロー**を作ります。
`hello` と入力すると `*** HELLO ***` が返ってきます。

```
入力 { message: "hello" }
   ↓
[ to-upper ]   大文字にする        → { formatted: "HELLO" }
   ↓
[ decorate ]   *** で囲む          → { result: "*** HELLO ***" }
   ↓
出力 { result: "*** HELLO ***" }
```

**LLM もツールも使いません。** 目的は便利なものを作ることではなく、8-1 で見た3つの要素
（`createStep()` / `createWorkflow()` / 実行）が実際にどう繋がるのかを、**最小の部品で1度通す**ことです。
ここで確認するのは次の3点です。

| 確認すること | どこに現れるか |
|---|---|
| ステップは `inputSchema` → `execute` → `outputSchema` の3点セットで書く | `to-upper` / `decorate` の定義 |
| **前のステップの `outputSchema` と、次のステップの `inputSchema` が同じ形である**必要がある | `to-upper` の出力 `{ formatted }` が、そのまま `decorate` の入力になっている |
| ステップは `.then()` で繋ぎ、`.commit()` で確定する | ファイル末尾の `createWorkflow(...).then(...).then(...).commit()` |

処理そのものが単純なぶん、**この「スキーマの受け渡し」だけに集中できます**。LLM を呼ぶ実用的な
ワークフローは 8-3 で作ります。

> 💡 **この `helloWorkflow` は `travelAgent` と何の関係もありません。** 同じ `new Mastra({...})` に並べて書きますが、
> **`new Mastra()` は結合ではなく登録簿**で、登録された部品に `storage` / `observability` を共通で配り、
> Studio と HTTP エンドポイントに露出させるだけです。並べたからといって両者が繋がるわけではありません。
>
> とはいえ実務では、エージェントとワークフローは次のどれかで繋がるのが普通です。**繋がるのは 8-3 からです。**
>
> | 繋ぎ方 | 書き方 | 登場する節 |
> |---|---|---|
> | ワークフロー → エージェント | ステップの中で `mastra.getAgentById('travel-agent')` | **8-3・8-5・8-6**（最も多い形） |
> | 共通のツールを介して | 同じツールを Agent の `tools` にも `createStep(tool)` にも使う | **8-3**（`weatherTool`） |
> | エージェント → ワークフロー | `new Agent({ workflows: { ... } })` に渡すと、LLM が判断して呼べる | **8-7** |
>
> 設計の定石は「**ワークフローを骨格にして、判断が必要なステップだけエージェントを呼ぶ**」です
> （判断表は座学 [§6-4](../../00-overview/06-workflow.md)）。8-2 はその骨格の書き方だけを、LLM を外して練習する節です。

### 書く

```ts
// src/mastra/workflows/hello-workflow.ts
import { createWorkflow, createStep } from '@mastra/core/workflows'
import { z } from 'zod'

const upperInputSchema = z.object({ message: z.string() })
const upperOutputSchema = z.object({ formatted: z.string() })
type UpperOutput = z.infer<typeof upperOutputSchema>

const upperStep = createStep({
  id: 'to-upper',
  inputSchema: upperInputSchema,
  outputSchema: upperOutputSchema,
  execute: async ({ inputData }): Promise<UpperOutput> => {
    return { formatted: inputData.message.toUpperCase() }
  },
})

const decorateInputSchema = z.object({ formatted: z.string() })
const decorateOutputSchema = z.object({ result: z.string() })
type DecorateOutput = z.infer<typeof decorateOutputSchema>

const decorateStep = createStep({
  id: 'decorate',
  inputSchema: decorateInputSchema,
  outputSchema: decorateOutputSchema,
  execute: async ({ inputData }): Promise<DecorateOutput> => {
    return { result: `*** ${inputData.formatted} ***` }
  },
})

export const helloWorkflow = createWorkflow({
  id: 'hello-workflow',
  inputSchema: z.object({ message: z.string() }),
  outputSchema: z.object({ result: z.string() }),
})
  .then(upperStep)
  .then(decorateStep)
  .commit()
```

<details>
<summary>📘 このコードで使っている TypeScript / Zod の記法（クリックで開く）</summary>

| 記法 | 何か | 使い方 |
|---|---|---|
| `z.object({...})` | Zod で**オブジェクトの形を定義する**関数。返るのは型ではなく**実行時に存在する値**（スキーマ） | `z.object({ message: z.string() })` で「文字列 `message` を持つオブジェクト」。`inputSchema` / `outputSchema` に渡すと、Mastra が実行時に入出力を検証し、Studio の入力フォームも自動生成する |
| `z.infer<...>` | Zod スキーマ（値）から**対応する TypeScript の型を取り出す**ユーティリティ型 | `z.infer<typeof upperOutputSchema>` → `{ formatted: string }`。同じ形を「スキーマ」と「型」に二重に書かずに済み、スキーマを直せば型も追随する |
| `typeof`（型の文脈） | TypeScript の**型演算子**で、「**値**から、その値の**型**を取り出す」 | `upperOutputSchema` は値なので型の位置には書けない。`typeof upperOutputSchema` として初めて `z.infer<...>` に渡せる。JavaScript の `typeof x === 'string'` とは名前が同じだけの別物 |
| `Promise<T>` | **「今はまだ無いが、あとで手に入る `T`」**を表す型。**「あとで値が入る箱」**と考えるとよい | `async` 関数は必ず Promise を返すので、`return { formatted: ... }` と書いても戻り値の型は `Promise<UpperOutput>`。受け取る側は `await` で中身を取り出す |

**「今はまだ無い」とは？** `await` を含む関数は、待ちに入った時点で**いったん呼び出し元に制御を返します**。
このとき返す値が要るのに中身はまだ出来ていないので、**空の箱を先に返す**わけです。

```js
const box = fetchWeather() // 中で 300ms 待つ関数。ここで即座に返る
console.log(box) //=> Promise { <pending> }  ← 箱（中身待ち）
console.log(box.formatted) //=> undefined    ← 中身がまだ無い
console.log('次の行が先に動く') // ← 300ms 待たずにここまで来る
```

**なぜ `UpperOutput` ではなく `Promise<UpperOutput>` と書くのか。** 上のとおり、呼び出し側が実際に受け取るのは
中身ではなく**箱**だからです。`UpperOutput` と書くと型と実物が食い違い、`box.formatted` のような
間違いを型チェックが見逃します。`Promise<UpperOutput>` にしておけば `await` の付け忘れで止まります
（`Property 'formatted' does not exist on type 'Promise<UpperOutput>'`）。
なお **`async` を付けたら `Promise<...>` 以外は書けません**（`TS1064` になります）。

> 💡 **型注釈は `Promise<UpperOutput>` なのに、`return` するのは裸の `{ formatted: ... }`** という非対称さに
> 戸惑うかもしれません。`async` 関数の中では **`return` した値を JavaScript が自動で Promise に包む**ので、
> 書き手は中身だけを書きます。型注釈のほうは、実際に外へ出ていくもの（箱）を書きます。
> `Promise.resolve(...)` で自分で包む必要はありません。

**`execute:` の右にある `=>`（アロー関数）は何をしている？**

`=>` は**関数を作る式**です（一般的な説明は座学 [§0-4](../../00-overview/00-typescript-primer.md#0-4-設定オブジェクトとアロー関数)）。
`execute:` に渡しているのは「実行した結果」ではなく**関数そのもの**で、**「呼ばれたら大文字にする手順書」を
Mastra に預けている**だけです。実際に呼ぶのはワークフローを実行したときの Mastra 側なので、
`createStep({...})` を書いた時点では `execute` の中身は1回も動きません。

`async ({ inputData }): Promise<UpperOutput> => { ... }` を分解するとこうなります。

| 部分 | 意味 |
|---|---|
| `async` | この関数は Promise を返す（中で `await` が使える） |
| `({ inputData })` | 引数。Mastra は `{ inputData, mastra, ... }` という**1つのオブジェクト**を渡してくるので、そこから `inputData` だけ取り出している（分割代入） |
| `: Promise<UpperOutput>` | 戻り値の型注釈。`=>` の**手前**に書く |
| `=> { ... }` | 関数の本体 |

**波カッコの有無で `return` の要否が変わります。** 8-4 以降のコードを読むときに効いてきます。

```ts
// ① 本体が { } → return が必要（8-2 はこちら）
execute: async ({ inputData }) => {
  return { advice: '傘を持っていきましょう' }
}

// ② 本体が式1つ → return を省略できる。ただしオブジェクトは ( ) で囲む（8-4 はこちら）
execute: async ({ inputData }) => ({ advice: '傘を持っていきましょう' })
```

②の `({ ... })` のカッコは飾りではありません。**囲まないと `{` が「関数本体の始まり」と解釈される**ため、
オブジェクトをそのまま返したいときだけ `()` が要ります。

**`execute` に `async` を付けているのはなぜ？**

- Mastra が `execute` の戻り値を **Promise として扱う**（`await` する）ため、シグネチャをそろえる意味がある
- 8-3 以降の `execute` は `await agent.generate(...)` のように**中で待つ処理**を書く。`await` は `async` 関数の中でしか使えないので、8-2 の時点から形をそろえておけば、処理を足すときに書き換えが要らない

**ほかに `async` を付けるもの / 付けないもの**

| | 対象 |
|---|---|
| 付ける | ステップの `execute`、`.map()` に渡す関数、`.branch()` の条件関数、`.dountil()` / `.dowhile()` の条件関数（8-3〜8-5 で登場） |
| 付けない | `createStep()` / `createWorkflow()` / `z.object()` の呼び出し自体、`.then()` などのメソッドチェーン。即座に値を返すだけで、待つ処理がない |

</details>

登録:

```ts
// src/mastra/index.ts（import と workflows の2行を追加。他は Step 7 のまま）
import { helloWorkflow } from './workflows/hello-workflow.ts'   // ← 追加

export const mastra = new Mastra({
  agents: { travelAgent },
  workflows: { helloWorkflow },   // ← 追加
  storage: new LibSQLStore({ id: 'mastra-storage', url: 'file:./mastra.db' }),
  observability: new Observability({
    configs: {
      default: {
        serviceName: 'travel-agent',
        exporters: [new MastraStorageExporter()],
      },
    },
  }),
})
```

> ⚠️ **`import` を忘れると起動時に「`helloWorkflow` が見つからない」で落ちます。** `workflows:` に名前を書いただけでは
> ファイルは読み込まれません。import の行は、既にある `travelAgent` などの import と並べてファイルの先頭に置きます。
> 拡張子 `.ts` まで書くのは Step 7 までと同じです。

> ⚠️ **パスは `'./workflows/'` ではなく、ファイルまで書いてください。** ディレクトリで止めると起動時に落ちます。
> **`npx tsc --noEmit` はこの間違いを見逃す**（`tsconfig.json` の `moduleResolution: "bundler"` がディレクトリ指定を
> 許すため）ので、**型チェックが通っても安心せず、実際に `npm run dev` で起動して確かめてください。**
>
> ```
> code: 'ERR_UNSUPPORTED_DIR_IMPORT'
> url: 'file:///.../src/mastra/workflows/'
> ```
>
> Node は ES Modules でディレクトリを指す import を認めておらず、`workflows/index.ts` を自分で用意しない限り
> 解決できません（Node 24.21.0 で確認）。本ハンズオンではそのファイルは作らないので、常にファイル名まで書きます。

Studio の **Workflows タブ**を開くと、
- 中央パネルにステップのグラフが可視化される
- 右サイドバーに `inputSchema` から自動生成された入力フォームが出る
- 実行中はグラフ上でステップの状態がリアルタイムに更新される

処理の流れが目で追えるので、ワークフローの動きを理解するのに最適です。

## 8-3. 実用的なワークフローを作る（45分）

「天気を取得 → エージェントに持ち物提案させる → 構造化して返す」という3ステップを組みます。

```ts
// src/mastra/workflows/trip-prep-workflow.ts
import { createWorkflow, createStep } from '@mastra/core/workflows'
import { z } from 'zod'
import { weatherTool } from '../tools/weather-tool.ts'

// Step 1: 天気を取得（既存のツールをそのままステップに変換する）
const fetchWeatherStep = createStep(weatherTool)

// Step 2: エージェントに提案させる
const suggestInputSchema = z.object({ city: z.string(), summary: z.string() })
const suggestOutputSchema = z.object({ suggestion: z.string() })
type SuggestOutput = z.infer<typeof suggestOutputSchema>

const suggestStep = createStep({
  id: 'suggest-items',
  inputSchema: suggestInputSchema,
  outputSchema: suggestOutputSchema,
  execute: async ({ inputData, mastra }): Promise<SuggestOutput> => {
    const agent = mastra.getAgentById('travel-agent')
    const response = await agent.generate(
      `行き先: ${inputData.city}\n天気: ${inputData.summary}\n\n` +
        `この天気を踏まえて持ち物を提案してください。天気の再取得は不要です。`,
    )
    return { suggestion: response.text }
  },
})

export const tripPrepWorkflow = createWorkflow({
  id: 'trip-prep-workflow',
  inputSchema: z.object({ city: z.string() }),
  outputSchema: z.object({ suggestion: z.string() }),
})
  .then(fetchWeatherStep)
  // ツールの出力（配列を含む）を、次のステップが求める形に整形する
  .map(async ({ inputData }): Promise<z.infer<typeof suggestInputSchema>> => {
    const summary: string = inputData.forecast
      .slice(0, 3)
      .map(
        (d): string =>
          `${d.date}: ${d.minTemp}〜${d.maxTemp}℃ 降水確率${d.precipitationProbability}%`,
      )
      .join(' / ')
    return { city: inputData.city, summary }
  })
  .then(suggestStep)
  .commit()
```

ここで使った書き方は2つあります。

| 書き方 | 役割 |
|---|---|
| `createStep(weatherTool)` | **既存のツールをそのままステップにする**。`inputSchema` / `outputSchema` はツールの定義がそのまま使われるので、書き直す必要がない |
| `.map(...)` | 前のステップの出力を、次のステップの `inputSchema` に合う形へ**変換する**。ここでは天気の配列を1本の要約文字列に畳んでいる |

<details>
<summary>📘 <code>.slice(0, 3)</code> と <code>(d)</code> は何をしている？（クリックで開く）</summary>

`inputData.forecast` は1要素が1日分の予報の配列です（日数を指定していないので Open-Meteo の既定の7日分）。

| 部分 | 意味 |
|---|---|
| `.slice(0, 3)` | 先頭から**3つ**（`[0]`〜`[2]`）を取り出した新しい配列を返す。終了位置の `3` 自体は含まない。元の配列は変わらない |
| `.map((d): string => ...)` | **配列の**メソッド。要素を1つずつ関数に通し、戻り値を集めた配列を作る |
| `(d)` | その関数の引数で、**1日分のオブジェクト**が順に入る。名前は day の頭文字というだけで自由。型は `forecast` の要素から推論される |
| `(d)` の `( )` | 引数が1つなら本来 `d => ...` と省略できるが、**戻り値の型 `: string` は `)` の直後に書く決まり**なので、型注釈を付けるときは省けない（`d: string => ...` は構文エラー） |
| `.join(' / ')` | 配列の文字列を ` / ` でつないで1本の文字列にする |

```
7日分の配列 → slice(0, 3) → 3日分 → map → ["10-03: 9〜18℃ 降水確率40%", …] → join → "10-03: … / 10-04: … / 10-05: …"
```

> ⚠️ **`.map` が2種類出てきます。** 外側の `.map(async ({ inputData }) => ...)` は**ワークフローの**メソッド（ステップ間の変換）、
> 内側の `forecast....map((d) => ...)` は**配列の**メソッドです。名前が同じだけの別物です。

</details>

作ったワークフローは `src/mastra/index.ts` に**登録しないと Studio に出てきません**。8-2 と同じ要領で追加します。

```ts
// src/mastra/index.ts
import { tripPrepWorkflow } from './workflows/trip-prep-workflow.ts'

export const mastra = new Mastra({
  agents: { travelAgent },
  workflows: { helloWorkflow, tripPrepWorkflow },   // ← 追加
  // storage / observability は Step 7 のまま
})
```

> ⚠️ ツールの `execute` を `weatherTool.execute({ city })` の形で**そのまま**呼ぶことはできません。
> `execute` は省略可能なプロパティ（型が `undefined` を含む）なので非 null の確認が要り、引数も
> `execute(input, context)` の2つが必要です（公式も `await tool.execute({ city }, { requestContext })`
> の形で紹介しています）。**ワークフローで使い回すときは `createStep(tool)` のほうが素直です** ——
> スキーマがそのまま引き継がれ、トレースにもステップとして現れます。

### `execute` が受け取る `mastra` とは

`suggestStep` の `execute` は、8-2 の `({ inputData })` に加えて **`mastra`** も受け取っています。

`mastra` は、`src/mastra/index.ts` で `new Mastra({...})` によって作った **Mastra インスタンスそのもの**です
（`export const mastra` と同じオブジェクト）。8-2 の記法の表で見たとおり、Mastra は `execute` を呼ぶときに
`{ inputData, mastra, ... }` という1つのオブジェクトを渡してくるので、そこから `mastra` も取り出しているだけです。
8-2 では使わなかったので書かなかっただけで、渡されてはいました。

| 項目 | 内容 |
|---|---|
| 正体 | `new Mastra({...})` で作った**登録簿**（8-2 の 💡） |
| 型 | `Mastra`。省略可能なプロパティではないので、`undefined` の確認は要らない |
| 使い道 | 登録済みの部品を取り出す。`mastra.getAgentById('travel-agent')` でエージェント、`mastra.getWorkflowById('hello-workflow')` でワークフロー |

`getAgentById()` に渡すのはエージェントの **`id`**（`new Agent({ id: 'travel-agent', ... })`）です。
`new Mastra({ agents: { travelAgent } })` の**キー**で引きたいときは `mastra.getAgent('travelAgent')` を使います。
8-5・8-6 のステップも同じ書き方でエージェントを取り出します。

> ⚠️ **id を打ち間違えても `tsc` は通ります。** 引数の `mastra` は「何が登録されているか」を知らない汎用の `Mastra` 型なので、
> `'travel-agentt'` のような誤りも型チェックをすり抜け、実行時に `Agent with id travel-agentt not found` で止まります
> （`@mastra/core` 1.67.0 で確認）。

**なぜ `travelAgent` を直接 import しないのか。** `import { travelAgent } from '../agents/travel-agent.ts'` と書いても、
実は同じオブジェクトが手に入り、動きます（`new Mastra()` は登録したエージェントを複製せず、その実体に storage などを配るため）。
それでも `mastra` 経由で取り出すのは、次の2つのためです。

| 理由 | 内容 |
|---|---|
| ワークフローがエージェントのファイルに依存しない | ワークフローが知っているのは `'travel-agent'` という id だけ。エージェントの実装ファイルを差し替えても、同じ id で登録されていればワークフローは書き換えずに済む |
| ファイルどうしの循環を避ける | 8-7 では、エージェントのファイルがワークフローを import する。ワークフロー側からもエージェントを import すると2つのファイルが互いを読み込み合い、読み込み順しだいで壊れる構成になる |

引数の一覧は公式リファレンス https://mastra.ai/reference/workflows/step にあります。

Studio の Workflows タブから `city: "Sapporo"` を入力して実行し、グラフ上でステップが順に進むのを観察します。

> ⚠️ ここは**ローマ字で入力してください。** `"札幌"` と入れると Open-Meteo の geocoding が0件を返し、最初のステップが `都市が見つかりません` で止まります（[day1/04-tools.md](../day1/04-tools.md) §4-2 の注意を参照）。

## 8-4. 分岐で処理を振り分ける — `.branch()`（45分）

「降水確率が高ければ雨の備え、低ければ晴れの備えを提案する」を、**条件で枝を振り分けるワークフロー**として組みます。
この節は **LLM を呼びません**（天気 API とコードだけ）。実行のたびに費用がかからず、毎回同じ判定になるので、
分岐の挙動そのものに集中できます。

8-3 で覚えた `createStep(weatherTool)` と `.map()` をそのまま使うので、**新しく出てくるのは `.branch()` だけ**です。

```ts
// src/mastra/workflows/packing-branch-workflow.ts
import { createWorkflow, createStep } from '@mastra/core/workflows'
import { z } from 'zod'
import { weatherTool } from '../tools/weather-tool.ts'

// Step 1: 天気を取得（8-3 と同じ）
const fetchWeatherStep = createStep(weatherTool)

const branchInputSchema = z.object({ city: z.string(), maxPrecipitation: z.number() })
const adviceOutputSchema = z.object({ advice: z.string() })
type AdviceOutput = z.infer<typeof adviceOutputSchema>

// 枝1: 降水確率が高いとき
const rainyStep = createStep({
  id: 'rainy',
  inputSchema: branchInputSchema,
  outputSchema: adviceOutputSchema,
  execute: async ({ inputData }): Promise<AdviceOutput> => ({
    advice: `${inputData.city} は降水確率が最大 ${inputData.maxPrecipitation}%。折りたたみ傘とレインウェアを持っていきましょう`,
  }),
})

// 枝2: 降水確率が低いとき
const dryStep = createStep({
  id: 'dry',
  inputSchema: branchInputSchema,
  outputSchema: adviceOutputSchema,
  execute: async ({ inputData }): Promise<AdviceOutput> => ({
    advice: `${inputData.city} は降水確率が最大 ${inputData.maxPrecipitation}%。日焼け止めと帽子があると安心です`,
  }),
})

// 合流: 動かなかった枝は undefined で届くので .optional() が要る
const mergeInputSchema = z.object({
  rainy: adviceOutputSchema.optional(), // ← キーは枝ステップの id
  dry: adviceOutputSchema.optional(),
})
const mergeOutputSchema = z.object({ result: z.string() })
type MergeOutput = z.infer<typeof mergeOutputSchema>

const mergeStep = createStep({
  id: 'merge-advice',
  inputSchema: mergeInputSchema,
  outputSchema: mergeOutputSchema,
  execute: async ({ inputData }): Promise<MergeOutput> => ({
    result:
      [inputData.rainy?.advice, inputData.dry?.advice].filter(Boolean).join(' / ') ||
      '判定できませんでした',
  }),
})

export const packingBranchWorkflow = createWorkflow({
  id: 'packing-branch-workflow',
  inputSchema: z.object({ city: z.string() }),
  outputSchema: z.object({ result: z.string() }),
})
  .then(fetchWeatherStep)
  // 直近3日の降水確率の最大値を1つ取り出し、分岐の材料にする
  .map(async ({ inputData }): Promise<z.infer<typeof branchInputSchema>> => ({
    city: inputData.city,
    maxPrecipitation: Math.max(
      ...inputData.forecast.slice(0, 3).map((d): number => d.precipitationProbability),
    ),
  }))
  .branch([
    // [条件, 条件が true のとき動かすステップ] の組を並べる
    [async ({ inputData }): Promise<boolean> => inputData.maxPrecipitation >= 50, rainyStep],
    [async ({ inputData }): Promise<boolean> => inputData.maxPrecipitation < 50, dryStep],
  ])
  .then(mergeStep)
  .commit()
```

<details>
<summary>📘 合流ステップの <code>.optional()</code> と <code>?.</code> は何をしている？（クリックで開く）</summary>

**合流ステップに届くもの。** `.branch()` の後ろには、枝ステップの `id` をキーにしたオブジェクトが届きます。
ただし動くのは条件が true の枝だけなので、**片方は欠けて届きます**。

| 降水確率 | `mergeStep` の `inputData` |
|---|---|
| 50% 以上 | `{ rainy: { advice: '…' } }`（`dry` が無い） |
| 50% 未満 | `{ dry: { advice: '…' } }`（`rainy` が無い） |

**それを受けるための2つの書き方。**

| 記法 | 書く場所 | 意味 | 無いとどうなる |
|---|---|---|---|
| `.optional()` | スキーマ（`mergeInputSchema`） | zod の「**このキーは無くてもよい**」。型は `{ advice: string } \| undefined` になる | 両方必須になり、合流ステップの**入力検証で毎回落ちる** |
| `?.` | 読む側（`mergeStep`） | **オプショナルチェーン**。左側が `undefined` / `null` なら先を読まずに `undefined` を返す | `rainy` が無いと `Cannot read properties of undefined` で落ちる。ただし型に `undefined` が入っているので、**付け忘れは `tsc` が止める** |

**読んだ後の後始末 — `.filter(Boolean)`。** `?.` で `undefined` になった方を配列から取り除きます。
`Boolean` は値を `true` / `false` に変える組み込み関数で、`.filter()` は true になった要素だけを残すので、
`.filter((x) => Boolean(x))` と同じ意味になります。

| 動いた枝 | `[rainy?.advice, dry?.advice]` | `.filter(Boolean)` | `.join(' / ')` | 最終結果 |
|---|---|---|---|---|
| `rainy` のみ | `['…傘…', undefined]` | `['…傘…']` | `'…傘…'` | `'…傘…'` |
| どちらも動かない（実験②） | `[undefined, undefined]` | `[]` | `''` | `'' \|\| '判定できませんでした'` で右側が使われる |

`filter` を通さずに `join` すると、`undefined` が空文字として扱われ `'…傘… / '` のように区切りだけが残ります。

> ⚠️ `Boolean` は `undefined` 以外に `null`・`''`・`0`・`false`・`NaN` も false にするので、これらも消えます。
> ここは「文字列か `undefined`」しか来ないので問題ありませんが、`0` を残したい数値の配列などでは
> `.filter((x) => x !== undefined)` と条件を明示します。

</details>

`src/mastra/index.ts` に登録します（8-2・8-3 と同じ要領です）。

```ts
// src/mastra/index.ts
import { packingBranchWorkflow } from './workflows/packing-branch-workflow.ts'

export const mastra = new Mastra({
  agents: { travelAgent },
  workflows: { helloWorkflow, tripPrepWorkflow, packingBranchWorkflow },   // ← 追加
  // storage / observability は Step 7 のまま
})
```

### 動かして確かめる

Studio の Workflows タブで `packing-branch-workflow` を開き、`city` を **`Sapporo`** と **`Naha`** で1回ずつ実行します。
グラフ上で実行されるのは片方の枝（`rainy` か `dry`）だけで、もう一方は動きません（実行済みのステップがハイライトされます）。

> ⚠️ 8-3 と同じく**ローマ字で入力してください**（`"札幌"` は geocoding が0件を返して最初のステップで止まります）。
> また予報は日々変わるので、どちらの都市がどちらの枝に入るかは実行日で変わります。**結果が違っても問題ありません**
> —— 動いた枝が、表示された降水確率と条件（50% 以上か未満か）に合っているかを見てください。

### 実験：条件を書き換えて、分岐の仕様を体感する

座学 [§6-2](../../00-overview/06-workflow.md) には「`.branch()` は `if / else if` ではない」とあります。読んだだけでは
腑に落ちにくいので、実際に壊してみます。`.branch([...])` の条件を書き換えて、そのつど Studio で `Sapporo` を実行します。

| 実験 | 書き換える条件 | 予想してから実行する | 分かること |
|---|---|---|---|
| ① 両方 true | `maxPrecipitation >= 0` と `maxPrecipitation <= 100` | 何本の枝が動くか？ | **true になった枝はすべて動く**。合流ステップの出力が `枝1の文 / 枝2の文` の2件になる |
| ② 両方 false | `maxPrecipitation > 100` と `maxPrecipitation < 0` | エラーになるか？ | **エラーにならず成功する**。どの枝も動かず、合流ステップには空の入力が届いて `判定できませんでした` が返る |

実験が終わったら、条件を元の `>= 50` / `< 50` に戻してください。

> ⚠️ **② が成功してしまうのは、合流ステップの `inputSchema` が `.optional()` だから**です。`.optional()` を外すと、
> 分岐が空振りしたときに検証エラーで止まります。「片方の枝しか動かない」が前提の設計だと、条件が漏れたときに
> 気づきにくい（成功として通り抜ける）点に注意してください。排他にしたいときは、本編のように条件どうしを打ち消し合う形に書きます。

> ⚠️ Studio の `Debug` トグルを ON にすると、`.branch()` は**条件が true の最初の1本だけ**を実行して止まります
> （[reference/mastra-studio.md](../../reference/mastra-studio.md) §5-3）。実験 ① で「両方動く」ことを見るときは
> **Debug を OFF** にしてください。

> 💡 **`.parallel()` との違い。** 「条件に関係なく**必ず全部**動かしたい」なら `.branch()` ではなく `.parallel([stepA, stepB])`
> です。全部動くので `.optional()` は要りません。次のステップには `{ stepA の id: その出力, stepB の id: その出力 }` の形で届きます
> （キーは**ステップの `id`**）。

## 8-5. 条件付きループで作り直させる — `.dountil()`（45分）

「持ち物の提案を作り、**エージェント自身に十分か判定させ、足りなければ書き直させる**」ループを組みます。
[Deep Search テンプレート](../../02-templates/05-deep-search.md)（§4-2）が検索でやっているのと同じ型を、自分の手で小さく書きます。

入力は都市名と**天気の要約**（自分で書く文字列）にします。天気 API は呼ばないので、費用がかかるのは LLM の呼び出しだけです
（1周ごとに1回、最大3回）。

```ts
// src/mastra/workflows/refine-workflow.ts
import { createWorkflow, createStep } from '@mastra/core/workflows'
import { z } from 'zod'

// ループ本体の入力と出力は同じ形にそろえる（理由は下の表）
const refineSchema = z.object({
  city: z.string(),
  summary: z.string(),
  suggestion: z.string(),
  isEnough: z.boolean(),
})
type RefineData = z.infer<typeof refineSchema>

const refineResultSchema = z.object({ suggestion: z.string(), isEnough: z.boolean() })

const refineStep = createStep({
  id: 'refine-suggestion',
  inputSchema: refineSchema,
  outputSchema: refineSchema,
  execute: async ({ inputData, mastra }): Promise<RefineData> => {
    const agent = mastra.getAgentById('travel-agent')
    // 2周目以降は、前回の提案が inputData.suggestion に入っている
    const previous: string = inputData.suggestion
      ? `\n\n前回の提案:\n${inputData.suggestion}\n\n足りない点を直して、より良い提案にしてください。`
      : ''
    const response = await agent.generate(
      `行き先: ${inputData.city}\n天気: ${inputData.summary}\n\n` +
        `この天気を踏まえて持ち物を提案してください。天気の再取得は不要です。${previous}\n\n` +
        `提案が「必須」と「あると便利」に分かれ、各アイテムに理由が付いていれば isEnough を true にしてください。`,
      // Day1 Step 3-3 と同じ書き方: LLM に「この形で答えて」と伝える
      { structuredOutput: { schema: refineResultSchema } },
    )
    // city / summary は入力をそのまま引き継ぎ、suggestion / isEnough だけ上書きする
    return { ...inputData, ...response.object }
  },
})

export const refineWorkflow = createWorkflow({
  id: 'refine-workflow',
  inputSchema: z.object({ city: z.string(), summary: z.string() }),
  outputSchema: refineSchema,
})
  // 1周目の入力をループ本体の形にそろえる
  .map(async ({ inputData }): Promise<RefineData> => ({ ...inputData, suggestion: '', isEnough: false }))
  .dountil(refineStep, async ({ inputData, iterationCount }): Promise<boolean> => {
    if (inputData.isEnough) return true // 十分と判定された → ループを抜ける
    if (iterationCount >= 3) return true // 上限に達した → 諦めて抜ける
    return false // まだ不十分 → もう一周
  })
  .commit()
```

登録します。

```ts
// src/mastra/index.ts
import { refineWorkflow } from './workflows/refine-workflow.ts'

export const mastra = new Mastra({
  agents: { travelAgent },
  workflows: { helloWorkflow, tripPrepWorkflow, packingBranchWorkflow, refineWorkflow },   // ← 追加
  // storage / observability は Step 7 のまま
})
```

### 読むときの要点

| 要点 | 内容 |
|---|---|
| **本体の入力と出力を同じ形にする** | 2周目以降、`refineStep` の入力は「直前のステップの出力」ではなく**「`refineStep` 自身の前回の出力」**になる。出力に `city` / `summary` が無いと、2周目の入力検証で落ちる（`{ ...inputData, ...response.object }` で引き継いでいるのはこのため） |
| `.map()` で1周目をそろえる | 1周目の入力だけは直前のステップの出力。ワークフローの入力 `{ city, summary }` に `suggestion` と `isEnough` の初期値を足して、本体の形にする |
| 条件関数の `inputData` | 本体が**今回返した出力**。`isEnough` はここで読む |
| `iterationCount` | 初回の評価時に `1`。上限は自分で書く |

> ⚠️ **形が合っていなくても `tsc` は通ります。** 本体の出力から `city` を削っても型エラーにはならず、**2周目に入って初めて**
> `Step input validation failed` で止まります（`@mastra/core` 1.67.0 で確認）。ループを書いたら、エディタが黙っていても
> 「2周目が実際に動くか」を必ず実行して確かめてください。

上限に達したときの扱いは2通りあります。このコードは前者です。

| 書き方 | 結果 |
|---|---|
| `return true`（このコード） | ループを抜けて**成功として返る**。不十分なまま抜けたかどうかは、出力の `isEnough` が `false` のままであることで分かる |
| `throw new Error('…')`（座学 §6-2 の例） | `start()` が**例外で reject される**。`status: 'failed'` の結果としては返らないので、呼び出し側で `try / catch` が要る |

### 動かして確かめる

Studio の Workflows タブで `refine-workflow` を開き、次を入力して実行します。

```json
{ "city": "Sapporo", "summary": "3日間とも雨。最高気温 12℃、最低気温 6℃" }
```

Traces 画面で `refine-suggestion` が**何回**実行されたかを数えます。

> 💡 **1周で終わることもあります**（エージェントが最初の提案を十分と判定した場合）。ループが回るところを見たいときは、
> プロンプトの基準を厳しくします。たとえば `各アイテムに重量の目安（g）まで書かれていれば` を足すと、
> 1周目では満たせず、2周目以降で書き直す動きになりやすくなります（LLM の判断なので、回数は実行ごとに変わります）。

### 実験：`.dountil()` は後判定である

座学 [§6-2](../../00-overview/06-workflow.md) は「**`.dountil()` と `.dowhile()` は後判定**」と書いています。
条件を**最初から成立する**ものに書き換えて、それでも本体が動くかを確かめます。

```ts
.dountil(refineStep, async (): Promise<boolean> => true)   // 何を返されても、最初から「もう十分」
```

同じ入力で実行し、Traces で `refine-suggestion` を数えると **1回は必ず動いています**。条件は本体を実行した**後**に
評価されるので、「条件によっては1回も実行しない」は書けません（`@mastra/core` 1.67.0 で確認）。

| やりたいこと | 書き方 |
|---|---|
| 条件が最初から成立していたら**1回も実行しない** | ループの手前に `.branch()` を置き、ループごと迂回させる |
| 条件が成立する**間**繰り返す | `.dountil` を **`.dowhile`** に替え、条件を反転する（下のコード） |

```ts
// 上と同じ動き。dountil は「条件が true になるまで」、dowhile は「条件が true の間」
.dowhile(refineStep, async ({ inputData, iterationCount }): Promise<boolean> => !inputData.isEnough && iterationCount < 3)
```

`.dowhile()` も本体を先に実行してから条件を評価します（差は「条件が true で抜ける」か「true の間続ける」かだけです）。
実験が終わったら、条件を元の形に戻してください。

## 8-6. 人間の承認を挟む — suspend / resume（30分）

ワークフローは、**途中で止まって人間の判断を待つ**ことができます（human-in-the-loop）。
「`travelAgent` が旅程案を下書きし、人間が承認したら送信する」を組みます。送信のように**取り消せない処理の前**に
人間を挟みたい、という場面を想定しています。

座学 [§6-3](../../00-overview/06-workflow.md) の `approvalWorkflow` を雛形に、固定文字列だった下書きを
`travelAgent` の生成に差し替えます。

```ts
// src/mastra/workflows/approval-workflow.ts
import { createWorkflow, createStep } from '@mastra/core/workflows'
import { z } from 'zod'

const planSchema = z.object({ plan: z.string() })
type PlanOutput = z.infer<typeof planSchema>

// 下書き: travelAgent に旅程案を書かせる（ここは普通のステップ）
const draftPlanStep = createStep({
  id: 'draft-plan',
  inputSchema: z.object({ city: z.string() }),
  outputSchema: planSchema,
  execute: async ({ inputData, mastra }): Promise<PlanOutput> => {
    const agent = mastra.getAgentById('travel-agent')
    const response = await agent.generate(
      `行き先: ${inputData.city}\n\n2泊3日の旅程案を、日ごとに3行程度で作ってください。天気の取得は不要です。`,
    )
    return { plan: response.text }
  },
})

const approveOutputSchema = z.object({ plan: z.string(), approved: z.boolean() })
const suspendSchema = z.object({ plan: z.string(), question: z.string() })
const resumeSchema = z.object({ approved: z.boolean() })

// 承認待ち: ここで止めて人間の判断を待つ
const approveStep = createStep({
  id: 'approve',
  inputSchema: planSchema,
  outputSchema: approveOutputSchema,

  suspendSchema, // ワークフロー → 人間
  resumeSchema, // 人間 → ワークフロー

  // suspend() の戻り値は Mastra 内部の型なので、ここだけ戻り値の型注釈を付けない
  execute: async ({ inputData, resumeData, suspend }) => {
    // 1回目: resumeData が無い → 中断して人間の入力を待つ
    if (!resumeData) {
      return suspend({ plan: inputData.plan, question: 'この旅程で確定してよいですか？' })
    }
    // 2回目（resume 後）: resumeData が入っている → 続きを実行する
    return { plan: inputData.plan, approved: resumeData.approved }
  },
})

const sendPlanOutputSchema = z.object({ result: z.string() })
type SendPlanOutput = z.infer<typeof sendPlanOutputSchema>

// 送信: 承認されたときだけ本処理を行う
const sendPlanStep = createStep({
  id: 'send-plan',
  inputSchema: approveOutputSchema,
  outputSchema: sendPlanOutputSchema,
  execute: async ({ inputData }): Promise<SendPlanOutput> => ({
    result: inputData.approved ? `送信しました:\n${inputData.plan}` : '差し戻されました',
  }),
})

export const approvalWorkflow = createWorkflow({
  id: 'approval-workflow',
  inputSchema: z.object({ city: z.string() }),
  outputSchema: sendPlanOutputSchema,
})
  .then(draftPlanStep)
  .then(approveStep)
  .then(sendPlanStep)
  .commit()
```

> 💡 **`suspendSchema,` だけで済むのは、TypeScript のプロパティ省略記法です。** キー名と値の変数名が同じなら、
> `キー: 変数` を `変数` だけに短縮できます。
>
> ```ts
> suspendSchema,                  // 省略形
> suspendSchema: suspendSchema,   // 上と同じ意味
> ```
>
> `createStep` が受け取るキー名は `suspendSchema` / `resumeSchema` で、同名の変数を用意したので省略できています。
> 一方 `outputSchema: approveOutputSchema` は名前が違うため、キーを明示しています。
> `new Mastra({ agents: { travelAgent } })` も同じ省略形です（キー名は `travelAgent` になります）。

登録します。

```ts
// src/mastra/index.ts
import { approvalWorkflow } from './workflows/approval-workflow.ts'

export const mastra = new Mastra({
  agents: { travelAgent },
  workflows: {
    helloWorkflow,
    tripPrepWorkflow,
    packingBranchWorkflow,
    refineWorkflow,
    approvalWorkflow,   // ← 追加
  },
  // storage / observability は Step 7 のまま
})
```

### 動かして確かめる

Studio の Workflows タブで `approval-workflow` を開き、`city` に `Sapporo` を入れて実行します。

1. `draft-plan` が終わり、**`approve` ステップで `suspended`（入力待ち）になって止まる**。`suspendSchema` に渡した旅程案と質問が見える
2. `resumeSchema` の形（`approved`）の入力欄が出る。`true` で再開すると `send-plan` まで進み、`送信しました:` で終わる
3. もう一度最初から実行し、今度は `false` で再開する。`差し戻されました` で終わる
   - Form のチェックボックスは外すと「未入力」扱いになり `Required` で止まります。**`{ } JSON` タブに切り替えて
     `{ "approved": false }` と入力**してから Resume してください（[reference/mastra-studio.md](../../reference/mastra-studio.md) §5-2）

画面の見方は [reference/mastra-studio.md](../../reference/mastra-studio.md) §5-2 にまとめています。

| 確かめること | 見どころ |
|---|---|
| `approve` の `execute` は**2回動く** | 再開すると、止まったステップが**先頭から再実行**される。1回目は `resumeData` が空、2回目は入っている。`if (!resumeData)` はこの違いで見分けている |
| `draft-plan` は**再実行されない** | 再開で動き直すのは止まったステップだけ。LLM を呼ぶ下書きは1回きり。**再開のたびに旅程案が変わったりはしない** |

> ⚠️ **`suspend()` と Studio の `Debug` トグルは別物です。** どちらも「止まる」が、前者は**コードに書いた場所だけ**で止まる
> 承認待ち（状態は `suspended`、本番でも起きる）、後者は Studio が**全ステップの境界**で止める開発用のステップ実行
> （状態は `paused`）です。承認の動きを見るときは `Debug` を OFF にしてください。対比表は
> [reference/mastra-studio.md](../../reference/mastra-studio.md) §5-3。

> 💡 コードから呼ぶ場合、中断した実行の結果（`status === 'suspended'`）の `suspendPayload` は、
> `suspend()` に渡した値が**ステップの `id` でくるまれて**入ります（`{ approve: { plan, question } }`）。
> 呼び出し側の書き方は座学 [§6-3](../../00-overview/06-workflow.md) を参照してください。

**発展（時間があれば）**: 中断中の状態は Step 7 で設定した **Storage にスナップショットとして保存**されています。
それを確かめます。`approve` で止まったまま `Ctrl+C` で `npm run dev` を落とし、起動し直して、**同じ実行を再開**してみてください。
再開できれば、承認が翌日になってもサーバーの再起動を挟んでも続きから進められる、ということです。
再開できないときは、`src/mastra/index.ts` の `storage` が抜けていないか（メモリ上にしか残らない構成になっていないか）を疑います。

## 8-7. エージェントにワークフローを呼ばせる（30分）

8-3・8-5・8-6 はいずれも「**ワークフローがエージェントを呼ぶ**」向きでした。逆向き、つまり
**エージェントが自分の判断でワークフローを実行する**こともできます。8-2 の 💡 で触れた3つ目の繋ぎ方です。

### 何を作るのか

**8-4 で作った `packingBranchWorkflow` を `travelAgent` に持たせます。** 新しいワークフローは書きません。
チャットで「Sapporo に行きます。何を持っていけばいいですか？」と聞いたとき、エージェントが
**自分でワークフローを呼ぶかどうかを決める**——その判断を観察するのがこの節の目的です。

8-4 と比べると、同じワークフローの起動を**誰が決めるか**が違います。

| | 起動を決めるのは | 実行するかどうか |
|---|---|---|
| 8-4（Studio から実行） | 人間 | 必ず実行される |
| **8-7（エージェント経由）** | **LLM** | **呼ばれないこともある** |

### 1. ワークフローに `description` を付ける

**LLM はこの説明文を読んで呼ぶかどうかを決めます。** 無いと `Workflow: <名前>` という自動生成の
説明になり、判断材料がありません。8-4 のファイルに1行足します。

```ts
// src/mastra/workflows/packing-branch-workflow.ts（createWorkflow に1行追加）
export const packingBranchWorkflow = createWorkflow({
  id: 'packing-branch-workflow',
  description: '指定した都市の降水確率を調べ、雨の備えか日差しの備えかを判定して持ち物の助言を返す', // ← 追加
  inputSchema: z.object({ city: z.string() }),
  outputSchema: z.object({ result: z.string() }),
})
```

### 2. エージェントに渡す

```ts
// src/mastra/agents/travel-agent.ts
import { packingBranchWorkflow } from '../workflows/packing-branch-workflow.ts'   // ← 追加

export const travelAgent = new Agent({
  // id / name / instructions / model / tools / memory は Step 7 のまま
  workflows: { packingBranch: packingBranchWorkflow },   // ← 追加
})
```

### LLM からはこう見える

渡したワークフローは、**自動的にツール1つに変換されて** LLM に提示されます。

| 項目 | 値 |
|---|---|
| ツール名 | **`workflow-packingBranch`** —— `workflow-` + **オブジェクトのキー** |
| 説明 | `createWorkflow` の `description`（無ければ `Workflow: packingBranch`） |
| 引数 | **`{ inputData: { city: string } }`** —— ワークフローの `inputSchema` が `inputData` に**包まれる** |
| 戻り値 | 成功時 `{ result, runId }` / 失敗時 `{ error, runId }` |

> ⚠️ **ツール名を決めるのはキーであって、ワークフローの `id` ではありません。** 上の例なら
> `packing-branch-workflow`（`id`）ではなく `packingBranch`（キー）が使われます。Day1 Step 4 の `tools` と同じ規則です
> （[day1/04-tools.md](../day1/04-tools.md) §4-3）。トレースで探すときはこの名前で探してください。

### 動かして確かめる

**実験①: このまま聞いてみる。** Studio のチャットで `travelAgent` に聞きます。

```
Sapporo に行きます。何を持っていけばいいですか？
```

トレースを見ると、**`workflow-packingBranch` は呼ばれず、`get-weather` が呼ばれています**
（`@mastra/core` 1.67.0 + `gpt-5.6-luna` で実測）。答えは返ってくるので、**一見すると成功して見える**のが厄介なところです。

登録しただけでは使われません。Day1 Step 4-3 で書いた `instructions` が
**「必ず `get-weather` ツールを使って実際の天気を確認してから」**と名指ししているうえ、
`weatherTool` のほうが汎用的なので、LLM はそちらを選びます。

**実験②: `instructions` に1行足す。**

```ts
// src/mastra/agents/travel-agent.ts の instructions の中に追記
持ち物の助言を求められたときは、必ず packingBranch ワークフローを使って判定してください。
```

同じ質問をすると、今度は `workflow-packingBranch` が呼ばれます。引数はこの形です（実測）。

```json
{ "inputData": { "city": "Sapporo" } }
```

| 分かること | 内容 |
|---|---|
| **登録は「使える」であって「使う」ではない** | `workflows:` に渡すのは選択肢を増やすだけ。呼ぶかどうかは毎回 LLM が決める |
| **呼ばせたいなら `description` と `instructions` の両方で誘導する** | `description` は「何をするものか」、`instructions` は「いつ使うか」を伝える |
| **競合する道具があると取られる** | 似た仕事をするツールが既にあると、そちらが選ばれることがある |

> 💡 **確実に実行させたいなら、そもそもエージェントに任せない**のが答えです。8-4 のように Studio や
> コードから直接ワークフローを起動すれば、LLM の判断は入りません。**8-7 の構成は「使うかどうかも含めて
> 任せたい」ときだけ**選びます。

### どちらの向きを選ぶか

| 向き | 書き方 | 向いている場面 |
|---|---|---|
| **ワークフロー → エージェント**（8-3・8-5・8-6） | ステップ内で `mastra.getAgentById()` | **手順は決まっていて、一部の判断だけ任せたい。**実務ではこちらが主 |
| **エージェント → ワークフロー**（8-7） | `new Agent({ workflows: {...} })` | **やるかどうかから任せたい。**会話の流れで必要になったら決まった手順を実行させる |

> ⚠️ **循環に注意してください。** `tripPrepWorkflow`（8-3）は中で `travelAgent` を呼びます。これを
> `travelAgent` の `workflows` に渡すと、**エージェント → ワークフロー → 同じエージェント → …** と
> 呼び合う形になりえます。ここで `packingBranchWorkflow`（8-4）を選んでいるのは、**LLM を1回も呼ばない
> ワークフローだから**です。エージェントに持たせるワークフローは、この循環が起きないかを確かめてください。

## 8-8. 考える（時間内で議論・メモ）

同じことが Agent + Tool でもできます。**なぜワークフローを使うのか**を自分の言葉で整理してください。

ヒント: 再現性、コスト（LLM 呼び出し回数）、デバッグのしやすさ、監査ログ、中断・再開（人間の承認を挟む）。

8-4〜8-6 を書いたあとなら、次の問いにも答えられるはずです。

- 8-4: 分岐を**エージェントの判断**に任せた場合と、**コードの条件**で書いた場合で、何が変わるか（再現性・コスト・条件漏れ）
- 8-5: ループの終了判定を LLM に任せたとき、**無限に回らない**ためにどんな保険を書いたか。上限を超えた場合は、呼び出し側にどう伝えるのがよいか
- 8-6: 承認を待つあいだ、**サーバーはリクエストを握り続ける必要があるか**（スナップショットが保存されていると何が変わるか）
- 8-7: ワークフローを**確実に実行したい**とき、エージェントに持たせる構成は適しているか。適さないなら、代わりに何を選ぶか

**チェックポイント**
- [ ] Studio のグラフビューでワークフローの実行を追えた
- [ ] ステップ間のスキーマ整合を理解した
- [ ] `.branch()` で条件どおりに枝が振り分けられ、条件を両方 true にすると両方の枝が動くことを確かめた
- [ ] ループ本体の入力と出力を同じ形にする理由を説明できる
- [ ] 条件が最初から成立していても `.dountil()` の本体が1回動く（後判定）ことを確かめた
- [ ] `suspend()` で止まったワークフローを Studio から再開でき、承認と差し戻しの両方を試した
- [ ] エージェントにワークフローを持たせ、**登録しただけでは呼ばれない**ことと、`instructions` で呼ばせられることを確かめた
- [ ] Agent と Workflow の使い分けを説明できる
