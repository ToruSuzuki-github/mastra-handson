# Step 5. HTTP API として叩く（30分）

`mastra dev` は Studio だけでなく、エージェントを叩ける HTTP エンドポイントも公開しています。

## 5-1. エンドポイントを確認する（10分）

`mastra dev` を起動した状態で、まず利用可能なエンドポイントを確認します。

```bash
curl http://localhost:4111/api/agents
```

> **注意**: API のパスはエージェントの ID / 登録キーに依存します。公式ガイドの記載とも食い違うことがあるため、**必ず上記で実際のパスを確認**してください。

## 5-2. curl で呼ぶ（10分）

```bash
curl -X POST http://localhost:4111/api/agents/travel-agent/generate \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      { "role": "user", "content": "来週、金沢に2泊で行きます。何を持っていけばいい？" }
    ]
  }'
```

<details>
<summary>💡 この <code>curl</code> コマンドは何をしている？ <code>-H</code> / <code>-d</code> とは？（クリックで開く）</summary>

`curl` は、ターミナルから **HTTP リクエストを送るコマンド**です。ブラウザの代わりに URL へアクセスし、
返ってきた内容をそのまま表示します。5-1 のように URL だけ渡せば `GET`、データを送るときは `POST` を使います。

| オプション | 意味 | このコマンドでの役割 |
|---|---|---|
| `-X POST` | HTTP メソッドを指定する（`--request` の短縮形） | エージェントに「メッセージを送る」ので `POST` にする |
| `-H "..."` | リクエストの**ヘッダー**を追加する（`--header` の短縮形） | `Content-Type: application/json` で「送る中身は JSON です」とサーバーに伝える |
| `-d '...'` | リクエストの**ボディ（送るデータ）**を指定する（`--data` の短縮形） | エージェントへの質問を `messages` として JSON で渡す |

- **`-H`**: 付け忘れると、サーバーが中身を JSON として読めずエラーになることがあります。ヘッダーは複数指定でき、
  その場合は `-H` を繰り返します（認証トークンを付ける `-H "Authorization: Bearer ..."` など）。
- **`-d`**: `-d` を使うと、`-X POST` を省いても自動で `POST` になります。ただし手順書では意図を明示するため `-X POST` を残しています。
  JSON 全体を `'...'`（シングルクォート）で囲むと、中の `"` をエスケープせずに書けます。
- 行末の `\` は「コマンドが次の行へ続く」という意味のシェルの記法で、長いコマンドを読みやすく改行しているだけです。

</details>

<details>
<summary>💡 HTTPS で呼びたい場合は何が必要？（クリックで開く）</summary>

ここまでの `http://localhost:4111` は暗号化されない HTTP です。手元の学習では問題ありませんが、
社内システムから呼ぶ・インターネットに公開する場合は HTTPS が必要になります。
やることは**ローカルで試すのか、公開するのか**で変わります。

**① ローカルの `mastra dev` を HTTPS にする**

| 方法 | やること | 証明書 |
|---|---|---|
| フラグ | `npx mastra dev --https`（短縮形は `-s`） | CLI が自動で生成する（`localhost` 向け） |
| 設定 | `src/mastra/index.ts` の `server.https` に `key` と `cert` を渡す | 自分で用意する（`mkcert` など） |

両方を指定すると**設定側が優先**されます（起動時に警告が出ます）。設定の書き方は次のとおりです。

```ts
import { readFileSync } from 'node:fs'

export const mastra = new Mastra({
  // ...（agents などはそのまま）
  server: {
    https: {
      key: readFileSync('./certs/localhost-key.pem'),
      cert: readFileSync('./certs/localhost.pem'),
    },
  },
})
```

HTTPS にしたあとは次の点も変わります。

- **呼び出し先の URL** を `https://localhost:4111/...` に直す（curl も Studio もフロントも）
- **自己署名の証明書は curl に拒否される。** 試すだけなら `curl -k`（証明書の検証を無効化）で通る。
  ローカル CA を作る `mkcert` で発行した証明書なら、CA を信頼させれば `-k` は要らない
- ⚠️ **DevContainer では、ブラウザが Windows 側で動く**ため、コンテナ内で生成した証明書はそのままでは
  ブラウザに信頼されません。`--https` の自動生成がコンテナ内で信頼登録まで済ませられるかは
  **未検証**です。確実なのは、Windows 側で `mkcert` を実行して証明書を作り、`server.https` で読み込む方法です

**② 公開・本番で HTTPS にする**

Mastra のサーバー自体には TLS を持たせず、**手前のリバースプロキシで終端する**のが一般的です。

1. ドメインを取得し、DNS をサーバーに向ける
2. Caddy / nginx / ALB / Cloudflare などを Mastra の前に置き、証明書（Let's Encrypt など）を自動取得・更新させる
3. プロキシから `http://localhost:4111` へ転送する。ストリーミング応答（`/stream`）は、プロキシ側の
   バッファリングを切らないと分割して届かない
4. フロントを別オリジンに置くなら **CORS を絞る**（既定は `origin: '*'` で全オリジンを許可）
5. **認証を付ける。** HTTPS は通信を暗号化するだけで、「誰でも呼べる」状態は変わらない

</details>

## 5-3. 意味を考える（10分）

ここまでで「Mastra で作ったエージェントは、そのまま HTTP サービスとして他システムから呼べる」ことが確認できました。実務への適用を考えるとき、この点が重要になります。

- 既存の社内 Web アプリからエージェントを呼べる
- Slack ボットや業務システムのバックエンドとして組み込める
- フロントエンドのフレームワーク統合ガイドも用意されている（Next.js / React / Express / Hono など）

> 💡 **ここは `curl` で止めますが、実際に TypeScript のフロントエンドから呼ぶのが
> [day3/](../day3/)（フロントエンド編）です。** `curl` で叩けるだけなら他言語のフレームワークでも
> 同じで、Mastra の差が出るのは「エージェント側の zod スキーマをフロントでもそのまま使える」
> ところです。Day3 [Step 13](../day3/13-typed-client.md) でそこを扱います。

**チェックポイント**
- [ ] `/api/agents` でエンドポイント一覧を取得できた
- [ ] curl でエージェントから応答を得た
