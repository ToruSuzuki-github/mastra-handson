# Step 7. メモリ：会話を記憶させる（90分）

Day1 のエージェントは、毎回すべてを忘れます。会話履歴を持たせます。

## 7-1. ストレージを設定する（20分）

Mastra のメモリはストレージ層の上に成り立ちます。ローカル開発では libSQL（SQLite 互換）が最も手軽です。

```bash
npm install @mastra/memory @mastra/libsql
```

```ts
// src/mastra/index.ts
import { Mastra } from '@mastra/core'
import { Observability, MastraStorageExporter } from '@mastra/observability'
import { LibSQLStore } from '@mastra/libsql'
import { travelAgent } from './agents/travel-agent.ts'

export const mastra = new Mastra({
  agents: { travelAgent },
  // ↓ ここが今回の追加。observability は Day1 Step 2 で入れたものをそのまま残す
  storage: new LibSQLStore({
    id: 'mastra-storage',
    url: 'file:./mastra.db',   // ':memory:' にすると再起動で消える
  }),
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

> `LibSQLStore` には `id` を必ず指定してください。`url` は `:memory:`（インメモリ）、`file:xxx.db`（ファイル）、`libsql://...`（リモート）が使えます。ハンズオンでは永続化を体感したいのでファイルにします。

## 7-2. エージェントにメモリを付ける（20分）

```ts
// src/mastra/agents/travel-agent.ts
import { Agent } from '@mastra/core/agent'
import { Memory } from '@mastra/memory'
import { weatherTool } from '../tools/weather-tool.ts'

export const travelAgent = new Agent({
  id: 'travel-agent',
  name: 'Travel Agent',
  instructions: `
あなたは旅行の準備を手伝うアシスタントです。
行き先が伝えられたら、必ず get-weather ツールを使って実際の天気を確認してから提案してください。
過去の会話でユーザーが伝えた好みや制約（荷物を減らしたい、寒がり等）を覚えて反映してください。
日本語で回答します。
`,
  model: 'openai/gpt-5.6-luna',
  tools: { [weatherTool.id]: weatherTool },
  memory: new Memory({
    options: {
      lastMessages: 20,
    },
  }),
})
```

## 7-3. Studio で確認する（20分）

```bash
npm run dev
```

Studio で新しいチャットを開始し、複数ターンの会話をします。

```
1ターン目: 「私は極度の寒がりです。あと荷物は最小限にしたい。」
2ターン目: 「来週、札幌に行きます。」
```

2ターン目で「寒がり」「荷物最小限」が反映されていれば成功です。
Studio の**右サイドバーにメモリ関連の情報が表示される**ので、何が記憶されているか確認してください。

> **ここから Traces が「残る」ようになります。** Day1 でも `observability` は入れてあったので
> トレース自体は出ていましたが、保存先がインメモリだったため再起動で消えていました。`storage` を
> `file:` にしたことで、会話履歴（`mastra_threads` / `mastra_messages`）だけでなく**実行トレース
> （`mastra_ai_spans`）も同じ DB ファイルに永続化される**ようになります。
> dev サーバーを Ctrl+C して再起動しても前回のトレースが Traces 画面に残っていることを
> 確認してください。仕組みと消し方は
> [reference/mastra-studio.md](../../reference/mastra-studio.md) §7。

## 7-4. コードから thread / resource を指定する（20分）

会話は `resource`（誰の会話か）と `thread`（どの会話か）で管理されます。

```ts
// src/index.ts
import type { AgentMemoryOption } from '@mastra/core/agent'
import { mastra } from './mastra/index.ts'

async function main(): Promise<void> {
  const agent = mastra.getAgentById('travel-agent')

  const memory: AgentMemoryOption = { resource: 'user-taro', thread: 'trip-sapporo' }

  const r1 = await agent.generate('私は極度の寒がりです。', { memory })
  console.log('A1:', r1.text, '\n')

  const r2 = await agent.generate('来週、札幌に行きます。', { memory })
  console.log('A2:', r2.text)
}

main().catch((error: unknown): void => {
  console.error(error)
})
```

実行は Step 3 と同じく `--env-file` を付けます。

```bash
node --env-file=.env src/index.ts
```

`thread` を変えると記憶が引き継がれないことを確認してください。

> 💡 このスクリプトは 7-3 で Studio から話した内容を**覚えていません**。スレッドが違うだけでなく、
> 保存先の DB ファイル自体が別になるためです（理由は 7-5）。

## 7-5. 保存内容を覗く（10分）

VS Code の SQLite Viewer 拡張（環境構築時に入れたもの）で `mastra.db` を開き、メッセージがどう保存されているか見てみます。

まず**ファイルの場所を探してください**。`url: 'file:./mastra.db'` の相対パスは実行時のカレントから
解決されるため、**起動の仕方によって場所が変わります**。

```bash
find . -name '*.db' -not -path './node_modules/*'
```

| 起動方法 | できる場所 |
|---|---|
| `npm run dev`（Studio 経由） | `src/mastra/public/mastra.db`（[reference/mastra-studio.md](../../reference/mastra-studio.md) §7-2） |
| `node --env-file=.env src/index.ts`（7-4） | プロジェクト直下の `mastra.db` |

> ⚠️ **この2つは別のファイルです。** 同じ設定のまま両方の起動方法を使うと DB が2つでき、
> **Studio で見た会話履歴と 7-4 のスクリプトの履歴が繋がりません**。
> 「Studio で話したのにスクリプトが覚えていない」と感じたら、まず `find` で DB の数を確認してください。
>
> **なぜ2つできるのか**: `storage` の設定は `src/mastra/index.ts` の1か所だけですが、`file:./mastra.db` は
> **相対パス**なので、ファイルの場所ではなく**実行中のプロセスのカレントディレクトリ（cwd）**を基準に解決されます。
> `mastra dev` は Studio のサーバーを **cwd = `src/mastra/public/`** で起動し（自分で作った覚えのない
> `public/` フォルダは CLI が作ったもの）、7-4 のスクリプトは**コマンドを打ったプロジェクト直下**が cwd です。
> 基準が違うので、同じ設定から別々のファイルができます。設定ミスではありません。
> 詳しい仕組みと、1つにまとめる方法は [reference/mastra-studio.md](../../reference/mastra-studio.md) §7-2。

**開いたら何を見るか**

7-4 のスクリプトが作った**プロジェクト直下の `mastra.db`** を SQLite Viewer で開きます
（左のテーブル一覧から選ぶと行が表に出ます）。次の 4 つを順に確認してください。

1. **`mastra_threads`** — 1 行 = 1 スレッドです。`id` が 7-4 で渡した `thread`（`trip-sapporo`）、
   `resourceId` が `resource`（`user-taro`）と**書いた値そのまま**入っていることを確認します。
   `title` が空なのは、タイトルの自動生成（発展の `generateTitle`）を設定していないためです。
2. **`mastra_messages`** — 1 行 = 1 メッセージです。`thread_id` 列でスレッドごとに行が分かれ、
   `role` が `user` / `assistant` の順に並びます。7-4 で `thread` を変えて試したなら、
   **`thread_id` の値が違う行どうしは混ざっていない**はずです。これが「thread を変えると記憶が
   分離される」の実体です。
3. **`content` 列のセルをクリックして全文を読む** — JSON 文字列です。`parts` という配列の中に、
   発言の `text` だけでなく、天気を聞いた回のアシスタントには **`tool-invocation`** が入っています。
   開くと `toolName`（`get-weather`）・`args`（渡した都市名）・`result`（Day1 で作ったツールの戻り値）が
   そのまま見えます。**ツールの引数と戻り値まで会話履歴に保存されている**ことを確認してください。
4. **`content` の `metadata.traceId` をコピーして、`mastra_ai_spans` の `traceId` 列と照合する** —
   同じ値の行が複数並び、`agent run` → `memory: recall`（履歴を読む）→ `llm` → `memory: save`
   （履歴を書く）という実行の流れが読めます。**メッセージから、その返答を生んだ実行へ辿れる**ことが分かります。

> 💡 **`lastMessages: 20` は LLM へ渡す件数であって、保存件数ではありません。** 20 ターンを超えて
> 話しても `mastra_messages` には全件が残ります。ここが 7-2 の設定と Storage の役割の違いです。
> Memory は「どこまで LLM に見せるか」を決め、Storage は「全部を保存する」だけです。

> 💡 **Studio 側の DB（`src/mastra/public/mastra.db`）も同じ手順で開いて見比べてください。**
> `mastra_threads` の `id` が UUID、`resourceId` が `travel-agent` になっているはずです。コードでは
> `thread` / `resource` を**自分で指定できる**一方、Studio は自動で採番していることが分かります。
> また Studio 経由のユーザー発言は `role` が `user` ではなく `signal` で保存されます
> （[reference/mastra-studio.md](../../reference/mastra-studio.md) §7-3）。Studio の Traces 画面は
> こちらの DB を読むので、`/traces?traceId=<値>` で開けるのも Studio 側の `traceId` だけです。

`mastra_resources` や `mastra_agents` など、行数が 0 のテーブルが多数あります。使っていない機能の
テーブルなので**空で正常**です（`mastra_resources` は発展の working memory を試すと行が入ります）。
テーブルごとの役割と `content` の JSON 構造は
[reference/mastra-studio.md](../../reference/mastra-studio.md) §7-3。

**発展（時間があれば）**

Memory には他にも機能があります。公式ドキュメントを見て、興味があるものを1つ試してください。

- **semantic recall**: キーワード一致ではなく意味的な近さで過去の発言を呼び戻す（ベクトルストアが必要）
- **working memory**: ユーザーの属性など、常に保持しておきたい情報を明示的に管理する
- **generateTitle**: スレッドのタイトルを自動生成する

参照: https://mastra.ai/docs/memory/overview / https://mastra.ai/reference/memory/memory-class

**チェックポイント**
- [ ] 複数ターンで文脈が引き継がれた
- [ ] `thread` を変えると記憶が分離されることを確認した
- [ ] `mastra_messages` の `content` に、ツール呼び出しの引数と戻り値まで保存されていることを確認した
- [ ] Storage と Memory の役割の違いを説明できる
