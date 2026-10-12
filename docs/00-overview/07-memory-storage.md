# 7. Memory と Storage — 会話を覚えさせる（5分）

エージェントは既定では前回のやり取りを覚えていません。「さっき言った東京の件だけど」が通じるようにするのが Memory です。

```ts
import { Agent } from '@mastra/core/agent';
import { Memory } from '@mastra/memory';

export const memoryAgent = new Agent({
  id: 'memory-agent',
  name: 'Memory Agent',
  instructions: '...',
  model: 'openai/gpt-5.6-luna',
  memory: new Memory({ options: { lastMessages: 20 } }),   // 直近20件を文脈に含める
});
```

## 7-1. 記憶の4つの種類

| 種類 | 内容 |
|---|---|
| **Message History** | ユーザー発言・エージェント応答・ツール結果をそのまま保持する。最も基本的な記憶 |
| **Working Memory** | 名前・好み・目標といった構造化されたユーザー情報を永続保持する。会話をまたいで参照される |
| **Semantic Recall** | 意味の近さで過去メッセージを検索して取り出す。キーワードが一致しなくても関連発言を拾える |
| **Observational Memory** | 背後のエージェントが古いメッセージを観察ログへ圧縮する。長い会話でも文脈ウィンドウを小さく保てる |

## 7-2. thread と resource

会話は2つの ID で区別されます。

| ID | 意味 |
|---|---|
| **thread** | 1つの会話スレッド。チャットの「1部屋」に相当 |
| **resource** | 会話の持ち主。通常はユーザー ID。同じ人の複数スレッドを横断する情報はここに紐づく |

## 7-3. Storage が必須

Memory を使うには保存先（Storage）が要ります。`Mastra` インスタンスに設定します。

```ts
import { Mastra } from '@mastra/core';
import { LibSQLStore } from '@mastra/libsql';

export const mastra = new Mastra({
  agents: { memoryAgent },
  storage: new LibSQLStore({ id: 'mastra-storage', url: 'file:./mastra.db' }),
});
```

ローカル開発では LibSQL（SQLite 互換）のファイル1つで完結します。本番では PostgreSQL などに差し替えられます。
Storage は会話履歴だけでなく、ワークフローのスナップショットやトレースの保存先も兼ねます。
