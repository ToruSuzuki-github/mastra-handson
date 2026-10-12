# 4-0. モデル ID を GPT-5.6 系へ差し替える（全テンプレート共通・5分）

4-1 / 4-2 の両方で最初に行う共通作業です。**プロバイダは OpenAI のまま**なので、
必要なキーは `OPENAI_API_KEY` のままで変わりません（ハンズオン本体と同じキーです）。

**差し替え表**

| テンプレートの初期値 | 差し替え後 | どこで使われているか |
|---|---|---|
| `openai/gpt-5-mini` | **`openai/gpt-5.6-luna`** | Chat with Database の SQL Agent（1体）／Deep Search の Intent Clarifier（1体） |
| `openai/gpt-5.2` | **`openai/gpt-5.6-terra`** | Deep Search の残り3体（検索計画・結果評価・最終回答） |

<details>
<summary>💰 <b>補足：なぜ差し替えるのか</b>（作業には不要。根拠を知りたい人向け）</summary>

**1. 安くなる**（USD / 100万トークン。[OpenAI 公式料金](https://developers.openai.com/api/docs/pricing)、2026-08-15 時点）

| モデル | 入力 | 出力 |
|---|---|---|
| `gpt-5-mini`（初期値） | $0.25 | $2.00 |
| **`gpt-5.6-luna`（差し替え後）** | **$0.20**（-20%） | **$1.20**（**-40%**） |
| `gpt-5.2`（初期値） | $1.75 | $14.00 |
| **`gpt-5.6-terra`（差し替え後）** | **$2.00**（+14%） | **$12.00**（-14%） |

GPT-5.6 系は同じタスクでの**出力トークン数が従来比で少ない**と公表されており、出力課金が支配的な
Deep Search では表の差以上に安くなります。

**2. 初期値のモデルには停止期限がある**

`gpt-5-mini`（`gpt-5-mini-2025-08-07`）は **2026-12-11 に API から削除**されます
（[Deprecations](https://developers.openai.com/api/docs/deprecations)）。同世代の `gpt-5.2` も
`gpt-5.2-chat-latest` が 2026-08-10 に停止済みで、追随する見込みです。

**3. 性能も上**

GPT-5.6 の3ティア（Sol / Terra / Luna）はいずれも前世代を上回ります。

**なぜ Deep Search のメインを Luna にしないのか**

Luna は 512K〜1M トークン級の長文で精度が大きく落ちます（MRCR で Terra 72.5% に対し Luna 41.3%）。
Web ページ本文を大量に読み込む Deep Search のメインには Terra を充てます。逆に、入力が短い
Chat with Database では Luna で十分です。

なお OpenAI 公式の「推奨後継」は `gpt-5-mini` → `gpt-5.6-terra` ですが、これは性能基準の対応表です。
コスト重視の本ハンズオンでは、初期値より安い `gpt-5.6-luna` を選びます。

</details>

**差し替えコマンド**（テンプレートのディレクトリ内で実行）

```bash
# 1. 現在のモデル指定を確認する
grep -rn "openai/" src/

# 2. 一括で差し替える（該当しない行は何も起きない）
grep -rl 'openai/gpt-5\.2'   src/ | xargs -r sed -i 's#openai/gpt-5\.2#openai/gpt-5.6-terra#g'
grep -rl 'openai/gpt-5-mini' src/ | xargs -r sed -i 's#openai/gpt-5-mini#openai/gpt-5.6-luna#g'

# 3. 差し替え結果を確認する（gpt-5.6-* だけが残っていれば成功）
grep -rn "openai/" src/
```

> テンプレート側のモデル指定はいつ更新されるか分かりません。**手順3の `grep` で実際の値を必ず確認**し、
> すでに `gpt-5.6-*` になっていればこの節はスキップして構いません。

**差し替え先のモデルが実在するか確かめたい場合**は、テンプレートに同梱されているスクリプトが使えます
（`npm install` 済みのテンプレートのディレクトリ内で実行。詳細は
[reference/agent-skills.md](../reference/agent-skills.md) §4-2）。

```bash
node .agents/skills/mastra/scripts/provider-registry.mjs --provider openai
```
