<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Playwright 使い方

> Playwright Test の CLI・codegen・ロケーター・アサーション・設定・認証・モックを用途別にまとめたチートシート。

- URL: https://cs.taka1156.site/playwright.html
- タグ: 開発ツール
- 日付: 2026-11-24

---

Chromium・Firefox・WebKit を同じ API で動かす E2E テストフレームワーク Playwright Test のチートシート（v1.65 系・2026-11 時点）

**ざっくり言うと**

- Microsoft 製の E2E テストフレームワークで、 `@playwright/test` がテストランナーとブラウザ操作をまとめて提供する。
- 操作は要素が押せる状態になるまで自動で待ち、 `await expect(locator)` は条件を満たすまで再試行する。
- 要素は `getByRole` など見た目に近いロケーターで探し、固定の待ち時間や CSS の細かい指定に頼らない。

## 使う流れ

**導入** `npm init playwright@latest` で設定・サンプル・ブラウザを入れる

**書く** `tests/*.spec.ts` に書く。 `codegen` で操作を記録して下書きにしてもよい

**実行** `npx playwright test` 。既定はヘッドレス・並列

**調べる** HTML レポート・トレース・UI モードで失敗した操作を 1 つずつ見る

要件（v1.65）: Node.js 22・24・26 の最新版。Node.js 20 は v1.65 からテストされていない。OS は Windows 11 以上・macOS 14 以上・Debian 12/13・Ubuntu 22.04/24.04/26.04。

## 導入

1. **プロジェクトに入れる** `npm init playwright@latest` 。TypeScript か JavaScript か、テストの置き場所、GitHub Actions のワークフローを作るか、ブラウザを入れるかを聞かれる
2. **できるもの** `playwright.config.ts` ・ `tests/example.spec.ts` （と選べば `.github/workflows/playwright.yml` ）。 `package.json` の devDependencies に `@playwright/test` が入る
3. **ブラウザを入れる** パッケージだけではブラウザは入らない。clone した直後や CI では `npx playwright install --with-deps` を実行する
4. **上げる** `npm install -D @playwright/test@latest` のあと、もう一度 `npx playwright install --with-deps` （版ごとに対応するブラウザの版が違う）

## CLI

実行・絞り込み

デバッグ

CI・スナップショット

導入・管理

| コマンド | やること |
| --- | --- |
| `npx playwright test` | 全テストを全プロジェクト（ブラウザ）で実行 |
| `npx playwright test tests/login.spec.ts` | ファイルを指定。引数はファイルパスに対する正規表現なので `login` だけでもよい |
| `npx playwright test login.spec.ts:42` | その行のテストだけ |
| `--project=chromium` | プロジェクトを指定（複数・ `*` 可） |
| `-g "@smoke"` / `-G "@slow"` | タイトル（タグを含む）が一致するものだけ／一致しないものだけ |
| `--headed` | ブラウザの画面を出して実行 |
| `-j 4` （ `--workers` ） | 並列数。 `50%` のように CPU コア数の割合でも書ける（既定 50%） |
| `--last-failed` | 前回失敗したテストだけ |
| `--only-changed=main` | `main` から変わったテストファイルだけ（値なしなら未コミットの変更） |
| `--repeat-each 10` | 各テストを 10 回。不安定なテスト探しに |
| `--shuffle [seed]` | 順番をランダムに（v1.64〜）。シードが表示されるので再現できる |
| `-x` （ `--max-failures=N` ） | 最初の失敗（N 件）で止める |
| `--list` | 実行せずにテストの一覧だけ出す |

| コマンド | やること |
| --- | --- |
| `npx playwright test --ui` | UI モード。ウォッチ・ステップごとの画面・DOM を時間をさかのぼって見られる |
| `npx playwright test --debug` | Inspector で 1 ステップずつ実行。 `--headed --workers=1 --timeout=0` などが付く |
| `--trace on` | 設定に関係なくトレースを取る |
| `npx playwright show-report` | 前回の HTML レポートを開く（失敗があるときは自動で開く） |
| `npx playwright show-trace trace.zip` | トレースビューアーで開く。 [trace.playwright.dev](https://trace.playwright.dev) にドロップしてもよい |
| `npx playwright codegen localhost:3000` | ブラウザを操作してテストコードを記録。詳しくは 下の codegen の節 |
| `await page.pause()` | テストの途中に書くと、そこで止まって Inspector が開く（headed 時） |

| コマンド | やること |
| --- | --- |
| `--forbid-only` | `test.only` が残っていたら失敗にする |
| `--retries=2` | 失敗したテストを最大 2 回やり直す。やり直しで通ったものは flaky と表示 |
| `--fail-on-flaky-tests` | flaky が 1 件でもあれば失敗にする |
| `--shard=1/4` | 4 台に分けたうちの 1 番目だけ実行。reporter を `blob` にして集める |
| `npx playwright merge-reports --reporter html ./all-blob-reports` | シャードごとの blob レポートを 1 つにまとめる |
| `--pass-with-no-tests` | テストが 0 件でも成功にする（v1.65 から、シャード実行で全シャードを通じて該当が 0 件だと失敗になった。空のシャードがあるだけなら成功） |
| `--reporter=failures` | 失敗とやり直しで通ったものだけを出す（v1.65〜） |
| `--add-reporter=github` | 設定の reporter を残したまま追加（v1.63〜） |
| `-u` | スクリーンショット・aria スナップショットを更新。値なしは `changed` （変わったものだけ） |
| `--update-snapshots=all` | 全部書き直す。 `missing` は無いものだけ書いて成功、 `none` は書かない |
| `--ignore-snapshots` | スナップショットのアサーションを飛ばす |

| コマンド | やること |
| --- | --- |
| `npm init playwright@latest` | 新規導入（既存プロジェクトにも追加できる） |
| `npx playwright install` | 対応する全ブラウザを入れる。 `chromium` のように 1 つだけも可 |
| `npx playwright install --with-deps` | OS の依存パッケージも一緒に入れる（CI・Linux 向け） |
| `npx playwright install --only-shell` | ヘッドレス専用の `chromium-headless-shell` だけ入れる（CI を軽く） |
| `npx playwright install-deps` | OS の依存パッケージだけ入れる |
| `npx playwright --version` | 入っている版を確かめる |

（元のページで操作できます → https://cs.taka1156.site/playwright.html#cli）

## 操作を記録してコードを作る（codegen）

ブラウザを実際に操作すると、その操作がテストコードとして書き出される。ロケーターは役割・文字・test id を優先して選ばれ、複数の要素に当てはまるときは 1 つに絞れる形に直される。

1. **起動する** `npx playwright codegen localhost:3000` 。操作するブラウザと、コードが出る Inspector の 2 つの窓が開く
2. **操作する** クリック・入力・移動などがそのままコードになる
3. **確認を足す** ツールバーのアイコンを選んで要素をクリックすると、見えているか（assert visibility）・文字（assert text）・値（assert value）のアサーションが入る。aria スナップショットも作れる
4. **持ち出す** 録画ボタンで止めてコピーし、テストファイルに貼る。 `-o tests/new.spec.ts` を付けるとファイルに書く
5. **手で整える** 出てくるのは下書き。 `test.step` でまとめる、値を変数にする、確認を足すなどしてから使う

| オプション | やること |
| --- | --- |
| `-o, --output <file>` | 生成したコードをファイルに保存 |
| `--target <lang>` | 出力の形式。 `playwright-test` （既定）・ `javascript` ・ `python-pytest` ・ `java` ・ `csharp` など |
| `-b, --browser <name>` | `chromium` （既定）・ `firefox` ・ `webkit` |
| `--device="iPhone 13"` | 端末をまねて記録（画面サイズ・UA など） |
| `--viewport-size="800,600"` | 画面サイズを指定 |
| `--color-scheme=dark` | ダークモードで記録 |
| `--lang` / `--timezone` / `--geolocation` | 言語・タイムゾーン・位置情報をまねる |
| `--save-storage=auth.json` | 終わったときの Cookie・localStorage・IndexedDB を保存（ログインだけ先に記録） |
| `--load-storage=auth.json` | 保存した状態を読み込み、ログイン済みから記録 |
| `--http-credentials="user:pass"` | HTTP 認証（Basic 認証など）の奥のページを記録（v1.63〜） |
| `--test-id-attribute=data-pw` | `getByTestId` に使う属性名 |

### ほかの記録のしかた

- **VS Code 拡張** : テストのサイドバーの **Record new** で新しいファイルに記録、 **Record at cursor** で今のテストのカーソル位置から続きを記録
- **ロケーターだけ欲しい** : Inspector の **Pick Locator** （VS Code では Pick locator）で要素をクリックすると、その要素のロケーターが出る
- **途中から記録** : テストやスクリプトで `await page.pause()` を呼ぶと、そこで止まって記録用の窓が開く。 `route` などで準備した状態から記録できる（headed で動かす）

AI エージェントにテストを書かせたいときは、v1.56 から入った Test Agents（planner・generator・healer）がある。 `npx playwright init-agents --loop=claude` のように使う AI ツールを指定して定義を入れる。

## テストの書き方

```
import { test, expect } from '@playwright/test';

test.describe('ログイン', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/login');            // baseURL からの相対パス
  });

  test('正しいパスワードで入れる', { tag: '@smoke' }, async ({ page }) => {
    await page.getByLabel('メールアドレス').fill('a@example.com');
    await page.getByLabel('パスワード').fill(process.env.TEST_PASSWORD!);
    await page.getByRole('button', { name: 'ログイン' }).click();

    await expect(page).toHaveURL(/\/dashboard/);
    await expect(page.getByRole('heading', { name: 'ダッシュボード' })).toBeVisible();
  });
});
```

| 書き方 | 意味 |
| --- | --- |
| `test.only(...)` | このテストだけ実行（CI では `--forbid-only` で検出） |
| `test.skip(...)` / `test.fixme(...)` | 飛ばす／直すまで飛ばす。 `test.skip(browserName === 'webkit', '理由')` のように条件も書ける |
| `test.fail(...)` | 失敗するのが正しいテスト（通ると失敗になる） |
| `test.slow()` | タイムアウトを 3 倍にする |
| `await test.step('名前', async () => {...})` | レポート・トレースに段落として出る |
| `test.describe.configure({ mode: 'serial' })` | グループ内を順番に実行。1 つ失敗すると残りは飛ばす |
| `test.use({ locale: 'ja-JP' })` | そのファイル・グループだけ設定（ `use` ）を上書き |

## ロケーター（要素の探し方）

上から順に優先する。ユーザーが見るもの（役割・ラベル・文字）で探すと、CSS やマークアップが変わっても壊れにくい。

| メソッド | 探すもの・例 |
| --- | --- |
| `getByRole(role, { name })` | ARIA の役割と名前。 `getByRole('button', { name: '送信' })` ・ `getByRole('checkbox', { checked: true })` |
| `getByLabel(text)` | `<label>` ・ `aria-label` で結び付いた入力欄 |
| `getByPlaceholder(text)` | placeholder の文字 |
| `getByText(text)` | 表示された文字。既定は部分一致・大文字小文字無視。 `{ exact: true }` で完全一致、正規表現も可 |
| `getByAltText(text)` | 画像などの alt |
| `getByTitle(text)` | title 属性 |
| `getByTestId(id)` | 既定は `data-testid` 。属性名は設定の `use.testIdAttribute` で変える |
| `locator('css') / locator('xpath=…')` | 最後の手段。DOM の構造に依存するので壊れやすい |

### 絞り込み・組み合わせ

```
// 行（listitem）のうち「りんご」を含むものの中のボタン
page.getByRole('listitem').filter({ hasText: 'りんご' }).getByRole('button', { name: '削除' });

// 子に見出しを持つカード / 持たないカード
page.locator('.card').filter({ has: page.getByRole('heading', { name: '料金' }) });
page.locator('.card').filter({ hasNot: page.getByText('売り切れ') });

page.getByRole('button').filter({ visible: true });          // 見えているものだけ
page.getByRole('button').and(page.getByTitle('購読'));       // 両方に当てはまる
page.getByRole('button', { name: '新規' }).or(page.getByText('確認')); // どちらか
page.getByRole('listitem').nth(2);                           // 0 始まり。first() / last() も
page.frameLocator('#payment').getByLabel('カード番号');       // iframe の中
```

ロケーターは作った時点では探さず、操作やアサーションのたびに探し直す。条件に当てはまる要素が複数あるまま `click()` などをするとエラー（strict モード）になるので、絞り込むか `first()` などで 1 つに決める。

## 操作

| メソッド | やること |
| --- | --- |
| `page.goto(url)` | 移動。 `load` イベントまで待つ |
| `click()` / `dblclick()` | クリック。 `{ button: 'right' }` ・ `{ modifiers: ['Shift'] }` |
| `fill(value)` | 入力欄の中身を置き換える（ふつうはこれ） |
| `pressSequentially(text)` | 1 文字ずつキー入力。キー入力ごとの処理を確かめるときだけ |
| `press('Enter')` | キーを押す。 `'Control+A'` のような組み合わせも |
| `check()` / `uncheck()` / `setChecked(bool)` | チェックボックス・ラジオ |
| `selectOption('blue')` | `<select>` を value かラベルで選ぶ。配列で複数 |
| `setInputFiles('a.pdf')` | ファイル選択。 `[]` で解除 |
| `hover()` / `focus()` | ホバー・フォーカス |
| `dragTo(target)` | ドラッグ＆ドロップ |
| `page.waitForURL('**/done')` | URL が変わるまで待つ |

操作の前に、要素が 1 つに決まり・見えていて・動いておらず・ほかの要素に隠れておらず・有効（入力なら編集可）になるまで自動で待つ。v1.65 からは inert の間も待つ。

### 新しいタブ・ダイアログ・ダウンロード

```
// 起きるのを待つ Promise を、きっかけの操作より先に作る
const popupPromise = page.waitForEvent('popup');
await page.getByRole('link', { name: '別タブで開く' }).click();
const popup = await popupPromise;

page.once('dialog', (dialog) => dialog.accept());   // alert / confirm（何もしないと自動で閉じる）

const downloadPromise = page.waitForEvent('download');
await page.getByText('CSV をダウンロード').click();
const download = await downloadPromise;
await download.saveAs('./out/' + download.suggestedFilename());
```

## アサーション

`await expect(locator).…` の形（web-first）は、条件を満たすまで既定 5 秒再試行する。 `.not` で否定。

| アサーション | 確かめること |
| --- | --- |
| `toBeVisible()` / `toBeHidden()` | 見えている／見えていない（存在しないも含む） |
| `toHaveText(text)` | 文字が一致（文字列は完全一致・空白は正規化。正規表現も可） |
| `toContainText(text)` | 文字を含む |
| `toHaveValue(v)` / `toHaveValues([…])` | 入力欄・複数選択の値 |
| `toBeChecked()` / `toBeEnabled()` / `toBeDisabled()` | チェック・有効・無効 |
| `toHaveCount(n)` | 当てはまる要素の数 |
| `toHaveAttribute(name, v)` / `toHaveClass(…)` / `toContainClass(…)` | 属性・クラス |
| `toHaveAccessibleName(…)` / `toHaveRole(…)` | アクセシビリティ上の名前・役割 |
| `expect(page).toHaveURL(…)` / `toHaveTitle(…)` | ページの URL・タイトル |
| `toHaveScreenshot()` | 見た目の比較。初回は基準画像を書いて失敗する（以後は差分で判定） |
| `toMatchAriaSnapshot(`…`)` | アクセシビリティツリーを YAML 形式の見本と比べる |
| `expect(response).toBeOK()` | API レスポンスが 2xx |

```
await expect.soft(page.getByTestId('status')).toHaveText('完了'); // 失敗しても続ける（テストは失敗扱い）
await expect(locator).toBeVisible({ timeout: 10_000 });          // このアサーションだけ待ち時間を変える

// 画面以外の値を再試行で待つ
await expect.poll(async () => (await request.get('/api/jobs/1')).status()).toBe(200);

// ブロックごと通るまでやり直す（既定のタイムアウトは 0 = 無制限なので指定する）
await expect(async () => {
  const res = await page.request.get('/api/ready');
  expect(res.status()).toBe(200);
}).toPass({ timeout: 30_000 });
```

`toBe` ・ `toEqual` など `await` の付かないアサーションは 1 回しか確かめない。画面の状態に使うと不安定になる。

## 設定ファイルの例

```
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,                 // ファイル内のテストも並列に
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',  // page.goto('/login') が使える
    trace: 'on-first-retry',           // 1 回目のやり直しでトレースを取る
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    { name: 'setup', testMatch: /.*\.setup\.ts/ },
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'], storageState: 'playwright/.auth/user.json' },
      dependencies: ['setup'],
    },
    { name: 'mobile', use: { ...devices['Pixel 7'] } },
  ],
  webServer: {                         // テストの前にアプリを起動して URL が応答するまで待つ
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

| タイムアウト | 既定 | 変える場所 |
| --- | --- | --- |
| テスト 1 本（フック・フィクスチャ込み） | 30 秒 | `timeout` ・ `test.setTimeout()` |
| web-first アサーション | 5 秒 | `expect: { timeout }` |
| 操作（click など）・移動（goto など） | なし（テストのタイムアウトまで） | `use: { actionTimeout, navigationTimeout }` |
| 実行全体 | なし | `globalTimeout` |

## ログイン状態の使い回し

テストは 1 本ごとに新しいブラウザコンテキスト（Cookie・localStorage が空）で始まる。ログインを毎回 UI でやらないよう、setup プロジェクトで 1 回ログインして状態をファイルに保存し、ほかのプロジェクトが `storageState` で読み込む（設定は上の例）。

```
// tests/auth.setup.ts
import { test as setup, expect } from '@playwright/test';

const authFile = 'playwright/.auth/user.json';

setup('ログイン', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('メールアドレス').fill(process.env.TEST_USER!);
  await page.getByLabel('パスワード').fill(process.env.TEST_PASSWORD!);
  await page.getByRole('button', { name: 'ログイン' }).click();
  await page.waitForURL('/dashboard');      // リダイレクトで Cookie が揃うまで待つ
  await page.context().storageState({ path: authFile });
});
```

- `playwright/.auth` は `.gitignore` に入れる（Cookie やトークンが入っている）
- ログインしていない状態で始めたいテストは `test.use({ storageState: { cookies: [], origins: [] } })`
- テストがサーバー側の状態を書き換えるなら、ワーカーごとに別アカウントにする（ `test.info().parallelIndex` で選ぶ）

## フィクスチャ

テストの引数 `{ page }` などがフィクスチャ。使うものだけが用意され、テストごとに作り直される。組み込みは `page` ・ `context` ・ `browser` ・ `browserName` ・ `request` など。自分のものは `test.extend` で足す。

```
// fixtures.ts
import { test as base } from '@playwright/test';
import { TodoPage } from './todo-page';     // ページオブジェクト

export const test = base.extend<{ todoPage: TodoPage }>({
  todoPage: async ({ page }, use) => {
    const todoPage = new TodoPage(page);
    await todoPage.goto();                  // 準備
    await use(todoPage);                    // ここでテストが走る
    await todoPage.removeAll();             // 後片付け
  },
});
export { expect } from '@playwright/test';

// テスト側: import { test, expect } from './fixtures';
test('追加できる', async ({ todoPage }) => { /* ... */ });
```

ワーカーごとに 1 回だけ作るもの（DB 接続など）は `{ scope: 'worker' }` を付ける。全テストの前後に 1 回だけ動かしたい処理は、setup プロジェクトか設定の `globalSetup` に書く。

## ネットワークのモックと API テスト

```
// 返事を差し替える
await page.route('**/api/todos', (route) =>
  route.fulfill({ json: [{ id: 1, title: 'モック' }] }));

// 画像を読み込ませない
await page.route('**/*.{png,jpg,jpeg}', (route) => route.abort());

// 本物の返事を一部だけ書き換える
await page.route('**/api/user', async (route) => {
  const response = await route.fetch();
  const json = await response.json();
  await route.fulfill({ response, json: { ...json, plan: 'pro' } });
});

// 送ったリクエストの返事を待つ（Promise は操作の前に作る）
const saved = page.waitForResponse('**/api/save');
await page.getByRole('button', { name: '保存' }).click();
expect((await saved).status()).toBe(200);
```

```
// ブラウザを使わない API テスト（request フィクスチャ。baseURL が効く）
test('作成できる', async ({ request }) => {
  const res = await request.post('/api/items', { data: { name: 'りんご' } });
  await expect(res).toBeOK();
  expect(await res.json()).toMatchObject({ name: 'りんご' });
});
```

そのほか: `route.continue({ headers })` で書き換えて送る、 `route.fallback()` で次のハンドラーに回す、 `context.route()` でコンテキスト内の全ページに効かせる、 `routeFromHAR()` で記録した HAR から返す。

`page.route()` のモックは Playwright のテストの中だけで使える。同じモックを Vitest や開発中の画面でも使い回したいなら、 [MSW](https://cs.taka1156.site/md/msw.md) のハンドラーに書いて `@msw/playwright` で読み込む。

## 最近の版で変わったこと

| 版 | 変わったこと |
| --- | --- |
| 1.65（2026-11） | Node.js 22・24・26 が対象（20 はテスト対象外）。 `bunx --bun playwright test` で Bun でも動く。 `failures` reporter と、html・json・junit の `onlyFailures` 。click・fill などは inert が外れるまで待つ。シャード実行で全シャードを通じて該当が 0 件だと失敗（空のシャードは成功のまま）。Cookie の照合が RFC 6265 どおりに |
| 1.64（2026-10） | `--shuffle` 、 `locator.within()` 、 `testProject.default` （既定の実行から外すプロジェクト）。 `--update-snapshots=missing` は成功で終わるように。動画が VP8 から VP9 に |
| 1.63（2026-09） | テストのロック `{ lock: '名前' }` 。同じロック名を持つテストは、ファイル・ワーカー・プロジェクトをまたいでも同時に走らない。 `--add-reporter` 。実験的なコンポーネントテスト（ `@playwright/experimental-ct-react` など）は更新終了 |
| 1.62（2026-07） | MCP と CLI を同梱（ `npx playwright mcp` ・ `npx playwright cli` ）。 `retryStrategy: 'isolated'` でやり直しを最後にまとめて実行 |

## よくある誤解

✗ 誤解: waitForTimeout で数秒待たせればテストは安定する

✓ 正しくは: 固定の待ち時間は遅い環境では足りず、速い環境では無駄になる。操作は自動で待ち、 `await expect(locator)` は条件を満たすまで再試行するので、待ちたい状態をアサーションで書く。公式も `page.waitForTimeout()` は本番のテストで使わないよう書いている。

✗ 誤解: expect(await locator.textContent()).toBe() でも toHaveText() と同じ

✓ 正しくは: `textContent()` はその瞬間の値を 1 回取るだけで、再試行しない。表示が少し遅れると失敗する。画面の状態は `await expect(locator).toHaveText()` のような web-first アサーションで確かめる。

✗ 誤解: 前のテストでログインしていれば次のテストもログイン済みで始まる

✓ 正しくは: テストは 1 本ごとに新しいブラウザコンテキストで始まり、Cookie も localStorage も引き継がない。並び順に頼らず、setup プロジェクトで保存した `storageState` を読み込む。

✗ 誤解: npm install で @playwright/test を入れればブラウザも入る

✓ 正しくは: パッケージにはブラウザが入っていない。 `npx playwright install` （CI では `--with-deps` ）でブラウザを入れる。Playwright を上げたときも、対応するブラウザの版が変わるのでもう一度実行する。

## 関連メモ

- [Vitest 使い方](https://cs.taka1156.site/md/vitest.md) — 単体テストは Vitest、画面を通した E2E は Playwright と分ける。Vitest のブラウザモードは Playwright をプロバイダーに使える
- [MSW 使い方](https://cs.taka1156.site/md/msw.md) — Vitest・開発中の画面・E2E で同じ API モックを使い回す
- [Storybook 使い方](https://cs.taka1156.site/md/storybook.md) — `@storybook/addon-vitest` がストーリーを Playwright の Chromium で描画してテストする

## 参考

- [Installation | Playwright](https://playwright.dev/docs/intro)
- [Command line | Playwright](https://playwright.dev/docs/test-cli)
- [Test generator | Playwright](https://playwright.dev/docs/codegen)
- [Agents | Playwright](https://playwright.dev/docs/test-agents)
- [Locators | Playwright](https://playwright.dev/docs/locators)
- [Auto-waiting | Playwright](https://playwright.dev/docs/actionability)
- [Assertions | Playwright](https://playwright.dev/docs/test-assertions)
- [Test configuration | Playwright](https://playwright.dev/docs/test-configuration)
- [Timeouts | Playwright](https://playwright.dev/docs/test-timeouts)
- [Authentication | Playwright](https://playwright.dev/docs/auth)
- [Fixtures | Playwright](https://playwright.dev/docs/test-fixtures)
- [Mock APIs | Playwright](https://playwright.dev/docs/mock)
- [Release notes | Playwright](https://playwright.dev/docs/release-notes)

2026-11-24 に確認（Playwright 1.65 時点）。
