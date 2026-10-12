# 01. 事前準備（環境構築と LLM の用意）

本編（[02-templates/](../02-templates/) または [03-handson/](../03-handson/)）を始める前に済ませておく準備フェーズです。
**所要時間の合計は 1.5〜2.5 時間**（Docker イメージの pull など待ち時間込み）。本編と一緒にまとめてやると時間が足りません。

**ゴール**: DevContainer が起動し、`.env` に有効な API キーが入っていて、`mastra` コマンドが
コンテナ内で動く状態。

---

## 目次

| ファイル | 内容 | 所要目安 |
|---|---|---|
| [01-environment.md](./01-environment.md) | DevContainer の構築（Windows 11 Home + WSL2 + Docker Desktop）。`devcontainer.json` / `.gitignore` / `.env.example` の中身と、後片付け手順まで | 1〜2時間 |
| [02-llm.md](./02-llm.md) | 使う LLM の決定と API キー取得。OpenAI（標準）/ Gemini / Vertex AI の料金比較と手順 | 30分 |

**この順に進めてください。** 01 の `.env.example` を作ってから 02 でキーを埋める流れです。

---

## 関連する参照資料

- [reference/devcontainer.json](../reference/devcontainer.json) — `devcontainer.json` の正典。
  [01-environment.md](./01-environment.md) §3 はこのファイルの写し
- [Dev Container metadata reference](https://containers.dev/implementors/json_reference/) —
  `devcontainer.json` の各プロパティの公式仕様

---

次は [02-templates/](../02-templates/)（公式テンプレート実行）または
[03-handson/](../03-handson/)（ハンズオン本体）へ。
