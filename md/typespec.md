<!-- scripts/memo.py build が自動生成。手で編集しない -->

# TypeSpec 使い方

> tsp コマンド・tspconfig.yaml・言語の書き方・HTTP デコレーター・バージョニング・emitter をまとめたチートシート。

- URL: https://cs.taka1156.site/typespec.html
- タグ: API・バックエンド
- 日付: 2026-11-25

---

API を型で書いて OpenAPI・JSON Schema・コードを生成する Microsoft 製の言語（v1.18 時点）

**ざっくり言うと**

- TypeScript 風の構文で API の形を書く言語で、拡張子は `.tsp` 、CLI は `tsp` 。
- `tsp compile` が型を読み、 `tspconfig.yaml` で選んだ emitter が OpenAPI などを書き出す。
- 安定版は compiler・http・openapi3・json-schema など一部。クライアント／サーバー生成はまだプレビュー。

## 全体の流れ

**main.tsp** model・op・デコレーター

**tsp compile** 型チェック・linter

**@typespec/openapi3** openapi.yaml

**@typespec/json-schema** *.yaml（JSON Schema）

**@typespec/protobuf** *.proto

**http-client-* / http-server-*** クライアント・サーバーのコード

どの emitter を動かすかは `tspconfig.yaml` の `emit` か `--emit` で決める。何も指定しないと型チェックだけで、何も出力されない。

## セットアップ

1. **CLI を入れる** `npm install -g @typespec/compiler` （Node.js LTS が前提）。Node なしなら `curl -fsSL https://typespec.io/install.sh | bash` の単体版（実験的）。
2. **プロジェクトを作る** `tsp init` でテンプレートを選ぶ。REST なら「Generic REST API」（http・rest・openapi・openapi3 が入る）。
3. **依存を入れる** `tsp install` （ `package.json` の依存を入れる。 `npm install` でもよい）。
4. **コンパイル** `tsp compile .` 。保存のたびに作り直すなら `tsp compile . --watch` 。
5. **エディター** VS Code は Marketplace の「TypeSpec」拡張を入れる（ `tsp code install` は非推奨）。

| できるファイル | 中身 |
| --- | --- |
| `main.tsp` | 入口。 `tsp compile .` はこのファイルから読む |
| `tspconfig.yaml` | emitter とそのオプション、linter |
| `package.json` | compiler・ライブラリ・emitter の版 |
| `tsp-output/` | 出力先（既定は `{cwd}/tsp-output` ）。OpenAPI は `tsp-output/@typespec/openapi3/openapi.yaml` |

## tsp コマンド

サブコマンド

compile のオプション

そのほかの CLI

| コマンド | 用途 |
| --- | --- |
| `tsp init [テンプレートURL]` | 新規プロジェクト。URL で独自テンプレートも使える |
| `tsp install` | 依存のインストール |
| `tsp compile <path>` | コンパイル。path はファイルか、 `main.tsp` があるフォルダ |
| `tsp format <glob...>` | 整形。例: `tsp format "**/*.tsp"` |
| `tsp info` | compiler の版・設定の解決結果 |
| `tsp info <emitter>` | その emitter のオプション一覧（例: `tsp info @typespec/openapi3` ） |
| `tsp --version` | 版の確認 |

| オプション | 意味 |
| --- | --- |
| `--emit <name>` | 動かす emitter（複数回指定できる） |
| `--option <emitter>.<key>=<値>` | emitter のオプション。例: `--option @typespec/openapi3.file-type=json` |
| `--output-dir <dir>` | 出力先の上書き |
| `--config <path>` | 使う `tspconfig.yaml` （かそのフォルダ）を指定 |
| `--arg <key>=<値>` | 設定ファイルの `parameters` に値を渡す |
| `--watch` | 変更を見て再コンパイル |
| `--no-emit` | emitter を動かさず型チェックだけ（CI 向け） |
| `--warn-as-error` | 警告でも終了コードを非 0 に |
| `--list-files` | 書き出したファイルの一覧を表示 |
| `--no-linter` | linter を全部止める |
| `--linter-rule <rule>=true\|false` | ルール単位で上書き |
| `--ignore-deprecated` | deprecated の診断を出さない |
| `--import <lib>` | 追加で import するライブラリ |
| `--trace <area>` / `--debug` | 内部ログ（例: `import-resolution.*` ） |
| `--stats` | コンパイルの統計を表示 |

| コマンド | 用途 |
| --- | --- |
| `npx tsp-openapi3 openapi.yaml --output-dir ./tsp` | 既存の OpenAPI 3 を TypeSpec に変換（ `@typespec/openapi3` に同梱）。 `--namespace` で名前空間を指定 |
| `TYPESPEC_NPM_REGISTRY` | 環境変数。 `tsp init` ・ `tsp install` が使う npm レジストリ（社内ミラー用） |

（元のページで操作できます → https://cs.taka1156.site/typespec.html#cli）

## tspconfig.yaml

```
emit:
  - "@typespec/openapi3"
  - "@typespec/json-schema"

output-dir: "{project-root}/generated"   # 既定は {cwd}/tsp-output

options:
  "@typespec/openapi3":
    openapi-versions: ["3.1.0"]           # 既定は ["3.0.0"]。"3.2.0" も可
    file-type: yaml                       # yaml | json（配列で両方）
    emitter-output-dir: "{output-dir}/openapi"
    output-file: "openapi.{version}.yaml"

linter:
  extends:
    - "@typespec/http/all"
```

| キー | 意味 |
| --- | --- |
| `emit` | 動かす emitter（CLI の `--emit` ） |
| `options` | emitter ごとのオプション（CLI の `--option` ） |
| `output-dir` | 全体の出力先。各 emitter は既定で `{output-dir}/<emitter名>` に出す |
| `emitter-output-dir` | emitter 共通のオプション。その emitter だけ出力先を変える |
| `extends` | 別の `tspconfig.yaml` を継承。同じ emitter の設定は丸ごと上書き（キー単位でマージしない） |
| `parameters` | 自前の変数。既定値が必須で、 `--arg` で上書き |
| `environment-variables` | 使う環境変数の宣言（既定値が必須）。 `{env.NAME}` で参照 |
| `linter` | `extends` ・ `enable` ・ `disable` 。v1.16 から `"file:../rules.yaml"` で共有ファイルも参照できる |

変数は `{project-root}` （設定ファイルのあるフォルダ）・ `{cwd}` ・ `{output-dir}` ・ `{emitter-name}` など。環境変数は `environment-variables:` に既定値つきで宣言すると `{env.NAME}` で使える。設定ファイルは入口の .tsp と同じフォルダから親へ向かって探し、最初に見つかったものを使う。CLI の指定は設定ファイルより優先される。

### openapi3 emitter のよく使うオプション

| オプション | 値（既定） | 用途 |
| --- | --- | --- |
| `openapi-versions` | `3.0.0` / `3.1.0` / `3.2.0` （ `["3.0.0"]` ） | 出す OpenAPI の版 |
| `file-type` | `yaml` / `json` （yaml） | 出力形式 |
| `output-file` | `{service-name-if-multiple}.{version}.openapi.yaml` | ファイル名。 `{version}` は版ごとに展開 |
| `omit-unreachable-types` | 真偽値 | op から参照されない型を出さない |
| `seal-object-schemas` | 真偽値（false） | 余分なプロパティを禁止（3.0 は `additionalProperties: false` 、3.1 以降は `unevaluatedProperties: false` ） |
| `operation-id-strategy` | `parent-container` / `fqn` / `explicit-only` | operationId の付け方（既定は `Pets_list` の形） |
| `safeint-strategy` | `int64` / `double-int` （int64） | `safeint` の出し方 |

## 言語の基本

組み込み型

model

enum・union

テンプレート

検証デコレーター

import・namespace

| 型 | 中身・OpenAPI での形 |
| --- | --- |
| `string` / `boolean` / `bytes` | 文字列・真偽値・バイナリ（JSON では base64） |
| `int8` 〜 `int64` , `uint8` 〜 `uint64` | 整数（ `format: int32` など） |
| `integer` / `safeint` | 任意長の整数 ／ JS で安全な範囲の整数 |
| `float32` / `float64` / `decimal` / `numeric` | 小数。 `numeric` は数値全体の親 |
| `utcDateTime` / `offsetDateTime` | 日時（ `format: date-time` ） |
| `plainDate` / `plainTime` / `duration` | 日付・時刻・期間 |
| `url` | `format: uri` |
| `T[]` / `Record<T>` | 配列 ／ 値が T の辞書（ `additionalProperties` ） |
| `[string, int32]` | タプル |
| `unknown` / `void` / `never` / `null` | 何でも ／ 本文なし ／ ありえない ／ null |

```
/** ペット（doc コメントは description になる） */
model Pet {
  id: string;
  name: string;
  age?: int32;              // ? で任意
  tags: string[] = #[];     // 既定値（配列は #[ ]、オブジェクトは #{ }）
  owner: Owner | null;      // null 許容
  kind: "dog" | "cat";      // リテラル型の union
}

scalar petId extends string;                  // 独自のスカラー
@format("email") scalar email extends string; // 形式付き
```

再利用の書き方（ `extends` / `is` / `...` ）は下の「model の再利用」を参照。

```
enum Kind { dog, cat, bird }             // 値は名前と同じ文字列
enum Level { low: 1, high: 10 }          // 値を指定

union Status { "active", "disabled", string }  // 拡張可能な列挙（anyOf）

@discriminated(#{ envelope: "none", discriminatorPropertyName: "kind" })
union Shape { circle: Circle, square: Square } // oneOf + discriminator
model Circle { kind: "circle"; r: float64 }
model Square { kind: "square"; size: float64 }
```

`enum` は閉じた列挙。あとから値が増えるなら、最後に `string` を足した `union` にしておく。

```
model Page<T> {
  items: T[];
  nextLink?: url;
}
model Paged<T extends {}> { value: T[] }   // 制約付き
model Box<T = string> { v: T }            // 既定の型

alias PetPage = Page<Pet>;                // 別名（出力に新しい型は出ない）

op list(): Page<Pet>;
```

| デコレーター | 対象 | OpenAPI での形 |
| --- | --- | --- |
| `@doc("…")` / `/** … */` | なんでも | `description` |
| `@summary("…")` | op など | `summary` |
| `@minLength(n)` / `@maxLength(n)` | 文字列 | `minLength` / `maxLength` |
| `@minValue(n)` / `@maxValue(n)` | 数値 | `minimum` / `maximum` |
| `@minItems(n)` / `@maxItems(n)` | 配列 | `minItems` / `maxItems` |
| `@pattern("^…$")` | 文字列 | `pattern` |
| `@format("email")` | 文字列 | `format` |
| `@secret` | 文字列 | `format: password` |
| `@encode(…)` | 日時・bytes など | 直列化の形（例: `DateTimeKnownEncoding.rfc3339` ） |
| `@visibility(Lifecycle.Read)` | プロパティ | 読み取り専用（ `readOnly` ）。 `Create` ・ `Update` ・ `Query` もある |
| `@example(…)` / `@opExample(…)` | 型 ／ op | `examples` |
| `#deprecated "理由"` | なんでも | 非推奨（ディレクティブ） |

```
import "@typespec/http";     // ライブラリ（package.json に入れておく）
import "./models/pet.tsp";   // 自分のファイル

using Http;                  // 名前空間を開く（書かないと Http.Body のように完全名）

namespace PetStore;          // ファイル全体をこの名前空間に
namespace PetStore.Admin { } // ブロックでも書ける

interface Pets { list(): Pet[]; }   // op のまとまり
```

（元のページで操作できます → https://cs.taka1156.site/typespec.html#lang）

## model の再利用

| 書き方 | 意味 | OpenAPI での形 |
| --- | --- | --- |
| `model Staff extends User { … }` | 継承。親子関係が残る | `allOf: [$ref User]` ＋ 追加分 |
| `model Admin is User { … }` | コピー（デコレーターも引き継ぐ）。関係は残らない | 全プロパティを持つ別スキーマ |
| `model A { ...Timestamps; … }` | プロパティだけ展開（spread） | A に平らに入る |
| `alias PetPage = Page<Pet>;` | 別名。新しい型は作らない | 出ない |
| `OmitProperties<T, "id">` / `PickProperties` / `OptionalProperties` | 組み込みの型変換テンプレート | 変換後の新スキーマ |

## HTTP（@typespec/http）

デコレーター

レスポンス

認証

| デコレーター | 付ける所 | 用途 |
| --- | --- | --- |
| `@service(#{ title: "…" })` | namespace | サービスの入口。無いと paths が空の OpenAPI になる（ `no-service-found` の警告） |
| `@server("https://…", "説明")` | namespace | `servers` |
| `@route("/pets")` | namespace・interface・op | パス。親の route に連結される |
| `@get` `@post` `@put` `@patch` `@delete` `@head` | op | HTTP メソッド。省略時は本文があれば POST、無ければ GET |
| `@path` | 引数 | パス引数。 `@route` に無ければ `/{id}` が自動で付く |
| `@query` / `@query("q")` | 引数 | クエリ。 `@query(#{ explode: true })` で配列の展開 |
| `@header` / `@header("x-req-id")` | 引数・レスポンス | ヘッダー。名前を省くと `contentType` → `content-type` に変換 |
| `@body` | 引数・レスポンス | 本文そのもの（型がそのまま本文） |
| `@bodyRoot` | 引数・レスポンス | 本文の根。中の `@header` などは本文から外れる |
| （何も付けない引数） | 引数 | 本文のプロパティとしてまとめられる |
| `@statusCode` | レスポンスのプロパティ | ステータスコード（ `201` や `400 \| 404` ） |
| `@error` | model | エラー型。 `@statusCode` が無ければ `default` レスポンスになる |
| `@tag("Pets")` | namespace・interface・op | OpenAPI の tags（ `@typespec/openapi` ） |
| `@operationId("…")` | op | operationId の指定（ `@typespec/openapi` ） |

```
// 型を返すだけなら 200 + JSON
@get op read(@path id: string): Pet | NotFoundResponse;

// ステータスやヘッダーを付ける
@post op create(@body pet: Pet): {
  @statusCode _: 201;
  @header location: string;
  @body pet: Pet;
} | ApiError;

// 組み込みのレスポンス型と合成（read と同じパスなので別の route で）
@get @route("/v2") op get2(@path id: string): OkResponse & { @body body: Pet };

@delete op remove(@path id: string): NoContentResponse;   // 204

@error model ApiError {
  @statusCode _: 400 | 500;
  code: string;
  message: string;
}
```

組み込み: `OkResponse` （200）・ `CreatedResponse` （201）・ `AcceptedResponse` （202）・ `NoContentResponse` （204）・ `MovedResponse` （301）・ `NotModifiedResponse` （304）・ `BadRequestResponse` （400）・ `UnauthorizedResponse` （401）・ `ForbiddenResponse` （403）・ `NotFoundResponse` （404）・ `ConflictResponse` （409）。

```
@useAuth(BearerAuth)                                    // Authorization: Bearer
@useAuth(BasicAuth)
@useAuth(ApiKeyAuth<ApiKeyLocation.header, "X-API-KEY">)
@useAuth(ApiKeyAuth<ApiKeyLocation.header, "X-API-KEY"> | BearerAuth)  // どちらか
@useAuth([ApiKeyAuth<ApiKeyLocation.header, "X-API-KEY">, BearerAuth]) // 両方
@useAuth(NoAuth | BearerAuth)                           // 認証なしも可
@useAuth(OAuth2Auth<[MyFlow]>)                          // OAuth2（フローは model で定義）
```

namespace に付けると全体、op や interface に付けるとそこだけ上書き。

（元のページで操作できます → https://cs.taka1156.site/typespec.html#http）

## まとまった例

main.tsp

出力（抜粋）

```
import "@typespec/http";
import "@typespec/openapi";
using Http;

@service(#{ title: "Pet Store" })
@server("https://api.example.com", "本番")
@useAuth(BearerAuth)
namespace PetStore;

model Pet {
  @visibility(Lifecycle.Read) id: string;
  @minLength(1) @maxLength(50) name: string;
  @minValue(0) age?: int32;
  tags: string[] = #[];
}

@error model ApiError {
  @statusCode _: 400 | 404 | 500;
  code: string;
  message: string;
}

model Page<T> { items: T[]; nextLink?: url; }

@route("/pets")
@tag("Pets")
interface Pets {
  @get list(@query limit?: int32 = 20, @query("q") search?: string): Page<Pet> | ApiError;
  @get read(@path id: string): Pet | NotFoundResponse;
  @post create(@body pet: Pet): { @statusCode _: 201; @body pet: Pet } | ApiError;
  @patch update(@path id: string, @body pet: MergePatchUpdate<Pet>): Pet;
  @delete delete(@path id: string): NoContentResponse;
}
```

```
paths:
  /pets:
    get:
      operationId: Pets_list
      parameters:
        - name: limit
          in: query
          ...
    post:
      operationId: Pets_create
      responses:
        '201': ...
        '400': ...   # ApiError の 400 | 404 | 500 がそれぞれ出る
  /pets/{id}:          # @path id が自動でパスに付く
    get:    { operationId: Pets_read }
    patch:
      operationId: Pets_update
      requestBody:
        content:
          application/merge-patch+json: ...
    delete: { operationId: Pets_delete }
security:
  - BearerAuth: []
```

（元のページで操作できます → https://cs.taka1156.site/typespec.html#example）

## バージョニング（@typespec/versioning）

```
import "@typespec/versioning";
using Versioning;

@versioned(Versions)
@service
namespace PetStore;
enum Versions { v1, v2, v3 }

model Pet {
  name: string;
  @added(Versions.v2) nickname?: string;                  // v2 から
  @removed(Versions.v3) legacy?: string;                  // v3 で削除
  @renamedFrom(Versions.v2, "fullName") displayName: string;
  @typeChangedFrom(Versions.v2, int32) weight: float64;
}

@added(Versions.v2) op search(): Pet[];
```

openapi3 emitter は版ごとに 1 ファイル出す（ `openapi.v1.yaml` ・ `openapi.v2.yaml` …）。ほかに `@madeOptional` ・ `@madeRequired` ・ `@useDependency` （依存ライブラリの版を固定）がある。

## 主な emitter・ライブラリ

| パッケージ | 状態 | 用途 |
| --- | --- | --- |
| `@typespec/compiler` | 安定（1.x） | 言語本体と `tsp` |
| `@typespec/http` | 安定（1.x） | HTTP のデコレーター |
| `@typespec/openapi` | 安定（1.x） | `@tag` ・ `@operationId` ・ `@extension` など |
| `@typespec/openapi3` | 安定（1.x） | OpenAPI 3.0 / 3.1 / 3.2 を出す・OpenAPI からの変換 |
| `@typespec/json-schema` | 安定（1.x） | JSON Schema を出す |
| `@typespec/rest` | プレビュー（0.x） | リソース指向の `@resource` など |
| `@typespec/versioning` | プレビュー（0.x） | API の版管理 |
| `@typespec/protobuf` | プレビュー（0.x） | .proto を出す（gRPC） |
| `@typespec/http-client-js` ほか python・java・csharp | プレビュー | クライアント SDK の生成 |
| `@typespec/http-server-js` / `http-server-csharp` | アルファ | サーバーの雛形の生成 |

TypeScript のクライアントや型は、openapi3 で OpenAPI を出してから既存の OpenAPI 用ジェネレーターに渡す使い方も多い。

## ハマりどころ

1. **何も出力されない** `emit` も `--emit` も無いと型チェックだけで終わる。namespace に `@service` が無いと paths が空の OpenAPI になる。
2. **出力先がずれる** 既定の `output-dir` は `{cwd}/tsp-output` 。別のフォルダから `tsp compile path/to/proj` すると、実行した場所に出る。固定したいなら `output-dir: "{project-root}/tsp-output"` 。
3. **デコレーターが見つからない** `import "@typespec/http";` に加えて `using Http;` が要る（無ければ `@Http.route` と書く）。
4. **PATCH の本文が全部必須になる** `@patch(#{ implicitOptionality: true })` は非推奨の警告が出る。merge-patch なら `@body pet: MergePatchUpdate<Pet>` 、そうでなければ任意プロパティの model を自分で書く。
5. **OpenAPI が 3.0 で出る** `openapi-versions` の既定は `["3.0.0"]` 。3.1 が欲しければ明示する。
6. **使っていない型まで出る** 既定ではサービスの namespace 内の型を全部出す。 `omit-unreachable-types: true` で op から参照されるものだけにする。
7. **enum に値を足すと互換が崩れる** クライアントが閉じた列挙として生成するため。増える見込みがあれば `union X { "a", "b", string }` にする。

## よくある誤解

✗ 誤解: TypeSpec は Azure 専用の言語だ

✓ 正しくは: 汎用の API 記述言語で、 `@typespec/*` のライブラリだけで普通の REST API の OpenAPI を出せる。Azure 向けの規約は `@azure-tools/typespec-*` という別のライブラリに分かれている。

✗ 誤解: TypeSpec で書いた制約は実行時に検証される

✓ 正しくは: TypeSpec は設計時のツールで、出力は OpenAPI などの定義ファイル。リクエストの検証は、生成した定義やコードを使うサーバー側で別に行う。

✗ 誤解: extends と is は同じ意味だ

✓ 正しくは: `extends` は親子関係が残り OpenAPI では `allOf` になる。 `is` は中身をコピーした別の型で、関係は出力に残らない。

## 関連メモ

- [Swagger 使い方](https://cs.taka1156.site/md/swagger.md) — TypeSpec が出した OpenAPI を表示・試す側
- [Hey API openapi-ts 使い方](https://cs.taka1156.site/md/hey-api-openapi-ts.md) — 出した OpenAPI から TypeScript のクライアントを作る

## 参考

- [TypeSpec Installation](https://typespec.io/docs/)
- [CLI usage](https://typespec.io/docs/handbook/cli/)
- [Configuration](https://typespec.io/docs/handbook/configuration/configuration/)
- [@typespec/http デコレーター](https://typespec.io/docs/libraries/http/reference/decorators/)
- [@typespec/openapi3 emitter](https://typespec.io/docs/emitters/openapi3/reference/emitter/)
- [@typespec/versioning デコレーター](https://typespec.io/docs/libraries/versioning/reference/decorators/)
- [TypeSpec 1.0 GA](https://typespec.io/blog/typespec-1-0-GA-release/) ・ [1.16.0 リリースノート](https://typespec.io/release-notes/typespec-1-16-0/)

2026-11-25 に確認。@typespec/compiler 1.18.0・openapi3 1.18.0・versioning 0.88.0 で例をコンパイルして確かめた。
