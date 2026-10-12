# Step 12. ビルドと成果物確認（30分）

ここまでは `npm run dev`（開発サーバー）で動かしてきました。この Step では、アプリを**本番用にビルド**し、
できあがった成果物を**実際に起動して API から使い**、最後に**成果物を削除**するところまで体験します。
「自分の書いたコードが、どんな形で配れるのか」を掴むのが狙いです。

**前提**: Step 11 まで完了（`travel-agent/` で `npm run dev` が動くこと）。

> 💡 `npm run dev`（`mastra dev`）は開発用で、Studio・ホットリロード付きです。`npm run build`（`mastra build`）は
> **配布用の成果物を作るだけ**で、サーバーは起動しません。

---

## 12-1. 型チェック（5分）

ビルドの前に、型エラーが無いことを確かめます。

```bash
npx tsc --noEmit
```

- **何も表示されなければ成功**です（`--noEmit` は JS を出力せず、型の検査だけをします）。
- エラーが出たら `ファイル名(行,列): error TSxxxx` の形で場所が示されます。直してから先へ進みます。

> 💡 `mastra build` 自体は型エラーがあっても通ることがあります（コードを変換してまとめるだけで、厳密な型検査はしないため）。
> **「ビルドが通る」と「型が正しい」は別物**なので、両方を確認します。

---

## 12-2. ビルドする（5分）

```bash
npm run build
```

次のようなログが出て、最後に `Build successful` と表示されれば成功です（1分もかかりません）。

```
Analyzing dependencies...          ← 使っているパッケージを調べる
Bundling Mastra application        ← コードを1つにまとめる
Installing dependencies            ← 本番用の node_modules を作る
Build successful, you can now deploy the .mastra/output directory to your target platform.
To start the server, run: node .mastra/output/index.mjs
```

### 何が出力されたか

生成物はすべて `.mastra/` の下にできます。

```
travel-agent/.mastra/
├── build-manifest.json    ← ビルド時刻とソースのハッシュ
├── bundler-config.mjs
└── output/                ← ★ これが配布する本体
    ├── index.mjs          ← サーバーの入口（起動するのはこれ）
    ├── mastra.mjs         ← 自分の Mastra 定義（agents / workflows / scorers …）をまとめたもの
    ├── tools/             ← ツール
    ├── package.json       ← 本番で必要なパッケージだけを列挙
    ├── package-lock.json
    └── node_modules/      ← 本番用の依存パッケージ（ここだけで動く）
```

| 見るところ | わかること |
|---|---|
| `output/index.mjs` | `src/` の TypeScript が**1つの JavaScript にバンドル**されている。`.ts` を直接は実行していない |
| `output/package.json` | 本番に必要な依存だけが並ぶ。`@types/node` や `typescript` などの開発用は含まれない |
| `output/node_modules/` | 依存が**成果物の中に閉じている**ので、このフォルダだけを別の場所へ持っていける（サイズは約 280 MB） |

> 💡 `.mastra/` は `.gitignore` 済みです（[01-setup/01-environment.md](../../01-setup/01-environment.md) §4）。
> **成果物は Git で管理せず、ビルドするたびに作り直す**ものです。

---

## 12-3. 起動して API から使う（10分）

### ① ビルドした成果物を起動する

```bash
node --env-file=.env .mastra/output/index.mjs
```

- `--env-file=.env` で `OPENAI_API_KEY` を渡します。**ビルドした成果物は `.env` を自動では読みません**
  （`npm run dev` では読まれていました）。渡し忘れると、API を呼んだときに認証エラーになります。
- `Mastra API running { url: 'http://localhost:4111/api' }` と出れば起動しています。
- `npm run dev` が動いたままだと、ポート 4111 が埋まっています。**先に dev サーバーを止める**か、
  `PORT=4200 node --env-file=.env .mastra/output/index.mjs` のようにポートを変えます。

### ② ブラウザで開く

`http://localhost:4111/` を開くと、**「Mastra Server」という簡単な状態ページ**が出ます。

> ⚠️ **Studio は開発用なので、ビルドした成果物には入りません。** 本番では画面を持たない
> **API サーバーとして**動きます。Step 3〜11 で触ってきた画面（Agents / Traces / Scorers など）は
> `npm run dev` でだけ使えます。

### ③ API を呼ぶ

別のターミナルから、`curl` でエージェントに話しかけます。

```bash
# 登録されているエージェントの一覧
curl -s http://localhost:4111/api/agents | head -c 300

# travel-agent に質問する
curl -s -X POST http://localhost:4111/api/agents/travel-agent/generate \
  -H 'content-type: application/json' \
  -d '{"messages":"来週、札幌に行きます。持ち物を教えて"}'
```

返ってくる JSON に、エージェントの回答が入っています（`text` フィールド）。Memory を使うときは、
会話を束ねる `thread` と利用者を表す `resource` を渡します（Step 7）。

```bash
curl -s -X POST http://localhost:4111/api/agents/travel-agent/generate \
  -H 'content-type: application/json' \
  -d '{"messages":"私の名前は太郎です","memory":{"thread":"t1","resource":"taro"}}'
```

| エンドポイント | 用途 |
|---|---|
| `GET  /api/agents` | エージェントの一覧と設定 |
| `POST /api/agents/<id>/generate` | 1回で全文を返す |
| `POST /api/agents/<id>/stream` | 文字を順次返す（Day3 のフロントエンド編で使う） |
| `GET  /api/workflows` | ワークフローの一覧 |

> 💡 **このサーバーが「デプロイするもの」の正体です。** 社内のサーバーやコンテナ、クラウドの上で
> `node .mastra/output/index.mjs` を動かし、フロントエンドや他システムから HTTP で呼べば、
> 本番運用になります。なお、dev サーバーでは使えた `/swagger-ui` は、ビルドした成果物では既定で無効です（404 になる）。

### ④ データの置き場所に注意する

```bash
find . -name '*.db' -o -name '*.duckdb' | grep -v node_modules
```

ビルドした成果物を **`travel-agent/` で起動した場合、DB は `./mastra.db` と `./mastra.duckdb`（コマンドを打ったディレクトリ）に**
できます。`npm run dev` が使う `src/mastra/public/` 配下の DB とは**別のファイル**なので、
dev で溜めた会話履歴やトレースは引き継がれません（理由は [reference/mastra-studio.md](../../reference/mastra-studio.md) §7-2 と同じ）。

> ⚠️ 本番では、ストレージを `file:` の SQLite ではなく**外部のデータベース**（PostgreSQL など）にするのが普通です。
> サーバーを作り直すたびにファイルが消える環境（コンテナなど）では、ファイルの DB は残りません。

起動を止めるときは、起動したターミナルで `Ctrl + C` を押します。

---

## 12-4. ビルドを削除する（5分）

ビルド成果物は**いつでも消して構いません**。ソースコード（`src/`）から作り直せるためです。

```bash
# 停止してから削除（サーバーが動いたままだと、動作中のファイルを消すことになる）
rm -rf .mastra
```

| 疑問 | 答え |
|---|---|
| 消すとコードが失われる？ | **いいえ。** `.mastra/` は `src/` から生成されただけの出力で、元は `src/` にあります |
| 消すと困る？ | 再びビルドするまで `node .mastra/output/index.mjs` が使えなくなるだけ。`npm run build` で戻る |
| Git に影響する？ | しない。`.gitignore` 済みなので差分にも出ない |
| `npm run dev` の最中に消していい？ | **避ける。** dev サーバーも `.mastra/` を作業場所に使っているため、動作中に消すと不具合の原因になる。止めてから消す |
| `mastra.db` も消える？ | **消えない。** DB は `.mastra/` の外にある（12-3 ④）。会話履歴やスコアを消したいときは、その `.db` / `.duckdb` ファイルを別に消す |

DB も含めて**まっさらに戻したい**ときは、サーバーをすべて止めてから次のとおり消します。

```bash
rm -rf .mastra
rm -f mastra.db mastra.db-shm mastra.db-wal mastra.duckdb mastra.duckdb.wal   # ビルド成果物を起動して作られた DB
```

> ⚠️ `src/mastra/public/` の DB は `npm run dev` で溜めた Step 7〜11 のデータです。消すと**会話履歴・Dataset・
> 実験結果が失われる**ので、必要なら残しておきます。

---

## 12-5. デプロイ先を選ぶ（5分）

ここまでで、成果物は「`node` で起動する API サーバー」だとわかりました。置き場所は次のように考えます。

| 選択肢 | 向く場面 |
|---|---|
| 社内サーバー / VM | `node` と `output/` を置き、systemd などで常駐させる。社内向けで最も素直 |
| コンテナ（Docker） | `output/` をイメージに入れる。再現性が高く、クラウドにも移しやすい |
| Mastra Cloud / サーバーレス | 管理を任せたいとき。ファイルが残らない環境が多いので、外部 DB が前提になる |

Mastra のデプロイ手段と注意点は公式の [Deployment Overview](https://mastra.ai/docs/deployment/overview) を確認してください。

---

## チェックポイント

- [ ] `npx tsc --noEmit` で型エラーが無い
- [ ] `npm run build` が通り、`.mastra/output/index.mjs` ができた
- [ ] `node --env-file=.env .mastra/output/index.mjs` で起動し、`curl` でエージェントの回答が返った
- [ ] 「ビルドした成果物には Studio が入らない」「`.env` を自動では読まない」を説明できる
- [ ] `rm -rf .mastra` で成果物を消せること、DB は別に残ることを理解した
- [ ] 「型が正しい」と「ビルドが通る」が別物だと説明できる

> 動作確認は `mastra` CLI 1.30.0 / `@mastra/core` 1.67.0 で行いました（型チェック・ビルド・成果物の起動・`/api/agents` の応答・
> `.env` を渡さないと API キーを見つけられないこと・`rm -rf .mastra`）。回答の中身（`text`）までは、API キーを使う実呼び出しをしていないため未確認です。

---

**Day 2 はここまでです。** ここまでの画面はすべて **Studio（開発者向けの画面）**でした。
「エンドユーザーが見る画面」まで作りたい場合は、[day3/](../day3/)（フロントエンド編・任意）で
自作のチャット UI を作ります。ツール呼び出しが UI に表示されるところまで確認できます。
