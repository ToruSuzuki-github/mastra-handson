# 実施前の動作確認（ハンズオンを開催する人向け）

本手順書のコード例は **2026-08-15 時点の Mastra v1（`@mastra/core` 1.59.0 / `zod` 4.4.3）で型チェック済み**です。
ただし**型チェックで分かるのはコードの整合性までで、次の3つは検出できません**。

| 検出できないもの | なぜ |
|---|---|
| 外部 API のレスポンス形状の変化 | `weather-tool.ts` は API の戻り値を `any` のまま読んでいるため、形が変わっても型チェックは素通りし、実行時に `undefined` になる |
| LLM 側の変化 | モデル名の廃止、無料枠のレート制限、出力傾向の変化 |
| Studio の画面構成 | バージョンによって UI が変わる。[reference/mastra-studio.md](../reference/mastra-studio.md) の画面説明が古くなる |

加えて Mastra は更新が速いため、日が経つとコード例そのものが最新版と食い違う可能性があります。
**開催の1週間前を目安に、以下を DevContainer 内で実施してください。**

## 1. 通し実行（本命・所要2〜3時間）

[03-handson/day1/](../03-handson/day1/) と [03-handson/day2/](../03-handson/day2/) を、**自分で最後まで一度通してください。**
解説を読む時間が要らないぶん、受講者の所要時間（Day1 + Day2 で17.5時間）よりはるかに短く終わります。
これが最も確実な確認方法です。

通しながら、次の4点を特に見てください。

- [ ] Step 2 の `observability` 設定で、**Studio の Traces 画面に実行が出てくる**（空なら `observability` の設定漏れ。[reference/mastra-studio.md](../reference/mastra-studio.md) §7-1）
- [ ] Step 4 の weather tool が**実際の気温を返す**（`undefined` や `NaN` が混ざらない）
- [ ] Step 8 のワークフローが Studio のグラフ上で最後まで進む（`tripPrepWorkflow` の登録忘れに注意）
- [ ] Step 8-4 の `packing-branch-workflow` で、都市によって動く枝が変わる（`rainy` / `dry`）
- [ ] Step 8-5 の `refine-workflow` でループが回って止まる。**LLM を呼ぶので API キーが要る**（Traces で `refine-suggestion` の回数を数える）
- [ ] Step 8-6 の `approval-workflow` が `approve` で止まり、`true` / `false` の両方で再開できる（同じく LLM を呼ぶ）
- [ ] Step 9 のマルチエージェントで、サブエージェントが実際に呼ばれている（トレースで確認）

### フロントエンド編（Day3）を実施するなら

[03-handson/day3/](../03-handson/day3/) は**まだ実機検証されていません**（公式ドキュメントと型定義の
読解のみで書かれています）。Day3 まで扱う場合は、上記に加えて次も確認してください。

- [ ] ポート **5173**（Vite）と **3000**（Next.js）が DevContainer から実際に転送される
- [ ] Step 13 で `npx tsc --noEmit` がスキーマのリネームを検出する
- [ ] Step 14 で `curl http://localhost:3000/api/mastra/agents` が応答する（`prefix` の指定が効いているか）
- [ ] チャット UI に出るツール名が `tool-get-weather` である（`tools` のキーで決まる。[day1/04-tools.md](../03-handson/day1/04-tools.md) §4-3 の注記参照）

## 2. 外部 API だけ先に確認する（所要5分）

上記のうち最も壊れやすいのが Open-Meteo です。通し実行の前に、単体で叩いて形を確かめられます。

```bash
# 1) 都市名 → 緯度経度。results[0] に latitude / longitude / name があるか
#    都市名は必ずローマ字。日本語表記はヒットしない（後述）
curl -s -G 'https://geocoding-api.open-meteo.com/v1/search' \
  --data-urlencode 'name=Naha' -d 'count=1' -d 'language=ja' | head -c 400

# 2) 緯度経度 → 天気。current.temperature_2m と
#    daily.time / temperature_2m_max / temperature_2m_min / precipitation_probability_max があるか
curl -s 'https://api.open-meteo.com/v1/forecast?latitude=26.2124&longitude=127.6809&current=temperature_2m&daily=temperature_2m_max,temperature_2m_min,precipitation_probability_max&timezone=auto' | head -c 700
```

ここで挙げたキー名が1つでも消えていれば、[03-handson/day1/04-tools.md](../03-handson/day1/04-tools.md) §4-2 の補助関数（`geocode` / `fetchWeather`）の書き換えが必要です。

> 💡 curl で日本語を試すときは `--data-urlencode` を使ってください。URL に生の日本語を直接書くと **403 Forbidden** になります。
> なおハンズオンのコードは `URLSearchParams` が自動でエンコードするため、この問題は起きません。

## 3. 型チェックだけ手早く回す（所要15分）

通し実行の時間が取れないときの簡易版です。コード例が最新の Mastra と整合しているかだけを機械的に確認します。

```bash
# DevContainer 内
mkdir -p /tmp/doc-check && cd /tmp/doc-check
npm init -y && npm pkg set type=module
npm install typescript @types/node zod @mastra/core @mastra/memory @mastra/libsql @mastra/observability @mastra/evals
```

`tsconfig.json` は [03-handson/day1/02-first-agent.md](../03-handson/day1/02-first-agent.md) §2-1 に
載せているものをそのまま使います（`strict: true` を含む）。
そのうえで、**day1 と day2 のコードブロックを先頭コメントのパス（`// src/mastra/...`）どおりに配置**して実行します。

```bash
npx tsc --noEmit
```

同じパスのブロックが複数回登場する箇所（`travel-agent.ts` など）は手順書の進行に沿って上書きされるため、**手順書に出てくる順に配置**してください。

## 4. ズレを見つけたら

1. 手順書の該当箇所を修正する（**手順書だけ直して原因を放置しない**）
2. [README.md](./README.md) の §10 メモ欄に症状と対処を記録する
3. 本ファイル冒頭とこの節の「型チェック済み」の日付・バージョンを更新する
