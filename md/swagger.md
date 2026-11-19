<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Swagger 使い方

> OpenAPI 定義の書き方と、Swagger UI・Swagger Editor の導入方法・主な設定をまとめたチートシート

- URL: https://cs.taka1156.site/swagger.html
- タグ: API・バックエンド
- 日付: 2026-11-19

---

OpenAPI 定義を書いて、Swagger UI でドキュメント化し、ブラウザから試すまで

**ざっくり言うと**

- Swagger は OpenAPI 定義（YAML / JSON）を書く・見る・試すためのツール群で、仕様そのものの名前は OpenAPI。
- Swagger UI は定義ファイルを読み込んで、API の一覧と「Try it out」で実際にリクエストを送れる画面を作る。
- 正は定義ファイルの側なので、まず `openapi.yaml` を整え、UI やコード生成はそこから作る。

## ツールの関係

**Swagger Editor** ブラウザで定義を書く・検証する

**サーバー実装** FastAPI・springdoc などがコードから定義を出力

**openapi.yaml** OpenAPI 定義

2.0 / 3.0 / 3.1 / 3.2

**Swagger UI** 定義を読んで API ドキュメントと Try it out を表示

**コード生成** Swagger Codegen・ [Hey API](https://cs.taka1156.site/md/hey-api-openapi-ts.md) などで型やクライアントを生成

矢印の手前（PC では左、スマホでは上）が定義を「作る」側、先が定義を「使う」側。どちらも真ん中の定義ファイルだけを介してつながる。

## OpenAPI 定義の書き方

トップレベルの主なキー。 `openapi` ・ `info` は必須で、3.1 以降は `paths` ・ `components` ・ `webhooks` のどれか 1 つがあればよい。

openapi: 3.1.0 仕様のバージョン（2.0 の定義は `swagger: "2.0"` と書く）

info title・version（API 自体の版）・description

servers ベース URL の一覧。Try it out の送信先になる

tags エンドポイントのグループ。UI の見出しになる

paths `/users/{id}` → `get` ・ `post` … → parameters・requestBody・responses

components schemas・parameters・responses・securitySchemes など、 `$ref` で使い回す部品

security 全体に掛ける認証（各 operation で上書きできる）

最小例

パラメータ

リクエストボディ

認証

```
openapi: 3.1.0
info:
  title: Pet API
  version: 1.0.0
servers:
  - url: https://api.example.com/v1
paths:
  /pets:
    get:
      operationId: listPets      # 生成されるクライアントの関数名になる
      tags: [pets]
      summary: ペット一覧
      responses:
        '200':                   # ステータスコードは文字列で書く
          description: OK
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Pet'
components:
  schemas:
    Pet:
      type: object
      required: [id, name]
      properties:
        id:   { type: integer, format: int64 }
        name: { type: string }
        tag:  { type: [string, 'null'] }   # 3.1 の nullable の書き方
```

```
paths:
  /pets/{petId}:
    get:
      operationId: getPetById
      parameters:
        - name: petId
          in: path            # path / query / header / cookie
          required: true      # path は必ず true
          schema: { type: integer }
        - name: fields
          in: query
          schema:
            type: array
            items: { type: string }
          style: form         # ?fields=a&fields=b（explode: true が既定）
      responses:
        '200': { $ref: '#/components/responses/PetOk' }
        '404': { description: Not Found }
```

```
paths:
  /pets:
    post:
      operationId: addPet
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/NewPet' }
          multipart/form-data:          # ファイルアップロード
            schema:
              type: object
              properties:
                photo: { type: string, contentMediaType: image/png }
      responses:
        '201': { description: Created }
```

```
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
    apiKey:
      type: apiKey
      in: header
      name: X-API-Key
security:
  - bearerAuth: []        # 全エンドポイントに適用

paths:
  /health:
    get:
      security: []        # 空配列でこの operation だけ認証なし
```

Swagger UI の右上「Authorize」ボタンは、ここで定義した securitySchemes から作られる。

（元のページで操作できます → https://cs.taka1156.site/swagger.html#spec）

## Swagger UI を表示する

npm パッケージは 3 種類。バンドラーを使うなら `swagger-ui` 、React なら `swagger-ui-react` 、静的ファイルをサーバーから配るなら `swagger-ui-dist` 。

HTML（CDN）

npm

React

Express

Docker

```
<link rel="stylesheet" href="https://unpkg.com/swagger-ui-dist@5/swagger-ui.css">
<div id="swagger-ui"></div>
<script src="https://unpkg.com/swagger-ui-dist@5/swagger-ui-bundle.js" crossorigin></script>
<script>
  window.onload = () => {
    window.ui = SwaggerUIBundle({
      url: '/openapi.yaml',     // 定義ファイルの URL
      dom_id: '#swagger-ui',
    });
  };
</script>
```

本番では版を固定する（例 `swagger-ui-dist@5.35.0` ）。上部バーを出すなら `swagger-ui-standalone-preset.js` も読み込み、 `presets` と `layout: 'StandaloneLayout'` を指定する。

```
npm install swagger-ui

// main.js
import SwaggerUI from 'swagger-ui';
import 'swagger-ui/dist/swagger-ui.css';

SwaggerUI({
  dom_id: '#swagger-ui',
  url: '/openapi.yaml',
});
```

```
npm install swagger-ui-react

import SwaggerUI from 'swagger-ui-react';
import 'swagger-ui-react/swagger-ui.css';

export default function ApiDocs() {
  return <SwaggerUI url="/openapi.yaml" docExpansion="none" />;
}
```

```
npm install swagger-ui-express yaml

import express from 'express';
import swaggerUi from 'swagger-ui-express';
import { parse } from 'yaml';
import { readFileSync } from 'node:fs';

const spec = parse(readFileSync('./openapi.yaml', 'utf8'));
const app = express();

app.use('/docs', swaggerUi.serve, swaggerUi.setup(spec));
app.listen(3000);   // http://localhost:3000/docs
```

`swagger-ui-express` は Swagger 公式ではなくコミュニティのパッケージ。中身は `swagger-ui-dist` を配っている。

```
# 手元の定義ファイルを表示（/bar/openapi.yaml をマウント）
docker run -p 80:8080 \
  -e SWAGGER_JSON=/foo/openapi.yaml \
  -v /bar:/foo \
  docker.swagger.io/swaggerapi/swagger-ui

# 外部 URL の定義を表示
docker run -p 80:8080 \
  -e SWAGGER_JSON_URL=https://petstore3.swagger.io/api/v3/openapi.json \
  docker.swagger.io/swaggerapi/swagger-ui

# /swagger 配下で配る
docker run -p 80:8080 -e BASE_URL=/swagger ... docker.swagger.io/swaggerapi/swagger-ui
```

設定は下の表の「Docker 変数」列の環境変数で渡せる。コンテナ内のポートは 8080（ `PORT` で変更）。

（元のページで操作できます → https://cs.taka1156.site/swagger.html#ui）

## Swagger UI の主な設定

`SwaggerUIBundle({ ... })` ・ `<SwaggerUI ... />` の props・Docker の環境変数で同じ設定を渡せる。

| キー | Docker 変数 | 既定値 | 用途 |
| --- | --- | --- | --- |
| `url` | `URL` | — | 定義ファイルの URL |
| `urls` | `URLS` | — | `[{ url, name }]` で複数の定義を上部バーで切り替える（ `url` より優先） |
| `spec` | `SPEC` | `{}` | 定義をオブジェクトで直接渡す（ `url` は無視される） |
| `dom_id` | `DOM_ID` | — | 描画先の要素（ `domNode` で要素を直接渡してもよい） |
| `docExpansion` | `DOC_EXPANSION` | `"list"` | 初期の開き方。 `list` （タグだけ）・ `full` （全部）・ `none` （全部閉じる） |
| `deepLinking` | `DEEP_LINKING` | `false` | タグ・operation を URL の `#` で直接開けるようにする |
| `filter` | `FILTER` | `false` | タグ名・パスで絞り込む入力欄を出す（大文字小文字を区別しない部分一致。文字列なら初期値） |
| `tryItOutEnabled` | `TRY_IT_OUT_ENABLED` | `false` | 最初から Try it out の入力欄を開いておく |
| `displayRequestDuration` | `DISPLAY_REQUEST_DURATION` | `false` | Try it out の所要時間（ms）を表示 |
| `persistAuthorization` | `PERSIST_AUTHORIZATION` | `false` | Authorize で入れたトークンをリロード後も保持 |
| `defaultModelsExpandDepth` | `DEFAULT_MODELS_EXPAND_DEPTH` | `1` | 下部の Schemas 欄の展開の深さ。 `-1` で欄ごと隠す |
| `supportedSubmitMethods` | `SUPPORTED_SUBMIT_METHODS` | 全メソッド | Try it out を許すメソッド。 `[]` で送信を全部無効（閲覧専用） |
| `validatorUrl` | `VALIDATOR_URL` | swagger.io の検証サーバー | 定義の検証バッジ。 `null` で外部への送信を止める |
| `requestInterceptor` | — | `a => a` | Try it out のリクエストを書き換える（ヘッダー追加など） |
| `withCredentials` | `WITH_CREDENTIALS` | `false` | Try it out で Cookie などの認証情報を送る |
| `configUrl` | `CONFIG_URL` | — | 設定を外部の JSON / YAML から読む |

## Swagger Editor で定義を書く

- ブラウザ版: [editor.swagger.io](https://editor.swagger.io/) 。左で YAML を書くと、右に Swagger UI のプレビューとエラーが出る
- ローカル: `docker run -d -p 8080:80 docker.swagger.io/swaggerapi/swagger-editor:latest` → `http://localhost:8080` （SwaggerEditor v5。旧版 v4 は OpenAPI 3.1 に対応していない）
- VS Code で書くなら、OpenAPI 用の拡張機能（スキーマ補完・プレビュー）を使う方法もある

## 典型的な流れ

1. **定義を書く** `openapi.yaml` を Swagger Editor などで書き、エラーが無い状態にする。サーバーのフレームワークが定義を出力するならそれを使う
2. **リポジトリで管理する** 定義ファイルを API のコードと同じリポジトリに置き、変更をレビューする
3. **Swagger UI で公開する** 上の導入方法のどれかで `/docs` などに置く。定義ファイル自体も同じオリジンから配ると CORS で困らない
4. **型・クライアントを生成する** フロントエンドは定義から型付きのクライアントを生成する（例: [Hey API openapi-ts](https://cs.taka1156.site/md/hey-api-openapi-ts.md) ）

## よくある誤解

✗ 誤解: Swagger と OpenAPI は同じものの別名だ

✓ 正しくは: 仕様はもともと「Swagger 仕様」と呼ばれていたが、2015 年末に Swagger 2.0 が OpenAPI Initiative に寄贈され、「OpenAPI 仕様」と呼ばれるようになった（2.0 は OpenAPI 2.0 とも呼ぶ）。いまの Swagger は SmartBear 社のツール群（UI・Editor・Codegen）の名前。定義ファイルの先頭も 2.0 は `swagger: "2.0"` 、3.x は `openapi: 3.x.x` と違う。

✗ 誤解: Try it out はサーバー側から送るので CORS は関係ない

✓ 正しくは: Try it out はブラウザから直接 API に送る。Swagger UI と API のオリジンが違えば、API 側で CORS を許可しないと失敗する。Swagger UI を API と同じオリジンで配るのがいちばん簡単。

✗ 誤解: Swagger UI を置けば API のドキュメントは常に正しい

✓ 正しくは: Swagger UI は渡された定義を表示するだけで、実装と一致しているかは確かめない。定義を手で書く場合は、契約テストやコードからの自動出力で実装とのずれを防ぐ。

✗ 誤解: 本番環境に Swagger UI をそのまま出しても問題ない

✓ 正しくは: 内部向けのエンドポイントまで一覧され、Try it out で誰でもリクエストを送れる。公開するなら認証の内側に置くか、 `supportedSubmitMethods: []` で送信を止め、 `validatorUrl: null` で定義の外部送信も止める。

## 出典

- [Swagger UI – Installation](https://github.com/swagger-api/swagger-ui/blob/master/docs/usage/installation.md) （swagger-ui-dist 5.35.0 時点）
- [Swagger UI – Configuration](https://github.com/swagger-api/swagger-ui/blob/master/docs/usage/configuration.md)
- [Swagger Editor（GitHub）](https://github.com/swagger-api/swagger-editor)
- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [swagger-ui-express（npm）](https://www.npmjs.com/package/swagger-ui-express)

確認日: 2026-11-19
