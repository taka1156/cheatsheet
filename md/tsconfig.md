<!-- scripts/memo.py build が自動生成。手で編集しない -->

# tsconfig 設定例

> TypeScript の tsconfig.json の書き方と主要オプション、実行環境ごとの選び方を 1 枚で

- URL: https://cs.taka1156.site/tsconfig.html
- タグ: TypeScript
- 日付: 2026-11-16

---

TypeScript プロジェクトの tsconfig.json の書き方と、主要オプションの意味

**ざっくり言うと**

- `strict: true` を基本にし、型の抜けを早めに見つける
- `include` で対象のソース、 `outDir` で出力先を決める
- 実行環境（Node.js かバンドラーか）で `module` と `moduleResolution` を選ぶ

## 基本形（Node.js で ESM を動かす場合）

```
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}
```

`NodeNext` で ESM として出力する場合（ `package.json` に `"type": "module"` がある、または `.mts` を使う）、相対 import には拡張子（ `./util.js` ）が必要。CommonJS では不要。

## 主要オプション

| オプション | 意味 |
| --- | --- |
| `target` | 出力する JS の版（例: `ES2022` ） |
| `lib` | 使える標準ライブラリの型（例: `ES2022` 、ブラウザ向けなら `DOM` を追加） |
| `module` / `moduleResolution` | import の解決方法。実行環境で選ぶ（下の表） |
| `strict` | null チェックや暗黙の any などを厳しくチェックする |
| `esModuleInterop` | CommonJS パッケージを default import で読めるようにする（ `module` を `NodeNext` にすると暗黙に有効） |
| `skipLibCheck` | 型定義ファイル（.d.ts）の検査を省き、速くする |
| `outDir` / `rootDir` | 出力先と、元になるソースのフォルダ |
| `noEmit` | JS を出力せず型チェックだけ行う |
| `verbatimModuleSyntax` | 型だけの import を `import type` と明示させる（公式の推奨設定に含まれる） |
| `include` / `exclude` | 対象に含める・除くファイル |

## 実行環境ごとの選び方

| 実行環境 | module | moduleResolution |
| --- | --- | --- |
| Node.js（ESM） | `NodeNext` | `NodeNext` |
| バンドラー経由（Vite・webpack・esbuild など） | `ESNext` | `bundler` |

## +α: バンドラー固有の設定例

バンドラーの型定義を読み込む場合は `types` に追加する。Vite では `import.meta.env` の型が `vite/client` の型定義で提供される。独自の環境変数の型は `src/vite-env.d.ts` で `ImportMetaEnv` を拡張して定義する。これは Vite を使う場合だけの追加で、汎用の設定には含めない。

## 出典と確認日

- [TypeScript 公式: tsconfig](https://www.typescriptlang.org/tsconfig)
- [TypeScript 公式: moduleResolution](https://www.typescriptlang.org/tsconfig/moduleResolution.html)

確認日: 2026-11-16。TypeScript 6.0 で既定値が変わった（ `strict` は既定で true など）。既定値は版ごとに公式で確かめる。
