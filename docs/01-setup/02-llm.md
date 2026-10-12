# 02. LLM の準備

ハンズオンで使う LLM を決めて、API キーを取得します。所要時間 30 分。

---

## 1. 結論：OpenAI（GPT-5.6 Luna）を使う

**推奨構成**

| 項目 | 値 |
|---|---|
| プロバイダ | OpenAI API |
| キー取得元 | OpenAI Platform → https://platform.openai.com/api-keys |
| 環境変数 | `OPENAI_API_KEY` |
| モデル指定 | `model: 'openai/gpt-5.6-luna'` |
| 費用 | **有料**（無料枠なし。ハンズオン全体で数十円程度。少額のクレジットを事前にチャージする） |

理由は3つ。

1. **出力形式を強制しやすい。** LLM アプリ開発では、構造化出力（Zod スキーマで指定した形の JSON を返させる）が土台になります。Gemini は JSON Schema のサブセットしか受け付けず、`z.union` や `z.record` が使えないなど、スキーマの書き方に制約が出ます（詳細は 6章）。本ハンズオンのスキーマは基本形だけで組んであり、OpenAI ではそのまま通ります。
2. **本編とテンプレート編でキーが1本で済む。** [02-templates/](../02-templates/) の公式テンプレートは作者がモデルを所有しており、いずれも OpenAI 前提で書かれています。本編も OpenAI にそろえたので、`OPENAI_API_KEY` だけで両方を通せます。
3. **Mastra の model router が対応している。** `openai/...` と書くだけで、Mastra が環境変数を自動的に読みます。追加パッケージも不要です。

### 費用と注意点

- **無料枠はありません。** 少額のクレジットを事前にチャージしてください。ハンズオン（weather agent 程度）の消費トークンはせいぜい数万〜十数万で、`gpt-5.6-luna` なら **全体で数十円程度**です。
- ハンズオンで機密情報を入力しないでください。

---

## 2. 料金比較（公式ページ調査結果 / 2026-08-09 時点）

USD / 100万トークン、標準ティア、入力 / 出力。

| 区分 | OpenAI | Google Gemini |
|---|---|---|
| フラッグシップ | gpt-5.6-sol **$5.00 / $30.00** | Gemini 3.1 Pro **$2.00 / $12.00**（20万トークン超で $4/$18） |
| 中位 | gpt-5.6-terra **$2.00 / $12.00** | Gemini 3.6 Flash **$1.50 / $7.50** |
| 低価格 | gpt-5.6-luna **$0.20 / $1.20** | Gemini 3.1 Flash-Lite **$0.25 / $1.50** |
| 最安 | gpt-5.4-nano $0.20 / $1.25 | Gemini 2.5 Flash-Lite **$0.10 / $0.40** |
| バッチ処理 | 50% 割引 | 50% 割引 |
| **無料枠** | **なし** | **あり**（Flash / Flash-Lite 系） |

出典:
- OpenAI: https://developers.openai.com/api/docs/pricing
- Google: https://ai.google.dev/gemini-api/docs/pricing

### 読み取れること

- フラッグシップ帯では **Gemini 3.1 Pro が GPT-5.6 Sol の半額以下**。
- 中位帯は Gemini 3.1 Pro（$2/$12）と gpt-5.6-terra（$2/$12）が**完全に同額**でぶつかっています。
- 最安帯は Gemini 2.5 Flash-Lite（$0.10/$0.40）が最も安い。
- 本ハンズオンが使う低価格帯（`gpt-5.6-luna`）は、Gemini 3.1 Flash-Lite と同水準です。

### 「なぜ価格でなく GPT を選んだか」

価格だけで見れば Gemini（特に無料枠）が有利な場面があります。それでも本ハンズオンで GPT を主軸にしたのは、**コストではなく「出力形式を強制しやすいか」を判断軸にした**ためです。構造化出力は Day1 Step3 以降、Day3 のフロントエンドとのスキーマ共有まで一貫して使う土台で、ここで制約に当たると学習の本題から外れた切り分けに時間を取られます。

なお、これは「GPT のほうが総合的に性能が高い」という主張ではありません。同価格帯で直接競合しており、ベンチマークによって勝敗が入れ替わります。人に説明するときも、**特定モデルの優劣を断定せず「用途とコストで選ぶ」というトーン**にしておくのが無難です。

そもそも今回のハンズオンでは、**最も高価な gpt-5.6-sol を使っても数十円**で、コスト差は実質ゼロです。

---

## 3. 手順：OpenAI API キーの取得

1. https://platform.openai.com にアクセスしてログイン
2. Billing で少額のクレジットをチャージ（無料枠はありません）
3. https://platform.openai.com/api-keys で「Create new secret key」からキーを発行
4. コンテナ内の `.env` に記入

```bash
# .env
OPENAI_API_KEY=sk-...
MASTRA_TELEMETRY_DISABLED=1
```

5. コード側では文字列でモデルを指定するだけ

```ts
model: 'openai/gpt-5.6-luna'      // 本ハンズオンの標準（低価格帯）
// model: 'openai/gpt-5.6-terra'  // 中位。応答の質を比べたいとき
```

`@ai-sdk/*` などの追加パッケージは**不要**です。

### 疎通確認（キー取得直後にやっておく）

Day1 Step1 でエージェントが応答すれば OK ですが、先に単体で確認したい場合はコンテナ内で：

```bash
curl https://api.openai.com/v1/models \
  -H "Authorization: Bearer $OPENAI_API_KEY"
```

モデル一覧の JSON が返ればキーは有効です。

> 💡 `echo $OPENAI_API_KEY` は**シェルの環境変数**を見るコマンドです。`.env` に書いただけではシェルには
> 渡らず空のままなので、上の `curl` を試すなら先に `export $(grep OPENAI_API_KEY .env)` するか、
> キーを直接貼り付けて実行してください。

---

## 4. 代替案A：Gemini API を使う場合

無料枠を使いたい、または社内の請求先の都合で Google に寄せたい場合の選択肢です。**ハンズオン本体（Day1〜Day3）と [02-templates/](../02-templates/) は OpenAI で書かれている**ため、Gemini で進める場合は各所の `model` と環境変数を読み替える必要があります（差し替えの実例は [02-templates/07-provider-switch.md](../02-templates/07-provider-switch.md)）。

| 項目 | 値 |
|---|---|
| プロバイダ | Google Gemini API（Gemini Developer API） |
| キー取得元 | Google AI Studio → https://aistudio.google.com/app/apikey |
| 環境変数 | `GOOGLE_GENERATIVE_AI_API_KEY`（または `GOOGLE_API_KEY`） |
| モデル指定 | `model: 'google/gemini-2.5-flash'` |
| 費用 | **無料**（Flash / Flash-Lite 系は無料枠あり。レート制限あり。Pro 系は対象外） |

```bash
# .env
GOOGLE_GENERATIVE_AI_API_KEY=AQ...   # 2026年8月時点の新規発行キーはこの形式。既存の AIza... キーもそのまま使える
```

`@ai-sdk/*` などの追加パッケージは**不要**です。

### 無料枠の注意点

- レート制限（分あたり・日あたりのリクエスト数）があります。ハンズオンの規模なら足りますが、連打すると弾かれることがあります。
- **無料枠で送った内容は Google の製品改善に利用されます**（有料枠は利用されません）。ハンズオンで機密情報を入力しないでください。
- 構造化出力に制約があります（6章）。

### APIキーの形式について（2026年8月時点）

Google は Gemini API キーを、従来の `AIza` から始まる形式（Standard key）から、**`AQ.` から始まる形式（Auth key）へ移行中**です。2026-08-09 時点で Google AI Studio が新規発行するキーは自動的に `AQ.` 形式になります。

- **移行スケジュール**: 制限なしの Standard key（`AIza`形式）は既に拒否され始めており、**2026年9月には Standard key が制限の有無に関わらず完全に使用不可**になる予定です。既に `AIza...` 形式のキーを持っている場合は、`AQ.` 形式での再発行が必要になります。
- **影響**: `.env` の `GOOGLE_GENERATIVE_AI_API_KEY` にはどちらの形式のキーもそのまま貼り付ければよく、**コード側（`model: 'google/...'`）の変更は不要**です。Mastra はキーの中身を判定せず、環境変数の値をそのまま渡すだけのため。

出典: [Using Gemini API keys](https://ai.google.dev/gemini-api/docs/api-key)（Google AI for Developers）

### 疎通確認

```bash
curl "https://generativelanguage.googleapis.com/v1beta/models" \
  -H "x-goog-api-key: $GOOGLE_GENERATIVE_AI_API_KEY"
```

> 💡 認証は `?key=...` のようなURLクエリパラメータではなく、`x-goog-api-key` HTTPヘッダーで行ってください。公式ドキュメントの例も、Mastra が内部で使う AI SDK の Google プロバイダも、この方式です。`AQ.` 形式のキーではクエリパラメータ方式だと失敗する例が報告されています。

---

## 5. 代替案B：Vertex AI（Gemini Enterprise Agent Platform）経由

**ハンズオンでは非推奨。** Gemini を選んだうえで、組織のガバナンス要件により「Google Cloud 経由でなければならない」場合のみ検討してください。

### 事前に知っておくこと

- Vertex AI は Google Cloud Next 2026 で **Gemini Enterprise Agent Platform** に改称されました。API エンドポイント・モデル ID・認証方式は従来どおりで、変わったのは製品名とドキュメント URL です。
- **Mastra の model router に Vertex 用プレフィックスはありません。** `@ai-sdk/google-vertex` を追加インストールし、Agent の `model` にプロバイダインスタンスを渡す必要があります。

### 手順

```bash
npm install @ai-sdk/google-vertex
```

**Express mode の API キーを使う場合**

```bash
# .env
GOOGLE_VERTEX_API_KEY=<express mode の API キー>
```

```ts
import { Agent } from '@mastra/core/agent'
import { createGoogleVertex } from '@ai-sdk/google-vertex'

// Express mode ではプロジェクト ID / リージョンの指定は不要
const vertex = createGoogleVertex({
  apiKey: process.env.GOOGLE_VERTEX_API_KEY,
})

export const agent = new Agent({
  id: 'my-agent',
  name: 'My Agent',
  instructions: '...',
  model: vertex('gemini-2.5-flash'),
})
```

**サービスアカウント鍵を使う場合**

```bash
# .env
GOOGLE_VERTEX_PROJECT=<your-project-id>
GOOGLE_VERTEX_LOCATION=us-central1
GOOGLE_APPLICATION_CREDENTIALS=/home/node/.gcp/vertex-sa.json
```

鍵ファイルは DevContainer に読み取り専用でマウントします。

```jsonc
{
  "mounts": [
    "source=${localEnv:HOME}/.gcp/vertex-sa.json,target=/home/node/.gcp/vertex-sa.json,type=bind,readonly"
  ]
}
```

### Vertex 経由の落とし穴

| 項目 | 内容 |
|---|---|
| ADC 認証 | 個人アカウントだと `gcloud auth application-default login` が必要になりがち。コンテナ内ではブラウザ認証フローが面倒（gcloud CLI の追加インストールも必要） |
| バージョン整合 | `@ai-sdk/google-vertex` は AI SDK のメジャーに追随する。Mastra が依存する `ai` とメジャーがズレると型エラーになる。導入後 `npm ls ai @ai-sdk/google-vertex` と `npx tsc --noEmit` で確認 |
| Express mode の制約 | チューニング済みモデル（`endpoints/...`）と音声認識（Chirp）は API キー非対応 |
| 構造化出力 | Vertex は OpenAPI 3.0 のサブセットのため `z.union` / `z.record` が使えない。`providerOptions.vertex.structuredOutputs: false` で回避 |

---

## 6. 構造化出力の注意（Gemini / Vertex を選んだ場合）

**この制約は Google のモデル固有のものです。** 本ハンズオンのスキーマは `z.object` などの基本形だけで組んであるため、標準構成（OpenAI）では問題になりません。本リポジトリ内の構造化出力に関する注意書きは、すべてこの章を根拠にしています。他プロバイダへ差し替える際に読んでください。

> OpenAI の構造化出力にも独自のスキーマ規則がありますが、本リポジトリでは検証していません。
> 基本形の範囲を超えるスキーマ（`z.union` など）を使う場合は、公式ドキュメントで確認してください。

Google のモデルは JSON Schema のサブセットしかサポートしていません。Gemini / Vertex に切り替えて Day1 Step3 の構造化出力を扱う場合、以下は避けてください。

- `z.union(...)`
- `z.record(...)`

`z.object` / `z.array` / `z.string` / `z.number` / `z.boolean` / `z.enum` の範囲で組むのが安全です。

---

## 参考リンク

- OpenAI API 料金: https://developers.openai.com/api/docs/pricing
- OpenAI API キー取得: https://platform.openai.com/api-keys
- Mastra モデルプロバイダ環境変数一覧: https://mastra.ai/models/environment-variables
- Mastra 対応モデル一覧: https://mastra.ai/models
- Gemini API 料金: https://ai.google.dev/gemini-api/docs/pricing
- Gemini API キー取得: https://aistudio.google.com/app/apikey
- Using Gemini API keys（AQ. 形式への移行について）: https://ai.google.dev/gemini-api/docs/api-key
- AI SDK Google Vertex Provider: https://ai-sdk.dev/providers/ai-sdk-providers/google-vertex
- Gemini Enterprise Agent Platform 入門: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/start
