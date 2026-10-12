# 参照資料

手順書（[00-overview/](../00-overview/) 〜 [03-handson/](../03-handson/)）とは別に、
**必要になったときに開く**資料を集めた場所です。番号が付いていないのは実施順を持たないためです。

| 資料 | 内容 | どこから参照するか |
|---|---|---|
| [mastra-studio.md](./mastra-studio.md) | **Mastra Studio の画面操作と保存仕様の正典。** 左ナビの構成、Agents / Tools / Workflows / Traces 各画面の使い方、ワークフローの実行と suspend/resume、Debug トグルによるステップ実行、トレースと会話履歴の保存先 | [02-templates/](../02-templates/) / [03-handson/](../03-handson/) のどこからでも |
| [agent-skills.md](./agent-skills.md) | 公式テンプレートに同梱される `.agents/` フォルダと `skills-lock.json` の正体（Agent Skills）。`src/mastra/agents/` との違い、AI に Mastra のコードを書かせるときの使い方 | [02-templates/01-basics.md](../02-templates/01-basics.md) §2-4、[02-templates/03-model-switch.md](../02-templates/03-model-switch.md) |
| [devcontainer.json](./devcontainer.json) | **`devcontainer.json` の正典。** [01-setup/01-environment.md](../01-setup/01-environment.md) §3 はこのファイルの写しで、リポジトリ直下の `.devcontainer/devcontainer.json` も同内容 | [01-setup/](../01-setup/) |

> **正典（canonical）とは**: 同じ内容が複数箇所にある場合に「ここを直せば正」と決めた側のファイルです。
> 手順書側にだけ書いて正典を放置しないでください。
