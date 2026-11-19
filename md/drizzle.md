<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Drizzle ORM 使い方

> Drizzle ORM のスキーマ定義・クエリ・リレーション・drizzle-kit のマイグレーションと、v1 での変更点をまとめたチートシート

- URL: https://cs.taka1156.site/drizzle.html
- タグ: API・バックエンド
- 日付: 2026-11-20

---

TypeScript でテーブルを定義し、SQL に近い形で型付きクエリを書く ORM。マイグレーションは drizzle-kit（drizzle-orm 0.45・drizzle-kit 0.31 の安定版が前提。v1 RC との違いは最後の節）

**ざっくり言うと**

- Drizzle は TypeScript でテーブルを書き、SQL とほぼ同じ形のクエリを型付きで書ける ORM。
- スキーマのファイルが唯一の元で、型はそこから推論され、drizzle-kit が差分の SQL マイグレーションを作る。
- 安定版は 0.45 で v1 は RC。リレーションの書き方と移行フォルダが変わるので、記事の版を確かめる。

## 全体の流れ

**マイグレーション** （スキーマを変えたとき・開発者の手元や CI）

**schema.ts** テーブル・列・インデックスを TS で定義

generate

**drizzle/*.sql** 前回のスナップショットとの差分 SQL。Git に入れる

migrate

**データベース** 適用済みの記録は `__drizzle_migrations` テーブル

**アプリの実行時**

**drizzle(接続)** ドライバー（pg・D1・mysql2 など）を包んで `db` を作る

**db.select()…** クエリビルダーまたは `db.query` （リレーション）で書く

**型付きの結果** 選んだ列・JOIN の種類から戻り値の型が決まる

## セットアップ

PostgreSQL

SQLite（D1）

MySQL

```
npm i drizzle-orm pg
npm i -D drizzle-kit @types/pg
```

```
// src/db.ts
import { drizzle } from 'drizzle-orm/node-postgres';
import * as schema from './schema';

export const db = drizzle(process.env.DATABASE_URL!, { schema });
```

```
// drizzle.config.ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  dialect: 'postgresql',
  schema: './src/schema.ts',      // glob も可: './src/db/schema/*.ts'
  out: './drizzle',               // マイグレーションの出力先
  dbCredentials: { url: process.env.DATABASE_URL! },
});
```

ドライバーは用途で選ぶ: `drizzle-orm/postgres-js` （postgres.js）・ `drizzle-orm/neon-http` （Neon）・ `drizzle-orm/pglite` （WASM の PostgreSQL。テスト向き）など。

```
npm i drizzle-orm
npm i -D drizzle-kit
```

```
// Worker の中で、バインディング（env.DB）から作る
import { drizzle } from 'drizzle-orm/d1';
import * as schema from './schema';

export default {
  async fetch(req: Request, env: Env) {
    const db = drizzle(env.DB, { schema });
    const rows = await db.select().from(schema.users);
    return Response.json(rows);
  },
};
```

```
// drizzle.config.ts（push・studio で本番 D1 に HTTP でつなぐとき）
export default defineConfig({
  dialect: 'sqlite',
  driver: 'd1-http',
  schema: './src/schema.ts',
  out: './drizzle',
  dbCredentials: {
    accountId: process.env.CLOUDFLARE_ACCOUNT_ID!,
    databaseId: process.env.CLOUDFLARE_DATABASE_ID!,
    token: process.env.CLOUDFLARE_D1_TOKEN!,
  },
});
```

generate で作った SQL は、wrangler の設定で D1 の `migrations_dir` を `drizzle` にして `wrangler d1 migrations apply <DB名>` （本番は `--remote` ）で適用してもよい。

ローカルファイルの SQLite は `drizzle-orm/better-sqlite3` ・ `drizzle-orm/libsql` （Turso）など。

```
npm i drizzle-orm mysql2
npm i -D drizzle-kit
```

```
import { drizzle } from 'drizzle-orm/mysql2';
import * as schema from './schema';

// 0.x でリレーションを使うときは mode が要る（PlanetScale なら 'planetscale'）
export const db = drizzle(process.env.DATABASE_URL!, { schema, mode: 'default' });
```

```
export default defineConfig({
  dialect: 'mysql',
  schema: './src/schema.ts',
  out: './drizzle',
  dbCredentials: { url: process.env.DATABASE_URL! },
});
```

MySQL には `RETURNING` がないので、挿入した ID は `.$returningId()` で受け取る。

（元のページで操作できます → https://cs.taka1156.site/drizzle.html#setup）

## スキーマ定義

```
// src/schema.ts（PostgreSQL の例）
import { pgTable, integer, text, varchar, timestamp, index } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  name: text().notNull(),
  email: varchar({ length: 255 }).notNull().unique(),
  role: text({ enum: ['admin', 'member'] }).notNull().default('member'),
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').$onUpdate(() => new Date()),
});

export const posts = pgTable('posts', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  title: text().notNull(),
  authorId: integer('author_id').notNull()
    .references(() => users.id, { onDelete: 'cascade' }),   // 外部キー
}, (t) => [
  index('posts_author_idx').on(t.authorId),                  // 3 つ目の引数は配列で返す
]);

// 型はテーブルから推論する
export type User = typeof users.$inferSelect;     // SELECT の結果
export type NewUser = typeof users.$inferInsert;  // INSERT の入力（既定値のある列は省略可）
```

列名を省くとキー名がそのまま DB の列名になる。キーは camelCase・DB は snake_case にしたいときは、列ごとに名前を渡すか、 `drizzle(url, { casing: 'snake_case' })` と `drizzle.config.ts` の `casing: 'snake_case'` を両方に書く（v1 では書き方が変わる）。

### 列の型（DB ごとの import 元）

| 用途 | PostgreSQL（ `pg-core` ） | SQLite（ `sqlite-core` ） | MySQL（ `mysql-core` ） |
| --- | --- | --- | --- |
| テーブル | `pgTable` | `sqliteTable` | `mysqlTable` |
| 自動採番の主キー | `integer().primaryKey().generatedAlwaysAsIdentity()` （または `serial()` ） | `integer().primaryKey({ autoIncrement: true })` | `int().primaryKey().autoincrement()` （または `serial()` ） |
| 文字列 | `text()` ・ `varchar({ length })` | `text()` | `varchar({ length })` ・ `text()` |
| 真偽値 | `boolean()` | `integer({ mode: 'boolean' })` | `boolean()` |
| 日時 | `timestamp({ withTimezone: true })` | `integer({ mode: 'timestamp' })` （秒。ミリ秒は `'timestamp_ms'` ） | `datetime()` ・ `timestamp()` |
| JSON | `jsonb().$type<T>()` | `text({ mode: 'json' }).$type<T>()` | `json().$type<T>()` |
| 列挙 | `pgEnum('role', [...])` （DB の型を作る）・ `text({ enum })` | `text({ enum: [...] })` （型だけ） | `mysqlEnum('role', [...])` |

### 列の修飾子

| 書き方 | 意味 |
| --- | --- |
| `.notNull()` | NOT NULL。付けないと型が `T \| null` になる |
| `.primaryKey()` ・ `.unique()` | 主キー・一意制約。複合主キーは 3 つ目の引数で `primaryKey({ columns: [t.a, t.b] })` |
| `.default(v)` ・ `.defaultNow()` | DB 側 の既定値（DDL に入る）。式は `.default(sql`...`)` |
| `.$defaultFn(() => …)` | アプリ側 で INSERT のときに値を作る（例: `crypto.randomUUID()` ）。DDL には出ない |
| `.$onUpdate(() => …)` | Drizzle で UPDATE したときにアプリ側で値を入れる（ `updatedAt` など）。既定値がない列なら INSERT でも入る |
| `.references(() => t.id, opts)` | 外部キー。 `onDelete` ・ `onUpdate` に `'cascade'` ・ `'set null'` など |
| `.$type<T>()` | TypeScript の型だけを差し替える（DB の型は変わらない） |
| `index()` ・ `uniqueIndex()` | 3 つ目の引数の配列で `index('名前').on(t.col)` |

## クエリ

select

insert

update・delete

join・集計

トランザクション・SQL

```
import { and, desc, eq, like } from 'drizzle-orm';

const all = await db.select().from(users);              // User[]

const admins = await db
  .select({ id: users.id, name: users.name })           // 列を選ぶと型も絞られる
  .from(users)
  .where(and(eq(users.role, 'admin'), like(users.name, 'A%')))
  .orderBy(desc(users.createdAt))
  .limit(10)
  .offset(20);

// 条件を後から足す: undefined を渡した条件は無視される
const filters = [q ? like(users.name, `%${q}%`) : undefined, eq(users.role, 'member')];
await db.select().from(users).where(and(...filters));
```

```
await db.insert(users).values({ name: 'Ann', email: 'ann@example.com' });

// 複数行 + 挿入した行を返す（PostgreSQL・SQLite）
const [ann, bob] = await db.insert(users).values([a, b]).returning();
const ids = await db.insert(users).values(a).returning({ id: users.id });

// MySQL は RETURNING がない
const [{ id }] = await db.insert(users).values(a).$returningId();

// 重複したとき（upsert）
await db.insert(users).values(a).onConflictDoNothing();
await db.insert(users).values(a).onConflictDoUpdate({
  target: users.email,
  set: { name: a.name },
});
// MySQL: .onDuplicateKeyUpdate({ set: { name: a.name } })
```

```
await db.update(users)
  .set({ name: 'Bob' })
  .where(eq(users.id, 1))
  .returning();

await db.delete(users).where(eq(users.id, 1));

// 列の値を使って更新（posts に views 列があるとき）
await db.update(posts).set({ views: sql`${posts.views} + 1` }).where(eq(posts.id, id));
```

`where` を書かない `update` ・ `delete` は全行が対象になる。

```
import { count, eq, gt } from 'drizzle-orm';

// leftJoin なので author は string | null になる（innerJoin なら string）
const rows = await db
  .select({ title: posts.title, author: users.name })
  .from(posts)
  .leftJoin(users, eq(posts.authorId, users.id));

// 件数だけ
const n = await db.$count(users, eq(users.role, 'admin'));

// グループ化
const byAuthor = await db
  .select({ authorId: posts.authorId, n: count() })
  .from(posts)
  .groupBy(posts.authorId)
  .having(({ n }) => gt(n, 5));

// サブクエリ
const sq = db.select().from(posts).where(gt(posts.id, 100)).as('sq');
await db.select().from(sq);
```

```
await db.transaction(async (tx) => {
  const [u] = await tx.insert(users).values(a).returning();
  await tx.insert(posts).values({ title: 'hello', authorId: u.id });
  if (!ok) tx.rollback();      // 例外を投げて全体を戻す
});

// D1・libSQL・Neon HTTP: まとめて 1 回で送る（全部成功か全部失敗）
await db.batch([
  db.insert(users).values(a),
  db.update(users).set({ role: 'admin' }).where(eq(users.id, 1)),
]);

// 生の SQL（値は自動でパラメータになる）
import { sql } from 'drizzle-orm';
await db.select({ lower: sql<string>`lower(${users.name})` }).from(users);
await db.execute(sql`select * from ${users} where ${users.id} = ${id}`); // PG・MySQL
// SQLite は db.run(sql`…`)・db.all(sql`…`)
```

D1 は対話的なトランザクション（BEGIN … COMMIT）を使えないので、まとめたい書き込みは `db.batch()` にする。

（元のページで操作できます → https://cs.taka1156.site/drizzle.html#query）

### 条件の演算子（ `drizzle-orm` から import）

| 関数 | SQL |
| --- | --- |
| `eq` ・ `ne` | `=` ・ `<>` |
| `gt` ・ `gte` ・ `lt` ・ `lte` | `>` ・ `>=` ・ `<` ・ `<=` |
| `inArray` ・ `notInArray` | `IN (…)` ・ `NOT IN (…)` （空配列も渡せる） |
| `isNull` ・ `isNotNull` | `IS NULL` ・ `IS NOT NULL` |
| `like` ・ `notLike` ・ `ilike` | `LIKE` （ `ilike` は PostgreSQL の大文字小文字無視） |
| `between` ・ `notBetween` | `BETWEEN a AND b` |
| `and` ・ `or` ・ `not` | 条件の組み合わせ |
| `exists` ・ `notExists` | サブクエリの有無 |
| `asc` ・ `desc` | `orderBy` の並び順 |

## リレーショナルクエリ（ `db.query` ）

JOIN を自分で書かずに、ネストしたオブジェクトで関連データを取る API。0.x と v1 で定義も絞り込みも書き方が違う。

0.x（relations）

v1 RC（defineRelations）

```
// schema.ts に一緒に書く
import { relations } from 'drizzle-orm';

export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}));
export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, { fields: [posts.authorId], references: [users.id] }),
}));

// db を作るとき schema（relations を含む）を渡す
const db = drizzle(url, { schema });

const list = await db.query.users.findMany({
  columns: { id: true, name: true },
  where: (u, { eq }) => eq(u.role, 'admin'),
  orderBy: (u, { desc }) => [desc(u.createdAt)],
  with: { posts: { columns: { title: true }, limit: 5 } },
});
const one = await db.query.users.findFirst({ where: (u, { eq }) => eq(u.id, 1) });
```

```
// relations.ts: 全テーブルの関係を 1 か所で定義
import { defineRelations } from 'drizzle-orm';
import * as schema from './schema';

export const relations = defineRelations(schema, (r) => ({
  users: { posts: r.many.posts() },
  posts: {
    author: r.one.users({ from: r.posts.authorId, to: r.users.id }),
  },
}));

// schema ではなく relations を渡す
const db = drizzle(url, { relations });

// where・orderBy はオブジェクトで書く
const list = await db.query.users.findMany({
  where: {
    role: 'admin',
    id: { gt: 10 },
    OR: [{ name: { like: 'A%' } }, { name: { ilike: 'b%' } }],
    posts: true,                       // 投稿が 1 件以上ある人（false なら 0 件）
  },
  orderBy: { createdAt: 'desc' },
  with: { posts: { limit: 5 } },
});
```

v1 では絞り込みに `undefined` を明示すると例外になる。省くか、関連の条件なら `{}` を渡す。

（元のページで操作できます → https://cs.taka1156.site/drizzle.html#relations）

## drizzle-kit コマンド

| コマンド | やること |
| --- | --- |
| `drizzle-kit generate` | スキーマと前回のスナップショットの差分から SQL を作る。 `--name=add_posts` で名前、 `--custom` で空の SQL（データ移行用） |
| `drizzle-kit migrate` | まだ適用していない SQL を DB に適用する |
| `drizzle-kit push` | SQL ファイルを作らず、スキーマを DB に直接反映する（試作・ローカル向け） |
| `drizzle-kit pull` | 既存の DB を読んで `schema.ts` を作る |
| `drizzle-kit check` | 生成済みのマイグレーションの食い違い（ブランチの衝突など）を調べる |
| `drizzle-kit up` | 古い形式のスナップショットを今の形式に上げる（v1 への移行でも使う） |
| `drizzle-kit studio` | ブラウザで DB を見る・編集する Drizzle Studio を起動 |
| `drizzle-kit export` | スキーマを DDL の SQL にして出力する |

設定ファイルを分けるときは `--config=drizzle.prod.config.ts` 。アプリの起動時に適用するなら、ドライバーごとの `migrate` を呼ぶ。

```
import { migrate } from 'drizzle-orm/node-postgres/migrator';
await migrate(db, { migrationsFolder: './drizzle' });
```

### generate と push の使い分け

1. **試作・ローカル** `push` でスキーマをすぐ反映する。履歴が残らないので、列名の変更などはデータが消える形になることがある（確認が出る）
2. **チーム開発・本番** `generate` で SQL を作ってレビュー・コミットし、 `migrate` （または wrangler など各サービスの仕組み）で適用する
3. **元に戻すとき** 自動のダウンマイグレーションはない。戻す変更を新しいマイグレーションとして作る

## 検証スキーマを作る（Zod）

```
// 0.x: drizzle-zod パッケージ（v1 では drizzle-orm/zod に入った）
import { z } from 'zod';
import { createInsertSchema, createSelectSchema, createUpdateSchema } from 'drizzle-zod';

// 自動採番（generatedAlwaysAsIdentity）の id は最初から入らない
export const insertUserSchema = createInsertSchema(users, {
  name: z.string().min(1).max(50),      // 列ごとに差し替え
}).omit({ createdAt: true });

export const updateUserSchema = createUpdateSchema(users);   // 全項目が省略可
```

Zod 自体の書き方は [Zod 使い方](https://cs.taka1156.site/md/zod.md) 。valibot・TypeBox・ArkType 向けもある。

## v1 RC での主な変更

v1 は `npm i drizzle-orm@rc` ・ `npm i -D drizzle-kit@rc` で入り、最初に `npx drizzle-kit up` で既存のマイグレーションを新しい形式に変える。RC の間も破壊的変更が入るので、上げるたびに変更点を読む。

| 項目 | 0.x | v1 |
| --- | --- | --- |
| リレーション | テーブルごとの `relations()` | `defineRelations(schema, r => …)` を 1 か所に。 `where` ・ `orderBy` はオブジェクト |
| `drizzle()` の引数 | `{ schema }` | `{ relations }` |
| casing | `drizzle({ casing: 'snake_case' })` | 廃止。 `snakeCase.table('users', …)` のようにテーブル側で指定 |
| マイグレーションの形 | `meta/_journal.json` と連番の SQL | journal を廃止し、マイグレーションごとのフォルダに SQL とスナップショット |
| 適用の順番 | 最後に適用したものより新しいものだけ | 未適用のものを全部（フォルダ名で照合） |
| `push` | `--strict` で確認 | データが消える文は既定で確認（ `--force` で省く）。 `--explain` で SQL を見るだけ |
| 検証ライブラリ | `drizzle-zod` など別パッケージ | `drizzle-orm/zod` ・ `drizzle-orm/valibot` など本体に同梱 |
| 列の一覧 | `getTableColumns(t)` | `getColumns(t)` |
| リードレプリカ | 読み取りは自動でレプリカへ | 既定はプライマリ。レプリカは `db.$replica` で明示 |
| クエリのエラー | パラメータの値を含む | 含めない（ `paramsInErrors: true` で戻せる） |

## よくある誤解

✗ 誤解: relations を書けば外部キー制約も作られる

✓ 正しくは: `relations()` ・ `defineRelations()` は `db.query` のための情報で、DB には何も作らない。外部キー制約は列の `.references()` で書く（両方書くのが普通）。

✗ 誤解: push と generate はどちらを使っても同じ

✓ 正しくは: `push` は差分を直接当てて履歴を残さないので、ほかの環境で同じ変更を再現できない。本番やチームでは `generate` した SQL をコミットして `migrate` する。

✗ 誤解: $defaultFn で書いた値は DB の既定値になる

✓ 正しくは: `$defaultFn` ・ `$onUpdate` は Drizzle 経由の INSERT・UPDATE のときにアプリ側で値を入れるだけで、DDL には出ない。SQL を直接流したときにも効かせたいなら `.default(sql`…`)` を使う。

✗ 誤解: drizzle-kit はマイグレーションを自動で巻き戻せる

✓ 正しくは: ダウンマイグレーションの仕組みはない。戻すときは、逆向きの変更を新しいマイグレーションとして作って適用する。

関連: [Void](https://cs.taka1156.site/md/void.md) （ `void/db` が Drizzle）・ [Better Auth](https://cs.taka1156.site/md/better-auth.md) （ `drizzleAdapter` ）

## 出典

- [Drizzle ORM – Overview](https://orm.drizzle.team/docs/overview) ・ [Schema](https://orm.drizzle.team/docs/sql-schema-declaration) ・ [Select](https://orm.drizzle.team/docs/select) ・ [Filters](https://orm.drizzle.team/docs/operators)
- [Migrations with Drizzle Kit](https://orm.drizzle.team/docs/kit-overview) ・ [Cloudflare D1](https://orm.drizzle.team/docs/connect-cloudflare-d1)
- [Changes in v1](https://orm.drizzle.team/docs/v0-v1-changes) ・ [Relational Queries v1 to v2](https://orm.drizzle.team/docs/relations-v1-v2) ・ [Upgrading to Drizzle v1](https://orm.drizzle.team/docs/upgrade-v1)
- [Latest Releases](https://orm.drizzle.team/docs/latest-releases) （npm の latest: drizzle-orm 0.45.4・drizzle-kit 0.31.11、rc: 1.0.0-rc.5）

確認日: 2026-11-20
