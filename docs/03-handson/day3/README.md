# ハンズオン フロントエンド編（Day 3 / 約3時間）

Day1・Day2 で育てたエージェントに、**TypeScript で書いたフロントエンド**を繋ぎます。

**ゴール**: エージェントを「Studio で触るもの」から「自分のアプリに組み込む部品」に変える。
Mastra が TypeScript ネイティブであることの実利 —— **サーバーとフロントで型が1本に繋がる** ——
を手を動かして確認する。

**前提**: [day2/](../day2/) まで完了していること（`weatherTool` と Memory 入りの `travel-agent/`）。

---

## なぜこの回があるのか

Day1 の [Step 5](../day1/05-http-api.md) で「エージェントはそのまま HTTP API になる」ことは確認しました。
ただし `curl` で叩くだけなら、Python 製のフレームワークでも同じことができます。

Mastra の差別化軸は「TypeScript ネイティブであること」ですが、それが効くのは**フロントエンドと
同じ言語・同じ型定義を共有できる**場面です。Day 3 はその一点に絞った回です。

- Python 製フレームワーク: サーバーの出力型とフロントの型を**別々に書いて手で同期する**
- Mastra: zod スキーマを1つ書いて、**サーバーとフロントの両方から import する**

後者は、スキーマを変えた瞬間にフロントのビルドが落ちます。「直し忘れ」が構造的に起きません。

---

## Step 一覧

| Step | 内容 | 時間 | 手順 |
|---|---|---|---|
| 13 | 型付きクライアント：分離構成（Vite + `@mastra/client-js`） | 65分 | [13-typed-client.md](./13-typed-client.md) |
| 14 | Next.js に同居させてチャット UI を作る | 95分 | [14-nextjs-chat.md](./14-nextjs-chat.md) |
| 予備 | | 20分 | |

---

## 2つの繋ぎ方の使い分け

Day 3 では意図的に**性質の違う2つの構成**を作ります。この表が Day 3 の結論です。

| | 分離構成（Step 13） | 同居構成（Step 14） |
|---|---|---|
| 構成 | Mastra サーバー（:4111）＋ フロント（:5173） | Next.js 1つ（:3000） |
| エージェントの呼び方 | HTTP（`@mastra/client-js`） | `import { mastra }` で関数呼び出し |
| 型の繋がり方 | zod スキーマを共有して繋ぐ（意識的な作業） | import なので自明に繋がる |
| プロセス / デプロイ | 2つ | 1つ |
| Mastra Studio | そのまま使える | 使えない（Next.js のログで見る） |
| 認証・CORS | 別オリジンなので設計が要る | 同一オリジンなので不要 |
| エージェントの再利用 | 複数のシステムから叩ける | そのアプリ専用になりがち |
| 向く場面 | **複数システムから使う共通のエージェント基盤**にしたいとき | **既存の Web アプリに AI 機能を足す**とき |

> 💡 「Mastra をどう組み込むか」を考えるときは、この2択から入ると整理しやすくなります。
> 「まず同居構成で1アプリに入れてみて、他システムから使いたい要望が出たら分離構成に切り出す」という
> 移行の順序も、Mastra 側のコードを書き換えずに済みます（呼び出し方が変わるだけ）。

---

## Day 3 完了チェックリスト

- [ ] `@mastra/client-js` でブラウザからエージェントを呼び、ストリーム表示できた
- [ ] エージェントと同じ zod スキーマをフロントで import し、`z.infer` の型で描画できた
- [ ] スキーマを変えるとフロントの型エラーが出ることを確認した
- [ ] API キーがブラウザに渡らない構成であることを説明できる
- [ ] Next.js の中から `import { mastra }` でエージェントを関数として呼べた
- [ ] チャット UI にツール呼び出し（`tool-get-weather`）が表示されることを確認した
- [ ] リロードしても会話履歴が復元されることを確認した
- [ ] 分離構成と同居構成の使い分けを自分の言葉で説明できる

---

## 参考リンク

| テーマ | URL |
|---|---|
| Mastra client（SDK の使い方） | https://mastra.ai/docs/server/mastra-client |
| Client SDK リファレンス（Agents API） | https://mastra.ai/reference/client-js/agents |
| Next.js 統合ガイド（本 Step の下敷き） | https://mastra.ai/docs/frameworks/web-frameworks/next-js |
| Web フレームワークへのデプロイ | https://mastra.ai/docs/deployment/web-framework |
| サーバー設定（CORS・port） | https://mastra.ai/reference/configuration |
| 既成の Agentic UI（CopilotKit / assistant-ui） | https://mastra.ai/guides/build-your-ui/copilotkit |

> チャット UI をゼロから作らず既成品に乗る選択肢もあります。CopilotKit や assistant-ui は
> Mastra との統合が用意されているので、社内ツールなら検討する価値があります。

---

**ここまででハンズオンは完了です。** お疲れさまでした。

- 困ったときは [troubleshooting/](../../troubleshooting/) へ
- Studio の画面や `devcontainer.json` を調べ直したくなったら [reference/](../../reference/) へ
- 公式ドキュメントの現在地は [reference/links.md](../../reference/links.md) にまとめてあります
