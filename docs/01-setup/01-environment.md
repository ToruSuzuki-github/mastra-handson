# 01. 環境構築（Windows 11 Home + DevContainer）

Mastra ハンズオン専用の使い捨て開発環境を作ります。所要時間は 1〜2 時間（イメージ pull などの待ち時間込み）。

---

## 1. ホスト側の前提条件

### 1.1 必要なもの

| ソフトウェア | 備考 |
|---|---|
| Windows 11 64bit（21H2 以降、22H2 以降推奨） | **Home エディションで問題なし** |
| WSL 2 | `wsl --install` で導入。WSL 本体は 2.1.5 以上推奨 |
| Docker Desktop for Windows | **WSL 2 バックエンド**で構成 |
| Visual Studio Code | — |
| VS Code 拡張 `ms-vscode-remote.remote-containers` | Dev Containers |
| Git for Windows | — |

### 1.2 Windows 11 Home 固有の注意

- **Hyper-V バックエンドは使えません。** Docker Desktop の Hyper-V バックエンドは Pro / Enterprise / Education 向けです。Home では WSL 2 バックエンド一択ですが、機能・性能上の不利はありません。
- **BIOS/UEFI で仮想化支援（Intel VT-x / AMD-V, SVM）を有効化**しておきます。タスクマネージャー「パフォーマンス」タブで「仮想化: 有効」と表示されていれば OK。
- ハードウェア目安: RAM 8GB 以上、空きディスク 10GB 以上。

### 1.3 Docker Desktop の WSL 統合を有効化する（初回起動前に必須）

Docker Desktop をインストールしただけでは、WSL 側の Ubuntu から `docker` コマンドが使えず、DevContainer の起動も失敗します。**初回起動の前に**次を確認してください。

1. Docker Desktop の左下のクジラアイコンが**緑（Engine running）**になっているか確認する
2. **Settings → Resources → WSL Integration** を開き、使用するディストリビューション（例: `Ubuntu`）のトグルを**オン**にする
3. **Apply & Restart** を押す。**トグルを切り替えただけでは反映されません**
4. PowerShell で WSL を再起動する

```powershell
# PowerShell（ホスト側）
wsl --shutdown
```

WSL のシェルで次が通れば準備完了です。

```bash
docker version
```

> ⚠️ 統合が無効なままだと、WSL 内で `docker` を実行したときに `The command 'docker' could not be found in this WSL 2 distro.` というメッセージが出ます。これが出たら本節をやり直してください。Docker Desktop の再起動やアップデートの後に統合が外れることもあります。

> 💡 `wsl --shutdown` は WSL に接続中の VS Code ウィンドウやターミナルをすべて切断します。作業中のファイルは先に保存してください。

### 1.4 パフォーマンス上の重要設定（必ず読む）

プロジェクトフォルダは **Windows 側（`C:\Users\...`）ではなく WSL 2 のファイルシステム内**に置いてください。

Windows ↔ Linux のファイルシステム境界をまたぐバインドマウントは I/O が著しく遅く、`npm install` や Mastra の dev サーバー（ファイル監視）が体感で数倍遅くなります。ハンズオン中のストレスに直結する部分です。

```powershell
# PowerShell（ホスト側）
wsl
```

```bash
# WSL 内
mkdir -p ~/work/mastra-handson && cd ~/work/mastra-handson
code .          # ここから VS Code を起動すると WSL 側のパスで開く
```

### 1.5 WSL のリソース上限（任意）

`%USERPROFILE%\.wslconfig` を作成して調整できます。

```ini
[wsl2]
memory=6GB
processors=4
```

反映は `wsl --shutdown` 後に Docker Desktop を再起動。

---

## 2. ディレクトリ構成

```
mastra-handson/
├── .devcontainer/
│   └── devcontainer.json
├── .env                 # ← Git 管理外。API キーを書く
├── .env.example
└── .gitignore
```

---

## 3. `.devcontainer/devcontainer.json`

以下が正典です。実ファイルは [reference/devcontainer.json](../reference/devcontainer.json) に置いてあるので、コピーして使ってください（このブロックはその写しです）。

```jsonc
{
  // 正典は docs/reference/devcontainer.json。変更する場合はそちらを先に更新し、
  // docs/01-setup/01-environment.md §3 の掲載内容と同期すること
  "$schema": "https://raw.githubusercontent.com/devcontainers/spec/main/schemas/devContainer.schema.json",
  "name": "mastra-handson",

  // Node 24 系（現行 Active LTS。Mastra v1 要件の 22.13.0+ も満たす）
  // 更新頻度を抑えたい場合は "5-24-bookworm" のようにピン留めする
  // （タグ一覧: https://mcr.microsoft.com/v2/devcontainers/typescript-node/tags/list）
  "image": "mcr.microsoft.com/devcontainers/typescript-node:24-bookworm",

  // 4111: Mastra Studio / dev サーバー
  // 5173: Day3 Step13 の Vite（travel-agent/web/）
  // 3000: Day3 Step14 の Next.js（travel-next/）
  "forwardPorts": [4111, 5173, 3000],
  "portsAttributes": {
    "4111": {
      "label": "Mastra Studio",
      // 既定値。mastra dev 起動後に自動でブラウザを開きたい場合は "openBrowserOnce" に変更する
      "onAutoForward": "notify"
    },
    "5173": {
      "label": "Vite (travel-agent/web)",
      "onAutoForward": "notify"
    },
    "3000": {
      "label": "Next.js (travel-next)",
      "onAutoForward": "notify"
    }
  },

  // ホストの環境変数をコンテナへ引き渡す（本ハンズオンは .env を使うので既定では無効）
  // ※ ここに書いたキーは「継承済み」扱いになり、mastra dev は .env の値で上書きしない。
  //    ホスト側が未設定だと空文字が入り、.env に書いても空のままになる。
  // "remoteEnv": {
  //   "OPENAI_API_KEY": "${localEnv:OPENAI_API_KEY}"
  // },

  "containerEnv": {
    // Mastra CLI の匿名テレメトリを無効化（任意）
    "MASTRA_TELEMETRY_DISABLED": "1"
  },

  "postCreateCommand": "node --version && npm --version",

  "customizations": {
    "vscode": {
      "extensions": [
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode",
        "mikestead.dotenv",
        "qwtel.sqlite-viewer",
        // トラブル時にIDE内から相談できるようにしておく（任意）
        "anthropic.claude-code"
      ],
      "settings": {
        "terminal.integrated.defaultProfile.linux": "bash",
        "editor.formatOnSave": true,
        // formatOnSave だけでは TS/JS のフォーマッタが確定せず保存時整形が効かないため明示指定
        "editor.defaultFormatter": "esbenp.prettier-vscode"
      }
    }
  },

  // typescript-node イメージの既定非 root ユーザー
  "remoteUser": "node"
}
```

**補足**

- `qwtel.sqlite-viewer` は応用編（Day2 Step7）でメモリの保存先 SQLite を覗くのに使います。
- `forwardPorts` の 5173 と 3000 はフロントエンド編（[03-handson/day3/](../03-handson/day3/)）で使います。Day3 をやらない場合は転送されても害はないので、そのままで構いません。
- `anthropic.claude-code` はハンズオン中に詰まった際、DevContainer内でClaude Codeに相談できるようにするための拡張機能です（任意）。拡張機能はコンテナ側のVS Code Serverに個別インストールされるため、ホスト側に入れているだけではコンテナ内では使えません。Claude Code自体はコンテナからインターネットへ既定でアクセスできる（Dockerのデフォルトネットワーク）ため、通信のための追加設定は不要です。
- Node 22 系に固定したい場合は `typescript-node:22-bookworm` に変更しても動作要件（22.13.0+）は満たせます。ただし Node 22 は 2025-10-21 から Maintenance LTS のため、本ハンズオンでは現行 Active LTS の Node 24 を既定にしています。
- **`remoteEnv` は既定でコメントアウトしています。有効にすると `.env` が効かなくなります。** `${localEnv:...}` は WSL 側（VS Code が動いている側）の環境変数を読み、未設定なら**空文字**になります。`mastra dev` は起動時点で環境に存在するキーを「継承済み」とみなし、`.env` の値で上書きしません（空文字でも同じ）。そのため `remoteEnv` に書いたキーは、ホスト側が未設定だと `.env` に正しい値を書いても空のままで、API キーが未設定のように見えます。API キーは `.env` だけで管理してください。ホストの環境変数で渡したい場合のみ `remoteEnv` を有効にし、その場合は `.env` に同じキーを書かないでください。症状からの逆引きは [troubleshooting/README.md](../troubleshooting/README.md) の「コンテナ内で API キーが空」を参照。なお `remoteEnv` を有効にした場合、VS Code の設定で `Terminal > Integrated: Inherit Env` が有効（既定 true）でないと、統合ターミナルに値が反映されない点にも注意してください。
- `portsAttributes` の `onAutoForward` は既定値の `"notify"` にしています。`forwardPorts` で常時転送している 4111 に対して `"openBrowser"` にすると、コンテナ接続直後（`mastra dev` 起動前）に空タブが開くことがあるためです。`mastra dev` 起動後に自動でブラウザを開かせたい場合は `"openBrowserOnce"` を検討してください。

> 各プロパティの詳しい意味は公式の [Dev Container metadata reference](https://containers.dev/implementors/json_reference/) を参照してください。Node 24 を選んでいるのは、Mastra v1 の要件（22.13.0 以上）を満たす LTS（長期サポート）版だからです（[Node.js のリリース一覧](https://nodejs.org/en/about/previous-releases)）。

---

## 4. `.gitignore`

リポジトリ直下に実ファイルがあります（[.gitignore](../../.gitignore)）。

```gitignore
# Node / Mastra 生成物（travel-agent/, travel-next/, template-*/ など、どのサブプロジェクトにも共通）
node_modules/
dist/
.mastra/

# Next.js 生成物（travel-next/。Day3 Step14）
.next/
next-env.d.ts

# 02-templates/ の公式テンプレート実行で作る使い捨てプロジェクト（中身はテンプレート作者の
# 所有物。ハンズオン後は丸ごと削除する）。travel-agent/ は成果物なので対象外
template-*/

# Day1 Step1 の Quickstart で作る使い捨てプロジェクト（生成物を眺めるだけで、以降は使わない）
quickstart-demo/

# 環境変数（.env.example は追跡する）
.env
.env.local
.env.development
!.env.example

# Mastra Memory の SQLite ストレージ
*.db
*.db-journal
*.db-wal
*.db-shm

# Step 10 の Metrics 用に DuckDB を足したときにできる観測データ（src/mastra/public/）
*.duckdb
*.duckdb.wal

# OS / エディタが生成する一時ファイル
.DS_Store
Thumbs.db
desktop.ini
*.swp
*.swo
*~

# Claude Code の個人用ローカル設定（各自の環境の値なので共有しない）
.claude/settings.local.json

# ローカル専用メモ（自分の環境の Docker コンテナ名・イメージ名など。各自の値なので共有しない）
.local/

# エディタの個人設定（インデント幅などの好み。共有すべき設定は
# .devcontainer/devcontainer.json の customizations.vscode に置く）
.vscode/
```

---

## 5. `.env.example`

リポジトリ直下に実ファイルがあります（[.env.example](../../.env.example)）。`cp .env.example .env` してから値を埋めてください。

```bash
# このファイルを .env にコピーして値を埋める（.env は Git に絶対にコミットしない）
#   cp .env.example .env

# OpenAI API（推奨。01-setup/02-llm.md 参照）
OPENAI_API_KEY=

# Gemini を使う場合（代替案。01-setup/02-llm.md §4 参照）
# GOOGLE_GENERATIVE_AI_API_KEY=

# 任意
MASTRA_TELEMETRY_DISABLED=1
```

> `.env` は Git に絶対にコミットしないでください。

---

## 6. 構築手順

> 事前確認: §1.3 の **Docker Desktop の WSL 統合**が有効になっていること（`docker version` が WSL 内で通ること）。ここが未設定だと 3. の「Reopen in Container」で失敗します。

```bash
# 1) WSL 内にディレクトリを作成
wsl
mkdir -p ~/work/mastra-handson && cd ~/work/mastra-handson
mkdir .devcontainer

# 2) 上記のファイルを作成
#    §3 の devcontainer.json は本リポジトリの reference/devcontainer.json をコピーしてもよい
#    （このハンズオン用リポジトリを clone している場合）
code .
```

3. VS Code で `F1` → **Dev Containers: Reopen in Container**
   初回はイメージ pull で数分かかります。

4. コンテナ内ターミナルで確認

```bash
node --version   # v24.x（22.13.0 以上であること）
npm --version
```

ここまで通れば環境構築は完了です。次は [02-llm.md](./02-llm.md) へ。

---

## 7. 補足：プロジェクトを Windows 側に置かざるを得ない場合

`node_modules` を名前付きボリュームに逃がすとある程度改善します。

```jsonc
{
  "mounts": [
    "source=mastra-node-modules,target=${containerWorkspaceFolder}/node_modules,type=volume"
  ],
  "postCreateCommand": "sudo chown -R node:node ${containerWorkspaceFolder}/node_modules && node --version && npm --version"
}
```

ボリュームの初期所有者が root になるため `chown` が必要です。この方式にすると `node_modules` はホストから見えなくなる点に注意してください。

> ⚠️ `postCreateCommand` は1つの devcontainer.json に1つしか書けません。§3 の `postCreateCommand`（`node --version && npm --version`）と併用する場合は、上記のように **1本のコマンドにまとめてください**。そのまま追記すると §3 側が上書きされて実行されなくなります。

---

## 8. 後片付け

> **本節は Docker（コンテナ・イメージ）を消す手順です。** 03 で作った公式テンプレートの
> プロジェクト（`template-*/`、1つあたり約 337MB）の削除と API キーの失効は
> [02-templates/09-cleanup.md](../02-templates/09-cleanup.md) §8 にあります。**そちらを先に済ませてください。**

ハンズオン終了後、環境ごと消す場合：

```bash
# ホスト側 PowerShell（WSL のシェルでも可）

# 1) コンテナを削除する
#    DevContainer のコンテナ名はランダムに決まるため、ラベルで絞り込んで探す
docker ps -a --filter "label=devcontainer.config_file"
docker rm -f <上で確認したコンテナ名>

# 2) DevContainer が作った派生イメージを削除する
#    名前は vsc-<フォルダ名>-<パスのSHA-256>[-uid] 形式
docker images --filter "reference=vsc-mastra-handson-*"
docker image rm <上で確認したイメージ名>

# 3) ベースイメージを削除する
docker image rm mcr.microsoft.com/devcontainers/typescript-node:24-bookworm
```

**注意点**

- コンテナ名は devcontainer.json の `name` とは無関係のランダムな名前（`musing_driscoll` など）になります。イメージ名も `vsc-<フォルダ名>-<フォルダの絶対パスから作ったハッシュ値>-uid` のような長い名前になります。
- **環境構築が終わった時点で、自分のコンテナ名・イメージ名を控えておく**と、終了時に探し直さずに済みます。控え先としてリポジトリ直下の `.local/cleanup-notes.md` を使ってください（Git 管理対象外のファイルなので、リポジトリを clone した時点では存在しません。上記の `docker ps -a` / `docker images` の結果を貼って各自で作成してください）。
- **手順 2 を省くと、いちばん容量を食う派生イメージが残ります。**ベースイメージ（手順 3）だけを消しても解放されません。
- §7 の名前付きボリュームを使った場合は `docker volume rm mastra-node-modules` も実行してください。
- `docker system prune -a --volumes` は他プロジェクトにも影響するため、使う場合は範囲を確認してください。

---

## 参考リンク

- Docker Desktop WSL 2 バックエンド: https://docs.docker.com/desktop/features/wsl/
- devcontainers/images（typescript-node）: https://github.com/devcontainers/images/tree/main/src/typescript-node
- Mastra v1 移行ガイド（Node 要件の記載あり）: https://mastra.ai/reference/migrations/upgrade-to-v1/overview
- Dev Container metadata reference（devcontainer.json 仕様）: https://containers.dev/implementors/json_reference/
- VS Code: Environment variables in Dev Containers: https://code.visualstudio.com/remote/advancedcontainers/environment-variables
- VS Code: Adding a non-root user to your Dev Container: https://code.visualstudio.com/remote/advancedcontainers/add-nonroot-user
