# トラブルシューティング & FAQ

ハンズオン中に詰まったらここを見てください。解決した内容を追記していくと、自分用のナレッジになります。

本ファイルは §1〜§7（症状別の逆引き）・§8 FAQ・§9 調べ先・§10 メモ欄を1枚にまとめてあります。
**Ctrl+F で症状を検索**するのが最速です。

> **「実施前の動作確認」は [pre-check.md](./pre-check.md) に分けてあります。**
> ハンズオンを**開催する人・最新版で手順が通るか確かめたい人**向けの内容（通し実行・外部 API の疎通確認・型チェック）です。

---

## 1. 環境構築まわり

| 症状 | 原因と対処 |
|---|---|
| WSL 内で `The command 'docker' could not be found in this WSL 2 distro.` | Docker Desktop の WSL 統合が無効。Settings → Resources → WSL Integration で対象ディストロをオンにし、**Apply & Restart** を押してから `wsl --shutdown`（[01-setup/01-environment.md §1.3](../01-setup/01-environment.md)） |
| 「Reopen in Container」が始まらない / すぐ失敗する | 同上。加えて Docker Desktop のクジラアイコンが緑（Engine running）か確認 |
| コンテナが起動しない / Docker が落ちる | BIOS の仮想化支援が無効。タスクマネージャー「パフォーマンス」タブで「仮想化: 有効」を確認 |
| Docker Desktop が Hyper-V を要求してくる | Windows 11 Home では WSL 2 バックエンドを使う。インストーラの「Use WSL 2 instead of Hyper-V」を選択 |
| `npm install` が異常に遅い | プロジェクトが Windows 側（`C:\...`）にある。WSL 側（`~/work/...`）へ移す |
| dev サーバーのファイル監視が効かない | 同上。ファイルシステム境界をまたぐと監視が不安定になる |
| ポート 4111 にブラウザから繋がらない | `forwardPorts` の設定漏れ、または dev サーバー未起動。VS Code の「ポート」パネルを確認。**起動直後はまだ Listen していない**ので、`│ Studio: http://localhost:4111` の行が出てから開く（下の「起動シーケンス」参照） |
| **Studio を開いたら別プロジェクトの画面が出る** | 前のプロジェクトの dev サーバーが動いたままで、今回のプロジェクトが **4112 に逃げている**。下の「2つ目を起動したとき」参照 |
| `Another development server instance is already running in this directory` | 同じプロジェクトで dev サーバーが二重起動。メッセージに出る PID を `kill <PID>` するか、既存のプロセスを止める |
| コンテナ内で API キーが空 / `.env` に書いたのに Mastra が反応しない（401・「キー未設定」） | まず **`devcontainer.json` の `remoteEnv` に同じキーが書かれていないか**確認する。書かれていると、ホスト側（WSL）が未設定のとき**空文字が注入され、`mastra dev` は `.env` の値で上書きしない**（空文字でも「設定済み」扱い）。`.env` を直しても治らない。対処は `remoteEnv` の該当行を削除（またはコメントアウト）→ 再アタッチ（`Reopen Folder Locally` → `Reopen in Container`）→ `mastra dev` を再起動。切り分けは、コンテナの統合ターミナルで `env \| grep -c '^OPENAI_API_KEY='` を実行する（`.env` の値はシェルには入らないので、`.env` だけで運用していれば `0`）。**`1` と出たら `remoteEnv`（または `containerEnv`）が空文字を注入している**。`remoteEnv` を使っていない場合は、`.env` の場所（`travel-agent/` 直下）が違う、またはコンテナ／dev サーバーの再起動が必要（[01-setup/01-environment.md §3](../01-setup/01-environment.md) 補足、[VS Code: Environment variables in Dev Containers](https://code.visualstudio.com/remote/advancedcontainers/environment-variables)） |
| ポート 5173 / 3000 にブラウザから繋がらない | フロントエンド編（Day3）で使うポート。`forwardPorts` に入っているか（[01-setup/01-environment.md §3](../01-setup/01-environment.md)）と、Vite 側に `server.host: true` があるかを確認（[03-handson/day3/13-typed-client.md](../03-handson/day3/13-typed-client.md) §13-3） |

### 起動シーケンス — いつポートが開くか

`npm run dev` はすぐにポートを掴みません。**バンドルが終わった最後**に Listen を開始します
（2026-08-15 実測、DevContainer 内。所要時間はマシン性能で変わります）。

| 経過 | 出来事 | 4111 |
|---|---|---|
| 0.0s | `npm run dev` 開始（テンプレートはまず `tsx src/seed.ts`） | — |
| 1.3s | `.mastra/dev.lock` 生成 | — |
| 5.1s | `◐ Preparing development environment...` | — |
| 7.9s | `✓ Initial bundle complete` | — |
| **9.5s** | `mastra ready` → **Listen 開始・HTTP 応答開始** | **確保** |

**`│ Studio: http://localhost:4111` の行が出るまでブラウザで開いても繋がりません。**
また、`src/` を編集するとホットリロードで**約1.3秒間 Listen が切れて**から復帰します。

### 2つ目のプロジェクトを起動したとき

`travel-agent/`（[03-handson/](../03-handson/)）とテンプレート（[02-templates/](../02-templates/)）は
**どちらも 4111 を使う**ため、行き来するときに混乱しがちです。挙動は「同じディレクトリか否か」で分かれます。

| 状況 | 何が起きるか |
|---|---|
| **同じプロジェクト**で2つ目を起動 | `.mastra/dev.lock` を検知して**即エラー停止**。PID を示して止まるので、そのまま `kill` できる |
| **別プロジェクト**で2つ目を起動（前のを止め忘れた） | エラーにならず、**自動的に 4112 で起動する**。1つ目は 4111 のまま生き続ける |

後者が厄介です。**エラーが出ないため気づきにくく**、4111 を開くと前のプロジェクトの Studio が
表示されて「変更が反映されない」と誤解しがちです。

```bash
# 起動時のログで実際のポートを必ず確認する
# │ Studio: http://localhost:4112  ← 4111 でないことがある

# いま 4111 を掴んでいるプロセスを調べる
ss -ltnp | grep 4111

# 使い終わった dev サーバーは Ctrl+C で確実に止める
```

> **推奨**: [02-templates/](../02-templates/) のテンプレートと [03-handson/](../03-handson/) のハンズオンは
> **同時に動かさない**。切り替えるときは前の dev サーバーを Ctrl+C で止めてください。
> `forwardPorts` に明示しているのは 4111 / 5173 / 3000 の3つだけなので
> （[.devcontainer/devcontainer.json](../../.devcontainer/devcontainer.json)）、**4112 は明示されておらず**
> VS Code の自動検知に依存します。

---

## 2. Mastra プロジェクトのセットアップ

| 症状 | 原因と対処 |
|---|---|
| `Cannot find module` / モジュール解決エラー | `tsconfig.json` の `module` が `CommonJS`、`moduleResolution` が `node` になっている。`ES2022` / `bundler` に修正 |
| ローカルファイルの import が失敗する | 拡張子を付ける（`'../tools/weather-tool.ts'`） |
| 起動時に `ERR_UNSUPPORTED_DIR_IMPORT` で落ちる | import 先が**ディレクトリで止まっている**（`'./workflows/'`）。Node の ES Modules はディレクトリ指定を認めないので、`'./workflows/hello-workflow.ts'` とファイル名まで書く。**`tsc --noEmit` はこれを見逃す**（`moduleResolution: "bundler"` が許すため）ので、型チェックが通っても `npm run dev` で起動して確かめる（Node 24.21.0 で確認） |
| `require is not defined` 等の ESM 関連エラー | `package.json` に `"type": "module"` があるか確認 |
| `mastra: command not found` | `mastra` が devDependency に入っていない。`npm install -D mastra@latest` |
| Node のバージョンが古いと言われる | Mastra v1 は **Node.js 22.13.0 以上**が必須。イメージのタグを確認 |
| Vite で `The request url is outside of Vite serving allow list` | `web/` の外（`src/mastra/schemas/`）を import しているため。`vite.config.ts` に `server.fs.allow: ['..']` を足す（[03-handson/day3/13-typed-client.md](../03-handson/day3/13-typed-client.md) §13-3） |
| フロントで `document` や `HTMLInputElement` が型エラーになる | `tsconfig.json` の `lib` に `DOM` が入っていない。`"lib": ["ES2022", "DOM", "DOM.Iterable"]` にする（同上 §13-3） |
| `web/main.ts` の `schema: packingListSchema` に `TS2590: Expression produces a union type that is too complex to represent` が出る | `agent.generate(...)` に型引数が無く、TypeScript が `schema` から出力の型を推論しきれない。**実行時には影響せず Vite では動く**が、`npx tsc --noEmit` が通らなくなる。`agent.generate<PackingList>(...)` と型引数を明示する（[03-handson/day3/13-typed-client.md](../03-handson/day3/13-typed-client.md) §13-5） |
| ブラウザのコンソールに CORS エラーが出る | Mastra の CORS は既定が `origin: '*'` なので通常は出ない。`server.cors` を設定した後なら `origin` にフロントのオリジンが含まれているか確認する。`credentials: true` のときは `'*'` が使えず、具体的なオリジンの列挙が必須（同上 §13-6） |
| Next.js で `error TS5097: An import path can only end with a '.ts' extension` | `travel-agent/` からコピーしたコードは Mastra CLI の流儀で拡張子付き import になっている。`tsconfig.json` に `"allowImportingTsExtensions": true` を足す（[03-handson/day3/14-nextjs-chat.md](../03-handson/day3/14-nextjs-chat.md) §14-3） |
| `create-mastra` のテンプレート実行で `npm error code EUNSUPPORTEDPROTOCOL` / `Unsupported URL Type "catalog:"` が出て、**作られたはずのディレクトリごと消える** | テンプレートの `package.json` に pnpm 専用記法（`"tsx": "catalog:"`）が残っているため。npm も pnpm も入らない。**失敗してから直すことはできない**ので、`--no-install` でクローンしてから依存を直す。**コマンドと詳しい原因は [02-templates/04-chat-with-database.md](../02-templates/04-chat-with-database.md) が正典** |

---

## 3. エージェント・ツールの挙動

| 症状 | 原因と対処 |
|---|---|
| **ツールが呼ばれない・無反応** | 素のオブジェクトでツールを定義している。必ず `createTool()`（`id` / `description` / `inputSchema` / `execute`）で定義する。エラーが出ないまま実行されないので気づきにくい |
| ツールがたまにしか呼ばれない | `description` が曖昧。「いつ使うべきか」が LLM に伝わる文章にする。`instructions` 側で「必ず〜を使え」と明示するのも有効 |
| ツールに変な引数が渡る | `inputSchema` の各フィールドに `.describe()` で説明を付ける |
| エージェントが推測で答えてしまう | `instructions` に「推測せず必ずツールで確認する」と明記する |
| `mastra.getAgentById()` が `MastraError` を投げる | `Mastra` インスタンスへの登録漏れ、または ID の不一致。`new Agent({ id: ... })` の値と一致しているか確認。**見つからないときは `undefined` ではなく例外**なので、try/catch か事前の登録確認で対処する |
| エージェントを直接 import すると挙動が違う | 直接 import したエージェントは共有ストレージ・ロギング・観測にアクセスできない。`mastra.getAgentById()` を使う |
| **`都市が見つかりません: 札幌` で天気ツールが失敗する** | Open-Meteo の geocoding API は**日本語の地名にヒットしない**。`Sapporo` のようにローマ字で渡す。`language=ja` は返却される地名の言語指定であって、検索語の言語ではない。ツールの `.describe()` に「ローマ字で指定」と書いておくと LLM が変換してくれる（[03-handson/day1/04-tools.md](../03-handson/day1/04-tools.md) §4-2） |
| ワークフローで `weatherTool.execute(...)` が型エラーになる | ツールの `execute` は省略可能なプロパティ（非 null 確認が要る）で、引数も `execute(input, context)` の2つが必要。`createStep(weatherTool)` でステップに変換し、整形は `.map()` で行う（[03-handson/day2/08-workflow.md](../03-handson/day2/08-workflow.md) §8-3） |
| ループ（`.dountil()`）の**2周目で** `Step input validation failed` になる | ループ本体の2周目の入力は**本体自身の前回の出力**。出力に入力へ必要な項目（`city` など）が無いと落ちる。**`tsc` は通る**ので実行して初めて気づく。本体の入力と出力を同じ形にそろえ、`{ ...inputData, ...結果 }` で引き継ぐ（[03-handson/day2/08-workflow.md](../03-handson/day2/08-workflow.md) §8-5） |
| `.branch()` の直後のステップで `expected object, received undefined` になる | 動かなかった枝のキーは `undefined` で届く。合流ステップの `inputSchema` で枝ごとのキーに `.optional()` を付ける（§8-4） |

---

## 4. モデル・API キーまわり

| 症状 | 原因と対処 |
|---|---|
| 401 / 403 | API キーが無効、または環境変数名が違う。OpenAI なら `OPENAI_API_KEY`（Gemini に差し替えた場合は `GOOGLE_GENERATIVE_AI_API_KEY` または `GOOGLE_API_KEY`。**どちらの名前でも通る**ので、片方だけ動くように見えるときは名前ではなく下の `remoteEnv` を疑う）。`.env` の場所と、キー取得後にクレジットをチャージ済みかも確認する（[01-setup/02-llm.md §3](../01-setup/02-llm.md)）。**`.env` に正しく書いたのに空扱いになるなら、`devcontainer.json` の `remoteEnv` が原因のことがある**（§1「コンテナ内で API キーが空」参照） |
| 429（レート制限・クォータ超過） | OpenAI はクレジット残高の不足でも返る。Billing の残高を確認し、少し待って再実行する。短時間に連打していないかも確認する |
| モデル ID が見つからない | `mastra.ai/models` で現行のモデル ID を確認。`provider/model` は **スラッシュ区切り**（`openai:gpt-...` は誤り） |
| `openai/gpt-5.6-luna` が動かない | 綴り（`luna` / `terra` / `sol`）と `openai/` プレフィックスを確認。疎通だけ先に見たいなら [01-setup/02-llm.md §3](../01-setup/02-llm.md) の `curl` でキー自体の有効性を確かめる |
| （Gemini に差し替えた場合）APIキーが `AQ.` から始まる／`AIza` で始まらない | 2026年に導入された新形式（Auth key）。正常な状態なので気にせず `.env` にそのまま貼ってよい。単体で疎通確認する場合は `?key=...` ではなく `x-goog-api-key` ヘッダーを使うこと（[01-setup/02-llm.md §4](../01-setup/02-llm.md)） |
| （Gemini に差し替えた場合）429 | 無料枠の分あたり/日あたり制限。少し待つか、有料枠へ移行する |
| （Gemini に差し替えた場合）`google/gemini-3.x-...` が動かない | Mastra の model router が対応していないモデル ID の可能性。まず `google/gemini-2.5-flash` で疎通確認してから切り替える |
| （Vertex 経由の場合）型エラーが大量に出る | `ai` と `@ai-sdk/google-vertex` のメジャーバージョン不整合。`npm ls ai @ai-sdk/google-vertex` で確認 |
| （Vertex 経由の場合）チューニング済みモデルが 401 | Express mode の API キーは tuned model 非対応。サービスアカウント方式へ |
| フロントエンドに API キーが漏れないか不安 | 分離構成（Day3 Step13）ではブラウザの通信相手は Mastra サーバー（4111）だけで、キーはサーバー側に留まる。**Next.js で `NEXT_PUBLIC_` を付けた環境変数はブラウザに埋め込まれる**ので、キーには絶対に付けないこと（[03-handson/day3/14-nextjs-chat.md](../03-handson/day3/14-nextjs-chat.md) §14-3） |

---

## 5. 構造化出力

| 症状 | 原因と対処 |
|---|---|
| `response_schema.properties[...].type: must be specified` | **Gemini / Vertex に差し替えたときに出る。** Google のモデルは OpenAPI 3.0 のサブセットしか受け付けない。`z.union` / `z.record` を使わない構造に書き換える（[01-setup/02-llm.md](../01-setup/02-llm.md) 6章）。標準構成の OpenAI では基本形のスキーマは通る |
| どうしても union が必要 | Vertex 経由なら `providerOptions.vertex.structuredOutputs: false` で回避できる。ただし出力の安定性は落ちる |
| `response.object` が undefined | `generate()` の第2引数に `structuredOutput: { schema }` を渡しているか確認 |

---

## 6. メモリ・ストレージ

| 症状 | 原因と対処 |
|---|---|
| 会話が記憶されない | `Mastra` インスタンスに `storage` を設定しているか確認。`url: ':memory:'` だと再起動で消える |
| `LibSQLStore` でエラー | `id` の指定が必須 |
| **`mastra.db` が2つできている** / Studio で話した内容をスクリプトが覚えていない（逆も） | 仕様。`file:./mastra.db` は実行中プロセスの cwd 基準で解決され、`mastra dev` は cwd = `src/mastra/public/`、`node src/index.ts` はプロジェクト直下なので別ファイルになる。まとめたいなら絶対パスを渡す（[reference/mastra-studio.md](../reference/mastra-studio.md) §7-2） |
| スレッドが混ざる / 分離されない | `generate()` に渡す `memory: { resource, thread }` を確認 |
| 記憶が多すぎてコストが増えた | `lastMessages` を減らす。トレースで実際の入力トークン数を確認する |
| **dev サーバーを止めたのに前回のトレースが残っている** | 仕様。`storage` が `file:` なら会話履歴と一緒にトレース（`mastra_ai_spans`）も永続化される。エージェントを編集しても消えない（[reference/mastra-studio.md](../reference/mastra-studio.md) §7） |
| トレースや会話履歴を消したい | dev サーバーを止めて DB ファイルを削除する。**両方が同じ DB に同居している**点に注意（同上 §7-5） |
| Traces 画面に何も出ない | ①**`observability:` を `new Mastra({...})` に渡していない**（これが最頻。渡していないとそもそも1件も記録されない。`package.json` に `@mastra/observability` があるだけでは効かない）②1回も実行していない ③`storage` が observability ドメインを持たない（起動ログに `Traces will not be persisted.` が出る）④期間フィルタが既定の Last 24 hours のまま（同上 §6・§7-1） |
| **評価（Step 11）の点数が Scorers 画面にいつまでも出ない** | ライブ採点は応答のあとに**裏で**走る。LLM 採点は judge の呼び出しが終わるまで数十秒かかる。出ないままなら、Scorer を `new Mastra({ scorers })` に登録したか、`storage` を設定したかを確認する（[reference/mastra-studio.md](../reference/mastra-studio.md) §8-1） |
| **スクリプトで作った Dataset が Studio に出ない** | 仕様。DB が別ファイル（`node` で動かすとプロジェクト直下の `mastra.db`、`npm run dev` は `src/mastra/public/mastra.db`）。Studio で見る Dataset は Studio から作る（同上 §8-3、§7-2） |
| `runEvals` で `Failed to save messages to sub-agent memory`（`Thread ID is required`）が何件も出る | Memory 付き・サブエージェント付きのエージェントを、`{ input: '…' }` で流したときのログ。判定には影響しない。`{ inputs: ['…'] }` に書き換えると質問ごとにスレッドが用意されて消える（[03-handson/day2/11-evals.md](../03-handson/day2/11-evals.md) §11-9） |

---

## 7. 公式サンプルをそのまま実行するときの注意

公式ドキュメントのコードを試す際、以下は読み替えが必要です。

1. **モデル指定が `openai/gpt-5.6-sol` などの上位モデルになっている**
   ハンズオンと同じ OpenAI なので `OPENAI_API_KEY` のままで動く。費用を抑えたいなら
   `model: 'openai/gpt-5.6-luna'` に置き換える。Gemini で試したい場合は
   [02-templates/07-provider-switch.md](../02-templates/07-provider-switch.md) を参照。

2. **Chef Michel チュートリアルの実行コマンドが `npx bun src/index.ts`**
   Bun を入れていない場合は `node src/index.ts`（Node 22.18+）または `npx tsx src/index.ts`。

3. **API パスの表記ゆれ**
   Chef Michel の agent 定義は `id: 'chef-agent'` だが curl 例は `/api/agents/chefAgent/generate`。
   実際のパスは `curl http://localhost:4111/api/agents` で確認する。

4. **バージョン差**
   docs は最新版（v1）を前提にしています。`mastra.ai/docs/v0/...` の URL は旧版なので混ぜないこと。

---

## 8. FAQ

**Q. Mastra は Python から使える？**
A. Mastra は TypeScript フレームワークです。Python 版はありません。ただし `mastra dev` が HTTP API を公開するので、Python 側から HTTP で呼ぶことは可能です。

**Q. LangChain / LangGraph との違いは？**
A. Mastra は TypeScript ネイティブで、Studio による GUI デバッグと Workflow のグラフ可視化が標準装備という点が特徴です。優劣の断定は避け、「エコシステムの成熟度は LangChain、TypeScript での開発体験は Mastra」といった整理が無難です。ライセンスや機能の詳しい比較表は [00-overview/10-comparison.md](../00-overview/10-comparison.md) にあります。

**Q. Dify との違いは？**
A. Dify は GUI でアプリを組み立てるプラットフォーム、Mastra はコードで書くフレームワークです。**成果物をコードとして持つかどうか**が分かれ目で、Git 管理・型・テスト・既存アプリへの組み込みを重視するなら Mastra、非エンジニアも同じ画面で編集したいなら Dify。優劣ではなく用途の違いとして説明してください。比較表は [00-overview/10-comparison.md](../00-overview/10-comparison.md) にあります。

**Q. 本番環境ではどこにデプロイする？**
A. 公式にデプロイメントガイドがあります（Cloudflare、Vercel、AWS など）。https://mastra.ai/docs/deployment/overview

**Q. 無料枠のまま社内本番で使える？**
A. そもそも本ハンズオンの標準構成（OpenAI）に無料枠はなく、有料の API です。なお Gemini に差し替えた場合の無料枠は、送信内容が製品改善に利用され、レート制限もあるため、社内本番では使うべきではありません。業務利用なら有料枠、または組織の契約（OpenAI / Google Cloud / Azure / AWS など）を検討してください。

**Q. ハンズオン中の LLM 費用は？**
A. 標準の `gpt-5.6-luna` で、ハンズオン全体で数十円程度です。最上位の `gpt-5.6-sol` を使っても、この規模なら数十円程度です（料金は [01-setup/02-llm.md](../01-setup/02-llm.md) §2）。Gemini の無料枠に差し替えれば 0 円にできます。

**Q. テレメトリは送信される？**
A. `create-mastra` CLI は匿名の利用情報（OS、Mastra バージョン、Node バージョン）を収集します。`MASTRA_TELEMETRY_DISABLED=1` で無効化できます。

---

## 9. 詰まったときの調べ先

1. 公式ドキュメント内検索: https://mastra.ai/docs （右上の Search、および「Ask AI」）
2. リファレンス: https://mastra.ai/reference
3. GitHub Issues: https://github.com/mastra-ai/mastra/issues
4. 各ドキュメントページ末尾の「Edit this page on GitHub」から原文（.mdx）を読む

---

## 10. 自分用メモ欄

ハンズオン中に遭遇したエラーと解決法をここに追記してください。同じ場所で詰まったときの近道になります。

| 日付 | 症状 | 原因 | 解決法 |
|---|---|---|---|
| | | | |
