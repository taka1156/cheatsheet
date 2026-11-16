<!-- scripts/memo.py build が自動生成。手で編集しない -->

# oxlint・oxfmt 使い方

> ESLint・Prettier の代わりになる Rust 製の linter と formatter の使い方と、プラグイン一覧

- URL: https://cs.taka1156.site/oxlint-oxfmt.html
- タグ: oxlint, oxfmt, JavaScript
- 日付: 2026-11-16

---

ESLint と Prettier の代わりになる、高速な Rust 製のリンターとフォーマッター

**ざっくり言うと**

- oxlint は ESLint 互換の linter、oxfmt は Prettier 互換の formatter
- Vite+ では `vp check` と `vp fmt` で、設定は `vite.config.ts` の `lint` と `fmt` に書く
- 単体で使う場合の細かいオプションは、各公式ドキュメントで版ごとに確かめる

## oxlint（リンター）

| やりたいこと | Vite+ でのコマンド |
| --- | --- |
| リント | `vp check` |
| 自動修正を付ける | `vp check --fix` |

### 設定（vite.config.ts の lint）

```
lint: {
  ignorePatterns: ['dist/**'],
},
```

型情報を使うルール（type-aware）は、oxlint の設定で `typeAware` を有効にすると使える。

## oxfmt（フォーマッター）

| やりたいこと | Vite+ でのコマンド |
| --- | --- |
| 整形する | `vp fmt` |
| 整形と lint と型チェックを一度に | `vp check` |

### 設定（vite.config.ts の fmt）

```
fmt: {
  semi: true,
  singleQuote: true,
},
```

Prettier の設定（ `.prettierrc` ）の項目は、おおむね同じ名前で書ける。ただし細かい違いがあるため、移行後に差分を確かめる。

## ESLint・Prettier からの置き換え

| 今まで | 置き換え先 |
| --- | --- |
| `.eslintrc` / `eslint.config.js` | oxlint（設定は `lint` に統合） |
| `.prettierrc` | oxfmt（設定は `fmt` に統合） |

ESLint のカスタムプラグインを使っている場合は、移行前に `lint.jsPlugins` で同じルールが動くかを確かめる。

## Oxlint のプラグイン一覧

Oxlint の組み込みプラグインは `plugins` に名前を書いて有効にする。既定で有効なのは `eslint` ・ `typescript` ・ `unicorn` ・ `oxc` 。分野で絞り込める。

すべて

フレームワーク

テスト

スタイル・書き方

品質・バグ検出

言語

インポート・Node.js

アクセシビリティ

ドキュメント

| 分野 | プラグイン | 対象 | 内容 | 備考 |
| --- | --- | --- | --- | --- |
| フレームワーク | `react` | React | React と hooks のルール | — |
| フレームワーク | `react-perf` | React | React の性能に関するルール | — |
| フレームワーク | `nextjs` | Next.js | Next.js 向けのルール | — |
| フレームワーク | `vue` | Vue | Vue の script ブロック向けのルール | — |
| テスト | `jest` | Jest | テストファイル向けのルール | — |
| テスト | `vitest` | Vitest | テストファイル向けのルール | — |
| スタイル・書き方 | `unicorn` | 汎用 | コード品質のルール（eslint-plugin-unicorn） | 既定で有効 |
| 品質・バグ検出 | `eslint` | JavaScript | ESLint 本体のルール | 既定で有効 |
| 品質・バグ検出 | `oxc` | 汎用 | Oxc 独自のルール | 既定で有効 |
| 品質・バグ検出 | `promise` | Promise | Promise の使い方のルール | — |
| 言語 | `typescript` | TypeScript | @typescript-eslint のルール | 既定で有効 |
| インポート・Node.js | `import` | import / export | import / export に関するルール | — |
| インポート・Node.js | `node` | Node.js | Node.js 固有のルール | — |
| アクセシビリティ | `jsx-a11y` | JSX | アクセシビリティのルール | — |
| ドキュメント | `jsdoc` | JSDoc コメント | JSDoc のルール | — |

この分野のプラグインはない。

（元のページで操作できます → https://cs.taka1156.site/oxlint-oxfmt.html#plugins）

### Oxfmt のプラグイン

Oxfmt には現時点で Prettier プラグインの仕組みがない。ロードマップ（beta）には入っている。

## 出典と確認日

- [oxc: linter](https://oxc.rs/docs/guide/usage/linter/plugins)
- [oxc: formatter](https://oxc.rs/docs/guide/usage/formatter/config)
- [Vite+ 設定（fmt・lint）](https://viteplus.dev/config/)

確認日: 2026-11-16。Vite+ は 2026年9月に 1.0 の安定版となった。機能は発展途上のため、Oxlint・Oxfmt を含め版ごとに公式で確かめる。
