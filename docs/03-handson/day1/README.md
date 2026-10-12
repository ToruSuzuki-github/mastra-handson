# ハンズオン 基礎編（Day 1 / 約6時間）

Agent・Tool・Studio・HTTP API という Mastra の中核を一通り触ります。

**ゴール**: 実際の天気 API を叩いて旅行の持ち物を提案するエージェントを、ゼロから作って動かせる状態。

**前提**: [01-setup/01-environment.md](../../01-setup/01-environment.md) と
[01-setup/02-llm.md](../../01-setup/02-llm.md) が完了していること。

---

## Step 一覧

| Step | 内容                                                    | 時間 | 手順                                           |
| ---- | ------------------------------------------------------- | ---- | ---------------------------------------------- |
| 0    | 環境の最終確認                                          | 15分 | [00-preflight.md](./00-preflight.md)           |
| 1    | Quickstart：1コマンドでエージェント生成 → Studio で対話 | 45分 | [01-quickstart.md](./01-quickstart.md)         |
| 2    | 手動セットアップ：ゼロから Agent を作る                 | 60分 | [02-first-agent.md](./02-first-agent.md)       |
| 3    | 応答形式の3パターン：generate / stream / 構造化出力     | 60分 | [03-response-modes.md](./03-response-modes.md) |
| 4    | ツール作成：実 API を叩く weather tool                  | 90分 | [04-tools.md](./04-tools.md)                   |
| 5    | HTTP API として叩く（curl）                             | 30分 | [05-http-api.md](./05-http-api.md)             |
| 6    | 振り返りとメモ整理                                      | 30分 | [06-review.md](./06-review.md)                 |
| 予備 |                                                         | 30分 |                                                |

---

## Day 1 完了チェックリスト

- [ ] `npm create mastra@latest` で生成したプロジェクトを動かせた
- [ ] ゼロから Agent を定義して Studio で応答させた
- [ ] `generate()` / `stream()` / 構造化出力を使い分けられた
- [ ] `createTool()` で実 API を叩くツールを作り、エージェントから呼ばせた
- [ ] トレースでツール呼び出しの中身を確認した
- [ ] HTTP API 経由でエージェントを叩けた

次は [day2/](../day2/) へ。
ここで叩いた HTTP API を TypeScript のフロントエンドから使うのは
[day3/](../day3/)（フロントエンド編）です。
