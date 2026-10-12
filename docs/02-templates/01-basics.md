# 2. 基本の使い方

```bash
npx create-mastra@latest --template <スラッグ>
```

プロジェクト名を指定する場合:

```bash
npx create-mastra@latest my-project --template <スラッグ>
```

## 2-1. プロジェクトはどこに作られるか

**コマンドを実行したカレントディレクトリの直下**に、フォルダが1つ作られます。出力先を指定する
フラグ（`--dir` のようなもの）は無いため、**作りたい場所へ `cd` してから実行**します。

| ケース | どうなるか |
|---|---|
| プロジェクト名を指定した | その名前が**ディレクトリ名と `package.json` の `name` の両方**になる |
| プロジェクト名を省略した | カレントに直接展開されるのではなく、**対話でプロジェクト名を聞かれる** |
| 既存の Mastra プロジェクトの中で実行した | **プロジェクトが入れ子になる**（6章の注意点5はこの仕様が理由） |

```bash
cd ~/work/mastra-handson
npx create-mastra@latest my-project --template <スラッグ>
# → ~/work/mastra-handson/my-project/ が作られる
```

## 2-2. スラッグの指定方法

| やりたいこと | コマンド |
|---|---|
| スラッグを指定する | `npx create-mastra@latest my-project --template <スラッグ>` |
| 一覧から対話で選ぶ | `npx create-mastra@latest my-project --template`（値を省略） |
| 任意の GitHub リポジトリから作る | `npx create-mastra@latest my-project --template https://github.com/mastra-ai/template-text-to-sql` |

> **⚠️ ページ URL とスラッグは一致しないことがあります。** 例: Google Sheet Analysis はページ URL が
> `/templates/google-sheets-analysis` ですが、コマンドは `--template google-sheets` です。
> **必ず各テンプレートのページの「Get started」に書かれたコマンドをコピー**してください。

## 2-3. `--llm` フラグは効かない

テンプレートは作者が依存関係・モデル・環境変数・ソースコードを所有しています。そのため
`create-mastra` の `--llm` / `--llm-api-key` は**テンプレートには適用されません**。
モデルの差し替えは自分で行います（モデル ID だけなら 4-0、プロバイダごとなら5章）。

## 2-4. 見慣れない `.agents/` と `skills-lock.json` について

展開したテンプレートで `ls -a` すると、`src/` や `package.json` に混じって `.agents/` フォルダと
`skills-lock.json` が出てきます。これは **Claude Code などの AI コーディングエージェントに
「Mastra の正しい書き方」を教えるための指示書**（Agent Skills）で、**Mastra のランタイムとは無関係**です。
`mastra dev` は読みません。**触らなくてよく、テンプレートごと消して構いません。**

ただし同梱の `provider-registry.mjs` は、次章 4-0 でモデル ID を差し替える際に
**実在するモデル名かを確認できる**ので、人間が使っても便利です。

詳細は [reference/agent-skills.md](../reference/agent-skills.md) を参照してください。
