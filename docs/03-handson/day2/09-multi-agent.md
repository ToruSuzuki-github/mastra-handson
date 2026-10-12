# Step 9. マルチエージェント：役割分担させる（60分）

## 9-1. 専門エージェントを作る（20分）

持ち物提案に特化したエージェントを分離します。

```ts
// src/mastra/agents/packing-agent.ts
import { Agent } from '@mastra/core/agent'

export const packingAgent = new Agent({
  id: 'packing-agent',
  name: 'Packing Agent',
  description: '天気情報をもとに旅行の持ち物リストを作成する専門エージェント',
  instructions: `
あなたは旅行の荷造りの専門家です。
与えられた天気情報と旅程から、持ち物リストを作成してください。

- 「必須」と「あると便利」に分ける
- 各アイテムに一言で理由を添える
- 荷物を最小限にする観点を常に持つ
`,
  model: 'openai/gpt-5.6-luna',
})
```

## 9-2. サブエージェントとして登録する（20分）

Mastra では、エージェントを別のエージェントの配下に置けます。

```ts
// src/mastra/agents/travel-agent.ts
import { Agent } from '@mastra/core/agent'
import { Memory } from '@mastra/memory'
import { weatherTool } from '../tools/weather-tool.ts'
import { packingAgent } from './packing-agent.ts'

export const travelAgent = new Agent({
  id: 'travel-agent',
  name: 'Travel Agent',
  instructions: `
あなたは旅行相談の窓口です。

- 天気の確認が必要なら get-weather ツールを使う
- 持ち物の提案が必要なら packingAgent に任せる
- あなた自身は全体の進行と最終的なまとめに徹する
`,
  model: 'openai/gpt-5.6-luna',
  tools: { [weatherTool.id]: weatherTool },
  agents: { packingAgent },
  memory: new Memory({ options: { lastMessages: 20 } }),
})
```

Studio で対話し、トレースで **どちらのエージェントがどの部分を担当したか** を確認します。

> サブエージェントの詳細な構成方法は公式を参照してください: https://mastra.ai/docs/subagents

## 9-3. 設計を考える（20分）

1つの巨大なエージェントに全部載せる場合と、役割分担する場合のトレードオフを整理します。

| 観点 | 単一エージェント | マルチエージェント |
|---|---|---|
| プロンプトの管理 | 長大になり破綻しやすい | 責務ごとに短く保てる |
| コスト | 呼び出し1回 | エージェント間の受け渡しで増える |
| デバッグ | どこで間違えたか分かりにくい | 責任範囲が明確 |
| レイテンシ | 速い | 遅くなる |

**チェックポイント**
- [ ] サブエージェントが呼ばれることをトレースで確認した
- [ ] 単一 vs 分割のトレードオフを説明できる
