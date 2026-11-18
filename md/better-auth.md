<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Better Auth 使い方

> Better Auth のセットアップ（DB・ハンドラー・クライアント）、よく使う API、設定、プラグイン、CLI のチートシート

- URL: https://cs.taka1156.site/better-auth.html
- タグ: Better Auth, 認証, TypeScript
- 日付: 2026-11-19

---

TypeScript 用の認証ライブラリ。自分のデータベースにユーザーとセッションを持ち、メール・パスワードやソーシャルログインをプラグインで足していく（v1.7 時点）

**ざっくり言うと**

- Better Auth は自分のアプリと DB の中で動く認証ライブラリで、外部の認証サービスを使わずにログインを実装できる。
- サーバーの `auth` を `/api/auth/*` にマウントし、ブラウザは `authClient` から呼ぶとセッションの Cookie が付く。
- 2 要素認証や組織などはプラグインで足し、足したらスキーマの生成・マイグレーションをやり直す。

## 全体の構成

**authClient** ブラウザ。 `signIn` ・ `signUp` ・ `useSession`

**/api/auth/*** `auth.handler(request)` 。Cookie `better-auth.session_token` を発行

**データベース** `user` ・ `session` ・ `account` ・ `verification` の 4 テーブル

サーバー側の他の処理（API・SSR）は `auth.api.getSession({ headers })` で、同じ Cookie からセッションを読む。

## セットアップ

1. **インストール** `npm i better-auth` （クライアントとサーバーが別プロジェクトなら両方に）
2. **環境変数** `BETTER_AUTH_SECRET` （32 文字以上。 `openssl rand -base64 32` ）と `BETTER_AUTH_URL` （例 `http://localhost:3000` ）
3. **auth.ts を作る** ルート・ `lib/` ・ `utils/` （ `src/` 配下も可）に置き、 `auth` という名前か default で export する。CLI がここを探す
4. **テーブルを作る** `npx auth@latest generate` （ORM のスキーマや SQL を生成）か `npx auth@latest migrate` （組み込みの Kysely のときだけ、DB に直接適用）
5. **ハンドラーをマウント** `/api/auth/*` を `auth.handler` に渡す（下のタブ）
6. **クライアントを作る** `createAuthClient()` をフレームワーク用のパスから import する

auth.ts

DB の種類

マウント

クライアント

```
// lib/auth.ts
import { betterAuth } from 'better-auth';
import { Pool } from 'pg';

export const auth = betterAuth({
  database: new Pool({ connectionString: process.env.DATABASE_URL }),
  emailAndPassword: { enabled: true },
  socialProviders: {
    github: {
      clientId: process.env.GITHUB_CLIENT_ID as string,
      clientSecret: process.env.GITHUB_CLIENT_SECRET as string,
    },
  },
  trustedOrigins: ['http://localhost:5173'],   // 別オリジンのフロントエンドから呼ぶとき
});
```

```
// SQLite / PostgreSQL / MySQL はドライバーを直接渡す
database: new Database('./sqlite.db'),             // better-sqlite3
database: createPool({ ... }),                     // mysql2/promise

// ORM はアダプター経由
import { drizzleAdapter } from 'better-auth/adapters/drizzle';
database: drizzleAdapter(db, { provider: 'pg' }),  // 'mysql' | 'sqlite'

import { prismaAdapter } from 'better-auth/adapters/prisma';
database: prismaAdapter(prisma, { provider: 'postgresql' }),
```

アダプターを使うなら `better-auth/minimal` から `betterAuth` を import するとバンドルが小さくなる。DB を設定しない「ステートレス」も可能だが、多くのプラグインは DB が要る。

```
// Next.js App Router: app/api/auth/[...all]/route.ts
import { toNextJsHandler } from 'better-auth/next-js';
export const { GET, POST } = toNextJsHandler(auth);

// Hono
app.all('/api/auth/*', (c) => auth.handler(c.req.raw));

// Express（v5 は '/api/auth/*splat'）。express.json() はこの後に置く
import { toNodeHandler } from 'better-auth/node';
app.all('/api/auth/*', toNodeHandler(auth));
app.use(express.json());
```

Hono での詳しい使い方は [Hono 使い方](https://cs.taka1156.site/md/hono.md) 。CORS を使うなら Hono の `cors()` を auth のルートより前に置き、同じオリジンを `trustedOrigins` にも入れる。

```
// lib/auth-client.ts
import { createAuthClient } from 'better-auth/react';   // vue・svelte・solid・client（素の JS）
export const authClient = createAuthClient({
  baseURL: 'http://localhost:3000',   // 同じオリジンなら省略可
});
```

（元のページで操作できます → https://cs.taka1156.site/better-auth.html#setup）

## よく使う API

クライアント

サーバー

型

```
// 登録・ログイン（{ data, error } を返す。例外は投げない）
const { data, error } = await authClient.signUp.email({
  email, password, name, callbackURL: '/dashboard',
});
await authClient.signIn.email({ email, password, rememberMe: true });
await authClient.signIn.social({ provider: 'github', callbackURL: '/dashboard' });
await authClient.signOut();

// セッション（React などではフック。サインアウトなどが即座に反映される）
const { data: session, isPending } = authClient.useSession();
const { data } = await authClient.getSession();   // フックを使わないとき
```

```
// リクエストのヘッダーを渡してセッションを読む
const session = await auth.api.getSession({ headers: request.headers });
if (!session) return new Response('Unauthorized', { status: 401 });
session.user.id;

// サーバーから直接操作する（auth.api はレート制限の対象外）
await auth.api.signInEmail({
  body: { email, password },
  asResponse: true,          // Set-Cookie 付きの Response で受け取る
});
```

Next.js の Server Action から呼んで Cookie を設定させたいときは、 `plugins` の最後に `nextCookies()` （ `better-auth/next-js` ）を入れる。

```
type Session = typeof auth.$Infer.Session;   // { session, user }

// user に独自の列を足す
export const auth = betterAuth({
  user: {
    additionalFields: {
      role: { type: 'string', defaultValue: 'user', input: false },
    },
  },
});

// クライアントにも型を伝える（同じプロジェクトにあるとき）
import { inferAdditionalFields } from 'better-auth/client/plugins';
createAuthClient({ plugins: [inferAdditionalFields<typeof auth>()] });
```

（元のページで操作できます → https://cs.taka1156.site/better-auth.html#usage）

## 主な設定

| キー | 既定 | 意味 |
| --- | --- | --- |
| `emailAndPassword.enabled` | 無効 | メールとパスワードでの登録・ログイン |
| `emailAndPassword.requireEmailVerification` | `false` | メール確認が済むまでログインさせない |
| `emailVerification.sendVerificationEmail` | — | 確認メールを送る関数（ `{ user, url, token }` を受け取る）。タイミング攻撃を避けるため await しない |
| `socialProviders.<名前>` | — | GitHub・Google などの `clientId` ・ `clientSecret` |
| `session.expiresIn` | 7 日 | セッションの有効期限（秒） |
| `session.updateAge` | 1 日 | この間隔を過ぎて使われると期限を延長する |
| `session.cookieCache` | 無効 | `{ enabled: true, maxAge: 300 }` で短時間 Cookie にキャッシュし、DB の読み込みを減らす |
| `trustedOrigins` | `baseURL` のみ | リクエストを受け付けるオリジン（別オリジンのフロントエンド用） |
| `rateLimit` | 本番のみ 10 秒 100 回 | IP ごとのレート制限。開発環境では既定で無効 |
| `advanced.cookiePrefix` | `better-auth` | Cookie 名の接頭辞 |

## プラグイン

サーバー（ `betterAuth({ plugins })` ）とクライアント（ `createAuthClient({ plugins })` ）の両方に足す。テーブルが増えるものは、足したあとに `generate` / `migrate` をやり直す。

```
// auth.ts
import { twoFactor } from 'better-auth/plugins';
betterAuth({ appName: 'My App', plugins: [twoFactor()] });

// auth-client.ts
import { twoFactorClient } from 'better-auth/client/plugins';
createAuthClient({ plugins: [twoFactorClient({ twoFactorPage: '/two-factor' })] });

await authClient.twoFactor.enable({ password });
```

| サーバー側 | できること |
| --- | --- |
| `twoFactor()` | TOTP・バックアップコードの 2 要素認証 |
| `passkey()` | パスキー（WebAuthn）。 `@better-auth/passkey` から import |
| `magicLink()` ・ `emailOTP()` | メールのリンク・ワンタイムコードでログイン |
| `username()` | ユーザー名でのログイン |
| `admin()` | ユーザーの一覧・BAN・ロール・なりすまし |
| `organization()` | 組織・メンバー・招待・ロール（マルチテナント） |
| `jwt()` ・ `bearer()` | JWT の発行・JWKS、Cookie の代わりに Bearer トークン |
| `apiKey()` | API キーの発行と検証。 `@better-auth/api-key` から import |
| `openAPI()` | 認証 API の OpenAPI 定義を生成し、 `/api/auth/reference` で表示（ [Swagger 使い方](https://cs.taka1156.site/md/swagger.md) も参照） |

表のうち `passkey()` ・ `apiKey()` 以外は `better-auth/plugins` から import し、クライアント側は `〜Client()` （ `better-auth/client/plugins` ）を使う。

## CLI

| コマンド | 意味 |
| --- | --- |
| `npx auth@latest generate` | Prisma・Drizzle のスキーマや SQL を生成。 `--adapter prisma --dialect postgresql` で DB に接続せずに生成 |
| `npx auth@latest migrate` | 組み込みの Kysely アダプターのとき、DB にテーブルを直接作る |
| `npx auth@latest check` | 設定を確認する（ `check schema` でスキーマだけ） |
| `npx auth@latest create-admin --email …` | 管理者ユーザーを作る |
| `--config ./src/auth.ts` | 設定ファイルが既定の場所にないとき |

## よくある誤解

✗ 誤解: Better Auth はログイン用の外部サービスだ

✓ 正しくは: npm のライブラリで、自分のサーバーと自分のデータベースの中で動く。ユーザー情報は自分の DB の `user` テーブルなどに入り、外部に送られない。

✗ 誤解: プラグインは auth.ts に足すだけで使える

✓ 正しくは: クライアント側の `〜Client()` も足さないと `authClient` にメソッドが生えない。テーブルや列を増やすプラグインは、 `generate` / `migrate` でスキーマを更新しないと実行時に失敗する。

✗ 誤解: signIn が失敗すると例外が投げられる

✓ 正しくは: クライアントのメソッドは `{ data, error }` を返し、既定では例外を投げない。 `error` を確かめて画面に出す。

✗ 誤解: auth.api を呼べばクライアントからの呼び出しと同じ制限がかかる

✓ 正しくは: `auth.api` を使ったサーバー内の呼び出しは、レート制限の対象外。サーバーから呼ぶ処理を外部に公開するなら、自分で制限をかける。

## 出典

- [Better Auth – Installation](https://www.better-auth.com/docs/installation) ・ [Basic Usage](https://www.better-auth.com/docs/basic-usage) （better-auth 1.7.26 時点）
- [Session Management](https://www.better-auth.com/docs/concepts/session-management) ・ [Database](https://www.better-auth.com/docs/concepts/database) ・ [CLI](https://www.better-auth.com/docs/concepts/cli)
- [Rate Limit](https://www.better-auth.com/docs/concepts/rate-limit) ・ [Email](https://www.better-auth.com/docs/concepts/email) ・ [Cookies](https://www.better-auth.com/docs/concepts/cookies)
- [Hono Integration](https://www.better-auth.com/docs/integrations/hono) ・ [Two-Factor Authentication](https://www.better-auth.com/docs/plugins/2fa)

確認日: 2026-11-19
