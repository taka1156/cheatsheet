<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Zod 使い方

> Zod v4 でスキーマを定義して実行時に検証し、型を推論する方法と、エラー処理・変換・codec・compile のチートシート

- URL: https://cs.taka1156.site/zod.html
- タグ: Zod, TypeScript, バリデーション
- 日付: 2026-11-19

---

TypeScript のスキーマ定義・実行時検証ライブラリ。型を 1 か所で書いて、検証と型推論の両方に使う（v4.6 時点）

**ざっくり言うと**

- Zod はスキーマを書くと、実行時の検証と TypeScript の型（ `z.infer` ）を同時に手に入れられるライブラリ。
- `parse` は失敗で例外、 `safeParse` は結果オブジェクト、 `validate` は真偽値だけを返す。
- v4 で書き方が変わった所（ `z.email()` ・ `error` 引数など）が多いので、v3 の記事をそのまま写さない。

## 検証の流れ

**入力** JSON・フォーム・環境変数など型の分からない値（ `unknown` ）

**schema.parse()** 型チェック → `refine` → `transform` の順に通す

**成功** 入力の複製を、 `z.infer<typeof schema>` の型で返す

**失敗** `ZodError` （ `issues` に場所と理由の一覧）

## 基本

定義と parse

safeParse

validate

型推論

```
npm install zod

import * as z from 'zod';

const User = z.object({
  name: z.string().min(1),
  email: z.email(),
  age: z.number().int().nonnegative().optional(),
});

const user = User.parse(await req.json());   // 失敗すると ZodError を throw
```

TypeScript 5.5 以降と `tsconfig.json` の `"strict": true` が前提。非同期の `refine` ・ `transform` を含むなら `parseAsync` を使う。

```
const result = User.safeParse(input);
if (!result.success) {
  console.log(z.prettifyError(result.error));   // 人が読める文字列
  return;
}
result.data;   // { name: string; email: string; age?: number }
```

```
// 真偽値だけ返す（関数の形は 4.5〜、メソッドは 4.6〜）。エラーを組み立てないので速く、型ガードにもなる
if (User.validate(input)) {
  input.name;   // ここでは User の入力型に絞り込まれる
}
z.validate(User, input);   // 関数の形（4.5〜。zod/mini はこちら）
```

最初の失敗で止まるので、大きくて信用できない入力の可否だけを知りたいときに向く。

```
type User = z.infer<typeof User>;   // 出力の型（= z.output）

// transform などで入力と出力の型が違うとき
const Len = z.string().transform((s) => s.length);
type In  = z.input<typeof Len>;    // string
type Out = z.output<typeof Len>;   // number
```

（元のページで操作できます → https://cs.taka1156.site/zod.html#basic）

## スキーマの早見表

プリミティブ

文字列の形式

オブジェクト

配列・union

修飾子

| 書き方 | 意味 |
| --- | --- |
| `z.string()` | `.min(n)` ・ `.max(n)` ・ `.length(n)` ・ `.regex(re)` ・ `.trim()` ・ `.toLowerCase()` |
| `z.number()` | `.int()` ・ `.positive()` ・ `.min(n)` ・ `.max(n)` ・ `.multipleOf(n)` （ `NaN` ・ `Infinity` は通らない） |
| `z.int()` | 安全な整数の範囲の整数（ `z.int32()` なども） |
| `z.boolean()` ・ `z.bigint()` ・ `z.date()` | それぞれの型 |
| `z.literal('a')` | 決まった値。 `z.literal(['a', 'b'])` で複数 |
| `z.enum(['a', 'b'])` | 文字列の列挙。TypeScript の enum もそのまま渡せる（ `z.nativeEnum` は非推奨） |
| `z.coerce.number()` | 入力を `Number(x)` で変換してから検証（クエリ文字列向け） |
| `z.stringbool()` | `"true"` ・ `"yes"` ・ `"1"` などを真偽値に（環境変数向け） |
| `z.unknown()` ・ `z.any()` ・ `z.never()` | 何でも通す（型は unknown / any）・何も通さない |

v4 では形式チェックはトップレベルの関数。 `z.string().email()` の形は動くが非推奨。

| 書き方 | 意味 |
| --- | --- |
| `z.email()` | メールアドレス |
| `z.url()` ・ `z.httpUrl()` | URL・http(s) の URL だけ |
| `z.uuid()` ・ `z.uuidv4()` ・ `z.uuidv7()` | UUID（RFC 9562 に厳密）。緩くするなら `z.guid()` |
| `z.iso.datetime()` ・ `z.iso.date()` ・ `z.iso.time()` | ISO 8601 の日時・日付・時刻の文字列（4.5〜 datetime は秒が必須） |
| `z.ipv4()` ・ `z.ipv6()` ・ `z.cidrv4()` | IP アドレス・CIDR |
| `z.jwt()` ・ `z.base64()` ・ `z.hex()` | JWT・Base64・16 進 |
| `z.e164()` ・ `z.creditCard()` ・ `z.iban()` | 電話番号・カード番号・IBAN（後ろ 2 つは 4.5・4.6〜） |
| `z.templateLiteral([...])` | テンプレートリテラル型（例 ``${number}px`` ） |

```
z.object({ ... })        // 知らないキーは取り除く（strip）
z.strictObject({ ... })  // 知らないキーがあればエラー（v3 の .strict()）
z.looseObject({ ... })   // 知らないキーも残す（v3 の .passthrough()）

User.extend({ role: z.enum(['admin', 'user']) })   // キーを足す（.merge は非推奨）
User.pick({ name: true })  /  User.omit({ age: true })
User.partial()             // 全部 optional に（.required() はその逆）
User.shape.name            // 個々のスキーマを取り出す
User.keyof()               // キー名の enum

z.record(z.string(), z.number())   // { [k: string]: number }
```

```
z.array(z.string()).min(1)         // string[]（1 件以上）
z.tuple([z.string(), z.number()])  // [string, number]
z.union([z.string(), z.number()])  // string | number

// type などの判別キーで分岐（union より速く、エラーも分かりやすい）
z.discriminatedUnion('type', [
  z.object({ type: z.literal('ok'),  data: z.string() }),
  z.object({ type: z.literal('err'), message: z.string() }),
]);

z.xor([A, B])   // ちょうど 1 つだけに一致（排他的な union）
```

| 書き方 | 型 | 意味 |
| --- | --- | --- |
| `.optional()` | `T \| undefined` | 省略可 |
| `.nullable()` | `T \| null` | null 可 |
| `.nullish()` | `T \| null \| undefined` | 両方可 |
| `.default(v)` | `T` | `undefined` のとき v を返す（検証せずにそのまま返す） |
| `.prefault(v)` | `T` | `undefined` のとき v を入力として検証・変換する |
| `.catch(v)` | `T` | 検証に失敗したら v を返す |
| `.readonly()` | `Readonly<T>` | 結果を `Object.freeze` |
| `.brand<'UserId'>()` | ブランド型 | 同じ string でも別の型として区別する（実行時は何もしない） |

（元のページで操作できます → https://cs.taka1156.site/zod.html#schemas）

## 独自の検証と変換

refine

transform・pipe

codec

```
const Signup = z.object({
  password: z.string().min(8),
  confirm: z.string(),
}).refine((v) => v.password === v.confirm, {
  error: 'パスワードが一致しません',
  path: ['confirm'],             // エラーを付けるフィールド
});

// 複数の issue を自分で積む
z.string().superRefine((val, ctx) => {
  if (val.includes(' ')) ctx.addIssue({ code: 'custom', message: '空白不可' });
});
```

```
// 検証のあとで値を変える（出力の型が変わる）
const Trimmed = z.string().transform((s) => s.trim());

// 変換してから別のスキーマで検証する
const Port = z.string().pipe(z.coerce.number().int().min(1).max(65535));

// 検証の前に入力を整える
const Num = z.preprocess((v) => (v === '' ? undefined : v), z.number().optional());
```

```
// 4.1〜: 双方向の変換。decode（= parse）で入力→出力、encode で出力→入力
const IsoDate = z.codec(z.iso.datetime(), z.date(), {
  decode: (s) => new Date(s),
  encode: (d) => d.toISOString(),
});
IsoDate.decode('2026-11-19T00:00:00Z');   // Date
IsoDate.encode(new Date());              // string
```

API の JSON と内部の型を行き来する所に使う。よく使う変換は公式ドキュメントにコピペ用の codec がある。

（元のページで操作できます → https://cs.taka1156.site/zod.html#refine）

## エラーメッセージ

```
z.string({ error: '文字列で入力してください' });          // v4 は message ではなく error
z.string().min(8, { error: '8 文字以上' });
z.string({ error: (iss) => iss.input === undefined ? '必須です' : '形式が違います' });

// 日本語の既定メッセージ
import { ja } from 'zod/locales';
z.config(ja());
```

| 関数 | 形 | 向いている用途 |
| --- | --- | --- |
| `err.issues` | `[{ code, path, message, ... }]` | 元データ。自分で整形する |
| `z.prettifyError(err)` | 複数行の文字列 | ログ・CLI の出力 |
| `z.flattenError(err)` | `{ formErrors, fieldErrors }` | 1 階層のフォーム |
| `z.treeifyError(err)` | 入れ子のオブジェクト | 入れ子のフォーム（v3 の `.format()` の代わり） |

## そのほかの機能

| 機能 | 書き方 | メモ |
| --- | --- | --- |
| JSON Schema へ | `z.toJSONSchema(schema)` | `.meta({ title, description })` の情報も出る。逆向きの `z.fromJSONSchema()` は実験的 |
| 事前コンパイル | `z.compile(schema)` | 4.5〜。結果は同じで数倍速い。 `new Function` を使うので CSP で eval 禁止の環境では効かない |
| 全体をコンパイル | `import 'zod/compile'` | エントリーの先頭で 1 回。 `node --import zod/compile app.js` でもよい |
| 小さい版 | `import * as z from 'zod/mini'` | メソッドチェーンの代わりに関数で書く（ `z.optional(z.string())` ）。フロントエンドのバンドルを小さくしたいとき |
| 関数の検証 | `z.function({ input, output })` | `.implement(fn)` で引数と戻り値を検証する関数を作る |
| Standard Schema | そのまま渡す | Zod のスキーマは Standard Schema 対応。 [Hono](https://cs.taka1156.site/md/hono.md) の `sValidator` などにそのまま使える |

## よくある誤解

✗ 誤解: parse は入力オブジェクトをそのまま返す

✓ 正しくは: 成功時に返すのは入力の複製で、 `z.object` では定義にないキーが取り除かれる。 `transform` があれば値も変わる。元の変数ではなく戻り値を使う。

✗ 誤解: v3 の書き方は v4 でもそのまま推奨されている

✓ 正しくは: 多くは動くが非推奨になっている。 `z.string().email()` → `z.email()` 、 `{ message }` → `{ error }` 、 `.strict()` → `z.strictObject` 、 `.merge()` → `.extend()` 、 `err.format()` → `z.treeifyError()` 。

✗ 誤解: default は入力が null のときにも使われる

✓ 正しくは: `.default()` が効くのは `undefined` のときだけ。 `null` も置き換えたいなら `.nullish()` と `transform` を組み合わせるか、 `preprocess` で `null` を `undefined` にする。

✗ 誤解: z.infer の型があれば実行時の検証はいらない

✓ 正しくは: 型は TypeScript のコンパイル時だけのもの。外から来るデータ（API のレスポンス・フォーム・環境変数）は、実行時に `parse` しないと型どおりとは限らない。

## 出典

- [Zod – Basic usage](https://zod.dev/basics) ・ [Defining schemas](https://zod.dev/api) （zod 4.6.7 時点）
- [Customizing errors](https://zod.dev/error-customization) ・ [Formatting errors](https://zod.dev/error-formatting)
- [Codecs](https://zod.dev/codecs) ・ [JSON Schema](https://zod.dev/json-schema) ・ [AOT compilation](https://zod.dev/compile)
- [Migration guide（v3 → v4）](https://zod.dev/v4/changelog) ・ [Releases（GitHub）](https://github.com/colinhacks/zod/releases)

確認日: 2026-11-19
