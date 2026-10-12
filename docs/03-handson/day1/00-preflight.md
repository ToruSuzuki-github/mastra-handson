# Step 0. 環境の最終確認（15分）

DevContainer 内のターミナルで確認します。

```bash
node --version    # v22.13.0 以上
npm --version
```

API キーは、**リポジトリ直下の `.env` に値が入っていること**を確認します。
この時点ではまだ Mastra プロジェクトを作っていないので、確認先はリポジトリ直下の `.env` です
（[01-setup/01-environment.md](../../01-setup/01-environment.md) §5 で作ったもの）。

```bash
cd ~/work/mastra-handson
grep OPENAI_API_KEY .env    # 値が = の右側に入っているか
```

> `.env` が無い場合は `cp .env.example .env` してから値を貼ります。
> 各プロジェクト（`quickstart-demo/` や `travel-agent/`）の `.env` は Step 1・Step 2 で別途作ります。

> 💡 `echo $OPENAI_API_KEY` は**シェルの環境変数**を見るコマンドなので、`.env` に
> 書いただけでは空のままです（それが正常）。値が出るのは `devcontainer.json` の `remoteEnv` 経由で
> ホスト側に export してある場合だけで、`.env` 運用なら空で問題ありません。

**チェックポイント**
- [ ] Node.js のバージョンが 22.13.0 以上
- [ ] `.env` に API キーが入っている
