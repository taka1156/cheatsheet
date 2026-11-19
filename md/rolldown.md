<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Rolldown 使い方

> Rust 製バンドラー Rolldown の基本と、Vite 8 での使われ方を図解

- URL: https://cs.taka1156.site/rolldown.html
- タグ: 開発ツール
- 日付: 2026-11-16

---

Rust 製バンドラー。Vite 8 の本番ビルドに使われ、Rollup 互換の API を持つ

**ざっくり言うと**

- Rolldown は Rollup と互換の API を持つ、Rust 製のバンドラー（2026年5月に 1.0 の安定版）
- Vite 8 では本番ビルド（ `vite build` ）の裏側で動くので、普段は設定を直接書かなくてよい
- 単体で使うときは `rolldown.config.js` （または `.mjs` ）に入力と出力を書き、CLI か JS API で実行する

## 基本の流れ

1. **入力** エントリーのファイル（例: `src/main.ts` ）を指定
2. **変換** TypeScript・JSX などを Oxc で変換し、依存関係をたどる
3. **出力** フォルダと形式（ `esm` ・ `cjs` ）を指定して書き出す

## 設定例（単体で使う場合）

### rolldown.config.mjs

```
import { defineConfig } from 'rolldown';

export default defineConfig({
  input: 'src/main.ts',
  output: {
    dir: 'dist',
    format: 'esm',
  },
});
```

### JS API で実行

```
import { rolldown } from 'rolldown';

const bundle = await rolldown({ input: 'src/main.ts' });
await bundle.write({ dir: 'dist', format: 'esm' });
await bundle.close();
```

## Vite 8 から使うとき

| やりたいこと | 書く場所 |
| --- | --- |
| 開発サーバー | `vp dev` （または `vite` ） |
| 本番ビルド | `vp build` （または `vite build` ）。Rolldown が使われる |
| 出力先の変更 | `vite.config.ts` の `build.outDir` |

Vite の設定を変えれば Rolldown の設定も変わる。Vite 8 ではバンドラー単体の設定を別に書く場面は少ない。

## 出典と確認日

- [Rolldown 公式](https://rolldown.rs/)
- [VoidZero: Vite+ alpha の発表](https://voidzero.dev/posts/announcing-vite-plus-alpha)

確認日: 2026-11-16。Rolldown は 2026年5月に 1.0 の安定版となった。使う前に公式で版を確かめる。
