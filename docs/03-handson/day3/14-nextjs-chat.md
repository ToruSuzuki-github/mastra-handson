# Step 14. Next.js に同居させてチャット UI を作る（95分）

[Step 13](./13-typed-client.md) は「Mastra サーバー（4111）＋ フロント（5173）」の**分離構成**でした。
ここでは**同居構成** —— Next.js アプリの中に Mastra を組み込み、1つのプロセス・1つのデプロイに
まとめる形を作ります。

**この Step のゴール**: ツール呼び出しの様子まで見えるチャット UI を作り、Mastra を「既存の Web アプリに
組み込む部品」として扱えるようになる。

**前提**: [Step 13](./13-typed-client.md) 完了。`travel-agent/` に `weatherTool` と Memory が入っていること。

---

## 14-1. 2つの繋ぎ方を整理する（10分）

| | 分離構成（Step 13） | 同居構成（本 Step） |
|---|---|---|
| プロセス | Mastra サーバー + フロント（2つ） | Next.js だけ（1つ） |
| 呼び出し | HTTP（`@mastra/client-js`） | **関数呼び出し**（`import { mastra }`） |
| デプロイ | 2箇所 | 1箇所 |
| Studio | そのまま使える | 使えない（後述） |
| エージェントの共有 | 複数のクライアントから叩ける | そのアプリ専用になりがち |
| 向く場面 | 複数システムから使う共通基盤にする | 1つの Web アプリに AI 機能を足す |

同居構成の本質は、**エージェントの呼び出しがネットワーク越しではなく関数呼び出しになる**ことです。
API の形を決める必要がなく、`await mastra.getAgentById('travel-agent').generate(...)` をサーバー
コンポーネントや API ルートから直接書けます。Step 13 で作った「型が繋がる」性質は、ここでは
import そのものによって自明に成立します。

> 💡 どちらが正解ということはありません。1つのエージェントを複数システムから使うなら分離構成、
> 「この業務アプリにチャットを付けたい」なら同居構成が素直です。判断材料は
> [README.md](./README.md) の使い分け表にまとめてあります。

## 14-2. Next.js プロジェクトを作る（15分）

`travel-agent/` とは別のプロジェクトとして、リポジトリ直下に作ります。

```bash
cd ~/work/mastra-handson
npx create-next-app@latest travel-next --yes --ts --eslint --tailwind --src-dir --app --turbopack --no-react-compiler --no-import-alias
cd travel-next
```

<details>
<summary>💡 このコマンドは何をしている？（クリックで開く）</summary>

**ひとことで**: Next.js の雛形（ひな形）を作るツール `create-next-app` を、インストールせずに一度だけ実行し、
本来は対話で聞かれる設定項目を**フラグ（`--xxx`）で先に答えている**コマンドです。

### 1. コマンドの骨格

| 部分 | 意味 |
|---|---|
| `npx` | npm パッケージを**インストールせずに**実行するコマンド。手元に無ければ一時的にダウンロードして走らせる |
| `create-next-app@latest` | Next.js 公式の雛形生成ツール。`@latest` は「最新版を使う」という指定 |
| `travel-next` | 作るプロジェクトの**フォルダ名**（`package.json` の `name` にもなる） |

### 2. フラグ（質問への先回り回答）

| 分類 | フラグ | 意味 | 本 Step での関わり |
|---|---|---|---|
| 質問の省略 | `--yes` | 指定していない項目を、保存済みの設定または既定値で決めて**対話を出さない** | これでコマンド1行で完結する |
| 言語・品質 | `--ts` | TypeScript で作る（`.tsx` / `tsconfig.json`） | 以降のコードはすべて TypeScript |
| | `--eslint` | ESLint（コードの書き方を自動チェックするツール）の設定を入れる | `npm run lint` が使える |
| 見た目 | `--tailwind` | Tailwind CSS を入れる。`className="p-4 text-xs"` のように**小さなクラスを組み合わせて**装飾する CSS フレームワーク | 14-5 のチャット UI の `className` が効く |
| 構造 | `--app` | **App Router** を使う。`app/` 配下のフォルダ構成がそのまま URL になる方式（旧方式は Pages Router） | 14-4 の `src/app/api/chat/route.ts` が `/api/chat` になる |
| | `--src-dir` | ソースを**ルート直下ではなく `src/` の中**に置く | 14-3 で `src/mastra` にコピーする前提 |
| ビルド | `--turbopack` | Turbopack（Rust 製の高速バンドラ。ソースを1つにまとめて配信する道具）を使う | 下の補足を参照 |
| | `--no-react-compiler` | React Compiler を**使わない**（`--no-` は「無効にする」） | 下の補足を参照 |
| | `--no-import-alias` | 既定の `@/*` を使う | 下の注記を参照 |

**補足**

- **`--turbopack`**: 現行の Next.js では既定が Turbopack なので、付けても付けなくても結果は同じです。
  最近の `create-next-app` のヘルプには載っていませんが、指定してもエラーにはなりません。
- **`--no-react-compiler`**: React Compiler は、コンポーネントの無駄な再描画を**ビルド時に自動で最適化**する
  仕組みです。便利ですがビルドが遅くなり、このハンズオンの本筋（Mastra の組み込み）とも関係しないので切っています。

### 3. 指定していなくても起きること

| 起きること | 理由・補足 |
|---|---|
| 依存パッケージが `npm` でインストールされ、`node_modules/` ができる | `npx` で起動したので npm が選ばれる |
| `git init` はされない | すでに Git リポジトリの中なので、入れ子のリポジトリは作られない |
| `AGENTS.md` が生成される | `create-next-app` の既定。AI コーディングエージェントに Next.js の最新作法を伝えるためのファイルで、無くても動く |

### 4. `cd travel-next`

作成されたフォルダへ移動するだけです。以降の `npm install` や `npm run dev` は、**必ずこのフォルダの中**で
実行します（`travel-agent/` と取り違えないよう注意）。

</details>

> `--no-import-alias` は「エイリアスを設定しない」ではなく「**既定の `@/*` を使う**（対話で聞かない）」
> という意味です。以降のコードで出てくる `@/mastra` はこれに依存しています。

依存を入れます。

```bash
npm install @mastra/core@latest @mastra/memory@latest @mastra/libsql@latest \
            @mastra/observability@latest @mastra/duckdb@latest @mastra/evals@latest \
            @mastra/ai-sdk@latest @mastra/next@latest \
            ai @ai-sdk/react hono zod@^4
```

| パッケージ | 役割 |
|---|---|
| `@mastra/core` / `memory` / `libsql` / `observability` | `travel-agent/` と同じ。エージェント本体が使う |
| `@mastra/duckdb` / `@mastra/evals` | Day2 の [Step 10](../day2/10-observability.md)（Metrics）と [Step 11](../day2/11-evals.md)（評価）で足したもの。14-3 でコピーする `src/mastra` が import しているため、**入れ忘れると `Module not found: Can't resolve '@mastra/duckdb'` で動かない** |
| `@mastra/ai-sdk` | Mastra の応答を **AI SDK のストリーム形式**に変換する橋渡し |
| `@mastra/next` | Mastra の API 一式を Next.js のルートとして生やす（14-6） |
| `hono` | `@mastra/next` が内部で使うルーター。**入れ忘れると 14-6 で動かない** |
| `ai` / `@ai-sdk/react` | Vercel AI SDK。`useChat()` でチャット UI の状態管理を担う |

## 14-3. エージェントを持ち込む（15分）

Day1〜Day2 で育てたエージェントをそのままコピーします。**書き直しは不要**です。

```bash
cp -r ../travel-agent/src/mastra ./src/mastra
```

`tsconfig.json` に1行足します。

```json
{
  "compilerOptions": {
    "allowImportingTsExtensions": true
  }
}
```

> **落とし穴**: `travel-agent/` のコードは `import { weatherTool } from '../tools/weather-tool.ts'` のように
> **拡張子付き**で import しています（Mastra CLI の流儀）。create-next-app が作る `tsconfig.json` は
> これを許可していないため、そのままでは `error TS5097: An import path can only end with a '.ts' extension
> when 'allowImportingTsExtensions' is enabled` で `next build` が落ちます。
> `allowImportingTsExtensions` は `noEmit: true` が前提ですが、Next.js の `tsconfig.json` は元から
> `noEmit: true` なのでこの1行で済みます。
> （拡張子を全部外す方針でも構いませんが、`travel-agent/` 側と差分が出るのでコピーが面倒になります。）

`next.config.ts` に2行**足します**。create-next-app が生成した設定は消さずに残し、`nextConfig` の
中へ追記してください:

```ts
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  // ↓ 生成済みの設定（cacheComponents / turbopack など）はそのまま残す

  // ★ 追加: Mastra 関連パッケージはバンドルせず Node のまま実行させる
  serverExternalPackages: ['@mastra/*'],
  // ★ 追加: ビルドを1プロセスで行う（DuckDB のファイルを複数プロセスで取り合わないため）
  experimental: { cpus: 1 },
}

export default nextConfig
```

> ⚠️ **ファイル全体を上の内容で置き換えないでください。** 生成された `turbopack.rules` が Tailwind CSS を
> 処理しています。消すと `Parsing CSS source code failed`（`globals.css` の `@theme`）という警告が出て、
> Tailwind のクラスが一切生成されなくなります。ビルド自体は成功するため、画面の装飾が効かなくなって
> 初めて気づくことになります。

> 💡 **`experimental: { cpus: 1 }` を入れる理由**: `next build` は複数のプロセスで並列に処理し、それぞれが
> `src/mastra` を読み込みます。Day2 の [Step 10](../day2/10-observability.md) で入れた DuckDB は
> **1つのファイルを同時に1プロセスしか開けない**ため、後から来たプロセスが
> `Could not set lock on file "mastra.duckdb"` というエラーをログに出します（ビルドは成功し、
> `npm run dev` には影響しません）。プロセスを1つに絞れば取り合いが起きません。起動時に
> `Experiments (use with caution): cpus: 1` と表示されるのは、この設定が効いている印です。

`.env.local` を作成（Next.js はこのファイル名を読みます）:

```bash
OPENAI_API_KEY=<あなたのキー>
```

> ⚠️ `.env` ではなく **`.env.local`** です。また、`NEXT_PUBLIC_` を付けた環境変数は**ブラウザに埋め込まれます**。
> API キーには絶対に付けないでください。

## 14-4. API ルートを作る（20分）

チャットの実体です。`src/app/api/chat/route.ts` を作成:

```ts
// src/app/api/chat/route.ts
import { handleChatStream } from '@mastra/ai-sdk'
import { toAISdkMessages } from '@mastra/ai-sdk/ui'
import type { ChatStreamHandlerParams } from '@mastra/ai-sdk'
import { createUIMessageStreamResponse } from 'ai'
import { NextResponse } from 'next/server'
import { mastra } from '@/mastra'

// Day2 で学んだ resource / thread。本来はログインユーザーごとに振る
const RESOURCE_ID: string = 'user-taro'
const THREAD_ID: string = 'travel-next-demo'

export async function POST(req: Request): Promise<Response> {
  const params: ChatStreamHandlerParams = await req.json()

  const stream = await handleChatStream({
    mastra,
    agentId: 'travel-agent',
    version: 'v7', // AI SDK のメッセージ形式のバージョン
    params: {
      ...params,
      memory: { ...params.memory, thread: THREAD_ID, resource: RESOURCE_ID },
    },
  })

  return createUIMessageStreamResponse({ stream })
}

// 画面を開き直したときに過去の会話を復元するための履歴取得
export async function GET(): Promise<NextResponse> {
  const memory = await mastra.getAgentById('travel-agent').getMemory()

  let recalled = null
  try {
    recalled = await memory?.recall({ threadId: THREAD_ID, resourceId: RESOURCE_ID })
  } catch (error: unknown) {
    console.log('過去のメッセージはまだありません')
  }

  return NextResponse.json(toAISdkMessages(recalled?.messages ?? [], { version: 'v7' }))
}
```

読みどころは3つです。

1. **`import { mastra } from '@/mastra'`** — HTTP ではなく import。これが同居構成の全てです。
2. **`handleChatStream`** — Mastra のストリームを AI SDK が期待する形式に変換します。この1行のおかげで、
   フロント側は Mastra を知らない普通の AI SDK アプリとして書けます。
3. **`memory.recall()`** — Day2 で `LibSQLStore` に貯めた会話を読み出します。Studio で見ていたあの履歴を、
   自前の UI から取り出しているわけです。

<details>
<summary>💡 API ルートって何？コードを上から読む（クリックで開く）</summary>

### 1. API ルートとは

ブラウザ（14-5 のチャット画面）から届いた**リクエストを受け取り、レスポンスを返す関数**です。
Next.js では `route.ts` というファイルを置くだけで、その場所が URL になります。

| ルール | この Step での例 |
|---|---|
| ファイル名は `route.ts` | `src/app/api/chat/route.ts` |
| `src/app` 以下のフォルダ構成が URL になる | `app/api/chat/` → **`/api/chat`** |
| `export` する関数名が **HTTP メソッド**（`GET` / `POST` など）。同じ URL でもメソッドごとに別の関数が動く | `POST` と `GET` の2つ |

HTTP メソッドは「何をしたいか」を表す種類です（GET = 取得、POST = 送信。Day1 の
[Step 5](../day1/05-http-api.md) の `curl` と同じもの）。このファイルの2つの関数は、14-5 の画面から
次のように呼ばれます。

| 関数 | 呼ぶのは（14-5） | 役割 |
|---|---|---|
| `POST` | `useChat` の `sendMessage` | ユーザーの発言を受け取り、AI の返答を**ストリームで**返す |
| `GET` | 画面を開いたときの `fetch('/api/chat')` | 保存済みの過去の会話を**JSON で**返す |

### 2. `POST` — 返答を返す

| コード | 何をしているか |
|---|---|
| `req: Request` / `Promise<Response>` | `Request` / `Response` はブラウザの `fetch` と同じ **Web 標準の型**。「リクエストを受け取りレスポンスを返す」のが API ルートの全て。`async` なので返り値は Promise になる |
| `await req.json()` | リクエストの本文（JSON）を読む。主な中身は `messages`（**これまでの全発言**）。`await` は読み終わるまで待つ |
| `: ChatStreamHandlerParams` | 本文の型を**宣言しているだけ**で、実行時に中身を検査しているわけではない |
| `handleChatStream({ ... })` | エージェントを動かし、返答を AI SDK が読める形式のストリームにする（引数は下表） |
| `createUIMessageStreamResponse({ stream })` | ストリームを **HTTP レスポンスに包んで返す**。全文の完成を待たずに少しずつ送るので、画面に1文字ずつ流れる |

`handleChatStream` の引数:

| 引数 | 意味 |
|---|---|
| `mastra` | 14-3 で持ち込んだ Mastra インスタンス |
| `agentId` | 動かすエージェントの ID |
| `version` | AI SDK のメッセージ形式のバージョン。`GET` 側の `toAISdkMessages` と揃える |
| `params` | ブラウザから来た本文。`memory` だけ、下のとおりサーバーで上書きする |

**`memory` をサーバーで上書きする理由**: `{ ...params.memory, thread: THREAD_ID, resource: RESOURCE_ID }` は、
「ブラウザから来た `memory` を土台に、`thread` と `resource` だけ**後ろから上書き**」という意味です
（`...` は後ろに書いたものが勝つ）。ブラウザが何を送ってきても会話の保存先はサーバーが決めるので、
他人の会話に書き込まれる事故を防げます。

### 3. `GET` — 過去の会話を返す

| コード | 何をしているか |
|---|---|
| `getAgentById('travel-agent').getMemory()` | エージェントに付けた Memory を取り出す |
| `memory?.recall({ threadId, resourceId })` | 保存済みの会話を読み出す。`?.` は「`memory` が無ければ `undefined` のまま進む」 |
| `try { ... } catch { ... }` | **会話がまだ1つも無い初回は `recall` が失敗する**。画面が壊れないよう、ログだけ出して「履歴なし」として続ける |
| `recalled?.messages ?? []` | 履歴があればその `messages`、無ければ空配列 `[]`。`??` は「左が `null` / `undefined` のときだけ右を使う」 |
| `toAISdkMessages(..., { version: 'v7' })` | Mastra の保存形式を、`useChat` が読める AI SDK のメッセージ形式に変換する |
| `NextResponse.json(...)` | 値を JSON にして返す。`NextResponse` は Next.js が `Response` を拡張したもの |

### 4. 動作確認とつまずきやすい点

`GET` は LLM を呼ばないので、`curl` で単独で確かめられます（`npm run dev` の起動後）。

```bash
curl http://localhost:3000/api/chat    # 会話がまだ無ければ []
```

`POST` は LLM を呼ぶため、14-5 の画面から試すのが簡単です。

- **関数名は大文字の `GET` / `POST`**。定義していないメソッドで叩くと `405 Method Not Allowed` が返ります。
- 初回の `GET` で `過去のメッセージはまだありません` とサーバーのログに出るのは**正常**です（上の `catch`）。

</details>

<details>
<summary>💡 <code>Response</code> と <code>NextResponse</code> は import が要る？（クリックで開く）</summary>

**結論**: `Response` は import 不要、`NextResponse` は import が必要です。

| | 正体 | import | このコードでの出どころ |
|---|---|---|---|
| `Request` / `Response` | **Web 標準**（ブラウザの `fetch` と同じもの） | **不要** | 先頭の import 一覧に無い ＝ 最初から使える |
| `NextResponse` | **Next.js が提供する**クラス | **必要** | 先頭の `import { NextResponse } from 'next/server'` |

### `Response` が import 無しで使える理由

- **実行時**: `Response` / `Request` は Node.js に最初から入っている**グローバル**です（`Math` や `Date` と同じ扱い）。
- **型として**: `tsconfig.json` の `"lib": ["dom", ...]` が、ブラウザ標準 API の型定義を読み込んでいるため、
  `Promise<Response>` のように型注釈にも書けます。

コードを読むときは「**import の一覧に無い名前は、最初から使えるもの**」と見分けられます。

### `NextResponse` は何が違うのか

`NextResponse` は標準の `Response` を**継承して機能を足したもの**です（`class NextResponse extends Response`）。
`Response` として扱えるので、標準の機能はそのまま使えます。足されているのは Next.js 固有の機能です。

| 追加機能 | 用途 |
|---|---|
| `cookies` | Cookie の読み書き |
| `NextResponse.redirect()` / `rewrite()` | 別の URL への転送・書き換え |

このコードで使っているのは `NextResponse.json(...)`（値を JSON にして返す）だけで、標準の
`Response.json(...)` に置き換えても同じ結果になります。Cookie やリダイレクトが要らない間は、
どちらを使っても構いません。

### 戻り値の型の書き分け

| 関数 | 戻り値の型 | 理由 |
|---|---|---|
| `POST` | `Promise<Response>` | `createUIMessageStreamResponse` が標準の `Response` を返す |
| `GET` | `Promise<NextResponse>` | `NextResponse.json(...)` を返している |

`NextResponse` は `Response` の一種なので、`GET` も `Promise<Response>` と書いて動きます。

</details>

## 14-5. チャット UI を作る（25分）

`src/app/chat/page.tsx` を作成:

```tsx
// src/app/chat/page.tsx
'use client'

import { useEffect, useState } from 'react'
import { useChat } from '@ai-sdk/react'
import { DefaultChatTransport } from 'ai'
import type { UIMessage, UIMessagePart, UIDataTypes, UITools } from 'ai'

export default function ChatPage(): React.JSX.Element {
  const [input, setInput] = useState<string>('')

  const { messages, setMessages, sendMessage, status } = useChat({
    id: 'travel-next-demo', // チャットの識別名。省くと描画中に乱数で作られ、Next.js がエラーにする
    transport: new DefaultChatTransport({ api: '/api/chat' }),
  })

  // 初回マウント時に GET /api/chat で過去の会話を復元する
  useEffect((): void => {
    fetch('/api/chat')
      .then((res: Response): Promise<UIMessage[]> => res.json())
      .then((history: UIMessage[]): void => setMessages(history))
      .catch((): void => {})
  }, [setMessages])

  return (
    <main className="mx-auto flex h-screen max-w-2xl flex-col gap-4 p-6">
      <div className="flex-1 space-y-4 overflow-y-auto">
        {messages.map((message: UIMessage) => (
          <div key={message.id} className="space-y-1">
            <div className="text-xs text-gray-500">{message.role}</div>

            {message.parts?.map((part: UIMessagePart<UIDataTypes, UITools>, i: number) => {
              if (part.type === 'text') {
                return (
                  <p key={i} className="whitespace-pre-wrap">
                    {part.text}
                  </p>
                )
              }

              // ★ツール呼び出しもメッセージの「部品」として流れてくる
              if (part.type.startsWith('tool-')) {
                return (
                  <pre key={i} className="overflow-x-auto rounded bg-gray-100 p-2 text-xs">
                    🔧 {part.type}
                    {'\n'}
                    {'input' in part ? JSON.stringify(part.input, null, 2) : ''}
                  </pre>
                )
              }

              return null
            })}
          </div>
        ))}
      </div>

      <form
        className="flex gap-2"
        onSubmit={(e: React.SubmitEvent<HTMLFormElement>): void => {
          e.preventDefault()
          if (!input.trim()) return
          sendMessage({ text: input })
          setInput('')
        }}
      >
        <input
          className="flex-1 rounded border p-2"
          value={input}
          onChange={(e: React.ChangeEvent<HTMLInputElement>): void => setInput(e.target.value)}
          placeholder="来週、札幌に2泊で行きます"
          disabled={status !== 'ready'}
        />
        <button className="rounded bg-black px-4 py-2 text-white" type="submit">
          送信
        </button>
      </form>
    </main>
  )
}
```

<details>
<summary>💡 <code>.tsx</code> って何？コードを上から読む（クリックで開く）</summary>

### 1. `.tsx` ファイルと置き場所

`.tsx` は **TypeScript + JSX** のファイルです。JSX は `<main>…</main>` のような **HTML に似たタグを
TypeScript の中に直接書ける記法**で、`{ }` の中には TypeScript の式を書けます。

| 拡張子 | 書けるもの | この Step での例 |
|---|---|---|
| `.ts` | 処理だけ | `route.ts`（API。14-4） |
| `.tsx` | 処理 ＋ **画面（JSX）** | `page.tsx`（画面。本節） |

Next.js はファイル名で役割を決めます。**フォルダ構成が URL になる**点は 14-4 と同じです。

| ファイル | URL | 役割 |
|---|---|---|
| `src/app/api/chat/route.ts` | `/api/chat` | API（14-4） |
| `src/app/chat/page.tsx` | `/chat` | 画面（本節） |

> ⚠️ **`page.tsx` は `api/` の外（`src/app/chat/`）に作ります。** `route.ts` と同じフォルダに置くと、
> 同じ URL に API と画面が重なるため `Conflicting route and page at /api/chat` で動きません。

### 2. コードを上から読む

**先頭と関数の形**

| コード | 何をしているか |
|---|---|
| `'use client'` | このファイルを**ブラウザ側で動かす**宣言。Next.js の部品は既定でサーバー側で動くため、`useState` や入力への反応（`onSubmit` など）を使うには必要 |
| `export default function ChatPage()` | 画面の部品（**コンポーネント**）を定義する関数。`return` する JSX が画面になる。Next.js は `page.tsx` の `export default` を画面として表示する |

**状態を持つ**

| コード | 何をしているか |
|---|---|
| `useState<string>('')` | 入力欄の文字を覚える。`[input, setInput]` は「現在の値」と「値を変える関数」。値が変わると画面が描き直される |
| `useChat({ id, transport })` | チャットの状態管理一式。`DefaultChatTransport({ api: '/api/chat' })` は「送信先は `/api/chat`」の指定で、14-4 の `POST` に届く |
| `id: 'travel-next-demo'` | このチャットの識別名。14-4 の `THREAD_ID` と同じ値にして対応を分かりやすくしている。**省略できない理由**は下の「つまずきやすい点」を参照 |

`useChat` の戻り値:

| 名前 | 中身 |
|---|---|
| `messages` | これまでの発言の配列 |
| `setMessages` | 配列を丸ごと差し替える（履歴の復元で使う） |
| `sendMessage` | 発言を送る。返答はストリームで `messages` に追記されていく |
| `status` | `'submitted'`（送信直後）/ `'streaming'`（受信中）/ `'ready'`（待機中）/ `'error'` |

**履歴を復元する（`useEffect`）**

| コード | 何をしているか |
|---|---|
| `useEffect(..., [setMessages])` | 画面を表示した後に処理を走らせる仕組み。第2引数の中身が変わらない限り再実行されない。`setMessages` は通常変わらないため、実質**最初の1回だけ**動く |
| `fetch('/api/chat')` | 14-4 の `GET` を呼ぶ |
| `.then(res => res.json())` → `.then(history => setMessages(history))` | 返ってきた JSON を取り出し、`messages` に入れる |
| `.catch(() => {})` | 失敗しても何もしない（履歴が出ないだけで画面は動く） |

**発言を表示する**

| コード | 何をしているか |
|---|---|
| `messages.map(...)` | 発言を1件ずつ画面の部品に変換する。`key` は各要素を見分ける目印 |
| `message.role` | 誰の発言か（`user` / `assistant`） |
| `message.parts` | 1つの発言は複数の**部品**でできている（文章、ツール呼び出しなど） |
| `part.type === 'text'` | 文章 → `<p>` で表示する |
| `part.type.startsWith('tool-')` | ツール呼び出し（例 `tool-get-weather`）→ `<pre>` に入力値を JSON で表示する |
| `'input' in part ? ... : ''` | `input` を持つ部品のときだけ値を出す |
| `return null` | どれにも当てはまらない部品は何も表示しない |

**入力フォーム**

| コード | 何をしているか |
|---|---|
| `onSubmit` ＋ `e.preventDefault()` | 送信時のページ全体の再読み込み（ブラウザの標準動作）を止める |
| `e: React.SubmitEvent<HTMLFormElement>` | フォーム送信イベントの型。古い記事に出てくる `React.FormEvent` は `@types/react` 19 で**非推奨**（取り消し線が付く）なので、`SubmitEvent` を使う |
| `if (!input.trim()) return` | 空白だけなら送らない |
| `sendMessage({ text: input })` → `setInput('')` | 送信して、入力欄を空に戻す |
| `value={input}` ＋ `onChange` | 入力欄と `input` 変数を連動させる |
| `disabled={status !== 'ready'}` | 返答の受信中は入力欄を使えなくする |

### 3. `className` の読み方（Tailwind CSS）

`className` の1語が1つの見た目指定です。暗記は不要で、分からなければ Tailwind の公式ドキュメントで
クラス名を検索できます。

| クラス | 意味 |
|---|---|
| `mx-auto max-w-2xl` | 幅に上限をつけて中央寄せ |
| `flex flex-col gap-4` | 縦に並べて、間に余白 |
| `h-screen` | 高さを画面いっぱいに |
| `flex-1 overflow-y-auto` | 余った高さを使い、はみ出したら縦スクロール |
| `space-y-4` / `p-6` | 子要素どうしの縦の間隔 / 内側の余白 |
| `rounded border` | 角丸と枠線 |
| `text-xs text-gray-500` | 小さく灰色の文字 |
| `whitespace-pre-wrap` | 改行をそのまま表示する（LLM の返答の改行を保つ） |

### 4. つまずきやすい点

- **開くのは `/chat`**。`http://localhost:3000/` は create-next-app が作った初期画面のままです。
- **`'use client'` を消さない**。無いと `useState` などが使えずエラーになります。
- **`useChat` の `id` を省かない**。省くと `useChat` が描画中に乱数（`Math.random()`）で ID を作ります。
  create-next-app が有効にしている `cacheComponents`（画面を事前に描画して使い回す仕組み）は、描画中に
  毎回変わる値を読むことを許さないため、`Next.js encountered the unstable value Math.random() in a
  Client Component` というエラーになり、`next build` も失敗します。固定の `id` を渡せば乱数は使われません。

</details>

起動:

```bash
npm run dev    # → http://localhost:3000
```

http://localhost:3000/chat を開いて「来週、札幌に2泊で行きます」と送ってください。

**確認すること**

- [ ] 応答が1文字ずつ流れてくる（`stream` が効いている）
- [ ] 途中で `🔧 tool-get-weather` のブロックが出て、`{ "city": "Sapporo" }` が見える
- [ ] 続けて「荷物は最小限にしたい」と送ると、次の質問で反映される（Memory が効いている）
- [ ] **ブラウザをリロードしても会話が残る**（`GET /api/chat` の履歴復元が効いている）

> **ここがフロントエンド編の本命です。** Studio のトレース画面は「開発者向けの画面」ですが、この
> `🔧 tool-get-weather` は**エンドユーザーが見る画面に、エージェントの思考過程が出ている**状態です。
> 「AI が勝手に答えた」ではなく「AI が天気 API を見に行ってから答えた」と示せるので、利用者の
> 信頼を得るうえで効きます。

**発展: AI Elements で見た目を整える（任意）**

Vercel が配布しているチャット UI コンポーネント集を使うと、上記の `<pre>` や `<p>` を整った
吹き出し・折りたたみ式のツール表示に置き換えられます。

```bash
npx ai-elements@latest
```

`src/components/ai-elements/` にコンポーネントが展開されるので、`Conversation` / `Message` /
`Tool` / `PromptInput` に差し替えます。**ハンズオンの本筋ではない**ので、時間が余ったときだけで
構いません。コードは公式の [Next.js 統合ガイド](https://mastra.ai/docs/frameworks/web-frameworks/next-js)
にそのまま載っています。

## 14-6. Mastra の API 一式を生やす（10分）

`handleChatStream` はチャット専用の入口でした。**エージェント・ワークフロー・ツールの API を
まるごと**Next.js に生やすこともできます。

**作る場所**: 14-4 で作った `route.ts` の**隣**に、`mastra` フォルダと `[...mastra]` フォルダを順に掘って、
その中に `route.ts` を新規作成します（`travel-next/` の中で作業します）。

```text
travel-next/
└── src/
    └── app/
        └── api/
            ├── chat/
            │   └── route.ts          ← 14-4 で作成済み
            └── mastra/               ← ★ 新規フォルダ
                └── [...mastra]/      ← ★ 新規フォルダ（角括弧とドット3つを含めて、この名前のまま）
                    └── route.ts      ← ★ 新規ファイル（下のコード）
```

ターミナルで作る場合（`travel-next/` で実行。角括弧はシェルに解釈されないようクォートで囲みます）:

```bash
mkdir -p "src/app/api/mastra/[...mastra]"
touch "src/app/api/mastra/[...mastra]/route.ts"
```

VS Code のエクスプローラーで作る場合は、`api` フォルダを右クリックして「新しいフォルダー」で `mastra`、
その中に `[...mastra]` を作り、さらにその中に `route.ts` を作成してください。

```ts
// src/app/api/mastra/[...mastra]/route.ts
import { createNextRouteHandler } from '@mastra/next'
import { mastra } from '@/mastra'

export const { GET, POST, PUT, DELETE, PATCH, OPTIONS, HEAD } = createNextRouteHandler({
  mastra,
  // ルートの置き場所と一致させる（既定は '/api'）
  prefix: '/api/mastra',
})
```

Day1 の [Step 5](../day1/05-http-api.md) と同じ要領で叩けます。

```bash
curl http://localhost:3000/api/mastra/agents
```

> ⚠️ **`prefix` は catch-all ルートのマウント先と必ず一致させてください。** 既定値は `/api` なので、
> `src/app/api/[...mastra]/route.ts`（`mastra` フォルダを挟まない置き方）なら省略できます。
> 上のように `src/app/api/mastra/[...mastra]/route.ts` へ置く場合は `prefix: '/api/mastra'` が要り、
> 省くとどのリクエストもルートに届きません。
> （公式の Next.js 統合ガイドは、この配置のまま curl の例だけ `/api/agents` になっているので注意）

<details>
<summary>💡 <code>[...mastra]</code> って何？コードを上から読む（クリックで開く）</summary>

### 1. 14-4 との違い

14-4 の `route.ts` は `/api/chat` **という1つの URL だけ**を受けるファイルでした。今回は
`/api/mastra/agents`、`/api/mastra/workflows`、… と**行き先がたくさんある**ので、1つずつファイルを
作る代わりに、**`/api/mastra/` 以下のすべてを1つの `route.ts` で受ける**方式にします。

| | 14-4 | 14-6 |
|---|---|---|
| フォルダ | `api/chat/` | `api/mastra/[...mastra]/` |
| 受ける URL | `/api/chat` だけ | `/api/mastra/` 以下すべて |
| 中身を書く人 | 自分（`POST` / `GET` を実装） | `@mastra/next` が用意（自分は呼ぶだけ） |

### 2. フォルダ名 `[...mastra]` の読み方

Next.js では、フォルダ名の付け方でその階層の意味が変わります。

| フォルダ名 | 意味 | 受ける URL の例 |
|---|---|---|
| `chat` | 文字どおりの URL の一部 | `/api/chat` |
| `[id]` | 何でも1階層ぶん受ける | `/api/users/123` |
| **`[...mastra]`** | **何階層でもまとめて受ける**（catch-all ルート） | `/api/mastra/agents`、`/api/mastra/agents/travel-agent/generate` など |

`[...mastra]` の `mastra` の部分は**受けた残りのパスを入れる変数名**で、名前は何でも動きます
（`[...slug]` など）。ただし `@mastra/next` の慣習に合わせて `mastra` にしています。

> ⚠️ 手で打つときに**角括弧 `[ ]` とドット3つ `...` を落とさない**でください。`mastra` というただの
> フォルダになり、`/api/mastra/agents` のような深い URL が届かなくなります（404 になります）。

`mastra` フォルダが2つ出てきますが、役割は別です。

```text
api/mastra/[...mastra]/route.ts
    ↑          ↑
    │          └ その先の /agents や /workflows を受ける変数
    └ URL の /api/mastra の部分
```

### 3. コードを上から読む

| コード | 何をしているか |
|---|---|
| `createNextRouteHandler({ mastra, prefix })` | `@mastra/next` の関数。14-3 で持ち込んだ `mastra` を渡すと、Mastra の API（エージェント一覧・実行など）を**処理するハンドラー一式**を返す |
| `prefix: '/api/mastra'` | 「このファイルが置かれている URL の根元」を教える指定。ここを基準に残りのパスを読み取る。**フォルダの位置と必ず一致**させる（上の注意書き） |
| `export const { GET, POST, PUT, DELETE, PATCH, OPTIONS, HEAD } = ...` | 返ってきたハンドラーを、HTTP メソッドの名前で `export` している。14-4 では `export async function POST` と書いていたものを、ここでは**まとめて受け取って出し直している**だけ |

14-4 で「`export` する関数名が HTTP メソッドになる」と説明しました。ここでは Mastra の API が
`GET`（取得）だけでなく `POST`（実行）や `DELETE`（削除）なども使うため、7種類をまとめて並べています。
**自分でメソッドの中身を書く必要はありません。**

### 4. 動作確認とつまずきやすい点

```bash
npm run dev                                # 別のターミナルで起動しておく
curl http://localhost:3000/api/mastra/agents
```

エージェントの一覧（JSON）が返れば成功です。`travel-agent` が含まれているはずです。

| 症状 | 原因と対処 |
|---|---|
| `404 Not Found` | フォルダ名が `[...mastra]` になっていない／`api/` の外に作った。上の図と見比べる |
| ルートには届くが中身が空・エラー | `prefix` が置き場所と不一致。`/api/mastra` になっているか確認 |
| `Module not found: Can't resolve 'hono'` | 14-2 で `hono` を入れ忘れている。`npm install hono` |
| 編集したのに反映されない | `npm run dev` を一度止めて再起動する（新しいフォルダを検出しないことがある） |

</details>

**Studio はどうなるのか**

同居構成では `mastra dev`（＝ Studio）は使いません。デバッグは Next.js のサーバーログと、上記の
API で行うことになります。トレース画面が見たい場合は、`travel-agent/` 側を別ターミナルで
`npm run dev` して使ってください（**同じ `mastra.db` を見ているわけではない**点に注意。
`travel-next/` は自分のディレクトリに `mastra.db` を作ります）。

> 💡 `travel-next/` の中で `npx mastra dev` を実行すると、Mastra CLI が規約どおり `src/mastra` を
> 見つけて Studio が立ち上がることがあります。公式が保証する使い方ではないので、動かなければ
> 深追いせず `travel-agent/` 側で確認してください。

---

## 14-7. 片付け（任意）

`travel-next/` は `node_modules/` と `.next/` で 1GB 近くになります。ハンズオン後に消す場合:

```bash
cd ~/work/mastra-handson
rm -rf travel-next
```

`travel-agent/` に足した `web/` と Vite は軽いので、残しておいて構いません。

---

**チェックポイント**
- [ ] Next.js の中から `import { mastra }` でエージェントを関数として呼べた
- [ ] チャット UI で応答がストリーム表示され、ツール呼び出しが画面に出た
- [ ] リロードしても会話履歴が復元されることを確認した
- [ ] `@mastra/next` で API 一式を生やし、`curl` で叩けた
- [ ] 分離構成（Step 13）と同居構成（Step 14）の使い分けを自分の言葉で説明できる

次は [README.md](./README.md) の「完了チェックリスト」と「使い分け表」で振り返って、
ハンズオンは完了です。
