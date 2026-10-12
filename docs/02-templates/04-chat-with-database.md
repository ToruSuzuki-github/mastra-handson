# 4-1. Chat with Database（60分）★最推奨

**なぜこれか**

- **準備が要らない** — サンプルデータが起動時に自動生成されるため、**OpenAI キー以外に何も要りません**
- **用途が想像しやすい** — 「自然言語で DB を検索する」は最もユースケースを想像しやすい題材
- **読む価値のあるコード** — ツールの分割とプロンプトによる安全確保が実例として読めます

```bash
cd ~/work/mastra-handson
npx create-mastra@latest template-sql --template text-to-sql --no-install
cd template-sql

# ⚠️ テンプレート側の不具合を回避する（下の警告を必ず読むこと）
npm pkg set devDependencies.tsx="^4.23.1"
npm install

# .env.example を .env にコピーしてキーを記入
cp .env.example .env
# エディタで .env を編集し、OPENAI_API_KEY= に取得済みのキーを貼る

# モデル ID を GPT-5.6 系へ差し替える（4-0 参照。ここは1ファイル1行）
grep -rl 'openai/gpt-5-mini' src/ | xargs -r sed -i 's#openai/gpt-5-mini#openai/gpt-5.6-luna#g'
grep -rn "openai/" src/   # openai/gpt-5.6-luna になっていることを確認

npm run dev
```

> **⚠️ `--no-install` と `npm pkg set` を省略しないでください**（2026-08-15 時点）。省略すると
> 依存インストールで落ち、**作られたディレクトリごと消えます**。失敗してから直すことはできません。

<details>
<summary>⚙️ <b>補足：なぜ <code>--no-install</code> が要るのか</b>（詰まったとき・原因を知りたいときに開く）</summary>

素直に `npx create-mastra@latest template-sql --template text-to-sql` を実行すると、クローンは
成功するのに依存インストールで落ちます。

```
✔  Template "Chat with Database" cloned successfully to template-sql
✖  Failed to install dependencies: Command failed with exit code 1: npm install
npm error code EUNSUPPORTEDPROTOCOL
npm error Unsupported URL Type "catalog:": catalog:
```

**原因**: テンプレートの `package.json` に `"tsx": "catalog:"` という **pnpm 専用の記法**が残っている
ためです。`catalog:` は pnpm 9.5+ のカタログ機能の書式で、npm は対応していません（npm を新しくしても
直りません）。テンプレートは Mastra のモノレポから自動同期されており、同期時にカタログの実バージョンへ
展開し損ねている **Mastra 側の不具合**です。同期先にはカタログ定義（`pnpm-workspace.yaml`）自体が
無いため、pnpm に切り替えても同様に失敗します。

**なぜ `--no-install` が要るか**: create-mastra は一時ディレクトリにクローンしてから install し、
成功したときだけ最終位置へ移動します。install で落ちると一時ディレクトリごと巻き戻されるので、
**失敗してから `package.json` を直す、という復旧はできません**。`--no-install` でインストールを
止めておき、`npm pkg set` で `tsx` を実バージョン（Mastra モノレポのカタログと同じ `^4.23.1`）に
固定してから、自分で `npm install` します。`tsx` は `npm run dev` / `npm run seed` が使うため削除は不可です。

同じ不具合は **Docs Chatbot** テンプレートにもあります。上流が修正されたら、この回避策は不要になり
素の `npx create-mastra@latest template-sql --template text-to-sql` に戻せます。

</details>

<details>
<summary>🗄️ <b>補足：サンプル DB はいつ作られるのか</b>（`data.db` が見当たらないとき）</summary>

リポジトリに `data.db` は含まれていません。`npm run dev` が `tsx src/seed.ts && mastra dev` という
定義になっているため、**起動のたびに seed スクリプトが走り**、`src/mastra/public/data.db`（会社・
従業員・プロジェクトのサンプルデータ）を作ります。2回目以降は「already seeded」と出てスキップされます。
DB だけ作り直したいときは `npm run seed` を単体で実行できます。

なお、このテンプレートの `.env.example` は `OPENAI_API_KEY=` の**1行だけ**です。

</details>

http://localhost:4111 を開いて Studio でチャットします。公式が挙げている質問例:

- 「データベースにはどんなテーブルがある？」
- 「Acme Corp の従業員を全員見せて」
- 「部署ごとの平均給与は？」
- 「進行中のプロジェクトと合計予算は？」
- 「2023年に入社した従業員を給与順で」
- 「一番売上が高い会社は？」

<details>
<summary>🖥️ <b>このテンプレートを Studio でどう見るか</b>（Studio を初めて開く人はここを展開）</summary>

> **Studio 自体の使い方**（左ナビの構成、Traces 画面の読み方、トレースの保存先など）は
> [reference/mastra-studio.md](../reference/mastra-studio.md) にまとめてあります。**Studio が初めてならそちらを先に**。
> ここでは Chat with Database 固有の見どころだけを扱います。

左ナビの各画面に、このテンプレートの登録内容がどう現れるかの対応です。

| 画面 | このテンプレートでは |
|---|---|
| **Agents** | SQL Agent が1体 ← **ここがメイン** |
| **Tools** | `introspect-database` / `execute-sql` の2つ |
| **Workflows** | **空**（正常。理由は下記3） |
| **Traces** | 実行するまで0件 |

## 1. まずチャットする（Agents）

左ナビ **Agents** → **SQL Agent** → チャット画面が開きます（直リンク: http://localhost:4111/agents/sql-agent/chat ）。

最初の1発は上の質問例のうち **「データベースにはどんなテーブルがある？」** から始めてください。
エージェントは instructions で「まず `introspect-database` を呼べ」と縛られているため、**ツール呼び出しの流れが一番きれいに観察できます**。

回答本文だけ読んで終わらないこと。このテンプレートで見るべきものは2つです。

- **ツール呼び出しの折りたたみ**。`introspect-database` → `execute-sql` の順に2回呼ばれます。
  開くと**引数（＝エージェントが生成した SQL）と戻り値（＝実行結果）がそのまま**見えます。
  「自然言語 → SQL → 実行結果」の変換過程を目視できるのが、このテンプレートの肝です。
- **右パネルのモデル ID**。4-0 の差し替えが効いているか（`gpt-5.6-luna` になっているか）をここで確認できます。

## 2. ツールを単体で叩く（Tools）

**LLM を介さずツールだけを実行**できる画面です。このテンプレートでは次の2つを試せます。

- `introspect-database`: 引数なしで実行 → スキーマ JSON が返る
- `execute-sql`: `query` に `SELECT * FROM employees LIMIT 5` などを直接入れて実行

エージェントの回答がおかしいとき、「ツールが壊れているのか / LLM の SQL 生成が悪いのか」を
ここで切り分けられます。**API キーを消費しません**。

## 3. Workflows タブが空なのは正常

このテンプレートは `src/mastra/index.ts` に `agents` しか登録しておらず、ワークフローを1つも定義していないためです。

```ts
export const mastra = new Mastra({
  agents: { sqlAgent },   // ← workflows: {...} が無い
  ...
});
```

Workflows タブを使うのは **4-2 の Deep Search** です。そちらでは、suspend/resume で処理が止まり
人間の入力待ちになる挙動を、この画面から操作します。

## 4. トレース（Traces）で見るべきもの

下の「観察するポイント」1で言う「トレース」は `/traces` 画面のことです。**チャットを1回実行してから**開いてください。

このテンプレートでは、**1回の質問に対して LLM の推論が2回走っている**ことがツリーから読めます
（1回目でツールを選び、2回目でツールの結果を文章にする）。4-0 でモデルを差し替えた効果は、
ここに出るトークン数で数字として確認できます。

</details>

**観察するポイント**

1. エージェントが**スキーマを自分で調べてから** SQL を書いていること（トレースで確認）
2. ツールが2つ（`introspect-database` / `execute-sql`）に分かれている設計意図
3. `src/mastra/tools/` のコードを読み、Day1 で自作したツールと比較する
4. `src/mastra/agents/sql-agent.ts` の instructions が「**まず introspect-database を呼べ**」「**SELECT 以外は生成するな**」と手順と制約を明文化していること。プロンプトで安全性を担保する実例として読む

> **発展**: 「自然言語 → SQL → 実行結果」の流れは、自分で用意したダミーデータでも試せます。身近な題材のデータを入れた SQLite に差し替えると、用途のイメージがさらに具体的になります。

**チェックポイント**

- [ ] `npm run dev` が起動し、ログの `│ Studio: http://localhost:____` が **4111** になっている
- [ ] Studio の Agents に SQL Agent が1体だけ出る
- [ ] 「データベースにはどんなテーブルがある？」に答えが返る
- [ ] ツール呼び出しの折りたたみで、生成された SQL と実行結果が読める
- [ ] 右パネルのモデル ID が `gpt-5.6-luna` になっている
