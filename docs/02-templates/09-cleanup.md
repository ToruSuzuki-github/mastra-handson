# 8. 使い終わったテンプレートの片付け

テンプレートは**使い捨て**です。中身は作者の所有物で自分の
成果物ではないため、動作確認が済んだら丸ごと削除して構いません。**1つ約 337MB、推奨コースの
4-1 と 4-2 を両方作ると約 700MB** です。WSL のディスクを圧迫するので終わったら消してください。

## 8-1. 削除の手順

```bash
# DevContainer 内。テンプレートを作った親ディレクトリで実行する
cd ~/work/mastra-handson

# ディレクトリごと削除する（node_modules だけ消して残す意味は無い）
rm -rf template-sql template-deep-search

# Day1 Step1 の Quickstart で作ったものも残っていれば一緒に消せる
rm -rf quickstart-demo

# npm のキャッシュも消す場合（任意。他プロジェクトの再インストールも遅くなる）
npm cache clean --force
```

依存も生成物もすべてディレクトリ内で完結しているため、`rm -rf` だけで親リポジトリには何も残りません。
`template-*/` と `quickstart-demo/` は [.gitignore](../../.gitignore) で無視されるので、
消し忘れても `git status` は汚れません。

## 8-2. API キーは別途失効させる

`.env` に書いた `OPENAI_API_KEY` / `EXA_API_KEY` はディレクトリ削除で消えますが、
**キー自体はサービス側で有効なまま**です。ハンズオン用に発行したキーなら、片付けと同時に
失効させてください。

- OpenAI: https://platform.openai.com/api-keys
- Exa: https://dashboard.exa.ai/api-keys

> **DevContainer や Docker イメージごと消す**場合は
> [01-setup/01-environment.md](../01-setup/01-environment.md) §8 を参照してください。
> こちらのテンプレート削除を先に済ませてから実行します。

<details>
<summary>📦 <b>補足：内訳（2026-08-15 実測、Chat with Database）</b>（何がディスクを食っているか知りたい人向け）</summary>

| 中身 | サイズ | 何か |
|---|---|---|
| `node_modules/` | 316MB | 依存パッケージ |
| `.mastra/` | 19MB | `mastra dev` が作るビルド出力 |
| `src/mastra/public/data.db` | 1.1MB | seed が作る SQLite |
| テンプレート本体（`src/` 等） | 1MB未満 | |
| **合計** | **約 337MB** | |

> `create-mastra` のバージョンによっては `.git/`（約 420KB）が作られることがあります。
> 2026-09 時点の手元の `template-sql/` / `template-deep-search/` には**ありませんでした**。
> あってもディレクトリごと消すので手順は変わりません。

</details>
