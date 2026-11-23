<!-- scripts/memo.py build が自動生成。手で編集しない -->

# secretlint 使い方

> API キーや秘密鍵などの認証情報がコミットに混ざるのを防ぐ linter、secretlint v14 の CLI・設定・除外・ルール・pre-commit と CI への組み込みをまとめたチートシート

- URL: https://cs.taka1156.site/secretlint.html
- タグ: セキュリティ, 開発ツール
- 日付: 2026-11-23

---

API キー・トークン・秘密鍵などの認証情報 (credential) がリポジトリに入るのを止める、ルールを差し替えられる linter（v14 系・2026-11 時点）

**ざっくり言うと**

- ファイルの中身をサービスごとのルールで調べ、生のトークンや秘密鍵が書かれていないかを報告する linter。
- v14 から公式ルールは本体に内蔵され、設定ファイルが無くても推奨ルール（ `preset-recommend` ）で動く。
- 一度コミットされた秘密は無効化が要るので、pre-commit と CI の両方で入る前に止めるのが基本。

## 動く仕組み

**対象を集める** 引数の glob で集める。ignore ファイルで除外したものと `node_modules` は飛ばす

**ルールで調べる** `.secretlintrc.*` のルール。無ければ推奨ルール

**報告する** 見つけた値は既定で伏せ字（マスク）にして表示

**終了コード** 0: error なし（warning だけも 0）・1: error あり・2: 設定の誤りなどの異常

## 入れ方と最初の実行

npx で試す

プロジェクトに入れる

Docker

単体バイナリ

```
npx secretlint "**/*"     # 設定なしで推奨ルールを全ファイルに
```

何も出ず終了コードが 0 なら見つからなかった。glob はシェルに展開させないよう必ず `"` で囲む。v13 までの `npx @secretlint/quick-start` は非推奨。

```
npm install secretlint --save-dev   # Node.js 22 以上
npx secretlint --init               # .secretlintrc.json を作る
npx secretlint "**/*"
```

v14 からルールのパッケージは入れなくてよい（入れても読まれない）。グローバルインストールはルールが壊れることがあるので勧められていない。

```
docker run -v `pwd`:`pwd` -w `pwd` --rm -it secretlint/secretlint secretlint "**/*"
```

公式イメージは設定なしで動く。内蔵ルールに加えて SARIF 形式の formatter も入っている。

```
# GitHub の Releases から OS・CPU に合うファイルを落とす
chmod +x ./secretlint
./secretlint "**/*"
./secretlint --init
```

Node.js が無い環境向け。

（元のページで操作できます → https://cs.taka1156.site/secretlint.html#install）

## CLI オプション

| オプション | 意味 |
| --- | --- |
| `--init` | `.secretlintrc.json` を作る（推奨プリセット＋ `package.json` にあるルール。canary があれば推奨の代わりに canary） |
| `--format <name>` | 出力形式。既定は `stylish` 。 `github` ・ `json` ・ `table` ・ `junit` ・ `checkstyle` ・ `mask-result` など |
| `--output <path>` | 結果をファイルに書く。 このとき見つかっても終了コードは 0 |
| `--secretlintrc <path>` | 設定ファイルを指定。v14 はそのパスの 1 ファイルだけ（親を探さない・拡張子を補わない） |
| `--secretlintignore <path>` | 追加で読む除外ファイル（各ディレクトリの `.secretlintignore` は常に読む） |
| `--no-gitignore` | `.gitignore` で除外されたファイルも調べる（v13 から既定で `.gitignore` を尊重） |
| `--no-glob` | 引数を glob と見なさず、そのままのパスとして扱う。 `(group)` ・ `[id]` を含むパス向け |
| `--no-maskSecrets` | 見つけた値を伏せずに表示（既定はマスク。CI のログや AI エージェントへの漏れを防ぐため） |
| `--stdinFileName <name>` | 標準入力を調べる。ファイル名で効くルールが変わるので名前を付ける |
| `--profile` | 結果の代わりに、フェーズ・ルール・ファイルごとの所要時間を表示 |
| `--secretlintrcJSON '<json>'` | 設定を JSON 文字列で渡す |

```
secretlint "src/**/*" "config/*.ini"                     # 範囲を絞る
secretlint --no-glob "src/(auth)/login.ts"               # 括弧を含むパス
echo "$TEXT" | secretlint --stdinFileName=memo.txt       # 標準入力
secretlint --no-gitignore "dist/**/*"                    # ビルド成果物も確かめる
secretlint "**/*" --format=json --output=report.json     # 機械向け（終了コードは 0 になる）
secretlint .zsh_history --format=mask-result --output=.zsh_history   # ファイル内の秘密を伏せ字で上書き
```

## 設定ファイル（.secretlintrc）

形式は `.secretlintrc.{json,yml,js}` 。今のディレクトリからルートに向かって探し、見つからなければ推奨ルールで動く。各ルールに書けるのは `options` （ルールごとの設定）・ `disabled` ・ `allowMessageIds` （特定のメッセージ ID を出さない）の 3 つ。

基本

プリセット内のルールを調整

独自のパターン

ルールを足す

```
{
  "rules": [
    { "id": "@secretlint/secretlint-rule-preset-recommend" }
  ]
}
```

`--init` が作るのはこの形。 `id` は v13 までのパッケージ名と同じで、v14 でもそのまま使える。

プリセットの中のルールに設定を渡すときは、プリセットの `rules` の中に書く。

```
{
  "rules": [
    {
      "id": "@secretlint/secretlint-rule-preset-recommend",
      "rules": [
        {
          "id": "@secretlint/secretlint-rule-aws",
          "options": { "allows": ["/dummy_.+/i"] },
          "allowMessageIds": ["AWSAccountID"]
        },
        { "id": "@secretlint/secretlint-rule-basicauth", "disabled": true }
      ]
    }
  ]
}
```

`allows` は正規表現風の文字列 ( `/…/i` ) で、一致した値を報告しない。メッセージ ID は報告の `[EXAMPLE_MESSAGE]` の部分か、各ルールの説明で分かる。

社内トークンなど決まったルールが無いものは、内蔵の `secretlint-rule-pattern` で正規表現やファイル名を指定する（推奨プリセットには入っていないので `id` を書く）。

```
{
  "rules": [
    { "id": "@secretlint/secretlint-rule-preset-recommend" },
    {
      "id": "@secretlint/secretlint-rule-pattern",
      "options": {
        "patterns": [
          { "name": "env files", "filePathGlobs": ["**/.env", "**/.env.*"] },
          {
            "name": "internal token",
            "patterns": ["/INTERNAL_TOKEN\\s*=\\s*(?<token>[A-Za-z0-9]{16,})/"]
          }
        ]
      }
    }
  ]
}
```

`filePathGlobs` だけならファイルがあるだけで報告、 `patterns` だけなら中身で判定、両方あれば両方に一致したときだけ報告。 `keywords` を書くと、その語を含むファイルだけで正規表現を動かすので速い。

```
{
  "rules": [
    { "id": "@secretlint/secretlint-rule-preset-recommend" },
    { "id": "@secretlint/secretlint-rule-preset-canary" },
    { "id": "@secretlint/secretlint-rule-azure" },
    { "id": "@secretlint/secretlint-rule-no-dotenv" }
  ]
}
```

canary・azure は内蔵なので `id` を書くだけ。 `no-dotenv` など別パッケージのルールは `npm i -D @secretlint/secretlint-rule-no-dotenv` で入れてから書く。

（元のページで操作できます → https://cs.taka1156.site/secretlint.html#config）

## ルールの一覧

| グループ | 動く条件 | 主なルール（ `@secretlint/secretlint-rule-` の後ろ） |
| --- | --- | --- |
| `preset-recommend` | 設定が無いとき・プリセットを書いたとき | aws・gcp・github・gitlab・npm・privatekey・basicauth・slack・sendgrid・shopify・stripe・openai・anthropic・grafana・groq・linear・1password・database-connection-string・databricks・hashicorp-vault・vercel・docker・figma・cloudflare・tailscale・huggingface・notion・filter-comments |
| `preset-canary` | プリセットか各 id を書いたとき | digitalocean・doppler・nuntiq・postman・pypi・replicate・rubygems・sentry・supabase・telegram（将来 recommend に入る候補。パッチ版でも動きが変わりうる） |
| 内蔵・プリセット外 | id を書いたときだけ | azure・pattern |
| 別パッケージ | インストールして id を書く | no-dotenv（ `.env` のコミットを禁止）・no-homedir（自分のホームのパス）・no-k8s-kind-secret（k8s の `kind: Secret` ）・secp256k1-privatekey（暗号資産の秘密鍵） |

## 除外のしかた

| 方法 | 効く範囲 |
| --- | --- |
| 組み込みの除外 | `.git` ・ `node_modules` ・ `.secretlintrc*` ・ `.secretlintignore*` は常に調べない |
| `.gitignore` | 各ディレクトリのものを Git と同じように重ねて尊重（ `--no-gitignore` で無効） |
| `.secretlintignore` | `.gitignore` と同じ書き方。各ディレクトリで重ねて読む |
| `--secretlintignore` | 追加の除外ファイル。パターンは今のディレクトリから見る |
| コメント（ `filter-comments` ） | ファイル内の一部。推奨プリセットに入っているので設定なしで効く |
| `allows` ・ `allowMessageIds` | ルール単位で、特定の値・種類を出さない |

```
// secretlint-disable-next-line
const SAMPLE = "テスト用のダミー値";

/* secretlint-disable @secretlint/secretlint-rule-github */
… この範囲は GitHub のルールだけ止める …
/* secretlint-enable @secretlint/secretlint-rule-github */

# secretlint-disable-line   ← // 以外に # や /* でもよい
```

除外のもと（組み込み・ `.gitignore` ・ `.secretlintignore` ・ `--secretlintignore` ）はそれぞれ独立に判定し、どれか 1 つが除外すれば飛ばす。 `!` で戻せるのは同じファイルが除外したものだけ。ignore ファイルでは `{a,b}` のような波括弧の展開は使えない。

## pre-commit と CI に組み込む

lint-staged

pre-commit

GitHub Actions

SARIF

```
// package.json（Husky などで pre-commit から lint-staged を呼ぶ）
{
  "lint-staged": {
    "*": ["secretlint --no-glob"]
  }
}
```

lint-staged はステージしたファイルのパスをそのまま渡すので、 `(group)` ・ `[param]` などで誤動作しないよう `--no-glob` を付ける。

```
# .pre-commit-config.yaml（pre-commit フレームワーク・Docker で動かす）
-   repo: local
    hooks:
    -   id: secretlint
        name: secretlint
        language: docker_image
        entry: secretlint/secretlint:latest secretlint
```

全リポジトリに効かせたいなら、Git の `core.hooksPath` にフックを置く方法もある（公式の例: `secretlint/git-hooks` ）。

```
# .github/workflows/secretlint.yml
name: Secretlint
on: [push, pull_request]
permissions:
  contents: read
jobs:
  secretlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with:
          node-version: 22
      - run: npm ci
      - run: npx secretlint --format github "**/*"   # PR の該当行に注釈が付く
```

変更したファイルだけ調べるなら、差分のファイル一覧を取って `npx secretlint --no-glob <files>` に渡す。

```
npm install -D @secretlint/secretlint-formatter-sarif
npx secretlint --format @secretlint/secretlint-formatter-sarif "**/*" > secretlint.sarif
```

SARIF はコードスキャンの結果を集める共通の形式。GitHub の code scanning などに取り込める。

（元のページで操作できます → https://cs.taka1156.site/secretlint.html#integrate）

## v13 → v14 で変わったこと

| 変更 | やること |
| --- | --- |
| 公式ルール・プリセットが `@secretlint/core` に内蔵 | `@secretlint/secretlint-rule-*` を uninstall してよい（id はそのまま） |
| 設定が無くても推奨ルールで動く | v13 は「config is not found」で終了コード 2 だった。CI の前提を見直す |
| `--secretlintrc` は指定した 1 ファイルだけ | 親ディレクトリにある設定は `../.secretlintrc.json` のようにパスで書く |
| `--secretlintignore` は追加のファイルへのパス | v13 は各ディレクトリで探すファイル名として扱っていた |
| aws ルールの `enableIDScanRule` は真偽値だけ | `"true"` （文字列）を `true` に |
| `--format json` にファイルの中身が入らない | 結果を読む自作の formatter は `getSource(filePath)` を使う |

## よくある誤解

✗ 誤解: secretlint は見つけた秘密を自動で直してくれる

✓ 正しくは: 自動修正はしない。 `--format=mask-result --output=…` でファイル内の値を伏せ字に置き換えられるだけ。コミット・push 済みなら、まず発行元で鍵やトークンを無効化して作り直す。

✗ 誤解: .gitignore している .env の中身も毎回チェックされている

✓ 正しくは: v13 から `.gitignore` で除外されたファイルは既定で調べない。手元の `.env` も確かめたいなら `--no-gitignore` を付ける。 `.env` 自体のコミットを禁止したいなら `no-dotenv` ルールを足す。

✗ 誤解: CI で --output を付けても、見つかればジョブは失敗する

✓ 正しくは: `--output` を付けると、見つかっても終了コードは 0 になる。ファイルに残しつつ失敗させたいなら、表示用と保存用で 2 回実行するか、結果のファイルを見て判定する。

✗ 誤解: secretlint を通ったので、リポジトリに秘密は無い

✓ 正しくは: 調べるのはルールが知っている形式と、今のファイルの中身だけ。独自形式のトークンは `pattern` ルールで足し、Git の履歴に残ったものは別の方法で確かめる。

## 関連メモ

- [Trivy 使い方](https://cs.taka1156.site/md/trivy.md) — `secret` スキャナーでイメージやファイルの秘密も探せる。依存の脆弱性とまとめて見るならこちら
- [OSV-Scanner 使い方](https://cs.taka1156.site/md/osv-scanner.md) — 秘密ではなく依存パッケージの既知の脆弱性を調べる、CI で並べて使う相方

## 参考

- [secretlint/secretlint（README）](https://github.com/secretlint/secretlint)
- [Configuring Secretlint](https://github.com/secretlint/secretlint/blob/main/docs/configuration.md)
- [Migration guide: Secretlint v13 to v14](https://github.com/secretlint/secretlint/blob/main/docs/migrations/v13-to-v14.md)
- [secretlint-rule-pattern](https://github.com/secretlint/secretlint/blob/main/docs/rules/pattern.md) ・ [secretlint-rule-filter-comments](https://github.com/secretlint/secretlint/blob/main/docs/rules/filter-comments.md)
- [secretlint（npm）](https://www.npmjs.com/package/secretlint)

確認日: 2026-11-23（secretlint 14.1.0）。
