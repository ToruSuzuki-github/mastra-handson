# mastra-handson

[Mastra](https://mastra.ai/)（TypeScript の AI エージェントフレームワーク）を学ぶための、日本語のハンズオン手順書です。
座学で全体像を掴み、DevContainer で環境を作り、旅行アシスタントのエージェントを Day1〜Day3 で少しずつ育てていきます。

> 本リポジトリは個人が作成した非公式の学習資料であり、Mastra の開発元とは関係ありません。
> 記載内容は各ドキュメントに記した調査日時点のものです。Mastra は更新が速いため、最新の情報は
> [公式ドキュメント](https://mastra.ai/docs) を確認してください。

## はじめに読む

手順書の目次と進め方は [docs/README.md](./docs/README.md) を参照してください。
Mastra が初めてなら、まず [docs/00-overview/](./docs/00-overview/)（座学 67分）で全体像を掴んでから手を動かすのが効率的です。
時間が取れない場合は [docs/02-templates/](./docs/02-templates/) の 60分コースから始められます。

## 必要なもの

- Docker と VS Code（Dev Containers 拡張機能）。手順書は Windows 11 Home + WSL 2 + Docker Desktop を想定しています
- OpenAI の API キー（有料。Gemini で進める代替案もあります）

詳細は [docs/01-setup/](./docs/01-setup/) を参照してください。

## 使い方

```bash
git clone <このリポジトリの URL>
cd mastra-handson
cp .env.example .env   # .env に API キーを書く（.env は Git にコミットしない）
```

VS Code でフォルダを開き、「Dev Containers: Reopen in Container」を実行すると、
[.devcontainer/devcontainer.json](./.devcontainer/devcontainer.json) の定義で Node.js 24 の開発環境が立ち上がります。
以降は [docs/README.md](./docs/README.md) の順に進めてください。

## ディレクトリ構成

```
mastra-handson/
├── docs/                  ハンズオン手順書一式（フェーズ別フォルダ）
│   ├── 00-overview/       座学（事前学習）
│   ├── 01-setup/          環境構築と LLM の準備
│   ├── 02-templates/      公式テンプレート実行（最短ルート）
│   ├── 03-handson/        ハンズオン本体（day1/ + day2/ + day3/）
│   ├── troubleshooting/   トラブルシューティング & FAQ（随時参照）
│   └── reference/         参照資料・参考リンク集（随時参照）
├── .devcontainer/         DevContainer 定義
├── .env.example           環境変数のテンプレート
├── .gitignore             Git の追跡から外すファイルの一覧（.env など）
└── LICENSE                ライセンス（CC BY 4.0）
```

番号付きフォルダが実施推奨順、番号なしフォルダは随時参照です。

ハンズオンで作るプロジェクト（`travel-agent/` など）は本リポジトリには含まれていません。
手順書に沿って、リポジトリ直下に自分で作成します。

## ライセンス

本リポジトリの内容（手順書の文章、コード例、設定ファイル）は
[クリエイティブ・コモンズ 表示 4.0 国際（CC BY 4.0）](https://creativecommons.org/licenses/by/4.0/deed.ja) で提供します。
全文は [LICENSE](./LICENSE) を参照してください。

クレジットを表示すれば、商用・非商用を問わず、複製・改変・再配布ができます。表示の例:

```
出典: mastra-handson（© 2026 ToruSuzuki-github, CC BY 4.0）<このリポジトリの URL>
改変した場合は、その旨も併記してください。
```

手順書の中で引用している公式ドキュメントの記述や、公式テンプレートのコードは、それぞれの著作者の
権利とライセンスに従います。
