# 5. Tool — エージェントに持たせる道具（4分）

ツールは `createTool()` で定義します。素のオブジェクトで書くとエラーにならないまま認識されないため、必ずこの関数を使います。

```ts
import { createTool } from '@mastra/core/tools';
import { z } from 'zod';

export const weatherTool = createTool({
  id: 'get-weather',
  description: '指定された都市の現在の天気を取得する',   // ← LLM はこの文章を読んで使うか決める
  inputSchema: z.object({ city: z.string().describe('都市名') }),
  outputSchema: z.object({ temperature: z.number(), condition: z.string() }),
  execute: async ({ city }) => {
    // 第1引数は inputSchema で検証済みの入力そのもの。ここで実際の API を呼ぶ
    return { temperature: 12, condition: '雨' };
  },
});
```

| 要素 | 役割 |
|---|---|
| `id` | ツールの識別子。トレースやエラーの表示に使われる。**LLM に見える名前ではない**（下の⚠️） |
| `description` | **どんなときに使う道具かの説明**。LLM はここだけを頼りに使うかどうかを判断する |
| `inputSchema` | 引数の形（Zod）。LLM への指示にもなり、返ってきた引数の検証にも使われる |
| `outputSchema` | 戻り値の形。TypeScript の型がここから導かれる |
| `execute` | 実際の処理。API 呼び出し、DB クエリ、計算など |

> ⚠️ **LLM に見えるツール名は `id` ではなく、`tools` に登録したときの「キー」です。**
> `tools: { weatherTool }` と書けば、`createTool({ id: 'get-weather' })` としていても LLM には
> `weatherTool` という名前で渡ります。チャット UI のツール表示（`tool-weatherTool`）も同じです。
> `id` の名前で揃えたいなら `tools: { [weatherTool.id]: weatherTool }` のように登録します
> （[4章](./04-agent.md) の「`tools` は配列ではなくオブジェクト」も参照）。

## 5-1. `description` が品質を決める

ツールが呼ばれない・間違ったツールが呼ばれるときの原因は、たいてい `description` です。
「何をするか」だけでなく **「どういうときに使うか」** を書くと、選択の精度が上がります。
`inputSchema` の各フィールドに `.describe()` を付けるのも同じ理由です。

## 5-2. 既製ツールを繋ぐ手段としての MCP

自作せずとも、**MCP**（Model Context Protocol）に対応した外部ツール群をそのままエージェントに接続できます。
GitHub、Slack、各種 SaaS の MCP サーバーが公開されており、自前実装なしでエージェントの手足を増やせます。ハンズオンでは扱いませんが、実務では最初に検討する価値があります。
