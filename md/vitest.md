<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Vitest 使い方

> Vite の設定で動くテストランナー Vitest 5 の書き方・モック・CLI・設定と、v4 からの移行で落ちる点を用途別にまとめたチートシート

- URL: https://cs.taka1156.site/vitest.html
- タグ: 開発ツール
- 日付: 2026-11-16

---

Vite の設定・プラグインでテストを変換して動かすテストランナー。Jest とほぼ同じ書き方（v5.1 系・2026-11 時点）

**ざっくり言うと**

- Vite と同じ設定・プラグインでテストファイルを変換して実行する、Jest 互換 API のテストランナー。
- `vitest` は端末ではウォッチ、CI や非 TTY では 1 回だけ実行。Vite+ では `vp test` （1 回）を使う。
- v5 は `clearMocks` が既定で有効、 `vi.mock` はトップレベル必須など、v4 から上げると落ちる点がある。

## 動く仕組み

**設定を読む** `vitest.config.*` か `vite.config.*` の `test` （v5 は親ディレクトリを探さない）

**Vite で変換** TS・JSX・エイリアス・プラグインをアプリと同じように処理

**ワーカーで実行** 既定は `pool: 'forks'` 。ファイルごとに分離（ `isolate` ）

**レポート** 端末に表示。html・json・junit などは `.vitest/` に書く（v5）

要件（v5）: Node.js 22.12.0 以上・Vite 6.4.0 以上。 `vite` は peer 依存になったので、Yarn では `vite` を自分で入れる。

## テストの書き方

基本

非同期

each・修飾子

スナップショット

型テスト

```
// src/sum.test.ts（既定の対象は **/*.{test,spec}.?(c|m)[jt]s?(x)）
import { describe, test, expect, beforeEach } from 'vitest'
import { sum } from './sum'

describe('sum', () => {
  beforeEach(() => { /* 各テストの前 */ })

  test('2 つの数を足す', () => {
    expect(sum(1, 2)).toBe(3)              // === で比較
    expect({ a: 1 }).toEqual({ a: 1 })     // 中身で比較
    expect(() => sum()).toThrow('引数')    // 文字列はメッセージの部分一致
  })
})
```

`describe` ・ `test` ・ `expect` は既定では import が要る。 `globals: true` にすると Jest のように import なしで使える（TS では `types: ["vitest/globals"]` も足す）。

```
test('取得できる', async () => {
  await expect(fetchUser(1)).resolves.toMatchObject({ id: 1 })
  await expect(fetchUser(-1)).rejects.toThrow('not found')
})

test('いずれ true になる', async () => {
  await expect.poll(() => queue.size).toBe(0)   // v5: 時間切れで失敗
})
```

v5 では `resolves` ・ `rejects` ・ `toMatchFileSnapshot` を `await` し忘れるとテストが失敗する（v4 は警告だけ）。

```
test.each([
  [1, 1, 2],
  [-1, 1, 0],
])('sum(%i, %i) は %i', (a, b, expected) => {
  expect(sum(a, b)).toBe(expected)
})

test.only('これだけ動かす', () => {})      // CI では allowOnly が false なので失敗する
test.skip('飛ばす', () => {})
test.todo('あとで書く')
test.concurrent('並行に動かす', async () => {})
test('並行にしない', { concurrent: false }, async () => {})  // v5 で .sequential の代わり
```

```
expect(render(<Button />)).toMatchSnapshot()          // __snapshots__/*.snap に保存
expect(format(data)).toMatchInlineSnapshot()          // 初回にテストの中へ書き込まれる
await expect(html).toMatchFileSnapshot('./out.html')  // 任意のファイルと比べる
```

意図して変えたときは `vitest -u` （ `--update` ）で更新。ウォッチ中は `u` キー。

```
// src/types.test-d.ts（*.test-d.ts が型テストの既定の対象）
import { expectTypeOf, assertType, test } from 'vitest'

test('型', () => {
  expectTypeOf(mount).toBeFunction()
  expectTypeOf(mount).parameter(0).toExtend<{ name: string }>()
  // @ts-expect-error name は string
  assertType(mount({ name: 42 }))
})
```

実行は `vitest --typecheck` （中で `tsc` か `vue-tsc` を呼ぶ）。

（元のページで操作できます → https://cs.taka1156.site/vitest.html#write）

## モック（ `vi` ）

| API | やること |
| --- | --- |
| `vi.fn(impl?)` | 呼び出しを記録する関数を作る。 `.mockReturnValue()` ・ `.mockResolvedValue()` ・ `.mockImplementationOnce()` で振る舞いを決める |
| `vi.spyOn(obj, 'method')` | 既存のメソッドを包んで記録する。元に戻せる |
| `vi.mock('./mod', factory?)` | モジュールを差し替える。ファイルの先頭に巻き上げられる（hoist）。factory なしなら自動モック |
| `vi.hoisted(() => …)` | `vi.mock` の factory から使う値を、巻き上げより前に用意する |
| `vi.importActual('./mod')` | factory の中で本物を読み、一部だけ差し替える |
| `vi.mocked(fn)` | TS でモックの型を付けるだけ（実行時は何もしない） |
| `vi.when(spy).calledWith(…).thenReturn(…)` | 引数ごとに返す値を決める（v5 で追加）。 `thenResolve` ・ `thenThrow` などもある |
| `vi.useFakeTimers()` | タイマーと日時を偽物にする。 `vi.advanceTimersByTime(ms)` ・ `vi.runAllTimers()` ・ `vi.setSystemTime(date)` で進める。v5 は `Temporal` も対象 |
| `vi.stubEnv` ・ `vi.stubGlobal` | 環境変数・グローバル変数を差し替える。 `vi.unstubAllEnvs()` などで戻す |

```
import { vi, test, expect } from 'vitest'
import { getUser } from './api'
import { greet } from './greet'

vi.mock('./api', async (importOriginal) => ({
  ...(await importOriginal<typeof import('./api')>()),
  getUser: vi.fn(),               // getUser だけ差し替え
}))

test('名前で挨拶する', async () => {
  vi.mocked(getUser).mockResolvedValue({ name: 'taka' })
  expect(await greet(1)).toBe('こんにちは taka')
  expect(getUser).toHaveBeenCalledWith(1)
})
```

### 後片付けの 3 段階

| メソッド | 呼び出し履歴 | 実装 | 自動でやる設定（既定） |
| --- | --- | --- | --- |
| `mockClear()` / `vi.clearAllMocks()` | 消す | そのまま | `clearMocks` （v5 から true ） |
| `mockReset()` / `vi.resetAllMocks()` | 消す | 作ったときの実装に戻す（ `vi.fn()` なら `undefined` を返す） | `mockReset` （false） |
| `mockRestore()` / `vi.restoreAllMocks()` | `mockRestore` は消す。 `vi.restoreAllMocks` は消さない | `vi.spyOn` したものを本物に戻す | `restoreMocks` （false） |

## CLI

コマンド

絞り込み

CI・レポート

| コマンド | 意味 |
| --- | --- |
| `vitest` | 端末ならウォッチ、CI・非 TTY なら 1 回実行（ `vitest watch` ・ `vitest dev` も同じ） |
| `vitest run` | ウォッチせず 1 回だけ |
| `vitest related src/a.ts` | そのソースを（静的 import で）使うテストだけ。lint-staged では `--run` を付ける |
| `vitest list` | 対象のテスト名を一覧。v5 は実行せず静的に解析（ `--filesOnly` ・ `--json` ） |
| `vitest doctor` | pool・isolate などを変えて実測し、速くなる設定を勧める（v5 で追加） |
| `vitest bench` | ベンチマーク（ `bench()` ）だけ実行 |
| `vitest init browser` | ブラウザモードの設定を作る |
| `vitest --ui` | ブラウザの UI。v5 は表示された `?token=…` 付きの URL で開く |

```
vitest user                       # パスに "user" を含むファイル（正規表現・glob ではない）
vitest src/user.test.ts:10        # 10 行目のテスト（フルのファイル名が要る）
vitest -t 'login'                 # テスト名で絞る
vitest -t 'auth > login'          # v5: describe とテスト名は ' > ' でつないだ名前に一致
vitest --project unit             # projects の名前で絞る（--project='!e2e' で除外）
vitest run --changed              # git で変わったファイルに関係するテスト
vitest run --last-failed          # 前回失敗したテストだけ（v5.1）
vitest run --since-last-run       # 前回から依存が変わったテストだけ（v5.1・実験的）
```

```
vitest run --coverage                          # カバレッジ（既定の provider は v8）
vitest run --reporter=junit --outputFile=junit.xml
vitest run --bail 1                            # 1 件失敗したら止める
vitest run --repeats 20                        # 結果に関係なく各テストを繰り返す（不安定なテスト探し）
vitest run --passWithNoTests                   # テストが無くても成功

# シャード: 分けて実行 → blob を集めてまとめる（既定の置き場所は .vitest/blob/）
vitest run --shard=1/3 --reporter=blob
vitest --merge-reports --reporter=junit
```

（元のページで操作できます → https://cs.taka1156.site/vitest.html#cli）

## 設定

```
// vitest.config.ts（vite.config.ts の test に書いてもよい）
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    include: ['src/**/*.test.ts'],
    environment: 'node',        // DOM が要るなら 'jsdom' か 'happy-dom'（別途インストール）
    globals: false,             // true で describe などを import なしで使える
    setupFiles: ['./test/setup.ts'],
    // clearMocks: false,       // v4 と同じにしたいとき
    coverage: {
      provider: 'v8',           // または 'istanbul'（@vitest/coverage-* を入れる）
      include: ['src/**'],
      thresholds: { lines: 80 },
    },
    projects: [                 // 種類ごとに設定を分ける
      { test: { name: 'unit', include: ['**/*.unit.test.ts'] } },
      { test: { name: 'dom', environment: 'happy-dom', include: ['**/*.dom.test.ts'] } },
    ],
  },
})
```

v5 では、インラインで書いた project はルートの設定（plugins・ `resolve.alias` ・ `setupFiles` など）を既定で引き継ぐ（ `extends: true` ）。配列は上書きではなく足される。引き継がないなら `extends: false` 。

### Vite+ で使うとき

**Vitest 単体** `vitest` はウォッチ、 `vitest run` で 1 回。import は `'vitest'` 。設定は `vitest.config.ts` でもよい

**Vite+** `vp test` は 1 回、 `vp test watch` でウォッチ。import は `'vite-plus/test'` 。設定は `vite.config.ts` の `test`

## v4 → v5 で落ちやすい所

| 変更 | 症状 | 直し方 |
| --- | --- | --- |
| `clearMocks` が既定で true | トップレベルや `beforeAll` で記録した呼び出しが回数に入らなくなる | 各テストの中で呼ぶ。一時しのぎは `clearMocks: false` |
| `vi.mock` ・ `vi.hoisted` はトップレベルだけ | `describe` の中などに書くとエラー（v4 は警告） | ファイルの一番外に移す |
| 非同期アサーションの await 忘れ | テストが失敗する | `await expect(…).resolves` |
| `-t` の名前のつなぎ方 | `-t 'math adds'` が一致しない | `-t 'math > adds'` か 1 区切りだけで書く |
| `.sequential` の削除 | `test.sequential` が無い | `{ concurrent: false }` |
| 設定ファイルを親ディレクトリで探さない | サブディレクトリで実行すると設定が読まれない | `--config` で渡し、 `--dir` で範囲を決める |
| 出力先が `.vitest/` に集約 | CI のアーティファクトのパスが空 | パスを `.vitest/` に。 `.gitignore` も 1 行で済む |

## よくある誤解

✗ 誤解: vitest コマンドはいつもウォッチモードで止まらない

✓ 正しくは: CI や非 TTY（パイプ・lint-staged など）では自動で 1 回実行になる。ただし lint-staged から `vitest related` を呼ぶときは、公式も `--run` を付けるよう勧めている。

✗ 誤解: describe の中に vi.mock を書けば、そのブロックだけ差し替わる

✓ 正しくは: `vi.mock` はファイルの先頭に巻き上げられるので、ファイル全体に効く。v5 ではトップレベル以外に書くとエラーになる。テストごとに変えたいなら、モックした関数の振る舞いをテストの中で `mockReturnValue` などで決める。

✗ 誤解: v5 で clearMocks が既定になったので、モックの実装も毎回リセットされる

✓ 正しくは: 消えるのは呼び出しの履歴だけで、 `mockReturnValue` などで決めた実装は残る。実装も戻すなら `mockReset: true` 、 `vi.spyOn` を本物に戻すなら `restoreMocks: true` 。

## 関連メモ

- [Vite+ と vp コマンド チートシート](https://cs.taka1156.site/md/vite-plus.md) — Vitest を `vp test` として同梱する統合ツールチェーン
- [tsconfig 設定例](https://cs.taka1156.site/md/tsconfig.md) — `globals: true` のときに足す `types` の書き場所

## 参考

- [Vitest Guide](https://vitest.dev/guide/)
- [Vitest: Command Line Interface](https://vitest.dev/guide/cli)
- [Vitest: vi API](https://vitest.dev/api/vi)
- [Vitest: Migrating to Vitest 5.0](https://vitest.dev/guide/migration)
- [Vitest 5.0 is out!](https://vitest.dev/blog/vitest-5) ・ [Vitest 5.1 is out!](https://vitest.dev/blog/vitest-5-1)
- [Vite+ 設定（test）](https://viteplus.dev/config/)

確認日: 2026-11-23（Vitest 5.1.1）。
