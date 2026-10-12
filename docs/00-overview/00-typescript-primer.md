# 0. 準備：この資料のコードを読むための TypeScript（3分）

> 💡 **TypeScript を書いたことがある人はこの章を飛ばしてください。** 本資料に出てくるコードは短く、以下の4点が読めれば理解できます。

TypeScript は **JavaScript に「型」を足したもの**です。実行前に型の食い違いを検出できるのが最大の利点で、Mastra はその性質をツールの入出力にまで効かせています。

## 0-1. import / export — 部品を受け渡す

```ts
import { Agent } from '@mastra/core/agent';   // 他のパッケージ・ファイルから持ってくる
export const travelAgent = new Agent({ /* ... */ });   // 他のファイルから使えるように公開する
```

| 書き方 | 意味 |
|---|---|
| `import { X } from '...'` | 指定した名前 `X` だけを取り出す |
| `'@mastra/core/agent'` | `@` や英字で始まるものはインストール済みパッケージ |
| `'../tools/weather-tool.ts'` | `.` で始まるものは自分で書いたファイル |
| `export const X = ...` | `X` を他のファイルから import できるようにする |

> ⚠️ Mastra では**自分のファイルを import するとき拡張子 `.ts` が必要**です。ハンズオンで最もつまずきやすい点のひとつです（[troubleshooting/](../troubleshooting/) §2 参照）。

## 0-2. 型注釈 — 「この値はこの形」と宣言する

```ts
const city: string = 'Tokyo';                 // : の後ろが型
function greet(name: string): string {        // 引数の型 / 戻り値の型
  return `こんにちは、${name}さん`;            // `${ }` は文字列に値を埋め込む書き方
}
```

型が合わなければ、実行する前にエディタが赤線で教えてくれます。
ただし**型は書かなくても推論される**ことが多く、本資料のコードにも明示的な型注釈はほとんど出てきません。

Mastra では型を手書きする代わりに、**Zod**（`z.object({ ... })`）でデータの形を宣言します。
Zod はデータ検証ライブラリで、書いた形から TypeScript の型が自動的に導かれます。「1回書けば、検証と型の両方が手に入る」と考えてください。

## 0-3. async / await — 時間のかかる処理を待つ

API 呼び出しやデータベースアクセスは、結果が返るまで時間がかかります。

```ts
const res = await fetch('https://api.example.com/weather');   // 結果が返るまでここで待つ
```

この `await` は、`async` を付けた関数の中でしか使えません。

```ts
async function getWeather(city: string) {          // await を使うので async が必要
  const res = await fetch(`https://api.example.com/weather?city=${city}`);
  return res.json();                                // 戻り値は自動的に Promise に包まれる
}
```

`async` 関数を呼ぶと、返ってくるのは結果そのものではなく **Promise（あとで結果が入る箱）** です。中身を取り出すにはまた `await` します。

```ts
const promise = getWeather('Tokyo');   // 型は Promise<...>。この時点ではまだ中身は入っていない
const weather = await promise;         // await で箱を開けて中身を取り出す

const weather2 = await getWeather('Osaka');   // 実際はこの1行にまとめて書くことがほとんど
```

| 書き方 | 意味 |
|---|---|
| `await 式` | その処理が終わるまで待って、結果を受け取る |
| `async` | `await` を使う関数には必ず付ける。`execute: async ({ city }) => { ... }` の形で頻出 |
| Promise | `async` 関数の戻り値。「あとで結果が入る箱」。呼ぶ側も `await` して取り出す |

> 💡 `await` を書き忘れると、中身の代わりに Promise そのものを使ってしまい、`undefined` や `[object Promise]` になります。よくある間違いです。

## 0-4. 設定オブジェクトとアロー関数

Mastra の API は、ほぼすべて **「設定をまとめたオブジェクトを1つ渡す」** 形です。

```ts
new Agent({ id: 'travel-agent', name: 'Travel Agent', model: 'openai/gpt-5.6-luna' });
```

引数の順番を覚える必要がなく、必要な項目だけ書けば済みます。
また `(引数) => { 処理 }` は**アロー関数**（無名関数の短い書き方）で、`execute` などに処理を渡すときに使います。

以上が読めれば十分です。分からない記法が出てきても、**「何をしている行か」が掴めれば先に進んで構いません。**
