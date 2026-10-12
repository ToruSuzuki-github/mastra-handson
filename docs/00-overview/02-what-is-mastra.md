# 2. Mastra とは（4分）

**TypeScript で AI エージェントとアプリケーションを構築するためのフレームワーク**です。公式は次のように名乗っています。

> Build, observe, and improve agents that run for days with Mastra, the leading TypeScript agent framework.
> （何日も動き続けるエージェントを、作り・観測し・改善する）

## 2-1. 基本情報

| 項目 | 内容 |
|---|---|
| 開発元 | Kepler Software, Inc.（Gatsby を作ったチームが中心。Y Combinator W25 卒業） |
| 主言語 | TypeScript（Node.js 22.13.0 以上が必要） |
| ライセンス | **Apache 2.0**。ただし `ee/` という名前のディレクトリ配下（認証機能の一部）のみ別ライセンスの source-available |
| リポジトリ公開 | 2024年8月 |
| v1.0 リリース | 2026年1月 |
| GitHub | https://github.com/mastra-ai/mastra （27,000 stars 超 / 2026-08-11 時点） |
| 価格 | フレームワーク本体は無料。座席課金・利用量ティアなし。マネージドの Mastra Cloud は別途 |

<details>
<summary>📖 <b>補足：<code>ee/</code> ディレクトリと Enterprise Edition ライセンス</b>（座学では読み飛ばして構いません。本番導入を検討する段階でここに戻ってください）</summary>

上表の「`ee/` 配下のみ別ライセンス」について、社内導入を検討する際に効いてくる点を補足します（2026-08-15 時点で公式リポジトリの `LICENSE.md` および `ee/LICENSE` を直接確認）。

**どこが対象か**

`LICENSE.md` は対象を「リポジトリ内の **`ee/` という名前のディレクトリ配下すべて**」と定めており、例として `packages/core/src/auth/ee/` を挙げています。ただし *including but not limited to*（これらに限らない）という書き方なので、**将来 `ee/` ディレクトリが増えればその中身も自動的に対象になります**。逆に言えば、それ以外はすべて Apache 2.0 です。

**現時点で何が入っているか**

実体は認証まわりのごく一部です。公式ドキュメントで EE 扱いと明記されている機能は **FGA（Fine-Grained Authorization）**、つまり「この利用者は**この特定のリソースに対して**その操作をしてよいか」というリソース単位の権限チェックです。マルチテナントの B2B 製品向けの機能で、役割単位で判定する RBAC より細かい制御を行います。

> Fine-Grained Authorization is part of the Mastra Enterprise Edition. Production deployments require a valid EE license.

一方、**Auth0 / Clerk / Firebase / Okta / Supabase / WorkOS などの認証プロバイダ連携パッケージ自体はリポジトリ直下の `auth/` にあり、Apache 2.0 です**。「認証機能を使うと有償になる」わけではない点に注意してください。

**ライセンスの中身**

`ee/LICENSE` は「Mastra Enterprise Edition (EE) License」という独自ライセンスです。OSI 承認のオープンソースライセンスではなく、いわゆる source-available（ソースは公開されているが利用は制限される）に当たります。

| 項目 | 内容 |
|---|---|
| **本番利用** | Kepler Software, Inc. との**書面による契約が必要**。契約なしの本番利用は認められない |
| **開発・テスト** | **契約なしで可**。自社システム上でのローカル実行、ステージング環境での評価・改変・テストが該当する |
| 「本番」の定義 | 「開発・テストを超えるすべての利用」とライセンス本文で明示的に定義されている |
| 再配布 | **禁止**（copy / merge / publish / distribute / sublicense / sell のいずれも不可） |
| 改変 | 自社の開発・テスト目的なら可。ただし改変部分の権利は Kepler 側に帰属すると定められている |
| 違反時 | 権利は通知なく自動的に終了する。是正可能な違反なら、認識から30日以内の是正で回復する |
| 保証 | 無保証（AS IS / WITH ALL FAULTS） |

**ハンズオンおよび社内検討への影響**

本ハンズオンで扱う Agent・Tool・Workflow・Memory・Studio はすべて Apache 2.0 の範囲であり、影響はありません。
社内で本番利用を検討する段階では、**FGA を使うかどうかだけ確認すれば足ります**。使わないなら Apache 2.0 のみで完結します。

> ⚠️ ライセンス構成は変わり得ます。採用判断の際は必ず https://github.com/mastra-ai/mastra/blob/main/LICENSE.md を最新版で確認し、本番利用の可否は法務にも確認してください。

</details>

## 2-2. 公式サイトに掲載されている採用事例

| 企業 | 用途 |
|---|---|
| Replit | 毎日数千のサンドボックス実行 |
| Sanity | 30,000 以上の組織へコンテンツエージェントを展開 |
| SoftBank | 数時間かかる文書作業をプレゼン資料に変換 |
| WorkOS | 評価から本番運用まで4週間以内 |
| MongoDB | 月間 50,000 エージェント実行 |

## 2-3. Mastra を選ぶとしたらどこが理由になるか

| 特徴 | 意味 |
|---|---|
| **TypeScript ネイティブ** | Python のラッパーではなく最初から TypeScript。ツールの入出力の型が、そのままエディタの補完と型チェックに効く。**フロントエンドと同じ zod スキーマを共有できる**のが実利の出どころ（[03-handson/day3/](../03-handson/day3/) で実際に作る） |
| **必要なものが最初から入っている** | エージェント・ワークフロー・メモリ・評価・可観測性が同一パッケージ群。別サービスを契約しなくてもローカルで一通り動く |
| **ローカル GUI が同梱** | Mastra Studio（後述）が開発サーバーに付属。エージェントとの対話も、内部で何が起きたかの確認も、ブラウザでできる |
| **単体でサーバーになる** | 作ったエージェントがそのまま HTTP API として公開される。Next.js や Express に組み込むこともできる（2つの繋ぎ方の比較は [03-handson/day3/README.md](../03-handson/day3/README.md)） |

> ⚠️ 「Mastra が他より優れている」という話ではありません。用途によって適した選択肢は変わります。比較は [10. 他フレームワークとの比較](./10-comparison.md) で扱います。
