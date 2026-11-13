<!-- scripts/memo.py build が自動生成。手で編集しない -->

# cf/config チートシート

> Cloudflare の新 CLI cf の設定ファイル cloudflare.config.ts の書き方（bindings・triggers・mode）をまとめた。

- URL: https://cs.taka1156.site/cf-config.html
- タグ: Cloudflare, Workers, TypeScript
- 日付: 2026-11-13

---

# cf/config

Cloudflare の新 CLI「cf」の設定ファイル `cloudflare.config.ts` を書くためのモジュール（open beta）

**ざっくり言うと**

- `cf/config` は `cloudflare.config.ts` で import するヘルパー集。 `wrangler.jsonc` / `wrangler.toml` の代わりに、Worker の設定を **型付きの TypeScript** で書く
- 中心は `defineConfig` （default export）。バインディングは `bindings.*` 、起動条件は `triggers.*` 、Durable Object などは `exports.*` で組み立てる
- 関数形式にすると `{ mode, isPreview }` を受け取れ、 `--mode` やプレビューごとに設定を切り替えられる。2026 年 9 月 28 日開始の open beta なので形式は変わりうる

## 全体像（どのコマンドが何を読むか）

**cf/config** defineConfig / bindings / triggers / exports などを提供

npm i -D cf で解決できるようにする

import → ↓ import

**cloudflare.config.ts** Worker・Containers・アカウントの設定

cf は一番近いこのファイルを読む

↓ 読み込むコマンド

**cf dev / build / deploy** 設定を丸ごと読み、ビルドツールに渡す

Cloudflare Vite プラグイン（既定）か Wrangler 4.136.0 以降

**cf d1 list などの API コマンド** 読むのは `accountId` と `complianceRegion` だけ

**cf workers types** 設定を評価して

`.cloudflare/types/index.d.ts` （ `Env` 型）を書き出す

cf 自体は dev サーバーもバンドラーも持たず、Vite（など）か Wrangler に処理を渡す。 `package.json` の scripts も実行しないので、追加のビルド手順は自分の `build` スクリプトにつなげる。

## 動かす前提

| 項目 | 内容 | 守らないと |
| --- | --- | --- |
| Node.js 22.18 以上 | Bun は非対応 | `cloudflare.config.ts loading is not supported on Bun` |
| `"type": "module"` | `package.json` に書く | 未指定だと毎回警告。 `"commonjs"` だと `Cannot use import statement outside a module` |
| `cf` を devDependency に | `npm i -D cf` （ `cf init` で作ったプロジェクトには入っている） | `cf/config` が解決できない |
| import に `.ts` 拡張子 | 別ファイルの設定を import するとき（Node は拡張子を補わない） | `ERR_MODULE_NOT_FOUND` |
| `tsconfig.json` | `allowImportingTsExtensions` を有効にし、 `.cloudflare/types` を `include` に明示 | 型が効かない（ワイルドカードはドットで始まるフォルダを拾わない） |

```
// tsconfig.json
{
  "compilerOptions": {
    "allowImportingTsExtensions": true,
    "noEmit": true
  },
  "include": ["src", "cloudflare.config.ts", ".cloudflare/types"]
}
```

## cf/config が export するもの

| 名前 | 用途 |
| --- | --- |
| `defineConfig` | 設定本体。 `export default` で使う。オブジェクト・Promise・ `({ mode, isPreview }) => 設定` の関数のどれでも渡せる |
| `defineWorker` | 別の設定ファイルから import できる Worker 定義を作る（default export の代わりにはならない） |
| `defineContainer` | Container の定義を Durable Object の export と top-level の `containers` で共有する |
| `bindings` | `worker.env` に入れるバインディングのビルダー（ `bindings.d1()` など） |
| `triggers` | `fetch` / `queue` / `scheduled` / `email` / `connect` の起動条件ビルダー |
| `exports` | `worker` / `durableObject` / `workflow` の名前付き export のビルダー |

## 最小構成

```
// cloudflare.config.ts
import { defineConfig } from "cf/config";
import * as entrypoint from "./src/index.ts" with { type: "cf-worker" };

export default defineConfig({
  worker: {
    name: "example-worker",          // 必須
    entrypoint,                      // コードを動かす Worker は必須
    compatibilityDate: "2026-11-13", // 必須（yyyy-mm-dd、ランタイムの版）
  },
});
```

`entrypoint` はパス文字列でもよいが、 `with { type: "cf-worker" }` で import したモジュールを渡す形が基本。アセットだけの Worker なら省略できる。

## 設定キー一覧

### トップレベル

| キー | 説明 |
| --- | --- |
| `accountId` | 既定のアカウント。環境変数 `CLOUDFLARE_ACCOUNT_ID` が優先 |
| `complianceRegion` | `"public"` か `"fedramp-high"` 。 `CLOUDFLARE_COMPLIANCE_REGION` が優先 |
| `worker` dev・build で必須 | Worker の定義。アカウント設定だけのファイルでも API コマンドには使えるが、Vite プラグインと Wrangler には `worker` が要る |
| `containers` | Container アプリケーションの定義（ `defineContainer` の配列） |

### `worker` の中

| キー | 説明 |
| --- | --- |
| `name` 必須 | Worker 名 |
| `compatibilityDate` 必須 | `yyyy-mm-dd` 。ランタイムの版を決める |
| `entrypoint` | パス文字列か `cf-worker` で import したモジュール。コードを持つ Worker では必須 |
| `compatibilityFlags` | 文字列の配列（例: `["enable_ctx_exports"]` ） |
| `env` | バインディング。値が `false` / `null` / `undefined` の項目は省かれるので、条件付きバインディングが書ける |
| `triggers` | 起動条件の配列（ `triggers.*` ） |
| `exports` | 名前付きの Worker / Durable Object / Workflow export |
| `assets` | 実行時の振る舞いだけ（HTML の扱い、not found の扱い、 `basePath` 、 `runWorkerFirst` ）。アセットの置き場所は指定しない |
| `domains` | カスタムドメイン。文字列は本番のみ、オブジェクトなら本番・Preview・両方を選べる |
| `cache` | `enabled` 、 `crossVersionCache` |
| `limits` | `cpuMs` 、 `subrequests` （Standard 使用量モデルのみ） |
| `observability` | ログ・トレース・サンプリング、 `redactQueryString` 、 `issues` |
| `placement` | `smart` / `off` 、またはリージョン・ホスト・ホスト名を指定する配置 |
| `previewUrls` / `workersDev` | 真偽値のスイッチ |
| `logpush` | トレースイベントを Workers Logpush に送るか（真偽値） |
| `tailConsumers` | Tail Worker の指定。 `streaming` フラグ付き可 |
| `access` | dev での Cloudflare Access のシミュレーション（ `aud` と疑似 `identity` ） |
| `unsafe` | `metadata` と `capnp` の素通し |

（元のページで操作できます → https://cs.taka1156.site/cf-config.html#keys）

## bindings（worker.env に入れるもの）

値

ストレージ

AI・メディア

連携・その他

`text` ・ `json` ・ `secret` 。 `secret()` は値を書かず、ローカル dev での存在チェックも有効になる。

`d1` ・ `kv` ・ `r2` ・ `hyperdrive` ・ `analyticsEngineDataset` ・ `artifacts` ・ `pipeline`

`ai` ・ `aiSearch` ・ `aiSearchNamespace` ・ `agentMemory` ・ `browser` ・ `images` ・ `media` ・ `stream` ・ `vectorize`

**メッセージング:** `queue` （本文の型をジェネリクスで渡せる）・ `sendEmail`

**Worker 間:** `assets` ・ `dispatchNamespace` ・ `durableObject` ・ `worker` ・ `workerLoader` ・ `workflow`

**ネットワーク・セキュリティ:** `mtlsCertificate` ・ `rateLimit` ・ `secretsStoreSecret` ・ `vpcNetwork` ・ `vpcService`

**メタデータ:** `flagship` ・ `logfwdr` ・ `versionMetadata`

```
import { bindings, defineConfig } from "cf/config";
import * as entrypoint from "./src/index.ts" with { type: "cf-worker" };

type Job = { id: string; operation: "index" | "delete" };

export default defineConfig({
  worker: {
    name: "example-worker",
    entrypoint,
    compatibilityDate: "2026-11-13",
    env: {
      API_ORIGIN: bindings.text("https://api.example.com"), // → 文字列リテラル型
      FEATURES: bindings.json({ search: true }),           // → { search: true }
      DATABASE: bindings.d1({ name: "application-db" }),   // → D1Database
      JOBS: bindings.queue<Job>({ name: "application-jobs" }), // → Queue<Job>
      API_KEY: bindings.secret(),                          // → string
    },
  },
});
```

右のコメントは生成される `Env` 型。Vite プラグインが dev・build 中に `.cloudflare/types/index.d.ts` を書き、Vite を使わないなら `cf workers types` で同じファイルを作る。

（元のページで操作できます → https://cs.taka1156.site/cf-config.html#bindings）

## triggers（いつ起動するか）

| ビルダー | 主な引数 | 用途 |
| --- | --- | --- |
| `triggers.fetch` | `{ pattern, zone? }` | ルート（例: `api.example.com/*` ） |
| `triggers.queue` | `{ name, maxBatchSize, maxRetries, ... }` | Queue のコンシューマー |
| `triggers.scheduled` | `{ schedule }` | Cron（例: `"0 * * * *"` ） |
| `triggers.email` | `{ addresses }` | メール受信 |
| `triggers.connect` | `{ protocol: "tcp" \| "udp", port, ... }` | TCP / UDP 接続 |

triggers は「起動するきっかけ」だけを決め、 `env` のバインディングは作らない。Queue に送る側にもなるなら `bindings.queue` も別に書く。

## exports と Durable Object のライフサイクル

| ビルダー | 説明 |
| --- | --- |
| `exports.worker({ cache })` | WorkerEntrypoint の export |
| `exports.durableObject({ storage, container? })` | `storage` は `"sqlite"` （推奨）か `"legacy-kv"` 。Container を付けられるのは sqlite のクラスだけ |
| `exports.workflow({ name, ... })` | `name` はアカウント内で一意。 `jurisdiction` は作成後に変えられない |

Durable Object は `state` でクラスの移り変わりを宣言する（移行用の migration ブロックの代わり）。

| state | 意味 | 併せて書くもの |
| --- | --- | --- |
| `created` （省略時） | 稼働中のクラス | `storage` |
| `deleted` | 削除したクラスを退役させる | — |
| `renamed` | 名前を変えた | `renamedTo` |
| `expecting-transfer` | 別 Worker から namespace を受け取る側（先にデプロイ） | `transferFrom` |
| `transferred` | 渡した側の跡（同じアカウント内のみ） | `transferredTo` |

稼働中の export は `ctx.exports` に出るが、deleted などの跡（tombstone）は出ない。

## mode と isPreview で切り替える

```
import { bindings, defineConfig, triggers } from "cf/config";
import * as entrypoint from "./src/index.ts" with { type: "cf-worker" };

export default defineConfig(({ mode, isPreview }) => {
  const isStaging = mode === "staging";
  return {
    accountId: "<ACCOUNT_ID>",
    worker: {
      name: isStaging ? "example-staging" : "example-worker",
      entrypoint,
      compatibilityDate: "2026-11-13",
      env: {
        MESSAGE: bindings.text(isStaging ? "staging" : "production"),
      },
      triggers: [triggers.scheduled({ schedule: "0 * * * *" })],
    },
  };
});
```

| 状況 | `mode` の値 |
| --- | --- |
| `--mode` / `-m` を付けた | 指定した値 |
| Vite プラグインで `cf dev` | `development` |
| Vite プラグインでビルド（ `cf build` / `cf deploy` ） | `production` |
| Wrangler でのビルド・API コマンド | `undefined` |

- `isPreview` は `cf previews deploy` のとき `true`
- mode ごとに完全な設定を返す。wrangler の `[env.xxx]` のような部分的なマージはない
- 値は `process.env` や任意の TypeScript から作れる

## 関連コマンド

| コマンド | 内容 |
| --- | --- |
| `cf init my-worker` / `cf init .` | `cloudflare.config.ts` 付きのプロジェクトを作る／既存アプリに追加 |
| `cf dev` / `cf dev --mode staging` | ローカル開発 |
| `cf build` / `cf build -m staging` | ビルド |
| `cf deploy --dry-run` | デプロイ内容の確認だけ |
| `cf deploy --mode staging` | mode を指定してデプロイ |
| `cf deploy --prebuilt --mode production` | ビルド済みをデプロイ。ビルド時に記録された mode と一致させる |
| `cf previews deploy [--preview-name x]` | Preview を作る（名前の既定はブランチ名、 `--dry-run` なし） |
| `cf workers types` | `Env` 型を生成 |
| `cf migrate --dry-run` | wrangler の設定を `cloudflare.config.ts` に変換（試行） |

## 設定ファイルの例

triggers 全部

Durable Object

Worker 間

Container

```
import { defineConfig, triggers } from "cf/config";
import * as entrypoint from "./src/index.ts" with { type: "cf-worker" };

export default defineConfig({
  worker: {
    name: "example-worker",
    entrypoint,
    compatibilityDate: "2026-11-13",
    triggers: [
      triggers.fetch({ pattern: "api.example.com/*", zone: "example.com" }),
      triggers.queue({ name: "application-jobs", maxBatchSize: 10, maxRetries: 3 }),
      triggers.scheduled({ schedule: "0 * * * *" }),
      triggers.email({ addresses: ["support@example.com"] }),
      triggers.connect({ protocol: "tcp", port: 5432 }),
    ],
  },
});
```

```
import { defineConfig, exports } from "cf/config";
import * as entrypoint from "./src/index.ts" with { type: "cf-worker" };

export default defineConfig({
  worker: {
    name: "counter-worker",
    entrypoint,
    compatibilityDate: "2026-11-13",
    compatibilityFlags: ["enable_ctx_exports"],
    exports: {
      Counter: exports.durableObject({ storage: "sqlite" }),
      OldCounter: exports.durableObject({ state: "renamed", renamedTo: "Counter" }),
      UnusedCounter: exports.durableObject({ state: "deleted" }),
    },
  },
});
```

```
// api/cloudflare.config.ts
import { defineConfig, defineWorker, exports } from "cf/config";
import * as entrypoint from "./src/index.ts" with { type: "cf-worker" };

export const apiWorker = defineWorker({
  name: "api-worker",
  entrypoint,
  compatibilityDate: "2026-11-13",
  exports: {
    Admin: exports.worker(),
    Counter: exports.durableObject({ storage: "sqlite" }),
  },
});
export default defineConfig({ worker: apiWorker });

// web/cloudflare.config.ts
import { bindings, defineConfig } from "cf/config";
import { apiWorker } from "../api/cloudflare.config.ts"; // .ts 拡張子が必要
import * as entrypoint from "./src/index.ts" with { type: "cf-worker" };

export default defineConfig({
  worker: {
    name: "web-worker",
    entrypoint,
    compatibilityDate: "2026-11-13",
    env: {
      API: bindings.worker({ worker: apiWorker, exportName: "Admin" }),
      COUNTERS: bindings.durableObject({ worker: apiWorker, exportName: "Counter" }),
    },
  },
});
```

Worker 名の文字列でも参照できるが、import した定義を渡したときだけ `exportName` を型で検査できる。

```
import { bindings, defineConfig, defineContainer, exports } from "cf/config";
import * as entrypoint from "./src/index.ts" with { type: "cf-worker" };

const imageProcessor = defineContainer({
  name: "image-processor",
  image: { dockerfile: "./Dockerfile" },
  instanceType: "lite",
  maxInstances: 1,
});

export default defineConfig({
  worker: {
    name: "image-worker",
    entrypoint,
    compatibilityDate: "2026-11-13",
    exports: {
      ImageProcessor: exports.durableObject({ storage: "sqlite", container: imageProcessor }),
    },
    env: {
      IMAGE_PROCESSOR: bindings.durableObject({
        worker: "image-worker",
        exportName: "ImageProcessor",
      }),
    },
  },
  containers: [imageProcessor],
});
```

Container 名は一意で、1 つの Durable Object export にだけ紐づく。Container の変更が反映されるのは `cf deploy` （ `cf workers versions create` はイメージ準備のみ）。

（元のページで操作できます → https://cs.taka1156.site/cf-config.html#examples）

## ハマりどころ

- **wrangler.jsonc は読まれない:** `cf dev` / `build` / `deploy` が見るのは `cloudflare.config.ts` だけ。フレームワークのない Worker プロジェクトでは失敗するので、先に `cf migrate` で変換する（変換結果は `cloudflare.config.ts` と、Wrangler でビルドする場合の `wrangler.config.ts` ）
- **アセットの置き場所は書かない:** `assets.directory` はない。Vite なら client ビルドの出力（ `publicDir` 含む）、Wrangler なら生成された `wrangler.config.ts` で決まる
- **API コマンドも設定を読む:** `cf d1 list` のようなコマンドも一番近い `cloudflare.config.ts` を読み込む（使うのは `accountId` / `complianceRegion` ）。 `accountId` を mode で変えるなら、すべてのコマンドに `--mode` を付ける
- **exports の変更はデプロイ時に効く:** アップロードでは反映されない。exports を変えたバージョンを 100% に上げてからトラフィックを分ける（exports が食い違うバージョン同士の分割デプロイは拒否される）
- **ログインは別:** cf は独自の認証情報を持つので、Wrangler でログイン済みでも `cf auth login` が要る
- **Preview の制約:** Durable Object 管理の Container は Preview 非対応。detached HEAD で CI の変数もないときは `--preview-name` が必須

## 出典

- [Programmatic configuration (cloudflare.config.ts) — Cloudflare Docs](https://developers.cloudflare.com/cf/projects/cloudflare-config/)
- [Projects (how cf runs your project) — Cloudflare Docs](https://developers.cloudflare.com/cf/projects/)
- [Cloudflare CLI is now in beta — Cloudflare Changelog](https://developers.cloudflare.com/changelog/post/2026-09-28-cloudflare-cli-beta/)
- [Introducing cf: the agentic CLI for the entire Cloudflare API — Cloudflare Blog](https://blog.cloudflare.com/cloudflare-cf-cli-launch/)

確認日: 2026-11-13（cf open beta 時点。正式版までに形式が変わる可能性あり）。例の `compatibilityDate` は確認日を入れた仮の値。
