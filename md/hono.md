<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Hono 使い方

> Hono のルーティング・Context・ミドルウェア・バリデーション・RPC・エラー処理をまとめたチートシート

- URL: https://cs.taka1156.site/hono.html
- タグ: Hono, TypeScript, Cloudflare
- 日付: 2026-11-19

---

Web 標準の Request / Response で動く軽量な Web フレームワーク。Cloudflare Workers・Bun・Deno・Node.js で同じコードが動く（v4.13 時点）

**ざっくり言うと**

- Hono は `app.get(path, handler)` でルートを書く Express 風の軽量フレームワークで、多くのランタイムで動く。
- ハンドラーは Context `c` を受け取り、 `c.req` で入力を読み、 `c.json()` などで Response を返す。
- バリデーターと RPC（ `hc` ）を組み合わせると、サーバーの型をそのままクライアントで使える。

## 始め方

create-hono

Cloudflare Workers / Bun

Node.js

Deno

```
npm create hono@latest my-app   # テンプレート（cloudflare-workers・bun・nodejs …）を選ぶ
cd my-app && npm i
npm run dev
```

```
import { Hono } from 'hono';

const app = new Hono();
app.get('/', (c) => c.text('Hello Hono!'));

export default app;   // Workers・Bun はこれだけでよい
```

```
npm i hono @hono/node-server

import { serve } from '@hono/node-server';
import { Hono } from 'hono';

const app = new Hono();
app.get('/', (c) => c.text('Hello Node.js!'));

const server = serve({ fetch: app.fetch, port: 3000 });
process.on('SIGTERM', () => server.close());
```

Node.js 20 以降（ `@hono/node-server` v2 の要件。v1 系なら 18.14.1 以降でも動く）。 `serve()` は `node:http` のサーバーを返すので、終了処理は自分で書く。

```
import { Hono } from 'hono';   // deno add npm:hono

const app = new Hono();
app.get('/', (c) => c.text('Hello Deno!'));

Deno.serve(app.fetch);
```

（元のページで操作できます → https://cs.taka1156.site/hono.html#start）

## ルーティング

| 書き方 | 意味 |
| --- | --- |
| `app.get('/posts', h)` | `get` ・ `post` ・ `put` ・ `delete` ・ `patch` ・ `all` （全メソッド） |
| `app.on(['GET', 'POST'], '/x', h)` | 複数のメソッドをまとめて |
| `/posts/:id` | パスパラメータ → `c.req.param('id')` |
| `/animal/:type?` | 省略できるパラメータ |
| `/post/:date{[0-9]+}` | 正規表現で制限 |
| `/wild/*/card` ・ `/api/*` | ワイルドカード |
| `app.route('/book', book)` | 別の `Hono` インスタンスを `/book` 配下にまとめる |
| `new Hono().basePath('/api')` | 全ルートに共通の接頭辞 |
| `app.get('/a', h1).post('/b', h2)` | チェーンで続けて書ける（RPC で型をつなぐときに必要） |

先に登録したルートが優先される。 `/book/a` を `/book/:slug` より前に書く、のように具体的な方を先にする。

## Context（c）でよく使うもの

入力（c.req）

出力

変数・環境

| メソッド | 返すもの |
| --- | --- |
| `c.req.param('id')` | パスパラメータ（引数なしで全部） |
| `c.req.query('q')` ・ `c.req.queries('tag')` | クエリ文字列（同名が複数なら `queries` で配列） |
| `c.req.header('Authorization')` | リクエストヘッダー |
| `await c.req.json()` | JSON のボディ |
| `await c.req.parseBody()` | フォーム（ `multipart/form-data` ・ `x-www-form-urlencoded` ）。ファイルは `File` |
| `c.req.valid('json')` | バリデーターを通った値（型付き） |
| `c.req.raw` | 元の `Request` （他のライブラリに渡すとき） |

```
return c.json({ ok: true }, 201);          // JSON とステータス
return c.text('hello');
return c.html('<h1>Hi</h1>');
return c.redirect('/login', 302);
return c.notFound();
return c.body(stream, 200, { 'Content-Type': 'text/plain' });

c.header('X-Request-Id', id);              // 返す前にヘッダーを足す
c.status(202);
```

```
type Env = {
  Bindings: { DB: D1Database; SECRET: string };   // Workers の環境変数・バインディング
  Variables: { userId: string };                   // c.set / c.get で渡す値
};
const app = new Hono<Env>();

app.use(async (c, next) => {
  c.set('userId', 'u_1');
  await next();
});
app.get('/', (c) => c.json({ user: c.get('userId'), hasDb: !!c.env.DB }));
```

（元のページで操作できます → https://cs.taka1156.site/hono.html#context）

## ミドルウェア

`await next()` の前がハンドラーの前、後がハンドラーの後に動く。登録した順に入り、逆の順に出る。

**logger 前半** 1 番目に登録

**cors 前半** 2 番目に登録

**handler** Response を返す

**cors 後半** `next()` の後

**logger 後半** 最後に出る

```
import { logger } from 'hono/logger';
import { cors } from 'hono/cors';
import { createMiddleware } from 'hono/factory';

app.use(logger());
app.use('/api/*', cors({ origin: 'https://example.com', credentials: true }));

// 自作（型付き）
const timing = createMiddleware(async (c, next) => {
  const start = Date.now();
  await next();
  c.header('X-Response-Time', `${Date.now() - start}ms`);
});
app.use(timing);
```

### 組み込みミドルウェア（ `hono/〜` から import）

| import 元 | 用途 |
| --- | --- |
| `hono/cors` | CORS ヘッダー |
| `hono/logger` ・ `hono/request-id` ・ `hono/timing` | アクセスログ・リクエスト ID・Server-Timing |
| `hono/secure-headers` ・ `hono/csrf` | セキュリティ系ヘッダー・CSRF 対策（ `Origin` ・ `Sec-Fetch-Site` の確認） |
| `hono/basic-auth` ・ `hono/bearer-auth` ・ `hono/jwt` ・ `hono/jwk` | 認証 |
| `hono/body-limit` ・ `hono/timeout` ・ `hono/ip-restriction` | ボディサイズ・時間・IP の制限 |
| `hono/etag` ・ `hono/cache` ・ `hono/compress` | キャッシュ・圧縮 |
| `hono/combine` | `some` ・ `every` ・ `except` でミドルウェアを組み合わせる |

## バリデーション

Zod

Standard Schema

```
npm i zod @hono/zod-validator

import { zValidator } from '@hono/zod-validator';
import * as z from 'zod';

app.post(
  '/posts',
  zValidator('json', z.object({ title: z.string(), body: z.string() })),
  (c) => {
    const { title } = c.req.valid('json');   // 型付き。失敗時は 400 が自動で返る
    return c.json({ title }, 201);
  },
);
```

第 1 引数は検証する場所: `json` ・ `form` ・ `query` ・ `param` ・ `header` ・ `cookie` 。スキーマの書き方は [Zod 使い方](https://cs.taka1156.site/md/zod.md) 。

```
npm i @hono/standard-validator

import { sValidator } from '@hono/standard-validator';
// Zod・Valibot・ArkType など Standard Schema 対応のどれでも渡せる
app.post('/posts', sValidator('json', schema), (c) => { ... });
```

（元のページで操作できます → https://cs.taka1156.site/hono.html#validation）

## RPC（型付きクライアント）

```
// server.ts — チェーンで書いて、その結果の型を export する
const route = app
  .get('/posts/:id', (c) => c.json({ id: c.req.param('id'), title: 'Hi' }))
  .post('/posts', zValidator('json', NewPost), (c) => c.json({ ok: true }, 201));
export type AppType = typeof route;

// client.ts
import { hc } from 'hono/client';
import type { AppType } from './server';

const client = hc<AppType>('http://localhost:8787/');
const res = await client.posts[':id'].$get({ param: { id: '1' } });
if (res.ok) {
  const post = await res.json();   // { id: string; title: string }
}
```

型を推論させるにはサーバーとクライアントの両方で `"strict": true` が必要。レスポンスの型だけ欲しいときは `InferResponseType<typeof client.posts.$post>` 。

## エラー処理とテスト

```
import { HTTPException } from 'hono/http-exception';

app.get('/admin', (c) => {
  throw new HTTPException(401, { message: 'Unauthorized' });
});

app.notFound((c) => c.json({ error: 'Not Found' }, 404));
app.onError((err, c) => {
  if (err instanceof HTTPException) return err.getResponse();
  console.error(err);
  return c.json({ error: 'Internal Server Error' }, 500);
});

// テスト（サーバーを立てずに Request を渡す）
const res = await app.request('/posts/1');
expect(res.status).toBe(200);
```

## API サーバーを作る手順

1. **雛形を作る** `npm create hono@latest` で動かす先のランタイムを選ぶ
2. **型を決める** `new Hono<{ Bindings; Variables }>()` で環境変数と `c.set` の型を書く
3. **ルートを機能ごとに分ける** ファイルごとに `new Hono()` を作り、 `app.route('/users', users)` でつなぐ
4. **共通処理をミドルウェアにする** logger・cors・認証（ [Better Auth](https://cs.taka1156.site/md/better-auth.md) など）を `app.use` で
5. **入力を検証する** `zValidator` を各ルートに付け、 `c.req.valid()` だけを使う
6. **テストする** `app.request()` か `testClient()` （ `hono/testing` ）で Request を直接渡す

## よくある誤解

✗ 誤解: Hono は Cloudflare Workers 専用のフレームワークだ

✓ 正しくは: Web 標準の `Request` ・ `Response` だけで書かれているので、Bun・Deno・Node.js（ `@hono/node-server` ）・AWS Lambda・Vercel などでも同じコードが動く。違うのはエントリーポイントだけ。

✗ 誤解: ルートを別々の文で登録しても RPC の型はつながる

✓ 正しくは: `app.get(...)` を別々の文で書くと、 `typeof app` にルートの型が積み上がらない。RPC で使うルートは `.get().post()` とチェーンで書き、その式の型を export する。

✗ 誤解: c.req.json() で読めば型も検証も済んでいる

✓ 正しくは: `c.req.json()` はパースするだけで検証しない。外からの入力はバリデーターを通し、 `c.req.valid('json')` で受け取る。

## 出典

- [Hono – Getting Started](https://hono.dev/docs/getting-started/basic) （hono 4.13.19 時点）
- [Routing](https://hono.dev/docs/api/routing) ・ [Context](https://hono.dev/docs/api/context) ・ [HonoRequest](https://hono.dev/docs/api/request)
- [Middleware](https://hono.dev/docs/guides/middleware) ・ [Validation](https://hono.dev/docs/guides/validation) ・ [RPC](https://hono.dev/docs/guides/rpc)
- [HTTPException](https://hono.dev/docs/api/exception) ・ [Testing](https://hono.dev/docs/guides/testing) ・ [Node.js](https://hono.dev/docs/getting-started/nodejs)

確認日: 2026-11-19
