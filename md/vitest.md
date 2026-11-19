<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Vitest 使い方

> Vite と設定を共有するテストランナー Vitest の基本的な書き方と実行方法

- URL: https://cs.taka1156.site/vitest.html
- タグ: 開発ツール
- 日付: 2026-11-16

---

Vite と設定を共有するテストランナー。Jest に近い書き方で、Vite+ には組み込み済み

**ざっくり言うと**

- テストは `describe` ・ `it` ・ `expect` で書き、Jest とほぼ同じ書き方
- Vite+ では `vp test` で実行でき、別途インストールしなくてよい
- 対象ファイルは `vite.config.ts` の `test.include` で決める

## テストの書き方

```
// src/sum.ts
export function sum(a: number, b: number): number {
  return a + b;
}

// src/sum.test.ts
import { describe, it, expect } from 'vitest';
import { sum } from './sum';

describe('sum', () => {
  it('2 つの数を足す', () => {
    expect(sum(1, 2)).toBe(3);
  });

  it.each([
    [0, 0, 0],
    [-1, 1, 0],
  ])('sum(%i, %i) は %i', (a, b, expected) => {
    expect(sum(a, b)).toBe(expected);
  });
});
```

## 実行コマンドと設定

| やりたいこと | コマンド・設定 |
| --- | --- |
| テストを実行 | `vp test` |
| 対象ファイル | `vite.config.ts` の `test.include` |
| 実行環境 | `test.environment` （例: `'node'` 、DOM は `'jsdom'` など） |

```
test: {
  include: ['src/**/*.test.ts'],
  environment: 'node',
},
```

## 出典と確認日

- [Vitest 公式](https://vitest.dev/)
- [Vitest: 設定](https://vitest.dev/config/)
- [Vite+ 設定（test）](https://viteplus.dev/config/)

確認日: 2026-11-16。Vite+ は 2026年9月に 1.0 の安定版となった。機能は発展途上のため、Vitest を含め版ごとに公式で確かめる。
