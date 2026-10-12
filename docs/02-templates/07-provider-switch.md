# 5.（任意）モデルを他プロバイダへ差し替える手順

**この章は読み飛ばして構いません。** ハンズオン本体（[03-handson/](../03-handson/)）もテンプレート編も
プロバイダは OpenAI で揃えてあるため、通常は差し替えの必要がありません。

次のような場合のみ、以下の手順で差し替えます。

- Mastra の **Model Router の効果**（`model` の文字列を1行変えるだけでプロバイダを替えられること）を実際に体験したい
- 社内の請求先の都合などで Gemini 無料枠に寄せたい

ここでは差し替え先の例として **Gemini** を使います（他プロバイダでも要領は同じです）。
Gemini のキー取得と注意点は [01-setup/02-llm.md](../01-setup/02-llm.md) §4 を参照してください。

## 5-1. 差し替えの手順

1. `.env.example` を見て、必要な環境変数を確認

```bash
cat .env.example
```

2. `.env` に Gemini のキーを設定

テンプレートの `.env.example` に `GOOGLE_GENERATIVE_AI_API_KEY` の行は**ありません**。自分で追記します。

```bash
GOOGLE_GENERATIVE_AI_API_KEY=<あなたのキー>
```

3. エージェント定義のモデル指定を書き換える

```bash
# モデル指定箇所を探す
grep -rn "model:" src/
```

```ts
// 変更前（テンプレートの初期値、または 4-0 で差し替えた後の値）
model: 'openai/gpt-5.6-luna',

// 変更後
model: 'google/gemini-2.5-flash',
```

> Gemini に寄せる場合、4-0 のモデル ID 差し替えは不要です（どのみち丸ごと置き換わるため）。

4. 再起動して動作確認

```bash
npm run dev
```

## 5-2. 差し替えがうまくいかない場合

| 症状 | 対処 |
|---|---|
| `import { openai } from '@ai-sdk/openai'` のように provider を直接 import している | model router 形式の文字列（`'google/gemini-2.5-flash'`）に置き換えれば import ごと削除できる |
| 埋め込み（embedding）モデルも OpenAI 指定になっている | RAG 系テンプレート（Chat with PDF など）で発生。Google の埋め込みモデルに差し替えるか、そのテンプレートだけ OpenAI キーを使う |
| 構造化出力でスキーマエラー | Google のモデルは `z.union` / `z.record` 非対応。該当箇所を書き換えるか、OpenAI のまま使う（[01-setup/02-llm.md](../01-setup/02-llm.md) 6章） |
| 書き換え箇所が複数ある | エージェント1体につき1箇所。Chat with Database は1ファイル1行だが、Deep Search は4ファイル（4体）すべてが対象 |
