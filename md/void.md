<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Void チートシート

> VoidZero 製の Vite フルスタックフレームワーク Void の、ファイル規約・API・void CLI を用途別にまとめたチートシート。

- URL: https://cs.taka1156.site/void.html
- タグ: Void, Cloudflare, Vite
- 日付: 2026-11-13

---

VoidZero 製、Vite の上で動くフルスタックフレームワーク ＋ Cloudflare へのデプロイ CLI

**ざっくり言うと**

- Vite のプラグイン `voidPlugin()` を足すだけで、API ルート・SSR ページ・DB・認証・KV・ストレージ・AI・キュー・cron が使えるフルスタックフレームワーク（npm パッケージ名は `void` 、MIT）
- `import { db } from 'void/db'` のように **使ったリソースをコードから推論** し、開発中はローカルで動かし、 `void deploy` で Cloudflare（D1・KV・R2 など）に自動で作ってデプロイする
- UI は React / Vue / Svelte / Solid。TanStack Start・Nuxt・SvelteKit・Astro などのメタフレームワークに「デプロイと DB だけ」足す使い方もできる

## 図でわかる：Void の各機能はどこにつながっているか

コードで import するもの・置くファイル（Void SDK）が、中でどの技術を使い、Cloudflare のどのリソースになるか。それを `void` コマンドで、自分の Cloudflare か Void Platform にデプロイする。

（図: Void の機能のつながり / Void SDK / import する・ファイルを置くだけ / routes/ / pages/ / void/db / db/schema.ts / void/auth / void/client / void/kv / storage・ai / queues/ / crons/ / Hono + Vite / API・SSR / React/Vue 等 / Drizzle / 型付きクエリ / マイグレーション / Better Auth / ログイン / GitHub・Google / import から / 推論して / 直接つなぐ / Cloudflare Workers / アプリは 1 つの Worker で動く / D1 / PostgreSQL・MySQL（Hyperdrive）· KV · R2 / Workers AI + AI Gateway · Queues · Cron Triggers · Durable Objects / Resource Inference / import を見て必要なリソースを判断 → ローカルは自動で用意 / 本番は void deploy が作成 / void CLI / npx void ・ pnpm void ・ vpx void / void init / セットアップ / void deploy / ビルド→作成→公開 / void db … / void secret / 本番の秘密値 / void project / ログ・戻す / デプロイ先をどちらか選ぶ / 自分の Cloudflare アカウント / --platform cloudflare / 個人のアプリ向け / Void Platform（任意） / --platform void / チームの共有基盤・利用者やプランを管理 / vite.config.ts に voidPlugin() を足すだけ / UI アダプター（@void/react など）・任意の void.config.ts ・ ビルドは Vite / Vite+ → https://cs.taka1156.site/void.html#stack）

| 図の段 | 中身 |
| --- | --- |
| Void SDK | `routes/` ・ `pages/` は Hono ＋ Vite でルーティングと SSR、 `void/db` は Drizzle、 `void/auth` は Better Auth を通る。 `void/kv` ・ `void/storage` ・ `void/ai` ・ `queues/` ・ `crons/` は推論で Cloudflare のリソースに直接つながる |
| Cloudflare Workers | アプリは 1 つの Worker で動き、D1（または Hyperdrive 経由の PostgreSQL・MySQL）・KV・R2・Workers AI・Queues・Cron Triggers・Durable Objects を使う |
| void CLI | `init` （セットアップ）・ `deploy` （ビルド → リソース作成 → マイグレーション → 公開）・ `db` ・ `secret` ・ `project` （ログ・ロールバック） |
| デプロイ先 | 自分の Cloudflare アカウントか、チームの Void Platform（任意）のどちらか → 下で説明 |

Nuxt・Astro などのメタフレームワークで使うときは、 `routes/` ・ `pages/` の列をそのフレームワークが受け持ち、Void はそれ以外（リソース・DB・デプロイ）を担当する → 3 種類のアプリ

## 3 つの部品

1 つのパッケージ `void` に入っている。Cloudflare のブログ（2026-10-15）で 3 つまとめてオープンソース化された。VoidZero は 2026 年 6 月から Cloudflare の一部。

### フレームワーク Void SDK

ファイルベースの API ルート（ `routes/` ）とページ（ `pages/` ）、DB からUIまで通る型、 `void/db` ・ `void/kv` などの API。中身のルーターは Hono。

### デプロイ Void CLI

`void deploy` 1 つでビルド → リソース作成 → マイグレーション → デプロイ。ログ・ロールバック・シークレット・ドメインも扱う。

### チーム用基盤（任意） Void Platform

会社の Cloudflare アカウント（Workers for Platforms）に入れる共有デプロイ環境。社員に Cloudflare の権限を配らずに、アクセス制御・上限・利用状況の管理つきでアプリを公開させられる。→ 下で説明

ツールチェーン（vp コマンド）の Vite+ は別物。Void は Vite+ を既定のビルドツールとして使う → [Vite+ と vp コマンド](https://cs.taka1156.site/md/vite-plus.md)

## ローカルと本番：同じコードで何が変わるか

**import { db } from 'void/db'** ・ **void/kv** ・ **void/storage** ・ **void/ai** ・ **void/queues** …

コードで使うだけ。バインディングの設定は書かない

↓ 起動時にソースを走査して必要なリソースを推論（Resource Inference）

ローカル（vite dev）

ローカルの D1・KV・R2 などをその場で用意。DB サーバーを別に立てなくてよい。 `.env` の値を使う

本番（void deploy）

Cloudflare に D1・KV・R2・Queue などを作成してバインド。マイグレーションを適用し、URL を表示。 `.env` はアップロードしない

## 3 種類のアプリ

| 種類 | 中身・判定 |
| --- | --- |
| Void アプリ | Void がルーティングとデプロイを担当。API だけ（ `routes/` ）／SPA ＋ API（ `src/` ＋ `routes/` ）／ページモード（ `pages/` で SSR）のどれでもよい |
| メタフレームワーク | TanStack Start・React Router v7・SvelteKit・Nuxt・Analog・Astro。ルーティングと SSR はフレームワーク側、Void はリソース推論・型付き DB・マイグレーション・デプロイを担当（Void 管理の認証は使えない） |
| 静的サイト | SPA や VitePress などのビルド結果。 `void deploy --dir <path>` でそのまま配信 |

種類はプロジェクト構成から自動判定。固定したいときは `void.config.ts` の `inference.appType` （ `"void"` / `"framework"` / `"spa"` / `"static"` ）。

## はじめ方

```
# 新規（空のディレクトリで）または既存の Vite アプリに追加
npm install -D void
npx void init        # UI フレームワーク・スターター・デプロイ先を選ぶ
npm run dev          # http://localhost:5173

npx void deploy      # 初回はブラウザでサインイン → ビルド・作成・マイグレーション・デプロイ
```

CLI は Node.js 24.22 以上 または 26.12 以上。 `void` はプロジェクトに入れて使う（CLI とランタイムの版を揃えるため）。pnpm なら `pnpm create void my-app` でディレクトリごと作れる。

## デプロイ先：自分の Cloudflare か Void Platform か

フレームワーク（アプリのコード）はどちらでも同じ。Void Platform はフレームワークとは **別に存在する任意の基盤** で、デプロイ先の選び方だけが変わる。

### 直接 Cloudflare

自分の Cloudflare アカウントへ。個人のアプリ向け。共同作業（メンバー招待）はなし。

```
void deploy --platform cloudflare
```

### Void Platform

管理者が会社の Cloudflare に入れた共有基盤へ。社員は Cloudflare の権限なしで、会社のログインでデプロイできる。利用者・プランの上限・ログ・ロールバックを管理者がまとめて管理。

```
void connect platform.example.com
void deploy --platform void --project my-app
```

違いとメリット・デメリットの比較は → [直接 Cloudflare vs Void Platform](https://cs.taka1156.site/md/void-deploy-targets.md) 、基盤のインストールや管理者コマンドは → [Void Platform チートシート](https://cs.taka1156.site/md/void-platform.md)

## ファイル規約

どのディレクトリも任意。使う分だけ作る。

routes/ API。ファイルのパスが URL。 `export const GET = defineHandler(...)`

pages/ SSR ページ。 `about.tsx` と対になる `about.server.ts` に `loader` ・ `action`

middleware/ 全リクエスト共通。 `01.logger.ts` のように数字で順番を決める

db/schema.ts Drizzle のテーブル定義（DB の型の元）

db/migrations/ `void db generate` で作る SQL。デプロイ時に順に適用

crons/ 定期実行。 `export const cron` ＋ `defineScheduled()`

queues/ キューの受け手。ファイル名がキュー名。 `defineQueue()`

email/ メール受信。ファイル名が宛先、 `_default.ts` はその他

durable-objects/ 状態を持つ処理。 `defineDurableState()`

src/ ・ public/ 共有コード ・ そのまま配信する静的ファイル

vite.config.ts `voidPlugin()` が必須

void.config.ts 任意。ルーティング・推論・Worker の設定

env.ts ・ .env 環境変数の型（コミットする）・ローカルの値（コミットしない）

.void/ 自動生成の型とローカル状態（gitignore）。 `void prepare` で作る

### ルーティングの書き方（routes/ と pages/ 共通）

| ファイル名 | 意味 |
| --- | --- |
| `users/[id].ts` | 動的セグメント → `/users/:id` 。値は `c.req.param('id')` |
| `[...slug].ts` | キャッチオール（ほかに合うものが無いとき最後に） |
| `(admin)/` | ルートグループ。URL には出ない |
| `_helper.ts` | `_` か `.` で始まるものはルートにならない |
| `debug.dev.ts` ・ `metrics.prod.ts` | 開発中だけ ・ 本番だけ（dev 専用で使うリソースは本番に作られない） |

## API チートシート

ルート

ページ

DB

KV・R2・AI

キュー・cron

認証・環境変数

```
// routes/api/users/index.ts → GET / POST /api/users
import { users, insertUserSchema } from '@schema';
import { defineHandler } from 'void';
import { db } from 'void/db';

export const GET = defineHandler(async () => {
  return await db.select().from(users);   // 配列 → JSON
});

export const POST = defineHandler.withValidator({
  body: insertUserSchema,                 // Drizzle スキーマから作った検証
})(async (c, { body }) => {
  await db.insert(users).values(body);
  return { created: true };
});
```

`c` は Hono の Context（ `c.json()` ・ `c.req` ・ `c.env` など全部使える）。戻り値は自動変換:

- オブジェクト・配列・数値・真偽値 → JSON
- 文字列 → `text/html`
- `null` / `undefined` → 204 No Content
- `Response` → そのまま

クライアントからは型付きの fetch で呼ぶ。ハンドラの戻り値を変えると呼び出し側も型チェックされる。

```
import { fetch } from 'void/client';
const { message } = await fetch('/api/hello');
```

ミドルウェア: `middleware/` に `export default defineMiddleware(async (c, next) => { …; await next(); })` 。Hono のミドルウェアもそのまま使える。

Inertia.js 風。 `.server.ts` の `loader` が返した値が、同名ページのコンポーネントの props になる。

```
// pages/blog.server.ts
import { defineHandler, type InferProps } from 'void';
import { db } from 'void/db';
import { posts } from '@schema';

export const loader = defineHandler(async () => ({
  posts: await db.select().from(posts),
}));
export type Props = InferProps<typeof loader>;

// pages/blog.tsx
import type { Props } from './blog.server';
export default function Blog({ posts }: Props) {
  return <ul>{posts.map((p) => <li key={p.id}>{p.title}</li>)}</ul>;
}
```

- `action` （POST / PUT / PATCH / DELETE）: 何も返さなければ loader を再実行して再描画、 `c.redirect(url)` なら遷移。1 ページに複数なら名前付き action
- フォーム: アダプターの `Form` と `useForm('/users', { name: '' })` 。値・エラーが action から型付けされる
- `export const ssr = false` でそのページだけ SSR しない（loader は動く）
- ほか: レイアウト（自動でネスト）、Markdown ページ、アイランド、View Transitions、プリレンダー

アダプターの入れ方（ `void init` なら自動）:

```
npm install @void/react      # @void/vue ・ @void/svelte ・ @void/solid

// vite.config.ts
import { voidReact } from '@void/react/plugin';
import { defineConfig } from 'vite';
import { voidPlugin } from 'void';

export default defineConfig({ plugins: [voidPlugin(), voidReact()] });
```

Drizzle ORM 同梱。同じスキーマで D1・PostgreSQL・MySQL を使える（D1 はローカルで DB サーバー不要なので最初の 1 つに向く）。

```
// db/schema.ts（@schema で import できる）にテーブルを定義
import { db } from 'void/db';
import { users } from '@schema';
const rows = await db.select().from(users);
```

| やること | コマンド |
| --- | --- |
| スキーマとルートのひな形を作る | `void gen model <name> [cols...]` |
| スキーマの変更から SQL マイグレーションを作る | `void db generate` （作った SQL は確認してコミット） |
| マイグレーションなしで直接反映（試作用） | `void db push` |
| ローカル / リモートの適用状況 | `void db status` |
| ローカルを作り直してシード | `void db reset` ・ `void db seed` |
| SQL を実行 | `void db execute "<sql>"` （本番は `--remote` ） |
| Drizzle Studio を開く | `void db studio` |
| 本番 D1 を復元ポイントに戻す | `void db restore --bookmark <b>` （自分の Cloudflare アカウントのとき） |

スキーマを変えたのにマイグレーションが無いと `void deploy` は止まる。ロールバックしてもマイグレーションは戻らない。

| import | 推論されるバインディング・使い方 |
| --- | --- |
| `import { kv } from 'void/kv'` | `KV` （KVNamespace）。 `kv.get<User>('user:1')` ・ `kv.put(key, value, { ttl: 3600 })` ・ `kv.delete(key)` 。値は自動で JSON に |
| `import { storage } from 'void/storage'` | `STORAGE` （R2）。 `storage.put(key, file, { httpMetadata })` ・ `storage.get(key)` ・ `storage.list()` |
| `import { ai } from 'void/ai'` | `AI` （Workers AI、AI Gateway も自動設定）。 `ai.run(model, { messages })` ・ `ai.stream(...)` ・ `ai.models()` |
| `import { getSandbox } from 'void/sandbox'` | Sandbox（Workers Paid と Containers が必要） |
| `import { defineLiveStream } from 'void/live'` | `VOID_LIVE` （Durable Object）でリアルタイム配信 |

`c.env.KV` のように env から触っても推論される。実行時にキーが決まる書き方は推論できないので `void.config.ts` の `inference.bindings` で明示する。Vue・Svelte のコンポーネントファイルは走査されないので、リソースの import は `.server.ts` などに書く。

```
// queues/emails.ts → キュー「emails」の受け手
import { defineQueue } from 'void';
interface Message { to: string; subject: string }

export default defineQueue<Message>(async (batch, env) => {
  for (const msg of batch.messages) {
    try { await send(msg.body); msg.ack(); }
    catch { msg.retry({ delaySeconds: 30 }); }
  }
});

// 送る側（body は Message 型でチェックされる）
import { queues } from 'void/queues';
await queues.emails.send({ to: 'a@example.com', subject: 'Hi' });
```

```
// crons/hourly.ts
import { defineScheduled } from 'void';
export const cron = '0 * * * *';          // 配列で複数指定も可
export default defineScheduled(async (controller, env) => {
  await env.KV.put('last-run', controller.scheduledTime.toString());
});
```

認証は Better Auth ベース。 `void/auth` か クライアントの `auth` を import すると有効になる。プロバイダーは `void.config.ts` の `auth.providers` （例 `['email', 'github', 'google']` ）。

```
// クライアント
import { auth } from 'void/client';
await auth.signIn.email({ email, password });

// サーバー（未ログインなら 401。ページなら throwRedirect でログイン画面へ）
import { requireAuth } from 'void/auth';
export const loader = defineHandler((c) => {
  const user = requireAuth(c, { throwRedirect: '/login' });
  return { email: user.email };
});
```

環境変数は `env.ts` で型を宣言し、 `void/env` から読む。

```
// env.ts
import { defineEnv, number, string, url } from 'void/env';
export default defineEnv({
  STRIPE_KEY: string(),
  PORT: number().default(3000),
  VITE_API_ORIGIN: url(),
});

// アプリ側
import { env } from 'void/env';
```

ローカルは `.env` 、本番のサーバー用の値は `void secret put NAME=value` 。公開してよい `VITE_*` はビルド時の環境から渡す。

（元のページで操作できます → https://cs.taka1156.site/void.html#api）

## void CLI チートシート

パッケージスクリプト以外では `npx void` ・ `pnpm void` ・ `vpx void` などで実行。 `void <command> --help` はサインインなしで見られる。

| コマンド | やること |
| --- | --- |
| `void init` | 新規・既存プロジェクトのセットアップ（コーディングエージェント用の `AGENTS.md` とスキルも） |
| `void init --github` | 保存したデプロイ先用の GitHub Actions ワークフローを作る |
| `void prepare` | Vite を起動せずに `.void/` の型を作る（CI の型チェック前に） |
| `void migrate` | 古い設定を今の形式に更新 |
| `void info [--json]` | リソースの永続名を確認（コードを移動する前に） |
| `void deploy` | ビルド → リソース作成 → マイグレーション → デプロイ。2 回目以降は保存した先へ |
| `void deploy --platform cloudflare` | 自分の Cloudflare アカウントへ（初回にアカウントを選ぶ） |
| `void deploy --platform void --project my-app` | チームの Void Platform へ |
| `void deploy --skip-migrations` | SQL マイグレーションを適用せずにコードだけデプロイ |
| `void deploy --dir <path>` | 静的ファイルのディレクトリをそのままデプロイ |
| `void connect <url>` | CLI を Void Platform につなぐ（ホスト名だけなら HTTPS） |
| `void auth login` | 保存したデプロイ先にログイン（無ければ選ぶ） |
| `void cloudflare login` ・ `void account login` | Cloudflare に ・ Void Platform にログイン |
| `void project link` | ディレクトリをプロジェクトに紐付け |
| `void project logs` | デプロイしたアプリの実行ログ |
| `void project requests` | リクエスト単位のトラフィック（ステータス・メソッド・時間）。Void Platform のみ |
| `void project rollback` | 前のデプロイに戻す（DB マイグレーションは戻らない） |
| `void project cancel` | 進行中のデプロイを取り消す |
| `void project purge-cache` | キャッシュしたページを全部消す（Void Platform） |
| `void build logs` | ビルドログを流す・追う・ダウンロード |
| `void gen model <name> [cols...]` | Drizzle のテーブルと API ルートのひな形 |
| `void gen route <path>` | API ルートを作る |
| `void db generate` ・ `push` ・ `status` | マイグレーション作成 ・ 直接反映 ・ 適用状況（詳しくは上の DB タブ） |
| `void db execute <sql> --remote` | デプロイ先の DB に SQL を実行 |
| `void secret put <name=value>` | 本番のシークレットを設定。 `void secret list` で一覧 |
| `void secret sync <file>` | dotenv ファイルからまとめて登録 |
| `void env check [--remote]` | `env.ts` のスキーマどおりか確かめる |
| `void env types` | `.void/env.d.ts` を作り直す |
| `void domain …` | カスタムドメイン |
| `void email setup` ・ `status` ・ `logs` | メール送受信の準備 ・ 状態 ・ 配信ログ |
| `void platform install` | 会社用の Void Platform を Cloudflare に入れる（認証・DB・ストレージを対話で設定） |

環境変数: `VOID_TOKEN` （保存した認証の代わりのトークン）、 `VOID_PROJECT` （deploy などの既定のプロジェクト）。

（元のページで操作できます → https://cs.taka1156.site/void.html#cli）

## 設定ファイルの例

vite.config.ts

void.config.ts

```
import { defineConfig } from 'vite';   // Vite+ なら 'vite-plus'
import { voidPlugin } from 'void';
import { voidVue } from '@void/vue/plugin';

export default defineConfig({
  plugins: [voidPlugin(), voidVue()],
});
```

メタフレームワークでも、そのフレームワークの Vite 設定に `voidPlugin()` を足す。

```
// すべて任意。TypeScript のパスエイリアス（@/…）は使えない
import { defineConfig } from 'void/config';

export default defineConfig({
  target: 'cloudflare',
  auth: { providers: ['email', 'github'] },
  routing: {
    revalidate: { '/': 60, '*': 30 },        // ISR の秒数
    headers: { '/assets/*': ['Cache-Control: public, max-age=31536000, immutable'] },
  },
  inference: {
    bindings: { db: true, kv: false },      // 推論の上書き
    appType: 'void',
  },
  worker: { compatibility_flags: ['nodejs_compat'] },
  cloudflare: { name: 'my-app' },
});
```

`void.config.ts` と、生成された `void.lock.json` （リソース ID とマイグレーション履歴）はコミットする。

（元のページで操作できます → https://cs.taka1156.site/void.html#config）

## よくある誤解

✗ 誤解: Void はデプロイサービスである

✓ 正しくは: 本体は **Vite の上のフルスタックフレームワーク（SDK）** で、デプロイ CLI と任意のチーム基盤がセット。3 月の発表時は「Vite ネイティブのデプロイ基盤」として紹介されたが、Cloudflare 合流後に全部オープンソースになり、デプロイ先は自分の Cloudflare アカウントか自前の Void Platform になった。

✗ 誤解: Vite+ を入れないと使えない

✓ 正しくは: `void init` で Vite+ か素の Vite を選べる。Vite+ を入れる場合もグローバルインストールは不要。

✗ 誤解: wrangler.jsonc にバインディングを書く必要がある

✓ 正しくは: 基本は不要（コードから推論）。既に Worker と `wrangler.jsonc` があれば `void deploy` がそれを紐付けて既存リソースを引き継げる。

✗ 誤解: Cloudflare 以外では動かせない

✓ 正しくは: `void deploy` のデプロイ先は Cloudflare か Void Platform（これも Cloudflare 上）だけ。ただし `void.config.ts` で `target: 'node'` （または `'bun'` / `'deno'` ）にすると、サーバーごと Node.js・Bun・Deno 向けにビルドでき、自前のサーバーやコンテナで動かせる（ `void/db` ・ `void/kv` ・ `void/auth` など Cloudflare のリソースを使う import は使えない）。 `output: 'static'` で全ページを事前生成すれば、Worker なしの静的サイトとしてどこにでも置ける。

## 出典

- [Void ドキュメント](https://void.cloud) （Quickstart・Supported App Types・Project Structure・Server Routing・Pages Routing・Resource Inference・Deployment・CLI / Config Reference）
- [voidzero-dev/void](https://github.com/voidzero-dev/void) （README と `docs/` 。パッケージ `void` 1.8.0 時点）
- [Deploy with Void](https://blog.cloudflare.com/deploy-with-void/) （Cloudflare Blog, 2026-10-15）

確認日: 2026-11-13
