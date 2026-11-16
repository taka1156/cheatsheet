<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Vite+ 統合設定

> Vite・Vitest・oxlint・oxfmt・Rolldown を vite.config.ts 1 つにまとめる設定

- URL: https://cs.taka1156.site/vite-plus-integration.html
- タグ: Vite+, Vite, 設定
- 日付: 2026-11-16

---

vite.config.ts 1 つに開発・テスト・リント・整形の設定をまとめる

**ざっくり言うと**

- 設定は `vite.config.ts` に集約し、 `vitest.config.ts` や `.oxlintrc.json` 、 `.prettierrc` は作らない
- コマンドは `vp dev` ・ `vp build` ・ `vp test` ・ `vp check` ・ `vp fmt` の 5 つで足りる
- Vite+ は 2026年9月に 1.0 の安定版となったが、機能は発展途上なので、版と移行手順は公式で確かめる

## vite.config.ts

以下は設定の一例です。オプション名は版によって異なる場合があるため、公式の設定リファレンスと照合してください。

```
import { defineConfig } from 'vite-plus';

export default defineConfig({
  plugins: [],
  build: {
    outDir: 'dist',
  },
  test: {
    include: ['src/**/*.test.ts'],
    environment: 'node',
  },
  lint: {
    ignorePatterns: ['dist/**'],
  },
  fmt: {
    semi: true,
    singleQuote: true,
  },
});
```

## package.json の scripts

```
{
  "scripts": {
    "dev": "vp dev",
    "build": "vp build",
    "test": "vp test",
    "check": "vp check",
    "fmt": "vp fmt"
  }
}
```

## 従来のツールとの対応

| 従来 | Vite+ での置き換え |
| --- | --- |
| Vite のスクリプト | `vp dev` / `vp build` |
| Vitest（ `vitest.config.ts` ） | `vp test` 、設定は `test` |
| ESLint（ `.eslintrc` ） | oxlint、設定は `lint` |
| Prettier（ `.prettierrc` ） | oxfmt、設定は `fmt` |

tsconfig.json の扱いは公式で確認できていないため、現行版で確かめる。型チェックは `vp check` に含まれる。

## 注意

✗ 誤解: vitest.config.ts を残せば共存できる

✓ 正しくは: Vite+ は設定を `vite.config.ts` にまとめる方針で、 `vp migrate` も `.oxlintrc` などをこのファイルへ統合する。別ファイルを残した場合の動作は公式に明記されていないため、残す場合は現行版で確かめる。

✗ 誤解: Vite+ は Vite 本体と置き換えになる

✓ 正しくは: Vite+ は Vite を含む統合 CLI。Vite 8 は Vite+ の中で動く。

## 出典と確認日

- [Vite+ 公式: 設定](https://viteplus.dev/config/)
- [Vite+ 公式 GitHub（README）](https://github.com/voidzero-dev/vite-plus)
- [VoidZero: Vite+ alpha の発表](https://voidzero.dev/posts/announcing-vite-plus-alpha)

確認日: 2026-11-16。Vite+ は 2026年9月に 1.0 の安定版となった。機能は発展途上のため、版ごとに公式で確かめる。
