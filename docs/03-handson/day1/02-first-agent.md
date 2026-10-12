# Step 2. 手動セットアップ：ゼロから Agent を作る（60分）

自動生成物はブラックボックスが多いので、最小構成を自分で組んで理解します。**ここからが本番のプロジェクト**です。

## 2-1. プロジェクト初期化（15分）

```bash
cd ~/work/mastra-handson                # 作業用フォルダへ移動する
mkdir travel-agent && cd travel-agent   # プロジェクト用フォルダを作り、その中へ入る
npm init -y                             # package.json を質問なし（既定値）で生成する

# 開発時だけ必要なパッケージ（devDependencies）
#   typescript … TypeScript コンパイラ（型チェック用）
#   @types/node … Node.js の型定義
#   mastra@latest … CLI（`mastra dev` / `mastra build` を提供）
npm install -D typescript @types/node mastra@latest

# 実行時に必要なパッケージ（dependencies）
#   @mastra/core@latest … Agent / Mastra 本体
#   @mastra/observability@latest … トレース記録（Step 2-3 で使用）
#   zod@^4 … スキーマ定義ライブラリ（4.x 系に固定。ツールの入出力定義で使う）
npm install @mastra/core@latest @mastra/observability@latest zod@^4
```

<details>
<summary>💡 <code>-D</code> を付けると何が変わる？（クリックで開く）</summary>

`-D` は `--save-dev` の短縮形で、`package.json` の **`devDependencies`** に記録されます（付けなければ `dependencies`）。

| | `dependencies` | `devDependencies` |
|---|---|---|
| コマンド | `npm install <pkg>` | `npm install -D <pkg>` |
| 用途 | アプリの実行時に必要 | 開発・ビルド・型チェックにだけ必要 |
| 本番向けインストール（`npm install --omit=dev` など） | 入る | **入らない** |

このハンズオンはローカルで `mastra dev` を動かすだけなので、どちらに入れても動作は変わりません。
違いが効くのは本番デプロイやパッケージ公開のときです。

</details>

`package.json` を編集:

```json
{
  "name": "travel-agent",
  "type": "module",
  "scripts": {
    "dev": "mastra dev",
    "build": "mastra build"
  }
}
```

> JSON にはコメントを書けないため、行ごとの説明は下表に分けています。コピーするときは上のブロックをそのまま使ってください。

| 行 | 内容 |
|---|---|
| `"name": "travel-agent"` | パッケージ（プロジェクト）名 |
| `"type": "module"` | `.js` / `.ts` を ESM（`import` / `export` 構文）として扱う設定 |
| `"scripts"` | `npm run <名前>` で実行できるコマンドの登録場所 |
| `"dev": "mastra dev"` | `npm run dev` で開発サーバーと Studio（ポート 4111）を起動する |
| `"build": "mastra build"` | `npm run build` で本番用にビルドする |

> `"type": "module"` を忘れないこと。

`tsconfig.json` を作成:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true,
    "noEmit": true,
    "allowImportingTsExtensions": true,
    "outDir": "dist"
  },
  "include": ["src/**/*"]
}
```

| 行 | 内容 |
|---|---|
| `"target": "ES2022"` | 想定する JavaScript のバージョン（ES2022 の構文をそのまま使う） |
| `"module": "ES2022"` | モジュール形式を ESM にする（`CommonJS` にしない） |
| `"moduleResolution": "bundler"` | import 先の解決規則をバンドラ方式にする（Mastra が内部でバンドルするため） |
| `"esModuleInterop": true` | CommonJS 製パッケージも `import x from '...'` で読めるようにする |
| `"forceConsistentCasingInFileNames": true` | ファイル名の大文字小文字が import と食い違ったらエラーにする（OS 間の差異対策） |
| `"strict": true` | 厳格な型チェックをまとめて有効にする |
| `"skipLibCheck": true` | `node_modules` 内の型定義ファイル（`.d.ts`）のチェックを省く（高速化・型定義同士の衝突回避） |
| `"noEmit": true` | `tsc` で JS ファイルを出力しない（型チェック専用。実際のビルドは `mastra` が行う） |
| `"allowImportingTsExtensions": true` | `import ... from './x.ts'` のように `.ts` 拡張子付きの import を許可する（`noEmit` とセットで必須） |
| `"outDir": "dist"` | 出力先フォルダ名（`noEmit` のため実質使われない） |
| `"include": ["src/**/*"]` | 型チェック対象を `src/` 配下の全ファイルに限定する |

> **重要**: `module` を `CommonJS`、`moduleResolution` を `node` にすると解決エラーになります。公式に明記されている落とし穴です。

`.env` を作成:

```bash
# OpenAI API のキー（model: 'openai/...' を呼ぶときに Mastra が自動で読む）
OPENAI_API_KEY=<あなたのキー>
# Mastra CLI の匿名テレメトリ（利用情報の送信）を無効にする
MASTRA_TELEMETRY_DISABLED=1
```

## 2-2. Agent を定義する（20分）

```ts
// src/mastra/agents/travel-agent.ts
import { Agent } from '@mastra/core/agent'

export const travelAgent = new Agent({
  id: 'travel-agent',
  name: 'Travel Agent',
  instructions: `
あなたは旅行の準備を手伝うアシスタントです。
ユーザーが行き先を伝えたら、その土地の気候を踏まえて持ち物を提案してください。

回答するときは:
- 箇条書きで簡潔に
- 「なぜそれが必要か」を一言添える
- 日本語で回答する
`,
  model: 'openai/gpt-5.6-luna',
})
```

## 2-3. Mastra インスタンスに登録する（10分）

```ts
// src/mastra/index.ts
import { Mastra } from '@mastra/core'
import { Observability, MastraStorageExporter } from '@mastra/observability'
import { travelAgent } from './agents/travel-agent.ts'

export const mastra = new Mastra({
  agents: { travelAgent },
  // これが無いとトレースは1件も記録されない（Studio の Traces 画面が空のままになる）
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

> ローカルファイルの import は**拡張子付き**（`.ts`）で書きます。

<details>
<summary>💡 <code>export const mastra</code> の変数名は変えてもいい？（クリックで開く）</summary>

**変えられません。`mastra` 固定です。** `mastra dev` / `mastra build` は `src/mastra/index.ts` から
`import { mastra } from '#mastra'` という形で**名前を指定して**読み込むためです。

```ts
export const mastra = new Mastra({ ... })   // ✅ 動く
export const myApp  = new Mastra({ ... })   // ❌ CLI が見つけられず起動に失敗する
```

`export` の付け忘れも同じく起動しません。`Mastra` クラス自体には名前の制約はなく、
制約は CLI の読み込み規約によるものです（CLI を使わず自分のスクリプトから使うだけなら自由）。

> 📎 **公式ドキュメントの記載について**: 公式の
> [Manual install](https://mastra.ai/docs/getting-started/manual-install)（手順 6）は
> `export const mastra = new Mastra({ ... })` と書いていますが、**「この名前でなければならない」と
> 明記した文章は、確認した範囲では見つかりませんでした**（Manual install / Reference: Configuration /
> Mastra class / CLI リファレンス / Deploy 系ページと、`@mastra/core` 同梱ドキュメント）。
> 上の「固定」は、`mastra` CLI の実装（`import { mastra } from '#mastra'`）に基づく説明です。
> 公式の例どおり `mastra` と書いておけば問題ありません。

</details>

> **トレースには「書き出すエクスポーター（`observability`）」と「保存先（`storage`）」の両方が要ります。**
> ここで `observability` を入れたのは、Step 4 でツール呼び出しをトレースで観察するためです。
> **これを渡さないと Traces 画面は最初から空のまま**で、`storage` をいくら足しても出てきません。
>
> 一方 `storage` はまだ設定していません。省くと Mastra はインメモリのストアにフォールバックするため、
> **Traces 画面では見えるが dev サーバーを止めると消えます**。Day2 の
> [day2/07-memory.md](../day2/07-memory.md) Step 7 で `LibSQLStore` を追加すると、会話履歴と一緒に
> トレースもファイルに残るようになります（仕組みは
> [reference/mastra-studio.md](../../reference/mastra-studio.md) §7）。

## 2-4. 動かす（15分）

```bash
npm run dev
```

http://localhost:4111 を開き、「来週バンコクに行きます」などと話しかけてみてください。

**理解を深めるための実験**
- `instructions` を書き換えて、口調や出力形式がどう変わるか試す
- `model` を `openai/gpt-5.6-terra`（中位ティア）に変えて、応答の質と速度を比べる

**チェックポイント**
- [ ] 自分で書いたエージェントが Studio で応答する
- [ ] `instructions` の変更が挙動に反映されることを確認した
- [ ] Agent / Mastra インスタンス / Studio の3者の関係を説明できる
