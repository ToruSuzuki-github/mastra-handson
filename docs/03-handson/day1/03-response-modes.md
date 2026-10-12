# Step 3. 応答形式の3パターン（60分）

Studio を使わず、コードからエージェントを呼びます。用途に応じた3つの取得方法を体験します。

## 3-1. `generate()` — 完全な応答を待つ（20分）

```ts
// src/index.ts
import type { FullOutput } from '@mastra/core/stream'
import { mastra } from './mastra/index.ts'

async function main(): Promise<void> {
  const agent = mastra.getAgentById('travel-agent')
  const query: string = '来週、北海道の富良野に3泊4日で行きます。何を持っていくべき？'

  console.log(`Query: ${query}\n`)

  const response: FullOutput<undefined> = await agent.generate(query)
  console.log(response.text)

  // 使用トークン数も取れる
  console.log('\n--- usage ---')
  console.log(response.usage)
}

main().catch((error: unknown): void => {
  console.error(error)
})
```

実行:

```bash
node --env-file=.env src/index.ts
```

> Node.js 22.18.0 以上なら TypeScript を直接実行できます。エラーになる場合は
> `npx tsx --env-file=.env src/index.ts` を使ってください。

> ⚠️ **`--env-file=.env` を忘れないでください。** `.env` を自動で読むのは `mastra dev` だけです。
> Mastra のライブラリ自体は dotenv を読み込まないため、素の `node` で動かすと API キーが未設定のまま
> 起動して認証エラーになります。

> **なぜ `mastra.getAgentById()` を使うのか**: エージェントを直接 import しても動きますが、Mastra インスタンス経由で取得しないと、共有ストレージ・ロギング・観測機能にアクセスできません。

## 3-2. `stream()` — 逐次表示する（20分）

`generate()` は応答が出揃うまで無音です。ユーザーに見せる UI では stream を使います。

```ts
// src/index.ts
import { mastra } from './mastra/index.ts'

async function main(): Promise<void> {
  const agent = mastra.getAgentById('travel-agent')
  const query: string = '来週、沖縄に行きます。何を持っていくべき？'

  console.log(`Query: ${query}\n`)

  const stream = await agent.stream(query)

  for await (const chunk of stream.textStream) {
    process.stdout.write(chunk)
  }

  console.log('\n\n✅ done')
}

main().catch((error: unknown): void => {
  console.error(error)
})
```

体感差を確認してください。人に見せるデモでは stream のほうが待ち時間を感じさせません。

<details>
<summary>💡 <code>await</code> は何をしているのか（クリックで開く）</summary>

エージェントの呼び出しは LLM への通信を伴い、結果が出るまで時間がかかります。JavaScript はこうした処理を
**Promise（「あとで結果を渡す」という引換券）** として返します。`await` は「その引換券が結果に
換わるまで、この関数の続きを実行せずに待つ」ための書き方です（待っている間も Node.js 全体は止まりません）。

このコードには `await` が2か所あり、**待つ対象が違います**。

| 書き方 | 待つもの | 待ち終わったとき |
|---|---|---|
| `await agent.stream(query)` | ストリームの用意（LLM への接続開始） | 応答の受け口 `stream` が手に入る。**応答の全文はまだ届いていない** |
| `for await (const chunk of stream.textStream)` | 断片（chunk）が1つ届くこと | 届くたびにループが1回まわる。全部届くとループが終わる |

`generate()` の `await agent.generate(query)` は「全文が揃うまで」を1回で待つので、その間は無音です。
`stream()` は待ちを細かく区切って、届いた断片から順に表示できるのが違いです。

覚えておく決まりごとは2つです。

- **`await` は `async` 関数の中でしか使えない**: `main()` に `async` が付いているのはこのためです。
- **`await` を付け忘れると Promise のまま進む**: `stream.textStream` を取り出せずエラーになります。
  Promise を返す関数（`generate()` / `stream()` など）には基本的に `await` を付けると覚えてください。

`async` / Promise の基礎は [座学 0-3（async / await）](../../00-overview/00-typescript-primer.md) にもまとめています。

</details>

## 3-3. 構造化出力 — 後続処理に渡す（20分）

エージェントの出力を人間ではなくコードに渡す場合、テキストではなく型付きオブジェクトで受け取ります。

```ts
// src/index.ts
import type { FullOutput } from '@mastra/core/stream'
import { z } from 'zod'
import { mastra } from './mastra/index.ts'

const packingListSchema = z.object({
  destination: z.string(),
  items: z.array(
    z.object({
      name: z.string(),
      reason: z.string(),
      priority: z.enum(['must', 'nice-to-have']),
    }),
  ),
})

type PackingList = z.infer<typeof packingListSchema>

async function main(): Promise<void> {
  const agent = mastra.getAgentById('travel-agent')
  const query: string = '来週、沖縄に3泊4日で行きます。持ち物リストを作って。'

  const response: FullOutput<PackingList> = await agent.generate(query, {
    structuredOutput: { schema: packingListSchema },
  })

  console.log(JSON.stringify(response.object, null, 2))
}

main().catch((error: unknown): void => {
  console.error(error)
})
```

<details>
<summary>💡 zod とは？（クリックで開く）</summary>

**zod** は「データの形（スキーマ）」を宣言するための TypeScript ライブラリです。`z.string()`（文字列）、
`z.array()`（配列）、`z.enum([...])`（候補から1つ）のように部品を組み合わせて、欲しいデータの形を書きます。

ここでは、その形を LLM に「この形で答えて」と伝えるために使っています。返ってきた値も zod で検証されるので、
`response.object` は形の保証された型付きオブジェクトとして安全に後続処理へ渡せます。
基礎は [座学 0-2（型注釈と Zod）](../../00-overview/00-typescript-primer.md) にもまとめています。

</details>

<details>
<summary>💡 <code>JSON.stringify(response.object, null, 2)</code> は何をしている？（クリックで開く）</summary>

`response.object` は JavaScript のオブジェクトです。`JSON.stringify()` は、それを **JSON 形式の文字列**に
変換する標準の関数で、ここでは画面に見やすく表示するために使っています。3つの引数の意味は次のとおりです。

| 引数 | 値 | 意味 |
|---|---|---|
| 1つ目 | `response.object` | 文字列に変換する対象 |
| 2つ目 | `null` | 値を加工する関数（replacer）。使わないので `null` |
| 3つ目 | `2` | 字下げのスペース数。指定すると改行と字下げが入る |

3つ目を省くと、全体が1行で出力されて読みにくくなります。

```text
{"destination":"沖縄","items":[{"name":"日焼け止め","reason":"紫外線","priority":"must"}]}
```

`2` を指定すると、入れ子の構造がそのまま読み取れます。

```json
{
  "destination": "沖縄",
  "items": [
    {
      "name": "日焼け止め",
      "reason": "紫外線",
      "priority": "must"
    }
  ]
}
```

表示のための整形なので、後続の処理には文字列にせず `response.object.items` のようにそのまま使えます。

</details>

> **注意**: このハンズオンの OpenAI では、`z.object` / `z.array` / `z.string` などの基本形のスキーマがそのまま通ります。なお Google のモデルは JSON Schema のサブセットしかサポートしておらず、`z.union` と `z.record` が使えません。Gemini など他プロバイダへ差し替える際に踏むので覚えておいてください（[01-setup/02-llm.md](../../01-setup/02-llm.md) 6章参照）。

**チェックポイント**
- [ ] 3つの呼び出し方の違いと使い分けを説明できる
- [ ] 構造化出力で型付きオブジェクトが取得できた
- [ ] `response.usage` でトークン消費を確認した
