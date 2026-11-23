<!-- scripts/memo.py build が自動生成。手で編集しない -->

# MSW 使い方

> MSW v3 のハンドラーの書き方、Vitest・ブラウザ・Vite・Playwright・Storybook への組み込み、v2 からの移行をまとめたチートシート。

- URL: https://cs.taka1156.site/msw.html
- タグ: 開発ツール
- 日付: 2026-11-24

---

ネットワークの層でリクエストを横取りして、モックの返事を返す API モックライブラリ Mock Service Worker のチートシート（v3.0 系・2026-11 時点）

**ざっくり言うと**

- fetch や axios をそのまま使ったまま、HTTP・GraphQL・WebSocket・SSE の返事をモックに差し替えるライブラリ。
- ハンドラーを 1 回書けば、Vitest・開発中のブラウザ・Storybook・Playwright で同じモックを使い回せる。
- v3 は ESM のみ・Node.js 22.12 以上で、 `http` は `msw/http` から読み込むなど v2 から書き方が変わった。

## どこで動かすか

**src/mocks/handlers.ts** `http.get(...)` などのハンドラーの配列。環境に依存しない

同じハンドラーを、それぞれの入口に渡す

**Vitest・Node** `setupServer` （ `msw/node` ）

**開発中のブラウザ** `setupWorker` （ `msw/browser` ）。Service Worker を使う

**Vite のアプリ** `msw/vite` プラグインと `virtual:msw`

**Playwright** `@msw/playwright` のフィクスチャ

**Storybook** `msw-storybook-addon` とストーリーの `parameters.msw`

要件（v3.0.7）: Node.js 22.12 以上（22・24・26 でテスト）、TypeScript 5.9 以上。パッケージは ESM のみ。

## 導入

1. **入れる** `npm install msw --save-dev`
2. **ハンドラーを書く** `src/mocks/handlers.ts` にハンドラーの配列を書いて export する（下の「ハンドラーの書き方」）
3. **環境ごとに組み込む** Node なら `setupServer(...handlers)` 、ブラウザなら `setupWorker(...handlers)` を作り、有効にする（下の「組み込み方」）
4. **ブラウザで使うなら** Service Worker のファイルを公開フォルダに置く: `npx msw init ./public --save` 。Vite の `msw/vite` プラグインを使うときは不要

## ハンドラーの書き方

基本

リクエストを読む

返事の作り方

遅延・素通し

GraphQL

```
// src/mocks/handlers.ts
import { http, HttpResponse } from 'msw/http'

export const handlers = [
  http.get('https://api.example.com/user', () => {
    return HttpResponse.json({ id: 'abc-123', name: 'John' })
  }),
  http.post('/api/todos', async ({ request }) => {
    const todo = await request.json()
    return HttpResponse.json({ id: 1, ...todo }, { status: 201 })
  }),
]
```

| 書き方 | 意味 |
| --- | --- |
| `http.get` / `post` / `put` / `patch` / `delete` / `head` / `options` | メソッドごとのハンドラー |
| `http.all('/api/*', …)` | メソッドを問わない（独自メソッドもこれ） |
| `'/user/:id'` | パスパラメーター。 `params.id` で読む |
| `'https://api.example.com/*'` | `*` はワイルドカード。正規表現も渡せる |
| 相対 URL（ `'/api/user'` ） | ブラウザでは `location.href` 基準で解決。Node では基準がないので絶対 URL で書くか、ベース URL を設定する |
| `{ once: true }` （第 3 引数） | 最初の 1 回だけ使うハンドラー |

クエリ文字列はパスに書かない（書いても取り除かれる）。中身はハンドラーの中で `new URL(request.url).searchParams` から読む。

```
http.post('/api/posts/:postId/comments', async ({ request, params, cookies }) => {
  params.postId                                  // パスパラメーター（文字列）
  new URL(request.url).searchParams.get('sort')  // クエリ
  request.headers.get('authorization')           // ヘッダー
  cookies.session                                // Cookie
  const body = await request.json()              // 本文（text() / formData() も）
  return HttpResponse.json({ ok: true })
})
```

ハンドラーが受け取る `request` は Fetch API の `Request` そのもの。 `requestId` も渡される。

（元のページで操作できます → https://cs.taka1156.site/msw.html#handlers）

### 型を付ける

```
http.post<{ postId: string }, { comment: string }, { url: string }>(
  '/api/posts/:postId/comments',
  async ({ params, request }) => {
    const { comment } = await request.json()   // { comment: string }
    return HttpResponse.json({ url: `/posts/${params.postId}` })
  },
)
// ジェネリクスの順番: パスパラメーター, リクエスト本文, レスポンス本文
```

| 書き方 | 返すもの |
| --- | --- |
| `HttpResponse.json(data, init)` | JSON。 `init` は `{ status, headers }` |
| `HttpResponse.text('ok')` | テキスト |
| `HttpResponse.html(…)` / `xml(…)` | HTML・XML |
| `HttpResponse.formData(fd)` / `arrayBuffer(buf)` | フォーム・バイナリ |
| `new HttpResponse(null, { status: 404 })` | 本文なしのエラー応答 |
| `HttpResponse.json({ message }, { status: 500 })` | 本文つきのエラー応答 |
| `HttpResponse.error()` | ネットワークエラー（オフライン・DNS 失敗など）。fetch が reject される |
| 何も返さない | このハンドラーでは決めず、次に当てはまるハンドラーへ回す |

`HttpResponse` は標準の `Response` を広げたもの。 `Set-Cookie` をモックの返事に付けられるなど、標準の `Response` ではできないことに対応している。

```
import { delay } from 'msw/utils/delay'
import { passthrough } from 'msw/utils/passthrough'
import { bypass } from 'msw/utils/bypass'

http.get('/api/slow', async () => {
  await delay()            // 実際のサーバーらしい待ち時間（ランダム）
  await delay(1000)        // 1 秒
  // await delay('infinite') でずっと返さない（読み込み中の表示を確かめる）
  return HttpResponse.json({ ok: true })
})

// 条件によって本物に送る
http.get('/api/user', ({ request }) => {
  if (request.headers.has('x-real')) return passthrough()
  return HttpResponse.json({ name: 'mock' })
})

// 本物の返事を取ってきて一部だけ書き換える
http.get('/api/user', async ({ request }) => {
  const real = await fetch(bypass(request)).then((r) => r.json())
  return HttpResponse.json({ ...real, plan: 'pro' })
})
```

ハンドラーの中で同じ URL を普通に `fetch` すると、自分自身に横取りされる。本物に送るときは `bypass(request)` で包む。

```
import { HttpResponse } from 'msw/http'
import { graphql } from 'msw/graphql'

const api = graphql.link('https://api.example.com/graphql')   // v3 はエンドポイントを先に決める

export const handlers = [
  api.query('GetUser', ({ variables }) =>
    HttpResponse.json({ data: { user: { id: variables.id, name: 'John' } } })),

  api.mutation('Login', () =>
    HttpResponse.json({ errors: [{ message: 'Invalid credentials' }] })),

  api.subscription('OnMessage', ({ subscription }) => {     // v3 から
    subscription.publish({ data: { message: { text: 'hi' } } })
  }),
]
```

v2 の `graphql.query()` などは v3 で削除された。 `graphql.link(url)` で作ったリンクにハンドラーを付ける。WebSocket は `msw/ws` の `ws.link()` 、Server-Sent Events は `msw/sse` の `sse()` で書く。

## 組み込み方

Vitest・Node

ブラウザ

Vite プラグイン

Playwright

Storybook

```
// src/mocks/node.ts
import { setupServer } from 'msw/node'
import { handlers } from './handlers'
export const server = setupServer(...handlers)

// vitest.setup.ts
import { beforeAll, afterEach, afterAll } from 'vitest'
import { server } from './src/mocks/node'

beforeAll(() => server.listen({ onUnhandledFrame: 'error' }))
afterEach(() => server.resetHandlers())   // テストで足した差し替えを消す
afterAll(() => server.close())

// vitest.config.ts
export default defineConfig({ test: { setupFiles: ['./vitest.setup.ts'] } })
```

Node 側は Service Worker を使わず、プロセス内で通信を横取りする。テストランナーに依存しないので、Jest やふつうの Node スクリプトでも同じ `setupServer` を使う。

```
// npx msw init ./public --save   （mockServiceWorker.js を公開フォルダに置く）

// src/mocks/browser.ts
import { setupWorker } from 'msw/browser'
import { handlers } from './handlers'
export const worker = setupWorker(...handlers)

// src/main.tsx
async function enableMocking() {
  if (process.env.NODE_ENV !== 'development') return
  const { worker } = await import('./mocks/browser')
  return worker.start()          // 登録が終わるまで待ってから描画する
}
enableMocking().then(() => { /* アプリを描画 */ })
```

成功すると開発者ツールのコンソールに `[MSW] Mocking enabled.` と出る。 `/mockServiceWorker.js` が 404 になるなら、 `init` で指定したフォルダが公開されていない。 `--save` を付けると `package.json` の `msw.workerDirectory` に記録される。v3 はインストールしても自動では更新しないので、 `msw` を上げたら `npx msw init` （引数なしなら記録したフォルダ全部）を実行し直す。

```
// vite.config.ts
import { defineConfig } from 'vite'
import { msw } from 'msw/vite'
export default defineConfig({ plugins: [msw()] })

// src/main.ts（アプリ本体は src/app.ts に移す）
if (import.meta.env.DEV) {
  const { network } = await import('virtual:msw')
  const { handlers } = await import('./mocks/handlers')
  network.configure({ handlers })
  await network.enable()
}
await import('./app')

// src/vite-env.d.ts（型が見つからないとき）
/// <reference types="msw/vite/client" />
```

- v3 で追加。開発中だけ `/mockServiceWorker.js` を配信するので、 `npx msw init` も公開フォルダへのコピーも要らない
- `virtual:msw` はブラウザでも SSR のサーバー側でも同じ書き方で有効になる
- 本番に含めないよう、セットアップは `import.meta.env.DEV` で囲む。囲めば `virtual:msw` の読み込みが消え、ワーカーも出力されない（囲まないと本番にもワーカーが出力される）
- 今の `setupWorker` の書き方を残したいなら `msw({ mode: 'worker-only' })` （ワーカーの配信だけ）

```
// npm i -D msw @msw/playwright
// tests/fixtures.ts
import { test as base } from '@playwright/test'
import type { AnyHandler } from 'msw'
import { defineNetworkFixture, type NetworkFixture } from '@msw/playwright'
import { handlers } from '../src/mocks/handlers'

export const test = base.extend<{ handlers: AnyHandler[]; network: NetworkFixture }>({
  handlers: [handlers, { option: true }],
  network: [
    async ({ context, handlers }, use) => {
      const network = defineNetworkFixture({ context, handlers })
      await network.enable()
      await use(network)
      await network.disable()
    },
    { auto: true },
  ],
})

// テスト側
test('ダッシュボードを表示', async ({ network, page }) => {
  network.use(http.get('/user', () => HttpResponse.json({ name: 'John' })))
  await page.goto('/dashboard')
})
```

`@msw/playwright` （0.x）は中で Playwright の `context.route()` ・ `context.routeWebSocket()` を使うので、Service Worker のファイルは要らない。テスト（Node）からブラウザ側のモックを直接差し替えられる。

```
// npm i -D msw msw-storybook-addon && npx msw init ./public --save

// .storybook/main.ts
export default { addons: ['msw-storybook-addon'] }

// .storybook/preview.ts
import { mswLoader } from 'msw-storybook-addon/csf3'
export default { loaders: [mswLoader()] }

// UserProfile.stories.ts
export const Default = {
  parameters: {
    msw: [http.get('/user', () => HttpResponse.json({ name: 'John Maverick' }))],
  },
}
```

`./public` は Storybook の静的ファイルのフォルダ（ `staticDirs` ）に合わせる。

（元のページで操作できます → https://cs.taka1156.site/msw.html#integrations）

## テストの中で差し替える

```
test('取得に失敗したらエラーを出す', async () => {
  server.use(                                          // 先頭に足す（初期のハンドラーより優先）
    http.get('https://api.example.com/user', () => new HttpResponse(null, { status: 500 })),
  )
  // ...
})
```

| メソッド | やること |
| --- | --- |
| `server.use(...handlers)` | ハンドラーを先頭に足す。ブラウザでは `worker.use()` |
| `server.resetHandlers()` | `use()` で足したものを消し、初期のハンドラーに戻す。引数を渡すと初期のハンドラーもそれに置き換える |
| `server.restoreHandlers()` | `once` のハンドラーを使う前の状態に戻す |
| `server.listHandlers()` | 今のハンドラーの一覧 |
| `server.boundary(fn)` | `fn` の中で行った `use()` などをその中だけに閉じ込める（Node のみ）。並列テストで差し替えがほかのテストに漏れないように |
| `server.events.on('request:start', …)` | リクエストの開始・一致・終了などを見る（デバッグ用） |

ハンドラーは先頭から順に照らし合わされ、最初に返事を返したものが使われる。 `use()` で足したものは先頭に入るので、初期のハンドラーより先に当たる。

## ハンドラーがないリクエスト

ハンドラーに当てはまらないリクエストは、既定ではそのまま本物に送られ、警告が出る。 `server.listen()` ・ `worker.start()` の `onUnhandledFrame` （v2 の `onUnhandledRequest` ）で変える。

| 値 | 動き |
| --- | --- |
| `'warn'` （既定） | 警告を出して、そのまま送る |
| `'error'` | エラーを出して止める。テストでモック漏れを見つけたいとき |
| `'bypass'` | 何も出さずにそのまま送る |
| `({ frame, defaults }) => {…}` | 自分で判断し、 `defaults.warn()` / `defaults.error()` を呼ぶ。この形にすると画像などの静的ファイルを無視する既定の動きも外れる |

## CLI・ユーティリティ

| コマンド・関数 | やること |
| --- | --- |
| `npx msw init <PUBLIC_DIR> --save` | `mockServiceWorker.js` を公開フォルダにコピーし、場所を `package.json` に記録 |
| `npx msw init <PUBLIC_DIR> --no-save` | 記録せずにコピー（CI など、聞き返されたくないとき） |
| `delay()` （ `msw/utils/delay` ） | 返事を遅らせる。引数なし・ミリ秒・ `'infinite'` |
| `passthrough()` （ `msw/utils/passthrough` ） | ハンドラーの中から、そのリクエストを本物に送る |
| `bypass(request)` （ `msw/utils/bypass` ） | 横取りされないリクエストを作る（本物の返事を取りたいとき） |
| `getResponse(handlers, request)` （ `msw` ） | ハンドラーに照らしてモックの返事を得る（自前の仕組みに組み込むとき） |

## v2 から v3 で変わったこと

| 項目 | v2 | v3 |
| --- | --- | --- |
| 読み込み元 | `import { http, HttpResponse, delay } from 'msw'` | `msw/http` ・ `msw/graphql` ・ `msw/ws` ・ `msw/sse` ・ `msw/utils/delay` など。 `http` ・ `delay` などは互換のため `msw` からも読み込めるが、次のメジャーで外す予定と発表されている。 `graphql` は `msw/graphql` からだけ |
| GraphQL | `graphql.query('GetUser', …)` | `graphql.link(url).query('GetUser', …)` 。subscription に対応 |
| 当てはまらないリクエスト | `onUnhandledRequest` 。コールバックは `(request, print)` | `onUnhandledFrame` （WebSocket も対象）。コールバックは `({ frame, defaults })` |
| 止める | `worker.stop()` | `await worker.stop()` （Promise を返す） |
| 二重の開始・停止 | エラーにならない | 開始済みで `listen()` ・ `start()` を呼ぶとエラー（停止も同じ） |
| イベント名 | `'connection'` | `'websocket:connection'` 。 `'websocket:error'` が増えた |
| 環境 | CJS・ESM、Node.js 18 以上 | ESM のみ、Node.js 22.12 以上、TypeScript 5.9 以上。 `graphql` は任意の peer 依存に |
| Vite | ワーカーを `public/` にコピー | `msw/vite` プラグインが配信 |
| React Native | `msw/native` | `@msw/react-native` に分離 |

移行用の codemod が公式に用意されている（移行ガイドからリンク）。

## Playwright の page.route() との使い分け

| 観点 | MSW | Playwright の `page.route()` |
| --- | --- | --- |
| 使える場所 | Vitest・ブラウザでの開発・Storybook・Playwright | Playwright のテストの中だけ |
| 書き方 | `http.get(url, resolver)` のハンドラーの配列 | `route.fulfill()` などをテストごとに書く |
| 向いている場面 | 同じモックを単体テスト・画面の開発・E2E で使い回したい | E2E だけで、そのテストの中で少し差し替えたい |

## よくある誤解

✗ 誤解: MSW を使うにはアプリの fetch を書き換える必要がある

✓ 正しくは: アプリのコードは変えない。ブラウザでは Service Worker、Node ではプロセス内の横取りで、送られたリクエストに返事を返す。fetch・axios などどのクライアントでもそのまま動く。

✗ 誤解: Node のテストでも npx msw init で Service Worker を置く必要がある

✓ 正しくは: `mockServiceWorker.js` が要るのはブラウザの `setupWorker` だけ。Vitest などの Node では `setupServer` を使い、ファイルは要らない。Vite のプラグインや `@msw/playwright` を使うときも置かなくてよい。

✗ 誤解: ハンドラーがないリクエストはエラーになるのでモック漏れにすぐ気づける

✓ 正しくは: 既定（ `'warn'` ）では警告を出して本物に送るだけで、テストは止まらない。モック漏れを失敗にしたいなら `server.listen({ onUnhandledFrame: 'error' })` にする。

✗ 誤解: server.use() で足したハンドラーはそのテストが終わると自動で消える

✓ 正しくは: 消えない。 `afterEach` で `server.resetHandlers()` を呼んで戻す。呼ばないと、次のテストにも差し替えが残る。

## 関連メモ

- [Playwright 使い方](https://cs.taka1156.site/md/playwright.md) — E2E では `@msw/playwright` で同じハンドラーを使うか、 `page.route()` で直接差し替える
- [Vitest 使い方](https://cs.taka1156.site/md/vitest.md) — `setupFiles` で `setupServer` を有効にする場所
- [Storybook 使い方](https://cs.taka1156.site/md/storybook.md) — `msw-storybook-addon` でストーリーごとに API の返事を変える
- [Hey API openapi-ts 使い方](https://cs.taka1156.site/md/hey-api-openapi-ts.md) — OpenAPI から MSW のハンドラーを作るプラグインがある

## 参考

- [Quick start | Mock Service Worker](https://mswjs.io/docs/quick-start)
- [2.x → 3.x | Mock Service Worker](https://mswjs.io/docs/migrations/2.x-to-3.x)
- [Introducing MSW 3.0](https://mswjs.io/blog/introducing-msw-3.0)
- [Default behaviors | Mock Service Worker](https://mswjs.io/docs/defaults)
- [http | Mock Service Worker](https://mswjs.io/docs/api/http)
- [HttpResponse | Mock Service Worker](https://mswjs.io/docs/api/http-response)
- [listen() | Mock Service Worker](https://mswjs.io/docs/api/setup-server/listen)
- [Browser integration | Mock Service Worker](https://mswjs.io/guides/integrations/browser)
- [Vite integration | Mock Service Worker](https://mswjs.io/guides/integrations/vite)
- [Storybook integration | Mock Service Worker](https://mswjs.io/guides/integrations/storybook)
- [TypeScript | Mock Service Worker](https://mswjs.io/guides/best-practices/typescript)
- [@msw/playwright（GitHub）](https://github.com/mswjs/playwright)

2026-11-24 に確認（msw 3.0.7・@msw/playwright 0.7.0 時点）。
