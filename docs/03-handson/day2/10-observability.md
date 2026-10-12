# Step 10. 観測とトレース（90分）

エージェントは「なぜその答えになったか」が見えにくい仕組みです。この Step では、**1回の実行を細かく読む**（トレース）、
**全体を数字で眺める**（メトリクス）、そして Mastra の観測・評価機能が**それぞれ何を答える道具か**を整理します。
各機能の位置づけ・用語は [00-overview/08-devops.md](../../00-overview/08-devops.md) §8-3 が正典です。

| 節 | 内容 | 時間 |
|---|---|---|
| 10-1 | トレースを読み込む（1回の実行の中身） | 30分 |
| 10-2 | メトリクスで全体を数字で見る（回数・速度・トークン・推定コスト） | 30分 |
| 10-3 | 観測・評価機能の全体像と使い分け | 30分 |

## 10-1. トレースを読み込む（30分）

Studio のトレース画面を、今度は「デバッグの道具」として使います。
画面の読み方（ツリーの構造、期間フィルタ、保存先のテーブル）は
[reference/mastra-studio.md](../../reference/mastra-studio.md) §6・§7 を参照してください。

確認すること:

- LLM への入力プロンプト全文（instructions + 会話履歴 + ツール定義が結合された結果）
- 各ステップのトークン消費
- ツール呼び出しの引数と戻り値
- レイテンシの内訳

**やってみる実験**

1. `lastMessages: 20` を `lastMessages: 3` に変えて、入力トークン数がどう変わるか比較
2. ツールを2つに増やして、プロンプトサイズの増加を確認
3. 「エージェントに渡されるコンテキストは自分が思っているより大きい」ことを実感する

> **コスト感覚の獲得**: ここで見たトークン数と [01-setup/02-llm.md](../../01-setup/02-llm.md) の料金表を突き合わせると、「このエージェントを1万回叩いたらいくらか」が概算できます。社内提案では必ず聞かれる観点です。

トレースは**1回の実行の顕微鏡**です。「この質問の回答がおかしい」の原因を探すのには強い一方、
「全体で何回動いて、いくらかかったか」「遅いのは普段からか」は、1件ずつ開いても分かりません。
次の 10-2 で、それを集計して見られるようにします。

## 10-2. メトリクスで全体を数字で見る（30分）

**メトリクス**は、実行の記録を**集計した数字**（回数・処理時間・トークン数・推定コスト）です。
トレースとの違いを先に押さえます。

| | トレース | メトリクス |
|---|---|---|
| 見るもの | **1回**の実行の中身（入れ子の時系列） | **全体**の傾向（合計・平均・パーセンタイル・時間推移） |
| 答える問い | 「この回答はなぜこうなった？」 | 「今日は何回動いた？ 遅くなっていない？ いくらかかった？」 |
| 画面 | Traces | Metrics |

### 準備: 保存先に分析用のデータベースを足す（10分）

**メトリクスは、Step 7 で入れた LibSQL（SQLite）だけでは保存できません。** 集計に向いた分析用（OLAP: Online Analytical Processing）のストアが別に要るためです。
ローカル開発では、ファイル1つで動く **DuckDB** を使います。`memory` や `workflows` は今までどおり LibSQL のまま、
**観測データだけを DuckDB に振り分ける**書き方（複合ストレージ）にします。

<details>
<summary>💡 OLAP とは何か、なぜ LibSQL では足りないのか（クリックで開く）</summary>

**OLAP**（Online Analytical Processing／オンライン分析処理）は、**大量の行をまとめて集計する**ことに向いた
データベースの作りを指します。対になる言葉が **OLTP**（Online Transaction Processing／オンライン取引処理）で、
こちらは**1件ずつ確実に読み書きする**ための作りです。

| | OLTP（オンライン取引処理） | OLAP（オンライン分析処理） |
|---|---|---|
| 得意な操作 | 1件の読み書き（会話履歴を1件追記する、ID で1件引く） | 大量の行の集計（10万件のトークン数を合計する、p95 を出す） |
| 典型的なクエリ | `INSERT` / `SELECT * WHERE id = …` | `SELECT sum(tokens) … GROUP BY agent, hour` |
| データの持ち方 | **行指向**（1行ぶんを隣り合わせに置く） | **列指向**（同じ列の値を隣り合わせに置く） |
| 製品の例 | SQLite / **LibSQL**、PostgreSQL、MySQL | **DuckDB**、ClickHouse、BigQuery |

**なぜ列指向だと集計が速いのか**: 「トークン数の合計」を出すとき、行指向のデータベースは全行を丸ごと読んでから
必要な1列を取り出します。列指向ならその**1列だけ**を読めばよく、同じ種類の値が並ぶので圧縮も効きます。
合計や平均のような計算も列単位でまとめて処理する作りになっているため、集計が桁違いに速くなります。

この節に当てはめると、**LibSQL（SQLite 系）は OLTP** です。会話履歴やワークフローの状態のように
「1件書いて1件読む」用途には向く一方、メトリクスの集計には向かないため、Mastra もメトリクスの保存先としては
対応していません。そこで**観測データだけを OLAP の DuckDB に振り分ける**のが、この節の複合ストレージです。
DuckDB は「分析用の SQLite」という位置づけ（プロセス内で動き、ファイル1つで済む）なので、ローカル開発に合います。
あとで本番向けとして出てくる **ClickHouse も OLAP** のデータベースです。

</details>

```bash
npm install @mastra/duckdb
```

```ts
// src/mastra/index.ts（storage の部分だけ差し替える）
import { MastraCompositeStore } from '@mastra/core/storage'
import { LibSQLStore } from '@mastra/libsql'
import { DuckDBStore } from '@mastra/duckdb'

export const mastra = new Mastra({
  // agents / workflows / observability などは今のまま
  storage: new MastraCompositeStore({
    id: 'composite-storage',
    // 会話履歴・ワークフローの状態・点数などは、これまでどおり LibSQL
    default: new LibSQLStore({ id: 'mastra-storage', url: 'file:./mastra.db' }),
    // トレース・メトリクス・ログだけ DuckDB へ
    domains: { observability: new DuckDBStore().observability },
  }),
})
```

| 書き方 | 意味 |
|---|---|
| `MastraCompositeStore` | データの種類（ドメイン）ごとに保存先を振り分ける。`default` が基本の保存先 |
| `domains: { observability: … }` | **観測データ（トレース・メトリクス・ログ）だけ**の保存先を上書きする |
| `new DuckDBStore().observability` | DuckDB の観測用ストア。DuckDB は観測ドメインしか持たないので、他のドメインは `default` が担当する |

`npm run dev` を再起動し、エージェントと数往復します（例: 「来週、福岡に行きます。持ち物を教えて」「札幌の天気を教えて」）。

> ⚠️ **切り替えると、これまでのトレースは Traces に出なくなります。** トレースの保存先が LibSQL から DuckDB に
> 移るためで、消えたわけではありません（LibSQL 側のファイルに残っています）。切り替えたあとに実行した分から見えます。
>
> ⚠️ **DuckDB のファイル（`mastra.duckdb` と `.wal`）が `src/mastra/public/` にできます。** コミットしないよう、
> リポジトリの [.gitignore](../../../.gitignore) には `*.duckdb` を入れてあります（保存先の理屈は
> [reference/mastra-studio.md](../../reference/mastra-studio.md) §7-2 と同じです）。
>
> 💡 DuckDB は**開発用**です（プロセス内で動き、ファイル1つに書く）。ファイルが残らない環境（サーバーレスなど）では動かず、
> 本番の観測データは ClickHouse などの分析用 DB、またはマネージドのサービスへ送ります。

### 動かして確かめる（10分）

Studio の左ナビ **Observability → Metrics**（`/metrics`）を開き、期間を **Last 24 hours** にします。
`Observability` はグループ見出しなので、その配下の `Metrics` を選んでください。

> ⚠️ **別グループの `Evaluation → Overview`（`/evaluation`）と間違えやすい**ので注意。こちらは Scorer の採点結果の画面で、
> [Step 11](./11-evals.md) をやるまでは空のままです。

公式のドキュメントによると、画面は次のカードで構成されます。

| カード | 読み取れること |
|---|---|
| 上段のサマリー | 実行回数・**モデルのコスト**・トークン消費・Scorer の平均点 |
| モデルごとの利用量とコスト | どのモデルにいくら使ったか |
| エージェントごとのトークン | **どのエージェントが重いか**（サブエージェントを分けた Step 9 以降は、内訳が分かれる） |
| トークンの時間推移と推定コスト | 急に増えた時間帯の特定 |
| トレース件数（完了 / エラー） | エラー率 |
| レイテンシ（p50 / p95） | 普段の速さと、遅い方の1割の速さ |

> 💡 **p50 と p95**: p50 は「真ん中の速さ」、p95 は「遅い方から数えて5%目の速さ」です。平均だけだと、
> たまに起きる遅い実行が見えません。「普段は2秒だが、20回に1回は15秒かかる」といった状態は p95 に出ます。

**コストの出し方**: 上段の合計コストを実行回数で割ると、**1回あたりの概算コスト**になります。
**これは推定値**です（モデルごとの単価をトークン数に掛けて見積もったもの）。実際の請求は、
[01-setup/02-llm.md](../../01-setup/02-llm.md) の料金表と、OpenAI の利用状況ページで突き合わせてください。

> 💡 画面ではなくコードや HTTP API からも集計を取れます。サーバーを起動した状態で、次のようにすると、
> 期間内の入力トークンの合計と推定コストが返ります。
>
> ```bash
> curl -s -X POST http://localhost:4111/api/observability/metrics/aggregate \
>   -H 'content-type: application/json' \
>   -d '{"name":["mastra_model_total_input_tokens"],"aggregation":"sum"}'
> # → {"value":7669,"estimatedCost":0.0084305,"costUnit":"USD"}（数値は環境で変わる）
> ```
>
> 取れるメトリクスの名前は `mastra_agent_duration_ms`（エージェントの処理時間）、`mastra_model_duration_ms`（LLM 呼び出しの時間）、
> `mastra_tool_duration_ms`（ツールの時間）、`mastra_model_total_input_tokens` / `mastra_model_total_output_tokens`（トークン）などです。
> 全一覧は `GET /api/observability/discovery/metric-names` で確認できます。

### 実験（10分）

10-1 の実験を、今度は**メトリクスで**確認します。

1. `lastMessages` を `3` にして同じ質問を5往復し、Metrics のトークン数（上段または時間推移）の伸び方を、`20` のときと比べる
2. 実行回数を横に置いて、**1回あたりのコスト**を手計算する
3. Step 9 の `packingAgent` 経由の質問と、天気だけを聞く質問で、エージェントごとのトークンがどう違うか見る

> 💡 Traces の一覧でも、メトリクスに対応した保存先なら、`Columns` から**入力トークン・出力トークン・推定コスト**の列を出せます
> （普通の LibSQL だけの構成ではこの列は出ません）。1件ずつのコストを並べて見たいときに便利です。

## 10-3. 観測・評価機能の全体像と使い分け（30分）

Mastra の観測・評価まわりには、名前の似た機能がいくつかあります。まず**何を答える道具か**で並べます。

| 機能 | 答える問い | 単位 | 見る場所（Studio） | このハンズオン |
|---|---|---|---|---|
| **Traces** | この回答は**なぜ**こうなった？ | 1回の実行（入れ子） | Traces | 10-1 |
| **Metrics** | 全体で**何回・どれくらい速く・いくら**？ | 集計 | Metrics | 10-2 |
| **Logs** | Mastra や自分のコードは**何を出力した**？ | 1行ずつ（トレースと相関） | Logs | 紹介のみ |
| **Feedback** | **人**はこの回答をどう評価した？ | 1件の評価（👍・点数・コメント） | トレースに紐づく | 紹介のみ |
| **Scorers（Evals）** | 出力は**基準を満たした**か？ | 1回の出力 → 0〜1 の点数 | Scorers | [Step 11](./11-evals.md) |
| **Datasets / Experiments** | **変えたら良くなった**？ | 質問集 × 実行 → 前後比較 | Datasets / Experiments | [Step 11](./11-evals.md) |

### 1回の実行から、何がどこに残るか

```
ユーザーの質問
   │
   ▼
エージェントの実行 ──→ Traces   : 1回分の入れ子のスパン（プロンプト全文・ツール引数・所要時間・トークン）
   │                    │
   │                    ├──→ Metrics  : スパンから作られる数値（処理時間・トークン・推定コスト）
   │                    ├──→ Logs     : 実行中に出た内部ログ（トレース ID で結びつく）
   │                    └──→ Feedback : あとから人が付ける評価（トレースやスパンに紐づく）
   ▼
応答 ──→ Scorers : 応答を採点した点数と理由（応答のあと裏で動く）
              └──→ Dataset の項目に保存 → Experiment で前後比較
```

### 機能ごとの補足

**Traces** — **すべての出発点**です。メトリクス・ログ・点数も、トレース ID でこの1回の実行に結びつきます。
OpenTelemetry や Datadog などの外部サービスへ送ることもできます（[00-overview/08-devops.md](../../00-overview/08-devops.md) §8-3）。

**Metrics** — 異常に**気づく**ための道具です。「コストが急に増えた」「遅い実行が増えた」を Metrics で見つけ、
原因の個別の実行を Traces で開く、という順番で使います。保存には分析用のデータベースが要ります（10-2）。

**Logs** — Mastra 内部のログと、アプリが出したログを、検索・絞り込みできます。ログがトレースと相関している場合は、
そのままトレースとスパンの時系列に移動できます。観測の設定をすると転送は**既定で有効**になります。
（[公式: Logging](https://mastra.ai/docs/observability/logging)）

**Feedback** — 👍👎・5段階評価・コメントなど、**人が付けた評価**を、トレースやスパンに紐づけて保存します。
Scorers が機械の採点なのに対し、Feedback は人の採点です。コードからは `mastra.observability.addFeedback()` で付けます。
ユーザーが見た回答のトレース ID は、会話の記録（メッセージの `content.metadata.traceId`）から取り出せます。
**社内のチャット UI に「役に立った / 立たなかった」ボタンを付ける**ときの受け皿になります
（[公式: Feedback](https://mastra.ai/docs/observability/feedback)）。

**Scorers / Datasets / Experiments** — **品質の判定**と**変更前後の比較**です。Traces や Metrics は「何が起きたか」を見せますが、
「それは良い結果か」は教えてくれません。それを数値にするのが Step 11 です。

### 使い分けの練習

次の場面で、**最初に開く画面**はどれでしょうか。考えてから、答えを開いてください。

1. 今日の請求が急に増えた。どのエージェントが原因か
2. ある会話の回答がおかしい。なぜか
3. プロンプトを直した。良くなったか
4. 「動作が遅い」という声がある。全体の傾向と、遅い1件の内訳を知りたい
5. 本番の回答に付いた「役に立たなかった」を、あとで分析したい
6. 社内の監視基盤（Datadog など）にも観測データを出したい

<details>
<summary>答え（クリックで開く）</summary>

1. **Metrics**（エージェントごとのトークンとコスト）→ 増えたエージェントの時間帯を **Traces** で開く
2. **Traces**（LLM に実際に送られた入力全文、ツールの引数と戻り値）
3. **Experiments**（同じ Dataset で前後を比べる。[Step 11](./11-evals.md)）
4. 全体の傾向は **Metrics** の p95、遅い1件の内訳は **Traces** の所要時間のバー
5. **Feedback**（トレースに紐づけて保存する。数値の評価は集計や時間推移のグラフにもできる）
6. **Traces** の外部送信（OpenTelemetry / Datadog などのエクスポーター）

</details>

### 公式ドキュメント

| 機能 | 内容 | URL |
|---|---|---|
| Traces | 実行トレースの記録・可視化 | https://mastra.ai/docs/observability/overview |
| Metrics | メトリクス収集 | https://mastra.ai/docs/observability/metrics/overview |
| Logs | ログの転送と検索 | https://mastra.ai/docs/observability/logging |
| Feedback | 人の評価をトレースに紐づける | https://mastra.ai/docs/observability/feedback |
| Evals | エージェント出力の自動評価（**次の [Step 11](./11-evals.md) で実際に使います**） | https://mastra.ai/docs/evals/overview |
| Datasets | 評価用データセット管理（同上） | https://mastra.ai/docs/datasets/overview |

**チェックポイント**
- [ ] トレースからプロンプト全文とトークン数を読み取れた
- [ ] 設定変更がトークン消費に与える影響を確認した
- [ ] 保存先を DuckDB を足した複合ストレージにし、Metrics 画面に数字が出た
- [ ] 総コストと実行回数から、1回あたりの概算コストを計算した
- [ ] トレースとメトリクスの違い（1回の中身 / 全体の傾向）を説明できる
- [ ] 6つの機能（Traces / Metrics / Logs / Feedback / Scorers / Datasets・Experiments）が答える問いの違いを説明できる
