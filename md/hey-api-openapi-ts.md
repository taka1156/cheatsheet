<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Hey API openapi-ts 使い方

> @hey-api/openapi-ts で OpenAPI 定義から TypeScript の型・SDK・クライアントを生成する手順と設定・プラグインのチートシート

- URL: https://cs.taka1156.site/hey-api-openapi-ts.html
- タグ: Hey API, OpenAPI, TypeScript
- 日付: 2026-11-19

---

@hey-api/openapi-ts で OpenAPI 定義から TypeScript の型・SDK・HTTP クライアントを生成する（v0.99 時点）

**ざっくり言うと**

- OpenAPI 定義（ファイル・URL）から、型（ `types.gen.ts` ）と型付きの API 関数（ `sdk.gen.ts` ）を生成する CLI。
- 何を生成するかはプラグインで決まり、既定は型＋SDK＋Fetch クライアントで、Zod や TanStack Query も足せる。
- まだ 0.x で破壊的変更が多いので版を固定し、生成物は手で直さず設定と定義の側を変える。

## 生成の流れ

**input** openapi.yaml・URL・API レジストリ

**parser** 定義を読み込み、フィルターや変換をかける

**plugins** 型・SDK・クライアント・Zod など、1 プラグイン＝生成物 1 種類

**output** `src/client/` に `*.gen.ts` を書き出す

## インストールと実行

お試し

プロジェクトに入れる

Node から呼ぶ

Vite プラグイン

```
npx @hey-api/openapi-ts -i ./openapi.yaml -o src/client
```

Node.js 22 系以上が必要（0.99 の `engines` は `>=22.18.0` ）。0.91 から ESM のみ。

```
# 0.x なので完全一致で固定する（-E = --save-exact）
npm install -D -E @hey-api/openapi-ts

// package.json
"scripts": {
  "openapi-ts": "openapi-ts"
}

npm run openapi-ts    # openapi-ts.config.ts を読んで生成
```

```
import { createClient } from '@hey-api/openapi-ts';

await createClient({
  input: './openapi.yaml',
  output: 'src/client',
});
```

```
npm install -D @hey-api/vite-plugin

// vite.config.ts（Vite 5〜8）
import { heyApiPlugin } from '@hey-api/vite-plugin';
import { defineConfig } from 'vite';

export default defineConfig({
  plugins: [
    heyApiPlugin({
      config: { input: './openapi.yaml', output: 'src/client' },
    }),
  ],
});
```

（元のページで操作できます → https://cs.taka1156.site/hey-api-openapi-ts.html#install）

### CLI オプション

| オプション | 意味 |
| --- | --- |
| `-i, --input` | OpenAPI 定義（パス・URL）。スペース区切りで複数（例 `-i a.yaml b.yaml` ） |
| `-o, --output` | 出力フォルダ。スペース区切りで複数 |
| `-c, --client` | HTTP クライアントのプラグイン（例 `@hey-api/client-axios` ） |
| `-p, --plugins` | 使うプラグイン（スペース区切りで複数） |
| `-f, --file` | 設定ファイルのパス |
| `-w, --watch` | 定義の変更を監視して再生成（間隔 ms を指定可。監視できるのは URL の入力だけ） |
| `--dry-run` | ファイルを書かずに実行 |
| `-d, --debug` / `-s, --silent` | デバッグログ / 出力を抑える |
| `-l, --logs` / `--no-log-file` | ログの出力先 / ログファイルを作らない |

## 設定ファイル

プロジェクトのルートに `openapi-ts.config.ts` （ `.mjs` ・ `.cjs` ・ `.js` も可）を置く。

```
// openapi-ts.config.ts
import { defineConfig } from '@hey-api/openapi-ts';

export default defineConfig({
  input: './openapi.yaml',          // パス・URL・'hey-api/backend' のようなレジストリ指定
  output: {
    path: 'src/client',
    postProcess: ['prettier'],      // 生成後に整形（'biome:format'・'oxfmt'・'eslint' なども可）
  },
  plugins: [
    '@hey-api/typescript',          // types.gen.ts
    '@hey-api/sdk',                 // sdk.gen.ts
    '@hey-api/client-fetch',        // client.gen.ts（既定のクライアント）
  ],
});
```

input の書き方

output の主な項目

複数の定義

```
// 認証が要る URL から取る
input: {
  path: 'https://api.example.com/openapi.json',
  fetch: { headers: { Authorization: 'Bearer xxx' } },
  watch: true,                      // 開発中に URL を監視して再生成
},
```

自己署名証明書の HTTPS から読むときは、環境変数 `NODE_TLS_REJECT_UNAUTHORIZED=0` が要る（開発時のみ）。

| キー | 意味 |
| --- | --- |
| `path` | 出力フォルダ |
| `postProcess` | 生成後に走らせるツール。 `prettier` ・ `biome:format` ・ `biome:check` ・ `oxfmt` ・ `oxlint` ・ `eslint` |
| `entryFile` | `false` で `index.ts` を作らない |
| `fileName.suffix` | ファイル名の接尾辞（既定 `.gen` 、 `null` で無し） |
| `module.extension` | import に付ける拡張子（Node の ESM なら `'.js'` ） |
| `source` | `true` で入力定義のコピー `source.json` も出力 |

```
// 定義ごとに別フォルダへ（入力と出力の配列を同じ長さに）
export default defineConfig({
  input: ['./foo.yaml', './bar.yaml'],
  output: ['src/foo', 'src/bar'],
});

// 1 つにまとめる（出力は 1 つ）
export default defineConfig({
  input: ['./foo.yaml', './bar.yaml'],
  output: 'src/client',
});

// 設定ごと分けるなら配列で export する
export default defineConfig([
  { input: './foo.yaml', output: 'src/foo' },
  { input: './bar.yaml', output: 'src/bar', plugins: ['zod'] },
]);
```

（元のページで操作できます → https://cs.taka1156.site/hey-api-openapi-ts.html#config）

## 生成されるファイル

src/client/ 出力フォルダ。手で編集しない（再生成で消える）

├ types.gen.ts スキーマ・リクエスト・レスポンスの型。型だけなのでバンドルに残らない

├ sdk.gen.ts operation ごとの関数（ `operationId` から名前が付く）

├ client.gen.ts SDK が使う既定の `client` インスタンス

├ client/ ・ core/ HTTP クライアント本体。定義に依存しない土台

└ index.ts 型と SDK の再 export（ `output.entryFile: false` で作らない）

## 生成したクライアントを使う

呼び出し

baseUrl・認証

起動時の設定

インターセプター

```
import { addPet, getPetById } from './client/sdk.gen';

// 引数は path・query・header・body に分けて渡す（paramsStructure: 'grouped' が既定）
const { data, error, response } = await getPetById({
  path: { petId: 1 },
});
if (error) {
  console.error(response?.status, error);   // error は定義の 4xx/5xx の型
} else {
  console.log(data.name);                   // data は 2xx の型
}

// 例外を投げさせて data だけ受け取る
const { data: pet } = await addPet({
  body: { name: 'Fluffy' },
  throwOnError: true,
});
```

既定（ `throwOnError: false` ）では HTTP エラーでも例外にならず、戻り値の `error` に入る。

```
import { client } from './client/client.gen';

client.setConfig({
  baseUrl: 'https://api.example.com',
  auth: () => localStorage.getItem('token') ?? undefined,  // securitySchemes がある operation にだけ付く
});

// 1 回だけ別の設定で呼ぶ
await getPetById({ path: { petId: 1 }, baseUrl: 'https://staging.example.com' });
```

```
// openapi-ts.config.ts
plugins: [
  { name: '@hey-api/client-fetch', runtimeConfigPath: '../hey-api.ts' },
],

// src/hey-api.ts（出力フォルダからの相対パスで解決される: 0.97〜）
import type { CreateClientConfig } from './client/client.gen';

export const createClientConfig: CreateClientConfig = (config) => ({
  ...config,
  baseUrl: import.meta.env.VITE_API_URL,
});
```

`setConfig()` より前にリクエストが走る心配がない。 `client.gen.ts` が初期化時にこの関数を呼ぶ。

```
import { client } from './client/client.gen';

const id = client.interceptors.request.use((request, options) => {
  request.headers.set('X-Request-Id', crypto.randomUUID());
  return request;
});
client.interceptors.response.use((response) => {
  if (response.status === 401) location.href = '/login';
  return response;
});
client.interceptors.request.eject(id);   // 外す（ID か関数の参照で）
```

（元のページで操作できます → https://cs.taka1156.site/hey-api-openapi-ts.html#usage）

## 主なプラグイン

文字列で名前だけ書くか、 `{ name, ...options }` で設定付きで書く。同じプラグインを 2 回書くと設定はマージされる（0.99〜）。

型・SDK

クライアント

検証・変換

状態管理

| プラグイン | 生成物・主なオプション |
| --- | --- |
| `@hey-api/typescript` | `types.gen.ts` 。 `enums: 'javascript'` で enum を実行時のオブジェクトとしても出す（既定 `false` は型だけ） |
| `@hey-api/sdk` | `sdk.gen.ts` 。 `operations.strategy: 'flat'` （既定・関数、tree-shaking 可）／ `'single'` （クラス）、 `paramsStructure: 'flat'` で引数を 1 階層に、 `validator` ・ `transformer` |
| `@hey-api/schemas` | `schemas.gen.ts` 。定義の JSON Schema をそのまま値として出す（フォーム生成などに） |

| プラグイン | 使う場面 |
| --- | --- |
| `@hey-api/client-fetch` | 既定。Fetch API の薄いラッパー |
| `@hey-api/client-axios` | Axios を使っている（ `axios` が別途必要） |
| `@hey-api/client-next` | Next.js の fetch 拡張（キャッシュ・revalidate）を使う |
| `@hey-api/client-nuxt` | Nuxt の `$fetch` ・ `useFetch` を使う |
| `@hey-api/client-ofetch` / `@hey-api/client-ky` | ofetch / ky を使う |
| `@hey-api/client-angular` | Angular の `HttpClient` を使う |

```
plugins: [
  { name: '@hey-api/sdk', validator: 'zod', transformer: true },
  { name: '@hey-api/transformers', dates: true },   // 日付文字列を Date に変換
],
```

| プラグイン | 生成物・用途 |
| --- | --- |
| `zod` | `zod.gen.ts` （ `zPet` など）。SDK の `validator` に指定すると実行時に検証する |
| `valibot` | `valibot.gen.ts` （ `vPet` など）。Zod より小さい |
| `@hey-api/transformers` | `transformers.gen.ts` 。 `dates: true` で `format: date-time` を `Date` に |

検証は実行時のコストがかかるので既定では無効。 `validator: { request: 'zod' }` のように片方だけにもできる。

```
plugins: ['@tanstack/react-query'],   // vue-query・svelte-query・solid-query なども

// 使う側
import { useQuery, useMutation } from '@tanstack/react-query';
import { getPetByIdOptions, addPetMutation } from './client/@tanstack/react-query.gen';

const { data } = useQuery(getPetByIdOptions({ path: { petId: 1 } }));
const mutation = useMutation(addPetMutation());
```

Vue 向けの `@pinia/colada` 、モック用の `msw` などもある（SWR は公式ドキュメント上は計画中。0.99 には文書化されていない `swr` プラグインが含まれている）。

（元のページで操作できます → https://cs.taka1156.site/hey-api-openapi-ts.html#plugins）

## 導入の手順

1. **定義を用意する** `operationId` を全 operation に付けておく（無いとパスとメソッドから関数名が作られ、読みにくい）。定義の書き方は [Swagger 使い方](https://cs.taka1156.site/md/swagger.md)
2. **インストールして設定を書く** `npm i -D -E @hey-api/openapi-ts` と `openapi-ts.config.ts`
3. **生成する** `npm run openapi-ts` 。CI では生成物との差分が無いことを確かめるか、ビルド前に毎回生成する
4. **クライアントを設定する** `baseUrl` ・ `auth` を `setConfig()` か `runtimeConfigPath` で入れる
5. **アップグレードは移行ノートを読んでから** 0.x のマイナー更新でも破壊的変更がある。版を上げたら再生成して型エラーを確かめる

## よくある誤解

✗ 誤解: 生成された .gen.ts を直接直せば済む

✓ 正しくは: 次の生成で上書きされて消える。名前や構造を変えたいときはプラグインのオプション（ `operations.nesting` など）か、入力側（定義・parser の設定）で直す。

✗ 誤解: SDK 関数は HTTP エラーのときに例外を投げる

✓ 正しくは: 既定では投げず、 `{ data, error, request, response }` の `error` に入れて返す。例外にしたいときは呼び出しごとか `client.setConfig()` で `throwOnError: true` にする。

✗ 誤解: 型を生成すれば実行時のレスポンスも型どおりだと保証される

✓ 正しくは: 生成されるのは TypeScript の型だけで、実行時には確かめない。サーバーが定義と違う値を返す可能性があるなら、 `validator: 'zod'` などで実行時の検証を有効にする。

✗ 誤解: 0.x でもマイナー更新なら ^ で自動的に上げて大丈夫

✓ 正しくは: 公式も「初期開発中なので版を完全に固定する」よう勧めている。0.96 で Node 20 の打ち切り、0.91 で CommonJS の打ち切りなど、マイナー版で破壊的変更が入る。

## 出典

- [Hey API – Get Started](https://heyapi.dev/docs/openapi/typescript/get-started) （@hey-api/openapi-ts 0.99.0 時点）
- [Hey API – Configuration](https://heyapi.dev/docs/openapi/typescript/configuration) ・ [Output](https://heyapi.dev/docs/openapi/typescript/output)
- [Hey API – Fetch API Client](https://heyapi.dev/docs/openapi/typescript/clients/fetch) ・ [SDK Plugin](https://heyapi.dev/docs/openapi/typescript/plugins/sdk)
- [Hey API – Migrating](https://heyapi.dev/docs/openapi/typescript/migrating)
- [hey-api/hey-api（GitHub）](https://github.com/hey-api/hey-api) （CLI オプションは 0.99.0 の `packages/openapi-ts/src/cli/index.ts` ）

確認日: 2026-11-19
