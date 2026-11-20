<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Storybook 使い方

> Storybook 10 のストーリーの書き方・設定・CLI・play テスト・モックをまとめた早見表

- URL: https://cs.taka1156.site/storybook.html
- タグ: 開発ツール
- 日付: 2026-11-20

---

UI コンポーネントをアプリから切り離して表示・文書化・テストする開発環境（Storybook 10 前提）

**ざっくり言うと**

- コンポーネントの「状態」ごとに 1 つずつストーリー (story) を書き、ブラウザ上のカタログとして並べるツール
- `*.stories.tsx` の default export がコンポーネントの設定、named export が各ストーリーになる
- Vite 系なら `addon-vitest` でストーリーをそのままテストにでき、 `play` で操作も検証できる

## 全体像

設定は `.storybook/` フォルダの 2 ファイルと、コンポーネントの隣に置くストーリーファイルの 3 つで成り立つ。

**.storybook/main.ts** どのファイルを読むか・フレームワーク・アドオン（Node 側で動く）

**.storybook/preview.ts** 全ストーリー共通の decorators・parameters・モック（ブラウザ側で動く）

**Button.stories.tsx** コンポーネントごとの meta と、状態ごとのストーリー

同じストーリーが次の 3 か所で使い回される:

**Canvas** サイドバーで選んで表示。Controls で args を変えて試す

**Docs** `autodocs` タグで props 表付きのページを自動生成

**テスト** Vitest がストーリーを描画し、 `play` の検証を流す

## セットアップと CLI

Storybook 10 は ESM のみで配布され、Node.js 20.16+ / 22.19+ / 24+ が必要。 `npm run storybook -- -p 9009` のように、npm 経由でオプションを渡すときは `--` を挟む。

| やりたいこと | コマンド | 補足 |
| --- | --- | --- |
| 導入する | `npm create storybook@latest` | フレームワークを自動判別し、 `.storybook/` ・サンプル・npm スクリプトを作る |
| 開発サーバー | `storybook dev -p 6006` | `--no-open` （ブラウザを開かない）、 `--ci` （対話なし）、 `--docs` （Docs だけのモード） |
| 静的ビルド | `storybook build` | 出力は `storybook-static/` 。 `-o dir` で変更。 `--test` はテスト用に UI 機能を省いた軽いビルド |
| アドオン追加 | `npx storybook add @storybook/addon-a11y` | インストールと `main.ts` への登録を同時に行う。外すときは `remove` |
| バージョンを上げる | `npx storybook@latest upgrade` | 関連パッケージをまとめて更新し、自動移行 (automigrate) を流す |
| 移行だけ流す | `npx storybook automigrate` | 例: `automigrate csf-factories` で CSF Next へ書き換え |
| 環境を診断 | `npx storybook doctor` | 版の食い違い・重複パッケージを探す |

## ストーリーの書き方

CSF (Component Story Format) で書く。今の標準は CSF 3。型安全な CSF Next（CSF Factories）は Preview 段階で、Storybook 11 で標準になる予定。

CSF 3

CSF Next (Preview)

```
// Button.stories.tsx
// '@storybook/react-vite' は使うフレームワークに合わせる（nextjs-vite・vue3-vite など）
import type { Meta, StoryObj } from '@storybook/react-vite';
import { fn } from 'storybook/test';
import { Button } from './Button';

const meta = {
  title: 'Components/Button',   // 省略するとファイルの場所から自動で決まる
  component: Button,
  tags: ['autodocs'],           // Docs ページを自動生成
  args: { onClick: fn() },      // 全ストーリー共通の props。fn() は呼ばれると Actions に出る
  argTypes: {
    variant: { control: 'select', options: ['primary', 'secondary'] },
  },
  parameters: { layout: 'centered' },
} satisfies Meta<typeof Button>;
export default meta;

type Story = StoryObj<typeof meta>;

// named export 1 つ = ストーリー 1 つ（UpperCamelCase）
export const Primary: Story = {
  args: { variant: 'primary', label: 'Button' },
};

export const Disabled: Story = {
  args: { ...Primary.args, disabled: true },   // 別ストーリーの args を流用
};
```

`const meta: Meta = …` ではなく `satisfies` を使うと、meta で渡した必須 props をストーリー側で省いても型エラーにならない。

```
// .storybook/main.ts
import { defineMain } from '@storybook/react-vite/node';
export default defineMain({
  framework: '@storybook/react-vite',
  stories: ['../src/**/*.stories.@(ts|tsx)'],
  addons: ['@storybook/addon-a11y'],
});

// .storybook/preview.ts
import { definePreview } from '@storybook/react-vite';
import addonA11y from '@storybook/addon-a11y';
export default definePreview({
  addons: [addonA11y()],
  parameters: { layout: 'centered' },   // ここも型が効く
});

// Button.stories.tsx
import preview from '#.storybook/preview';   // package.json の imports で設定
import { Button } from './Button';

const meta = preview.meta({ component: Button });
export default meta;

export const Primary = meta.story({
  args: { variant: 'primary', label: 'Button' },
});
```

型注釈が要らなくなり、 `Primary` をそのまま Vitest などから import して再利用できる。対応は React・Vue・Angular・Web Components（10.3 時点）。既存のファイルは `npx storybook automigrate csf-factories` で書き換えられる。

（元のページで操作できます → https://cs.taka1156.site/storybook.html#story）

### meta とストーリーに書けるキー

| キー | 置き場所 | 役割 |
| --- | --- | --- |
| `component` | meta | 対象コンポーネント。props 表や Controls の型推論に使う |
| `title` | meta | サイドバーの階層（ `/` 区切り）。静的に読める文字列で書く |
| `name` | story | サイドバーの表示名だけを変える（export 名はそのまま） |
| `args` | 両方・preview | コンポーネントに渡す props。Controls で画面から変えられる |
| `argTypes` | 両方・preview | Controls の種類や説明。 `control: 'text' \| 'boolean' \| 'number' \| 'range' \| 'color' \| 'date' \| 'select' \| 'radio' \| 'object'` など。 `table: { disable: true }` で非表示 |
| `parameters` | 両方・preview | アドオン向けの静的な設定（ `layout` ・ `backgrounds` ・ `a11y` など） |
| `decorators` | 両方・preview | ストーリーを包む関数。Provider・テーマ・余白などを足す |
| `tags` | 両方・preview | Docs 生成やテスト対象の切り替え（下の表） |
| `render` | 両方 | args の渡し方を変える描画関数。複数コンポーネントを組み合わせるとき |
| `play` | story・meta | 描画後に動く操作と検証（下の「操作のテスト」） |
| `beforeEach` | 両方・preview | 描画前の準備。戻り値の関数が後片付けになる |
| `loaders` | 両方・preview | 描画前に非同期でデータを取り、 `render` の第 2 引数の `loaded` に渡す |

### args・parameters・decorators の効き方

`args` と `parameters` は下の段ほど優先して上書きされる。 `parameters` はオブジェクトを深くマージするが、 `args` はキー単位の浅いマージで、オブジェクトの arg は丸ごと置き換わる。 `decorators` は上書きではなく重なり、ストーリーのものがいちばん内側、preview のものがいちばん外側に来る。

弱い（先に適用）

**preview.ts** 全ストーリー共通（グローバル）

**meta（default export）** そのファイルのストーリー全部

**story（named export）** そのストーリーだけ

**Controls・URL の args** 画面で変えた値（args のみ。保存はされない）

強い（最後に勝つ）

### よく使う tags と parameters

| 書き方 | 効果 |
| --- | --- |
| `tags: ['autodocs']` | そのコンポーネントの Docs ページを自動生成（ `@storybook/addon-docs` が必要）。preview に書けば全部に付く |
| `tags: ['!autodocs']` | preview で付けた `autodocs` を、そのファイル・ストーリーだけ外す。 `!` は打ち消し |
| `tags: ['!dev']` | サイドバーに出さない（Docs やテストには残る） |
| `tags: ['!test']` | テスト対象から外す（標準では全ストーリーに `test` が付いている） |
| `parameters.layout` | `'padded'` （既定）・ `'centered'` ・ `'fullscreen'` |
| `parameters.docs.description.component` | Docs ページの説明文。JSDoc のコメントからも自動で拾う |

## 設定ファイルの例

main.ts

preview.ts

```
// .storybook/main.ts
import type { StorybookConfig } from '@storybook/react-vite';

const config: StorybookConfig = {
  framework: '@storybook/react-vite',        // 必須
  stories: [                                  // 必須。main.ts からの相対 glob
    '../src/**/*.mdx',
    '../src/**/*.stories.@(js|jsx|mjs|ts|tsx)',
  ],
  addons: [
    '@storybook/addon-docs',
    '@storybook/addon-a11y',
    '@storybook/addon-vitest',
  ],
  staticDirs: ['../public'],                  // 画像などをそのまま配信
};
export default config;
```

Controls・Actions・Viewport・Backgrounds・Toolbars などは Storybook 9 から本体に入った。 `@storybook/addon-essentials` はもう入れない（Docs だけは `addon-docs` を別に入れる）。Vite の設定は `viteFinal` 、webpack は `webpackFinal` で足す。

```
// .storybook/preview.tsx
import type { Preview } from '@storybook/react-vite';
import { sb } from 'storybook/test';
import '../src/index.css';                      // グローバル CSS

sb.mock(import('../src/lib/session.ts'));      // モジュールのモック（下で説明）

const preview: Preview = {
  parameters: {
    controls: { matchers: { color: /(background|color)$/i, date: /Date$/i } },
  },
  decorators: [
    (Story) => (
      <ThemeProvider theme={theme}>
        <Story />
      </ThemeProvider>
    ),
  ],
  tags: ['autodocs'],
};
export default preview;
```

（元のページで操作できます → https://cs.taka1156.site/storybook.html#config）

## 操作のテスト（play 関数）

ストーリーを描画したあとに `play` が動き、Testing Library と Vitest の API で操作・検証する。結果はアドオンパネルの Interactions に 1 手ずつ出て、巻き戻して確かめられる。import 元は `storybook/test` （Storybook 9 で `@storybook/test` から変わった）。

```
import { expect, fn } from 'storybook/test';

const meta = {
  component: LoginForm,
  args: { onSubmit: fn() },     // 呼ばれたかを検証できるスパイ
} satisfies Meta<typeof LoginForm>;
export default meta;
type Story = StoryObj<typeof meta>;

export const Submitted: Story = {
  play: async ({ canvas, userEvent, args, step }) => {
    await step('入力する', async () => {
      await userEvent.type(canvas.getByLabelText('Email'), 'a@example.com');
      await userEvent.type(canvas.getByLabelText('Password'), 'secret');
    });
    await userEvent.click(canvas.getByRole('button', { name: 'Log in' }));
    await expect(args.onSubmit).toHaveBeenCalled();          // expect は await する
    await expect(await canvas.findByText('Welcome back')).toBeInTheDocument();
  },
};
```

### 要素の探し方（canvas のクエリ）

| 形 | 見つからない | 2 つ以上 | 使いどころ |
| --- | --- | --- | --- |
| `getBy…` | 例外 | 例外 | すでに表示されているはずの要素 |
| `queryBy…` | `null` | 例外 | 「無いこと」を確かめる |
| `findBy…` | 例外（待ってから） | 例外 | 非同期で出てくる要素。 `await` する |
| `getAllBy…` など | 例外（ `queryAll` は `[]` ） | 配列 | 複数ある要素 |

… には `Role` （推奨）・ `LabelText` ・ `Text` ・ `PlaceholderText` ・ `TestId` などが入る。描画前に日付などを固定したいときは、 `play` の引数から `mount` を受け取り、準備のあとで `await mount()` を呼ぶ。

## ストーリーを Vitest で流す

`@storybook/addon-vitest` は、ストーリーを Vitest のテストに変換し、ブラウザモード（Playwright の Chromium）で描画する。 `play` が無いストーリーも「描画でエラーが出ないか」のテストになる。Vitest そのものの使い方は [Vitest 使い方](https://cs.taka1156.site/md/vitest.md) を参照。

1. **アドオンを入れる** `npx storybook add @storybook/addon-vitest` 。Vite 系のフレームワーク（ `react-vite` ・ `vue3-vite` ・ `nextjs-vite` など）が前提で、Vitest が無ければ一緒に入る
2. **設定を確かめる** `vitest.config.ts` に `storybook` という名前のプロジェクトが足される（下の例）
3. **実行する** UI ではサイドバー下のテストウィジェットの Run、ターミナルでは `npm run test-storybook` （中身は `vitest --project=storybook` ）。CI では `vitest run --project=storybook`

```
// vitest.config.ts（Vitest 4 の例）
import { defineConfig } from 'vitest/config';
import { storybookTest } from '@storybook/addon-vitest/vitest-plugin';
import { playwright } from '@vitest/browser-playwright';

export default defineConfig({
  test: {
    projects: [{
      extends: true,
      plugins: [storybookTest({ configDir: '.storybook' })],
      test: {
        name: 'storybook',
        browser: {
          enabled: true,
          provider: playwright(),        // Vitest 3 では provider: 'playwright'
          headless: true,
          instances: [{ browser: 'chromium' }],
        },
      },
    }],
  },
});
```

`storybookTest({ tags: { include, exclude, skip } })` で対象を絞れる（既定は `include: ['test']` ）。webpack 系のフレームワークでは代わりに `@storybook/test-runner` を使う。

## モジュールのモック（sb.mock）

Storybook 9.1 から、コンポーネントが import する関数を `preview.ts` で差し替えられる（10 でも同じ）。Vite と webpack の両方で、開発サーバーにもビルドにも効く。

| 書き方 | 動き |
| --- | --- |
| `sb.mock(import('../lib/session.ts'), { spy: true })` | 本物の処理はそのまま動かし、呼び出しだけ記録する |
| `sb.mock(import('../lib/session.ts'))` | すべての export を何もしないモック関数に置き換える |
| `sb.mock(import('uuid'))` | `node_modules` のパッケージも同じように置き換えられる |
| `lib/__mocks__/session.js` を置く | そのファイルの実装がモックとして使われる（凝った振る舞いを使い回すとき）。パッケージならプロジェクト直下の `__mocks__/` 。中身は TypeScript ではなく ES モジュールの JS で書く |

```
// 個々のストーリーで戻り値を決める
import { mocked } from 'storybook/test';
import { getUser } from '../lib/session';

export const LoggedIn: Story = {
  beforeEach: () => { mocked(getUser).mockReturnValue({ name: 'Alice' }); },
};
```

登録できるのは `preview.ts` だけ。パスは preview からの相対で拡張子まで書き、 `@/` のような別名は使えない。API 通信ごと差し替えたいときは `msw-storybook-addon` （MSW）が向く。

## よくある誤解

✗ 誤解: ストーリーファイルは本番のバンドルにも入ってしまう

✓ 正しくは: ストーリーは開発用で、アプリのビルドには含まれない。Storybook 自体は `storybook build` で別のサイトとして書き出す。

✗ 誤解: Storybook 9 以降も addon-essentials と addon-interactions を入れる必要がある

✓ 正しくは: Controls・Actions・Interactions などは本体に統合された。古い記事のとおりに入れると版が合わずにエラーになることがある。Docs だけは `@storybook/addon-docs` を入れる。

✗ 誤解: テストは play 関数を書いたストーリーだけが対象になる

✓ 正しくは: addon-vitest では、 `test` タグの付いたストーリー（標準では全部）が描画のスモークテストになる。外したいストーリーには `tags: ['!test']` を付ける。

✗ 誤解: Controls で変えた値はストーリーに保存される

✓ 正しくは: 画面で変えた args は URL に一時的に載るだけで、ファイルは変わらない。残したい状態は新しいストーリーとして書く（Storybook の UI から保存する機能もあるが、それもファイルに書き出す形）。

## 出典と確認日

- [How to write stories](https://storybook.js.org/docs/writing-stories)
- [Writing stories in TypeScript](https://storybook.js.org/docs/writing-stories/typescript)
- [CSF Factories](https://storybook.js.org/docs/api/csf/csf-factories)
- [Configure Storybook](https://storybook.js.org/docs/configure)
- [CLI options](https://storybook.js.org/docs/api/cli-options)
- [Interaction tests](https://storybook.js.org/docs/writing-tests/interaction-testing)
- [Vitest addon](https://storybook.js.org/docs/writing-tests/integrations/vitest-addon)
- [Mocking modules](https://storybook.js.org/docs/writing-stories/mocking-data-and-modules/mocking-modules)
- [Storybook 10 リリース記事](https://storybook.js.org/blog/storybook-10/)

確認日: 2026-11-20。公式ドキュメントの最新は 10.6、Storybook 11 は beta。11 では CSF Next が標準になる予定なので、上げるときは移行ガイドを確かめる。
