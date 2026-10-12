# Mastra Studio の使い方（画面と仕様）

`mastra dev`（＝`npm run dev`）が立ち上げるローカル GUI **Mastra Studio** の、画面ごとの使い方と
「トレースはどこに保存されるのか」といった仕様をまとめた**参照資料**です。

- **対象読者**: [02-templates/](../02-templates/) / [03-handson/day1/](../03-handson/day1/) / [03-handson/day2/](../03-handson/day2/) を進めていて、Studio の画面で迷った人
- **Studio が「何のためにあるか」という位置づけ**は座学の [00-overview/08-devops.md](../00-overview/08-devops.md) §8-1 が正典です。本ファイルは重複を置かず、**画面操作と保存仕様**だけを扱います
- **調査日**: 2026-08-17（画面は **Studio v1.59.0** の実機で確認。§5-3 のみ 2026-08-19、§7-3 の「会話履歴の中身」は 2026-09-21 に追記）。**§7-6（Metrics と DuckDB）と §8（Evaluation）は 2026-10-03 に `mastra` CLI 1.30.0 / `@mastra/core` 1.67.0 で確認**（サーバーの HTTP API と同梱の Studio のソースで動作と画面の文言を確認した。ボタンの位置などの見た目までは実機で確かめていない）

> **⚠️ Studio の UI はバージョンで変わります。** 左ナビのラベルや URL が本ファイルと違っていたら、
> 実機を正としてください（左サイドバー最下部にバージョンが表示されています）。

---

## 1. 起動と URL

```bash
npm run dev
```

**起動ログに `│ Studio: http://localhost:4111` の行が出てから**ブラウザで開きます。
`mastra dev` はバンドルが終わった最後にポートを Listen するため、それ以前に開いても繋がりません。

> **⚠️ 4111 とは限りません。** 別のプロジェクトの dev サーバーを止め忘れていると、
> エラーにならず **4112 で起動**します。起動ログの `│ Studio: http://localhost:____` を必ず読んでください。
> 詳細と対処は [troubleshooting/ §1](../troubleshooting/)。

---

## 2. 画面構成

Studio の中身は、`src/mastra/index.ts` の `new Mastra({...})` に**登録したもの**です。
左ナビは3グループに分かれています。

| グループ | 項目 | URL | 何が出るか |
|---|---|---|---|
| **Primitives** | Agents | `/agents` | `agents: {...}` に登録したエージェント |
| | Workflows | `/workflows` | `workflows: {...}` に登録したワークフロー |
| | Processors | `/processors` | 入出力プロセッサ |
| | MCP Servers | `/mcps` | MCP サーバー連携 |
| | Tools | `/tools` | エージェントに紐づくツール |
| | Workspaces | `/workspaces` | ファイル操作・コード実行用のワークスペース |
| | Request Context | `/request-context` | 実行時に注入する値の設定 |
| **Evaluation** | Overview / Scorers / Datasets / Experiments | `/evaluation` 他 | 出力の自動評価まわり（§8、使い方は [03-handson/day2/11-evals.md](../03-handson/day2/11-evals.md)） |
| **Observability** | Metrics | `/metrics` | メトリクス（集計。**分析用ストアが要る**。§7-6、手順は [03-handson/day2/10-observability.md](../03-handson/day2/10-observability.md) §10-2） |
| | **Traces** | `/traces` | **実行トレース ← 最もよく使う** |
| | Logs | `/logs` | ログ |

**左ナビの項目は登録の有無にかかわらず常に表示されます。** 登録していない要素の画面は空になりますが、
これは正常です（例: `agents` しか登録していないプロジェクトでも Workflows の項目は出る。開くと空）。

---

## 3. Agents — 対話して挙動を見る

`/agents` でエージェント一覧、クリックでチャット画面（`/agents/<エージェントID>/chat`）が開きます。

一覧では **Name / Purpose（instructions の冒頭）/ Provider / Workflows / Agents / Tools の数**が並び、
Tools の数にカーソルを合わせると、そのエージェントが持つツール名と説明がポップアップします。

チャット画面で見るべきものは3つです。

- **ツール呼び出しの折りたたみ** — 応答の吹き出しの中に、呼ばれたツールが順に折りたたみ表示されます。
  開くと**渡された引数と返ってきた値がそのまま**見えます。「エージェントが何を判断して、何を受け取ったか」が読めます
- **右パネル** — モデル ID・instructions・紐づくツール一覧。**コードの変更が反映されているかの確認はここが早い**
- **スレッド** — エージェントに `memory` を付けている場合、会話履歴が保持されます。
  別の条件を試すときは**新しいスレッドを作る**と、前の文脈に引きずられず挙動が読めます

---

## 4. Tools — LLM を介さずツールだけ実行する

`/tools` から、**エージェント（＝LLM）を経由せずにツール単体**を実行できます。
ツールの入力スキーマ（zod）から**フォームが自動生成**されるので、値を入れて実行するだけです。

**なぜ便利か**: エージェントの回答がおかしいとき、原因は「ツールが壊れている」か
「LLM のツールの使い方が悪い」かのどちらかです。ここでツール単体が正しく動くと確認できれば、
後者に絞り込めます。**LLM を呼ばないので API キーを消費しません**。

---

## 5. Workflows — グラフで実行を追う

`/workflows` でワークフローをグラフ表示し、入力を与えて実行できます。ステップが順に進む様子が
グラフ上でハイライトされるため、どこまで進んだか・どこで止まったかが一目で分かります。

### 5-1. 実行する

一覧に並ぶのは **`workflows: {...}` に登録したワークフローだけ**です。サブワークフローは親の中の
ステップとして描かれ、一覧に別枠では出ません。

実行フォームは**ワークフローの `inputSchema`（zod）から自動生成**されます（§4 の Tools と同じ仕組み）。
`inputSchema` が `z.object({ initialQuery: z.string() })` なら、入力欄は `initialQuery` の1つだけです。

### 5-2. suspend で止まったとき

`suspend()` を含むワークフローは**途中で停止して入力待ち**になります。human-in-the-loop の挙動を
確認する場所です。関係するスキーマは2つあります。

| スキーマ | 向き | 画面での役割 |
|---|---|---|
| `suspendSchema` | ワークフロー → 人間 | `suspend({...})` に渡した値が表示される（例: 人間に聞きたい質問文） |
| `resumeSchema` | 人間 → ワークフロー | この形の入力フォームが出る。埋めて resume すると `resumeData` として渡る |

止まったステップの状態は `suspended` になり、**再開されるまで実行（run）が残り続けます**。
resume すると、そのステップの `execute` が**先頭から再実行**され、今度は `resumeData` が入った状態で
走ります。テンプレートで `if (!resumeData) { … suspend() }` という分岐をよく見るのはこのためです。

> ⚠️ **`z.boolean()` の項目を `false` で再開するときは、Form ではなく JSON タブを使います。** 再開パネルの
> 「Your response」の Form では、`z.boolean()` がチェックボックスになります。チェックを外した状態は「`false` を入力した」ではなく
> 「未入力」として扱われるため、`Required` で止まり Resume できません。**`{ } JSON`** タブに切り替え、
> `{ "approved": false }` のように値を直接書いて Resume します（Form が `Required` になるのは Studio 1.67.0 で確認）。

### 5-3. Debug — 1ステップずつ止めて実行する

実行フォームの **Run ボタンの左にある `Debug` トグル**（既定 OFF）は、**ワークフローを1ステップ進むごとに
停止させる「ステップ実行モード」**です。デバッガのステップ実行と同じものだと思ってください。

ON にして Run すると、各ステップが完了した時点でステータスが `success` ではなく **`paused`** になり、
そこで実行が止まります。止まっている間に、グラフ上の**次のステップノードの `⋮`（Step actions）メニュー**へ
2項目が増えます。

| メニュー | 動作 |
|---|---|
| **Run step** | そのステップ**1個だけ**を実行する。ダイアログが開き、**前ステップの出力（＝このステップへの入力）を書き換えてから**流せる |
| **Continue run** | そこから先を**通常実行で流し切る**。実行後は `Debug` トグルも自動的に OFF に戻る |

**Run step の「入力を書き換えられる」点が、この機能の一番の使いどころ**です。前のステップの出力を手で
差し替えれば、LLM の出力が特定の形になったときだけ落ちる後段のステップを、何度も最初から回さずに再現できます。

分岐やループの扱いも通常実行とは変わります。

- `.parallel()` — 並列ブランチを同時に走らせず、**先頭の1本だけ**実行して止まる
- `.branch()` — 条件が真になった**最初の1本だけ**実行して止まる
- サブワークフロー — 中のステップ単位で止まる（ネストしたステップ ID の読み方は §5-4）

**§5-2 の suspend とは別物**です。混同しやすいので対比しておきます。

| | `suspend()` | `Debug` トグル |
|---|---|---|
| 誰が止めるか | **コード**（`suspend()` を書いた箇所だけ） | **Studio**（全ステップの境界で機械的に） |
| 目的 | human-in-the-loop（人間の入力待ち） | 開発時のデバッグ |
| 止まった状態 | `suspended` | `paused` |
| 再開の入力 | `resumeSchema` のフォーム | 前ステップの出力を直接編集 |
| 本番でも起きるか | **起きる**（仕様の一部） | 起きない（Studio の実行時オプション） |

> **⚠️ トグルの状態は保存されません。** 画面をリロードしたり別のワークフローへ移動すると OFF に戻ります。
> Run ボタン左のスライダーアイコンから開く設定パネルにある **`Debug Mode`** も同じ値です（どちらを操作しても同じ）。

> **⚠️ 公式ドキュメントには記載がありません**（2026-08-19 時点）。本節は Studio v1.59.0 の実機と
> 同梱実装（`run.stream({ perStep: true })` が送られ、サーバー側が各ステップ完了時に `paused` へ落とす）で
> 確認した内容です。UI が変わっていたら実機を正としてください。

### 5-4. サブワークフローとループの見え方

`.dountil()` / `.foreach()` などはグラフ上で**ループのブロック**として描かれ、その中に対象のステップ
（サブワークフローなら、そのワークフローそのもの）が入ります。ネストしたステップの ID は
**`<サブワークフロー ID>.<ステップ ID>`** というドット区切りで扱われます。

### 5-5. 実行履歴（Runs）と Traces は別物

一度実行すると `runId` 単位で run が記録され、状態（`running` / `suspended` / `success` / `failed`）を
追えます。**これは Traces とは別の仕組み**です。

| | 何が残るか | 保存先テーブル（§7-3） |
|---|---|---|
| **Runs** | ワークフローの実行状態・suspend 中の再開情報 | `mastra_workflow_snapshot` |
| **Traces** | スパン（所要時間・トークン数・入出力） | `mastra_ai_spans` |

> **⚠️ Runs は見えるのに Traces が空、ということが起こります。** run は `storage` があれば残りますが、
> トレースには **`observability` の設定が別に必要**だからです（§7-1）。

---

## 6. Traces — 実行の中身を読む

`/traces`。**1回でも実行してから**開いてください（実行前は空です）。

**一覧**は Date / Time / Name / Input / Entity（＝どのエージェント・ワークフローの実行か）の表です。

> **⚠️ 既定の絞り込みは「Last 24 hours」です。** 昨日の実行を探して「消えた」と勘違いしやすいので、
> 見つからないときは期間を広げてください。`Add Filter` で条件を足すこともできます。

**1行クリック**で右側に詳細パネルが開きます（URL は `/traces?traceId=...`）。

- **ヘッダ**: Entity Id / Entity Type / Status / **Duration** / Started at / Ended at
- **ツリー**: 実行の入れ子構造。色でカテゴリ（Agent / Model / Tool / Memory / Other）が分かれ、
  各行の右に**所要時間のバー**が出ます。典型的にはこう並びます。

  ```
  agent run: '<エージェントID>'                12.454 s
  ├ input processor: message-history            0.001 s
  │ └ memory: recall                            0.001 s
  └ llm: '<モデルID>'                          12.446 s
    ├ step: 0
    │ └ inference: 0
    │   ├ chunk: 'reasoning'
    │   ├ chunk: 'tool-call'
    │   └ chunk: 'tool-result'
    │ └ tool: '<ツール名>'
    └ step: 1 …（ツールの結果を受けて2回目の推論）
  ```

- **トークン使用量**が各スパンに記録され、内部スパンの分は親スパンへ積み上げられます。
  ＝**実際の課金量**なので、`lastMessages` を減らす・ツールを増やすといった変更の影響をここで数字で確認できます
- **Evaluate Trace / Save as Dataset Item** — 記録済みのトレースを後から採点したり、評価用データセットに追加できます

チャット画面の折りたたみでも「何が呼ばれたか」は見えますが、**所要時間・トークン数・LLM に実際に送られた
入力の全文**はこの画面にしかありません。

---

## 7. トレースと会話履歴はどこに保存されるか

**Ctrl+C で dev サーバーを止めても、再起動するとトレースが残っている。** これは正常な挙動で、
トレースがメモリではなく**ストレージ（既定では SQLite ファイル）に永続化されている**ためです。

### 7-1. 仕組み

保存には**2つの設定が揃っている必要**があります。

```ts
// src/mastra/index.ts
export const mastra = new Mastra({
  agents: { myAgent },

  // ① 保存先
  storage: new LibSQLStore({ id: 'mastra-storage', url: 'file:./mastra.db' }),

  // ② トレースを ① へ書き出すエクスポーター
  observability: new Observability({
    configs: {
      default: {
        serviceName: 'my-app',
        exporters: [new MastraStorageExporter()],
      },
    },
  }),
})
```

| 設定 | トレースはどうなるか |
|---|---|
| **`observability:` を渡していない** | **そもそも記録されない。** 実行しても Traces 画面は空のまま（②が無いので書き出す先が無い） |
| `observability` はあるが `storage` が**無い** | **セッション中は見えるが再起動で消える。** Mastra が既定のインメモリストアにフォールバックするため（起動ログに `No 'storage' configured on Mastra — falling back to an in-memory store.` が出る） |
| `storage` はあるが observability ドメインを持たない | **残らない**。`MastraStorageExporter disabled: Observability storage not available. …` が出る |
| `url: ':memory:'` | セッション中は見えるが**再起動で消える**（プロセスと運命を共にする） |
| `url: 'file:...'` | **残る** ← Ctrl+C しても消えないのはこれ |

> **⚠️ 1行目だけ症状が決定的に違います。** `observability` 自体が無い場合は「再起動で消える」のではなく
> **最初から1件も出ません**。`package.json` の依存に `@mastra/observability` があっても、
> `new Mastra({...})` に渡していなければ機能しません（実例は [02-templates/05-deep-search.md](../02-templates/05-deep-search.md)）。
> **Traces 画面が空のときは、まず `observability:` を渡しているかを確認**してください。
> 渡したうえで残らない場合は、起動ログに `Traces will not be persisted.` が無いかを見ます。

> 本ハンズオンでは、[03-handson/day1/02-first-agent.md](../03-handson/day1/02-first-agent.md) Step 2 で
> `observability` を入れるのでトレース自体は Day1 から出ます。ただし `storage` はまだ無いため
> インメモリ止まりで、dev サーバーを止めると消えます。
> [03-handson/day2/07-memory.md](../03-handson/day2/07-memory.md) Step 7 で `LibSQLStore` を入れて
> 初めてファイルに残るようになります。「Day1 では消えたのに Day2 では残る」のはこの差です。

### 7-2. 実ファイルはどこにあるか

`url` の相対パスは、コードを書いたファイルの場所ではなく、**そのとき動いているプロセスのカレント
ディレクトリ（cwd）から**解決されます。`mastra dev` は `.mastra/output/index.mjs` を子プロセスとして
起動する際に **cwd を `src/mastra/public/` に設定する**ため（フォルダが無ければ CLI が作る）、
DB は `src/mastra/public/` の下に作られます（`mastra.db` 本体に加え、SQLite の `-wal` / `-shm` を伴う）。

```bash
# 実際の場所を探す
find . -name '*.db' -not -path './node_modules/*'
```

> **⚠️ 起動の仕方を変えると DB が2つできます。** `node --env-file=.env src/index.ts` のように
> スクリプトから `mastra` を直接使うと、cwd は**コマンドを打ったディレクトリ**（通常はプロジェクト直下）
> なので、同じ `file:./mastra.db` でもプロジェクト直下に別の `mastra.db` ができます。
>
> | 起動方法 | プロセスの cwd | できる DB |
> |---|---|---|
> | `npm run dev`（= `mastra dev`） | `src/mastra/public/` | `src/mastra/public/mastra.db` |
> | `node --env-file=.env src/index.ts` | コマンドを打ったディレクトリ | `./mastra.db` |
>
> 2つのファイルは中身を共有しないため、**スクリプトで溜めた会話履歴やトレースは Studio に出ず、
> その逆も同様**です。設定ミスではなく相対パスの仕様どおりの動作です。1つにまとめたい場合は、
> どちらの起動方法でも同じ場所を指す**絶対パス**を `url` に渡します（`import.meta.url` からの組み立ては、
> `mastra dev` がコードを `.mastra/output/` にバンドルして基準が変わるため使えません）。
> 挙動は `mastra` CLI 1.30.0 で確認しました。

`.mastra/output/preflight-local-paths.json` には `url` に書いた値（`file:./mastra.db`）と定義元ファイルが
記録されますが、**解決後の絶対パスではありません**。場所の確認は上の `find` で行ってください。

### 7-3. 何がどのテーブルに入るか

| テーブル | 内容 |
|---|---|
| `mastra_ai_spans` | **トレース／スパン本体**（Traces 画面が読んでいるのはここ。Metrics 用に DuckDB を足した場合は DuckDB 側に入り、ここは空。§7-6） |
| `mastra_threads` / `mastra_messages` | **会話履歴**（Memory）。中身は下の「会話履歴の中身」 |
| `mastra_resources` | resource 単位の永続データ（working memory など）。**使っていなければ空** |
| `mastra_workflow_snapshot` | ワークフローの実行状態（suspend 中の再開情報を含む） |
| `mastra_agents` / `mastra_agent_versions` | **Studio 上で作ったエージェント**の定義。コードで定義したものは入らない（§7-4） |
| `mastra_scorers` | **採点結果**（点数・理由・入出力。ライブ採点・トレース採点・Experiment のすべて）。保存には Scorer の `Mastra` への登録が要る（§8-1） |
| `mastra_datasets` / `mastra_dataset_items` / `mastra_dataset_versions` | **評価用データセット**とその項目、バージョン履歴（§8-3） |
| `mastra_experiments` / `mastra_experiment_results` | **Experiment** の実行記録と、項目ごとの結果（§8-4） |

上記以外にも `mastra_` で始まるテーブルが数十個でき（Datasets / Scorers / Skills など）、使っていない機能の
テーブルは**行数 0 のまま**です。空なのは異常ではありません。

#### 会話履歴の中身

`mastra_threads` は 1 行 = 1 スレッドで、主な列は `id`（thread ID）/ `resourceId` / `title` / `createdAt` /
`updatedAt` です。`mastra_messages` は 1 行 = 1 メッセージで、`thread_id` でスレッドに紐づきます
（主な列: `id` / `thread_id` / `role` / `type` / `content` / `createdAt` / `resourceId`）。

**本体は `content` 列**で、JSON 文字列として入っています。

```json
{
  "format": 2,
  "parts": [ { "type": "text", "text": "…" }, … ],
  "content": "…（テキストだけを連結したもの）",
  "metadata": { "modelId": "gpt-5.6-luna", "provider": "openai", "traceId": "…" }
}
```

| `parts[].type` | 入っているもの |
|---|---|
| `text` | 発言のテキスト |
| `reasoning` | 推論モデルの思考（要約が空でも、暗号化された思考の本体が `providerMetadata` に入る） |
| `tool-invocation` | ツール呼び出し。**`toolName` / `args`（渡した引数）/ `result`（戻り値）** がそのまま入る |
| `step-start` | ツール結果を受けた次の推論ステップの区切り |

- **ツールの引数と戻り値まで会話履歴に保存される**ため、DB を見るだけで「何を渡して何が返ったか」を追えます
- アシスタントの `metadata.traceId` は `mastra_ai_spans.traceId` と同じ値です。**メッセージから、その返答を
  生んだ実行トレースへ辿れます**（Traces 画面は `/traces?traceId=<値>` で直接開ける、§6）
- `lastMessages` は **LLM へ渡す件数**であり、**保存件数ではありません**。DB には全件が残ります

> **⚠️ 起動方法で `role` / `type` の値が変わります**（2026-09-21、Studio v1.59.0 / `mastra` CLI 1.30.0 で確認）。
> Studio のチャットから送ったユーザー発言は **`role = signal` / `type = user`** で保存され、`content` に
> `"content"` 文字列は無く、`metadata` には `traceId` ではなく `signal`（送信元の ID など）が入ります。
> `agent.generate()` をコードから呼んだ場合は **`role = user` / `type = v2`** です。アシスタント側は
> どちらも `role = assistant` / `type = v2` で `metadata.traceId` を持ちます。
> `role = 'user'` だけで絞り込むと Studio の発言が抜けるので注意してください。

> **補足**: `create-mastra` の最新スキャフォールドは `MastraCompositeStore` を使い、observability ドメイン
> だけ別のストア（DuckDB）に割り当てます。つまり保存先は「`storage` 全体」ではなく
> **ドメインごとの割り当て**で決まることがあります。`storage` の中身を確認してから探してください。

### 7-4. Agent を編集するとトレースは消えるか → **消えません**

スパンはエージェントを **`entityId`（文字列）で参照しているだけ**で、エージェントの定義とは独立した行として
残ります。実際のルートスパンはこうなっています。

```json
{ "name": "agent run: '<エージェントID>'", "spanType": "agent_run",
  "entityType": "agent", "entityId": "<エージェントID>", "entityName": "<表示名>" }
```

コードで定義したエージェントは**そもそも DB に登録されません**（`mastra_agents` / `mastra_agent_versions`
は空のまま。ここに行が入るのは Studio 上で作ったエージェントです）。したがって instructions・モデル・
ツールをいくら書き換えて再起動しても、過去のトレースは削除されません。

> **⚠️ 例外は「エージェントの id そのものを変えた」場合。** 古いトレースは旧 id に紐づいたまま残るため、
> Traces 画面で Entity 別に絞り込むと**見えなくなります**（消えたのではなく、参照先が変わっただけ）。

### 7-5. 消したいとき

```bash
# dev サーバーを止めてから実行する
rm -f src/mastra/public/mastra.db src/mastra/public/mastra.db-wal src/mastra/public/mastra.db-shm
```

次回起動時に空の DB が作り直されます。

> **⚠️ 会話履歴も同じ DB に同居しています。** ファイルごと消すとスレッド（Memory）も消えます。
> トレースだけ消したい場合は `mastra_ai_spans` を DELETE してください。
> また、アプリが業務データ用に持っている別の DB（seed で作られるサンプルデータ等）とは別物です。混同しないこと。

### 7-6. Metrics を使うと、観測データの保存先が変わる

**Metrics 画面（`/metrics`）は、LibSQL だけでは動きません。** 集計に向いた分析用（OLAP）ストアが別に要るためです
（公式: LibSQL・PostgreSQL などは Metrics 非対応）。ローカルでは **DuckDB** を足し、`MastraCompositeStore` で
**観測ドメインだけ** DuckDB に振り分けます（手順は [03-handson/day2/10-observability.md](../03-handson/day2/10-observability.md) §10-2）。

| データ | 振り分け前 | 振り分け後 |
|---|---|---|
| トレース（スパン） | LibSQL の `mastra_ai_spans` | **DuckDB の `span_events`**（LibSQL の `mastra_ai_spans` は空のまま） |
| メトリクス / ログ / 点数 / Feedback | （Metrics は保存不可） | DuckDB の `metric_events` / `log_events` / `score_events` / `feedback_events` |
| 会話履歴・ワークフローの状態・Dataset・Experiment | LibSQL | **LibSQL のまま**（`mastra_scorers` にも点数が入る） |

- DuckDB のファイルは、`LibSQLStore` と同じ理由（§7-2）で `mastra dev` なら `src/mastra/public/mastra.duckdb`（と `.wal`）にできます
- **切り替える前のトレースは Traces に出なくなります。** 保存先が変わっただけで、LibSQL 側には残っています
- DuckDB は開発用です。ファイルが残らない環境では動かず、本番は ClickHouse などを使います
- 挙動は `@mastra/duckdb` 1.12.1 / `mastra` CLI 1.30.0 で確認しました（`mastra dev` 起動 → エージェント実行 → `GET /api/observability/metrics` で
  メトリクスが返ること、`/metrics/aggregate` が推定コストを返すこと、テーブルの件数）。Metrics 画面の見た目までは実機で確かめていません

---

## 8. Evaluation — 採点・データセット・実験

左ナビの **Evaluation** には、出力を採点する **Scorers**、評価用の質問集 **Datasets**、質問集をエージェントに流して
採点する **Experiments** があります。Scorer の書き方と実験の進め方は
[03-handson/day2/11-evals.md](../03-handson/day2/11-evals.md) が手順書です。ここでは画面と仕様だけをまとめます。

### 8-1. Scorers — 点数と理由を読む

| 画面 | 見えるもの |
|---|---|
| Scorers 一覧 | **`new Mastra({ scorers })` に登録した** Scorer と、エージェントに付けた Scorer（エージェントに付けたものは自動で登録される）。説明と、付いているエージェント・ワークフローの数 |
| Scorer を選ぶ | その Scorer が出した点数の一覧（ページ送りあり） |
| 行を開く | 点数、**理由**（`generateReason` の文章）、入力、出力、judge に送られたプロンプト。ここから **Save as Dataset Item** で評価用データセットへ保存できる |

- **採点は応答のあとに非同期で走ります。** エージェントの応答は遅くなりませんが、点数が画面に出るまでに数十秒かかることがあります
  （LLM に判定させる Scorer は judge の呼び出しが終わるまで出ない）
- **点数が保存されるには `storage` が必要**で、Scorer が `Mastra` に登録されている必要があります。登録が無いと保存に失敗します
- ライブ採点では、エージェントに付けた Scorer が `sampling`（`{ type: 'ratio', rate: 0.1 }` なら10%）の割合で動きます。
  どの実行を採点するかは**トレース ID から決まる**ので、同じ割合の Scorer は同じ実行を採点します
- judge の LLM 呼び出しも1回の実行として記録されるため、**Traces に `scorer run: '<Scorer ID>'` というトレースが混ざります**
  （`agent run:` がエージェント本体）。トークンを使うので、評価のコストはここで確かめられます

### 8-2. 過去のトレースを採点する

Traces の詳細画面から **Evaluate Trace** で Scorer を選ぶと、**記録済みのトレースを後から採点**できます
（実行すると「Scorer triggered」と出て、結果は 8-1 の画面に溜まる）。選べるのは**登録済みの Scorer**だけです。
エージェントのトレースには `type: 'agent'` の Scorer が候補に出ます。

### 8-3. Datasets — 評価用の質問集

| 操作 | 内容 |
|---|---|
| **Create Dataset** | 名前・説明・（任意で）`input` と `groundTruth` のスキーマを決めて作る |
| **Add Item** | 項目を追加する。`input` と `groundTruth` は **JSON 欄**なので、文字列は `"` で囲む。CSV / JSON のインポートもできる |
| **Versions** / **Compare Versions** | 項目の追加・変更・削除のたびに**バージョンが上がり**、履歴を見られる。2つのバージョンの差分を比べられる |

- Experiment を **Dataset のバージョンに固定**して回せるので、「質問集を変えたせいで点数が変わった」のか
  「エージェントを変えたせいか」を切り分けられます
- エージェント・ワークフローを対象にする Experiment の `input` は、エージェントでは **文字列 / 文字列の配列 / メッセージの配列**
  でなければなりません（`agent.generate()` にそのまま渡されるため）

> **⚠️ スクリプト（`node ...`）で作った Dataset は Studio に出ません。** `mastra.datasets.create()` を
> コマンドで動かすと、DB はコマンドを打ったディレクトリ側にできます。`npm run dev` が読む DB とは別のファイルなので、
> Studio からは空に見えます（§7-2 と同じ理由）。Studio で見たいデータは Studio（または dev サーバーの HTTP API）から作ります。

### 8-4. Experiments — 質問集を流して採点する

Dataset の画面で **Run Experiment** を選び、ターゲット（登録済みのエージェント / ワークフロー / Scorer）、
採点に使う Scorer などを選んで実行します。

- **サーバー側で非同期に実行**され、Experiments タブには状態（pending → running → completed / failed）と件数が出ます。
  完了した実験を開くと、**質問ごとの出力・各 Scorer の点数と理由・トレース**が見えます
- **Compare** で2つ以上の実験を選ぶと、点数と結果を横並びで比べられます（プロンプトやモデルを変えた前後の比較に使う）
- エージェントが Memory を持っていても、**リソース ID を渡さなければ記憶なしで実行**されます（質問ごとに独立して評価できる）。
  リソース ID を渡すと、そのリソースの過去の会話や作業メモリが実験に影響します
- ツールの戻り値を固定したいときは、Dataset の項目に**ツールのモック**を付けます（外部 API に依存する回答の揺れを抑えられる）

---

## 9. つまずきやすい点まとめ

| 症状 | 見るところ |
|---|---|
| ブラウザで開いても繋がらない | 起動ログに `│ Studio: http://localhost:4111` が出たか（§1） |
| 変更したのに反映されない | ポートが 4112 に逃げていないか（§1、[troubleshooting/ §1](../troubleshooting/)） |
| Traces が空 | 実行したか／**`observability:` を渡したか**／`storage` を設定したか（§7-1）／期間フィルタが Last 24 hours（§6） |
| ワークフローは実行できたのに Traces が空 | Runs と Traces は別物。`observability:` の設定漏れを疑う（§5-5、§7-1） |
| Workflows タブが空 | `workflows: {...}` を登録していない。登録していなければ空が正常（§2） |
| サブワークフローが Workflows 一覧に出ない | 仕様。親のグラフの中にステップとして描かれる（§5-4） |
| suspend したまま先に進まない | `resumeSchema` の入力を埋めて resume する（§5-2） |
| 1ステップ進むごとに止まって最後まで走らない | `Debug` トグルが ON。ステップの `⋮` → `Continue run` で流し切る（§5-3） |
| 止めたのに前のトレースが残る | 仕様。`file:` で永続化されている（§7-1） |
| エージェントを直したらトレースが消えた | id を変えていないか（§7-4） |
| 点数がいつまでも出ない | 採点は応答のあとに非同期で走る（数十秒かかる）。出ないままなら、Scorer を `Mastra` に登録したか・`storage` があるか（§8-1） |
| スクリプトで作った Dataset が Studio に無い | DB が別ファイル。スクリプトは `./mastra.db`、`npm run dev` は `src/mastra/public/mastra.db`（§8-3、§7-2） |
| Traces に `scorer run:` が増えた | judge の LLM 呼び出し。仕様（§8-1） |
| Metrics 画面が空・エラー | LibSQL だけでは Metrics 非対応。DuckDB を足して `MastraCompositeStore` で観測ドメインを振り分けたか（§7-6） |
| DuckDB を足したら、前のトレースが Traces から消えた | 保存先が変わったため。LibSQL 側には残っている。切り替え後の実行分から見える（§7-6） |

---

## 参考リンク

- Studio 公式ドキュメント: https://mastra.ai/docs/studio/overview
- Observability 概要: https://mastra.ai/docs/observability/overview
- Storage 概要: https://mastra.ai/docs/storage

---

全体像: [README.md](../README.md)　関連: [00-overview/08-devops.md](../00-overview/08-devops.md) §8-1　困ったとき: [troubleshooting/](../troubleshooting/)
