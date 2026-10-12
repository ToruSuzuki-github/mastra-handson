# Step 4. ツール作成：実 API を叩く（90分）

ここが Mastra の核心です。エージェントに「自分で外部データを取りに行く能力」を与えます。

## 4-1. ツールの基本形を理解する（15分）

ツールは必ず `createTool()` で定義します。

```ts
import { createTool } from '@mastra/core/tools'
import { z } from 'zod'

export const exampleTool = createTool({
  id: 'tool-id',              // エージェントが認識する識別子
  description: '...',          // ★ここが最重要。LLM はこれを読んで使うか判断する
  inputSchema: z.object({ ... }),
  outputSchema: z.object({ ... }),
  execute: async (input, context) => {
    // input: inputSchema でバリデート済みのデータ
    // context: mastra, requestContext, abortSignal などの実行メタデータ
    return { ... }
  },
})
```

> **落とし穴**: 素のオブジェクトでツールを定義すると、**エラーにならないまま実行されません**。必ず `createTool()` を使ってください。公式に明記されている挙動です。

## 4-2. 天気取得ツールを作る（45分）

無料・キー不要の Open-Meteo API を使います。

天気の取得は「都市名 → 緯度経度」と「緯度経度 → 天気」の **2回の API 呼び出し**でできています。
すべてを `execute` に書くと長くなるので、1つの仕事ごとに関数へ切り出し、`execute` には流れだけを書きます。
同じ `weather-tool.ts` に、**① 補助関数 → ② ツール本体** の順で書いてください。

### ① 補助関数（API を1回ずつ叩く）

```ts
// src/mastra/tools/weather-tool.ts
import { createTool } from '@mastra/core/tools'
import { z } from 'zod'

// Open-Meteo ジオコーディング API のレスポンス形（使う部分だけ）
interface GeocodingResult {
  latitude: number
  longitude: number
  name: string
}

interface GeocodingResponse {
  results?: GeocodingResult[]
}

interface GeoLocation {
  latitude: number
  longitude: number
  name: string
}

// 1) 都市名 → 緯度経度
async function geocode(city: string): Promise<GeoLocation> {
  const url = new URL('https://geocoding-api.open-meteo.com/v1/search')
  url.searchParams.set('name', city)
  url.searchParams.set('count', '1')
  url.searchParams.set('language', 'ja')

  const res: Response = await fetch(url)
  if (!res.ok) {
    throw new Error(`ジオコーディング API がエラーを返しました: ${res.status}`)
  }
  const data: GeocodingResponse = await res.json()

  if (!data.results || data.results.length === 0) {
    throw new Error(`都市が見つかりません: ${city}`)
  }

  const { latitude, longitude, name } = data.results[0]
  return { latitude, longitude, name }
}

// Open-Meteo 天気予報 API のレスポンス形（使う部分だけ）
interface WeatherApiResponse {
  current: {
    temperature_2m: number
  }
  daily: {
    time: string[]
    temperature_2m_max: number[]
    temperature_2m_min: number[]
    precipitation_probability_max: number[]
  }
}

interface DailyForecast {
  date: string
  maxTemp: number
  minTemp: number
  precipitationProbability: number
}

interface WeatherData {
  currentTemperature: number
  forecast: DailyForecast[]
}

// 2) 緯度経度 → 天気
async function fetchWeather(latitude: number, longitude: number): Promise<WeatherData> {
  const url = new URL('https://api.open-meteo.com/v1/forecast')
  url.searchParams.set('latitude', String(latitude))
  url.searchParams.set('longitude', String(longitude))
  url.searchParams.set('current', 'temperature_2m')
  url.searchParams.set(
    'daily',
    'temperature_2m_max,temperature_2m_min,precipitation_probability_max',
  )
  url.searchParams.set('timezone', 'auto')

  const res: Response = await fetch(url)
  if (!res.ok) {
    throw new Error(`天気 API がエラーを返しました: ${res.status}`)
  }
  const w: WeatherApiResponse = await res.json()

  return {
    currentTemperature: w.current.temperature_2m,
    forecast: w.daily.time.map(
      (date: string, i: number): DailyForecast => ({
        date,
        maxTemp: w.daily.temperature_2m_max[i],
        minTemp: w.daily.temperature_2m_min[i],
        precipitationProbability: w.daily.precipitation_probability_max[i],
      }),
    ),
  }
}
```

| 関数 | 受け取るもの | 返すもの |
|---|---|---|
| `geocode` | 都市名 | 緯度・経度・地名。API がエラーを返したとき、または都市が見つからないときはエラーを投げる |
| `fetchWeather` | 緯度・経度 | 現在の気温と日別の予報（ツールの出力から `city` を除いた形）。API がエラーを返したときはエラーを投げる |

<details>
<summary>💡 <code>new URL()</code> と <code>url.searchParams.set()</code> は何をしている？（クリックで開く）</summary>

`URL` は、URL を**部品に分けて安全に組み立てる・読み取る**ための JavaScript 標準のクラスです。
Node.js では `import` なしでそのまま使えます。`new URL('https://…')` で、文字列の URL から `URL` オブジェクトを作ります。

このコードでは、URL の末尾につく `?name=Sapporo&count=1&language=ja` の部分（**クエリパラメータ**）を、
`url.searchParams.set(キー, 値)` で1つずつ追加しています。`searchParams` は `URL` オブジェクトが持つ
クエリパラメータ専用の窓口で、`set` は「そのキーの値を設定する（同じキーがあれば上書き）」という意味です。

```ts
const url = new URL('https://geocoding-api.open-meteo.com/v1/search')
url.searchParams.set('name', 'Sapporo')
url.searchParams.set('count', '1')
url.searchParams.set('language', 'ja')

url.href
// → 'https://geocoding-api.open-meteo.com/v1/search?name=Sapporo&count=1&language=ja'
```

**文字列連結（`` `...?name=${city}` ``）ではなく `URL` を使う理由は、特殊文字を自動で変換してくれるからです。**
URL には日本語・スペース・`&`・`=` などをそのまま入れられず、`%E6%9C%AD%E5%B9%8C` のような形
（パーセントエンコード）に変換する必要があります。`set` はこの変換を自動で行うので、
`city` に何が入っても URL が壊れません。

```ts
url.searchParams.set('name', '札幌')
// → …?name=%E6%9C%AD%E5%B9%8C   （自動でエンコードされる）
```

| 書き方 | 意味 |
|---|---|
| `new URL('https://…')` | URL 文字列から `URL` オブジェクトを作る |
| `url.searchParams.set('count', '1')` | クエリパラメータ `count=1` を追加する |
| `url.href` | 完成した URL 全体の文字列 |

補足:

- `set` に渡せる値は**文字列だけ**です。`count` が `'1'`、`fetchWeather` の緯度・経度が `String(latitude)` と
  書かれているのはこのためです（数値のまま渡すと TypeScript がエラーにします）。
- `const url` でも `url.searchParams.set(...)` で中身を変えられます。`const` が禁じるのは
  `url` 自体を別のオブジェクトに**付け替える**ことで、オブジェクトの中身の変更ではありません。

</details>

<details>
<summary>💡 <code>fetch()</code> とは？ <code>URL</code> クラスの関数？（クリックで開く）</summary>

**いいえ、`fetch()` は `URL` クラスの関数ではありません。別の仕組みです。**
2つは役割が分かれていて、`geocode` 内の流れは次のように読めます。

| 行 | 担当 | 役割 |
|---|---|---|
| `new URL(...)` と `url.searchParams.set(...)` | `URL` クラス | **どこへ**リクエストするか（宛先の URL）を組み立てる |
| `await fetch(url)` | `fetch()` 関数 | その URL へ **HTTP リクエストを送り**、応答を受け取る |
| `await res.json()` | `Response` オブジェクト | 応答の本文を JSON として読み取り、JavaScript のオブジェクトに変換する |

`fetch()` は、ブラウザや Node.js（本ハンズオンの Node 22 以降）に**標準で組み込まれたグローバル関数**で、`import` は不要です。
`URL` クラスとは無関係に単体で存在し、`fetch('https://…')` のように**文字列の URL を直接渡すこともできます**。
このコードで `URL` オブジェクトを作っているのは、上で説明したとおりクエリパラメータを安全に組み立てるためで、
`fetch` には完成した `URL` オブジェクトをそのまま渡せます。

`await` が2回出てくるのは、待つものが2段階に分かれているためです。

| 書き方 | 待つもの | 待ち終わったとき |
|---|---|---|
| `await fetch(url)` | サーバーからの応答の**先頭部分**（ステータスなど） | `Response` オブジェクト `res` が手に入る。本文はまだ読み込み中のことがある |
| `await res.json()` | 応答の**本文**を最後まで読んで JSON として解釈すること | JavaScript のオブジェクト `data` が手に入る |

> **注意**: `fetch()` は、サーバーが 404 や 500 を返しても**エラー（例外）を投げません**。
> 例外になるのは通信そのものに失敗したとき（オフラインなど）だけです。そのため、HTTP のエラーは
> `res.ok`（ステータスが 200〜299 なら `true`）で自分で確認します。上のコードで `if (!res.ok)` を
> 書いているのはこのためで、確認しないと、エラー応答の本文を成功時のデータとして読んでしまいます
> （たとえば `data.results` が無いだけに見えて、「都市が見つかりません」という実態と違うエラーになります）。

</details>

### ② ツール本体（流れだけを書く）

①と同じファイルの下に続けて書きます。

```ts
const weatherOutputSchema = z.object({
  city: z.string(),
  currentTemperature: z.number(),
  forecast: z.array(
    z.object({
      date: z.string(),
      maxTemp: z.number(),
      minTemp: z.number(),
      precipitationProbability: z.number(),
    }),
  ),
})

type WeatherOutput = z.infer<typeof weatherOutputSchema>

export const weatherTool = createTool({
  id: 'get-weather',
  description:
    '指定した都市の現在の天気と今後7日間の予報を取得する。気温・降水確率・風速がわかる。',
  inputSchema: z.object({
    city: z
      .string()
      .describe('都市名。英語表記（ローマ字）で指定する。例: Naha, Sapporo, Tokyo'),
  }),
  outputSchema: weatherOutputSchema,
  execute: async ({ city }): Promise<WeatherOutput> => {
    const { latitude, longitude, name } = await geocode(city)
    const weather = await fetchWeather(latitude, longitude)
    return { city: name, ...weather }
  },
})
```

`execute` の3行が「都市名 → 緯度経度 → 天気」という処理の流れそのものです。

- `geocode` と `fetchWeather` は `async` 関数（Promise を返す）なので、呼ぶ側にも `await` が要ります。
  詳しくは [Step 3 の `await` の解説](./03-response-modes.md) を参照してください。
- `...weather` は `weather` の中身（`currentTemperature` と `forecast`）をその場に展開する書き方です。
  `outputSchema` と同じ形のオブジェクトが返ります。

<details>
<summary>💡 <code>{ city: name, ...weather }</code> の <code>...</code> って何？（クリックで開く）</summary>

`...` は **スプレッド構文**（spread syntax）と呼ばれる JavaScript の書き方で、
オブジェクトの中身を**その場に展開してコピーする**働きをします。

`fetchWeather` の戻り値 `weather` が、たとえば次の形だったとします。

```ts
const weather: WeatherData = {
  currentTemperature: 12.3,
  forecast: [{ date: '2026-09-22', maxTemp: 20, minTemp: 11, precipitationProbability: 30 }],
}
```

このとき `{ city: name, ...weather }` は、キーを1つずつ書き出した次のコードと同じ結果になります。

```ts
return {
  city: name,                                    // '札幌市' など
  currentTemperature: weather.currentTemperature,
  forecast: weather.forecast,
}
```

**なぜ使うのか。** `fetchWeather` が返すのは `city` を持たない `{ currentTemperature, forecast }` です。
ツールの出力（`outputSchema`）には `city` も必要なので、「`city` を足しつつ、残りはそのまま使う」ために
`...weather` と書いています。スプレッドを使わないと、フィールドが増えるたびに `execute` 側にも書き足すことになります。

| 書き方 | 意味 |
|---|---|
| `{ city: name, ...weather }` | `city` を持つ新しいオブジェクトに、`weather` のキーをすべて並べる |
| `{ ...weather, city: name }` | 同じ内容。ただし `weather` に `city` があれば `name` で上書きされる（後ろが勝つ） |

補足:

- **同じキーがあれば、後ろに書いたほうが勝ちます。** `{ city: name, ...weather }` で `weather` にも `city` があれば、
  `weather` の値で上書きされます。並べる順序が意味を持ちます。
- **コピーは浅いです。** `forecast` の配列そのものは複製されず、`weather.forecast` と同じ配列を指します。
  ここでは返すだけなので問題になりません。
- `...` は関数の引数（`(...args) => {}`）や分割代入（`const { a, ...rest } = obj`）にも出てきます。
  書き方は同じですが、こちらは「残りをまとめて受け取る」という逆向きの働きで、**レスト構文**と呼ばれます。

</details>

<details>
<summary>💡 <code>z.string().describe()</code> の <code>describe()</code> って何？（クリックで開く）</summary>

`describe()` は zod のメソッドで、スキーマに**「この値は何か」を説明する文字列（説明文）を付ける**ものです。
`z.string()` は「文字列であること」しか表さないので、`city` に何を入れるべきかまでは伝わりません。
そこで `.describe('…')` を足して、値の意味・書式・例を補います。

```ts
city: z
  .string()                                  // 型: 文字列
  .describe('都市名。英語表記（ローマ字）で指定する。例: Naha, Sapporo, Tokyo')  // 説明文
```

**誰のための説明文か → LLM です。**
Mastra は `inputSchema` を LLM に渡すツール定義（JSON Schema）へ変換し、`describe()` の文字列は
各引数の `description` として一緒に送られます。LLM は、ツールの `description`（**いつ使うか**）を読んで
ツールを選び、引数の `describe()`（**何をどう渡すか**）を読んで引数の値を決めます。

| 書く場所 | 何を伝えるか | 読むタイミング |
|---|---|---|
| `createTool` の `description` | このツールは**何ができるか・いつ使うか** | ツールを使うか選ぶとき |
| 引数の `.describe()` | この引数に**何をどんな形式で入れるか** | 引数の値を決めるとき |

```
describe() なし → LLM に見えるのは「city: string」だけ
                  → 「札幌」のように日本語のまま渡してしまうことがある
describe() あり → 「英語表記（ローマ字）で。例: Sapporo」まで見える
                  → 「Sapporo」と渡してくれる
```

補足:

- `describe()` は**バリデーションをしません**。「ローマ字でなければエラー」といったチェックにはならず、
  あくまで LLM への**ヒント**です。形式を強制したいときは `.regex()` などの検証を別途足します。
- `city` が文字列でなくても、`z.number().describe('…')` や `z.enum([...]).describe('…')` のように
  **どの型にも付けられます**。引数が増えたら、1つずつ `describe()` を書くのがおすすめです。
- 説明文を書き換えると LLM の引数の渡し方が変わります。4-4 の実験（`description` を削る）と同様に、
  `describe()` を消して Studio のトレースで `city` の値がどう変わるか試すと、効果を確認できます。

</details>

> ⚠️ **都市名はローマ字で渡すこと。** Open-Meteo の geocoding API は**日本語表記にヒットしません**（`name=札幌` は結果0件、`name=Sapporo` なら見つかる）。`language=ja` は**返ってくる地名の言語**を指定するもので、検索語の言語ではありません。
> `inputSchema` の `.describe()` にローマ字と明記しているのはこのためです。ここを「日本語でも可」と書くと、LLM が日本語のまま渡して `都市が見つかりません` で失敗します。

## 4-3. エージェントに組み込む（15分）

```ts
// src/mastra/agents/travel-agent.ts
import { Agent } from '@mastra/core/agent'
import { weatherTool } from '../tools/weather-tool.ts'

export const travelAgent = new Agent({
  id: 'travel-agent',
  name: 'Travel Agent',
  instructions: `
あなたは旅行の準備を手伝うアシスタントです。

行き先が伝えられたら、**必ず get-weather ツールを使って実際の天気を確認してから**
持ち物を提案してください。推測で気候を語らないこと。

回答するときは:
- 最初に取得した天気の要点を1〜2行でまとめる
- そのあと持ち物を箇条書きで提案し、理由を一言添える
- 日本語で回答する
`,
  model: 'openai/gpt-5.6-luna',
  // キーが LLM に見えるツール名になる。ここを `weatherTool` にすると
  // LLM にもトレースにも `weatherTool` と出る（`id` の 'get-weather' ではない）
  tools: { [weatherTool.id]: weatherTool },
})
```

> ⚠️ **`tools` のキーが、そのまま LLM に渡るツール名になります。** `id` ではありません。
> `tools: { weatherTool }` と書くと（`{ weatherTool: weatherTool }` の省略形なので）ツール名は
> `weatherTool` になり、`createTool` で付けた `id: 'get-weather'` は使われません。
> ここでは `id` と名前を揃えたいので `{ [weatherTool.id]: weatherTool }` と書いています。
> この後のトレース確認やフロントエンドの表示（[day3/](../day3/)）も `get-weather` で統一されます。

## 4-4. 動作確認と観察（15分）

```bash
npm run dev
```

Studio で「来週札幌に行きます」と入力し、**トレースを開いて**以下を確認します。

- `get-weather` ツールが呼ばれているか
- ツールに渡された引数（`city`）は何か
- ツールが返した値をエージェントがどう使ったか

**理解を深めるための実験**

1. `description` を `'天気を取得する'` だけに削ってみる → ツールが呼ばれにくくなるか観察
2. `instructions` から「必ず get-weather ツールを使って」を消す → 挙動の変化を見る
3. 存在しない都市名（「ワカンダ」など）を渡す → エラーハンドリングの挙動を見る

> **これが Day1 で最も重要な体験です。** 「description を書き換えるとエージェントの判断が変わる」という実験は、プロンプトエンジニアリングとエージェント設計の関係を一発で理解させてくれます。

**チェックポイント**
- [ ] 自作ツールがエージェントから呼ばれた
- [ ] トレースで引数と戻り値を確認した
- [ ] `description` の書き方がツール選択に影響することを確認した
