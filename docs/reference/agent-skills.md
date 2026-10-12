# テンプレート同梱の `.agents/` フォルダとは（Agent Skills）

[02-templates/](../02-templates/) で公式テンプレートを展開したあと `ls -a` すると、見覚えのない
`.agents/` フォルダと `skills-lock.json` が出てきます。これが何なのかをまとめた**参照資料**です。

- **対象読者**: テンプレートの中身を読んでいて「このフォルダは何？消していいのか？」と迷った人
- **調査日**: 2026-08-22（`template-sql` / `template-deep-search` の実体と、`mastra-ai/skills` の公開情報で確認）

---

## 1. 結論（先に3行）

1. `.agents/` は **AI コーディングエージェント（Claude Code / Cursor 等）に読ませるための「スキル」置き場**で、
   Mastra のランタイムとは一切関係ありません。**`src/mastra/agents/` とは別物**です。
2. 中身は Mastra 公式の [mastra-ai/skills](https://github.com/mastra-ai/skills) を丸ごと同梱したもので、
   要旨は「**Mastra について記憶で書くな。必ず現行ドキュメントを引いてから書け**」という指示書です。
3. **ハンズオンの実行には不要**です。触らなくても動きますし、テンプレートごと削除して構いません。
   ただし **AI に Mastra のコードを書かせるなら効果があります**（→ §6）。

---

## 2. 実体

`template-sql` と `template-deep-search` の両方に、**まったく同じもの**が入っています
（`diff -r` で差分なしを確認）。合計 **104KB**、11ファイルだけです。

```
template-sql/
├── skills-lock.json              ← どこから取ってきたスキルかの記録（ロックファイル）
└── .agents/
    └── skills/
        └── mastra/               ← スキル1つ = 1フォルダ
            ├── SKILL.md          ← 本体（これだけが常時読まれる）
            ├── references/       ← 詳細資料 9本（必要になったときだけ読まれる）
            └── scripts/
                └── provider-registry.mjs
```

`node_modules/` 配下ではなく**プロジェクト直下**にある点に注意してください。npm パッケージではなく、
テンプレートのリポジトリに直接コミットされているファイル群です。

> **`create-mastra` が生成しているわけではありません。** `mastra` CLI のコードを検索しても
> `.agents` を書き出す処理はありません。テンプレート作者が事前に取り込み、
> テンプレートの中身としてクローンされてくるだけです。

---

## 3. `.agents/` とは何か

### 3-1. `src/mastra/agents/` との違い（ここが一番の混乱ポイント）

| | `.agents/` | `src/mastra/agents/` |
|---|---|---|
| 誰のためのもの | **開発者が使う AI**（Claude Code など） | **Mastra が実行するエージェント** |
| 中身 | Markdown の指示書 | TypeScript のコード |
| 実行時の役割 | **無し**。`mastra dev` は読まない | 本体。Studio に並ぶのはこちら |
| 消したらどうなる | AI の Mastra 知識が落ちるだけ | アプリが壊れる |

名前が紛らわしいだけで、**レイヤーがまったく違います**。`.agents/` は `.editorconfig` や
`.vscode/` と同じ「開発支援のための設定ファイル」の仲間だと考えると分かりやすいです。

### 3-2. Agent Skills という共通仕様

`SKILL.md`（YAML フロントマターに `name` と `description` を持つ Markdown）で AI の能力を拡張する、
ツール横断の共通仕様です。`npx skills` という CLI がパッケージマネージャの役割を担い、
**ホストごとに決められたディレクトリへ配置**します。

| ホスト | 配置先 |
|---|---|
| Claude Code | `.claude/skills/` |
| Cursor | `.cursor/skills/` |
| OpenCode | `.opencode/skills/` |
| Amp / Kimi ほか（ベンダー中立） | **`.agents/skills/`** |

> **⚠️ Claude Code は `.agents/skills/` を自動では読みません。** 標準の探索先は
> プロジェクト直下の `.claude/skills/` です。テンプレートに入っているのは
> あくまで「中立の置き場に置かれた配布物」なので、Claude Code で使いたい場合は §6 の手順で入れ直します。

### 3-3. `skills-lock.json`

`package-lock.json` の Agent Skills 版です。出所とハッシュを固定し、更新差分を追えるようにします。

```json
{
  "version": 1,
  "skills": {
    "mastra": {
      "source": "mastra-ai/skills",
      "sourceType": "github",
      "skillPath": "skills/mastra/SKILL.md",
      "computedHash": "6482dad39fb72d1fe9d02078c15c3ee708774fd026a8557b4c7dc37fb554977a"
    }
  }
}
```

---

## 4. `.agents/skills/mastra/` の中身

Mastra 公式スキル（`version: 2.1.0` / Apache-2.0 / author: Mastra）です。
`SKILL.md` の冒頭が、このスキルの存在理由をそのまま述べています。

> **Critical: Do not trust internal knowledge**
> — Mastra について知っていることは古いか間違っている可能性が高い。記憶に頼らず、常に現行ドキュメントで検証せよ。

Mastra は API の変更が速く、LLM の学習データには v0.x 系の古い書き方が大量に残っています。
それを踏ませないためのガードレールというわけです。具体的な指示は次の4点。

1. **調べる順番を固定する** — ①`node_modules/@mastra/*/dist/docs/` の埋め込みドキュメント（インストール済み
   バージョンと完全一致するので最優先）→ ②インストール済みのソースと型定義 → ③未インストールなら
   `https://mastra.ai/llms.txt` のリモートドキュメント。
2. **モデル名を記憶で書かない** — 必ず `"provider/model-name"` 形式にし、`provider-registry.mjs` を
   実行して実在を確認する。
3. **TypeScript は ES2022 必須** — CommonJS では動かない。
4. **型エラーが出たらまず自分の知識を疑う** — 「ユーザーのミス」と決めつけない。

### 4-1. `references/` の9本

`SKILL.md` は目次だけを持ち、必要になった資料だけを読ませる作りです（progressive disclosure）。

| ファイル | どんなときに読まれるか |
|---|---|
| `create-mastra.md` | プロジェクトの作成・インストール |
| `core-concepts.md` | Agent / Workflow / Tool / Memory / Storage の使い分け |
| `embedded-docs.md` | `node_modules` 内のドキュメントの引き方 |
| `remote-docs.md` | パッケージ未インストール時の公式ドキュメント参照 |
| `model-selection.md` | モデル指定の形式と検証 |
| `common-errors.md` | よくあるエラーと対処 |
| `migration-guide.md` | v0.x → v1.x の移行 |
| `mastra-api.md` | `mastra api` CLI でサーバー上の資源を操作する |
| `trace-intelligence.md` | トレースを横断分析してエージェントの改善点を探す（プラットフォーム機能） |

### 4-2. `provider-registry.mjs` は受講者にも役立つ

**利用可能なプロバイダとモデル名を実際に引くスクリプト**です。AI 用ですが、人間が叩いても有用です。
[02-templates/03-model-switch.md](../02-templates/03-model-switch.md) でモデル ID を GPT-5.6 系へ差し替えるとき、**打ち込む前に実在を確認**できます。

```bash
# テンプレートのディレクトリで実行する（node_modules を上へ辿って探す作り）
node .agents/skills/mastra/scripts/provider-registry.mjs --list
node .agents/skills/mastra/scripts/provider-registry.mjs --provider openai
```

```
OpenAI — 47 models

  gpt-5.6-terra
  gpt-5.6-sol
  gpt-5.6-luna
  gpt-5.6
  ...
```

参照しているのは `node_modules/@mastra/core/dist/provider-registry.json` なので、
**`npm install` 済みのプロジェクト内でのみ動きます**。

---

## 5. 片付けとの関係

`.agents/` は 104KB しかないため、[02-templates/09-cleanup.md](../02-templates/09-cleanup.md) の片付けで気にする必要はありません。
テンプレートごと `rm -rf` すれば一緒に消えます。`template-*/` は
[.gitignore](../../.gitignore) で無視されるので、親リポジトリにも残りません。

---

## 6. 自分のプロジェクトで使いたい場合

`travel-agent/` を Claude Code などに手伝わせて作るなら、同じスキルを入れておくと
**AI が古い v0.x 系 API を書いてくる事故がかなり減ります**。テンプレートからコピーするのではなく、
CLI で入れ直すのが確実です（ホストに合った場所へ配置され、`skills-lock.json` も更新されるため）。

```bash
# DevContainer 内、プロジェクトのルートで実行する
npx skills add mastra-ai/skills
```

対話でホスト（Claude Code など）を聞かれるので、使っているツールを選びます。Claude Code を選べば
`.claude/skills/mastra/` に入り、そのセッションから参照されるようになります。

- 一覧: `npx skills list`
- 更新確認 / 更新: `npx skills check` / `npx skills update`
- 削除: `npx skills remove mastra`

Mastra は [RFC 8615 の Well-Known URI](https://mastra.ai/.well-known/skills/index.json) でもスキルを公開しており、
対応ホストからは URL 指定なしで発見できます。

---

## 参考リンク

- Mastra 公式スキル: https://github.com/mastra-ai/skills
- スキル配信元（Well-Known URI）: https://mastra.ai/.well-known/skills/index.json
- skills CLI: https://github.com/antfu/skills-cli
- Claude Code のスキル（`.claude/skills/`）: https://docs.claude.com/en/docs/claude-code/skills
