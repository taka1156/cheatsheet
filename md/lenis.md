<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Lenis 使い方

> スムーススクロールの Lenis（v1.3）の導入、GSAP ScrollTrigger との同期、オプション・メソッド・中の箱のスクロールの扱い。

- URL: https://cs.taka1156.site/lenis.html
- タグ: フロントエンド
- 日付: 2026-11-24

---

ページのスクロールをなめらかにする軽量ライブラリ Lenis のチートシート（v1.3 系・2026-11 時点）

**ざっくり言うと**

- ホイールのスクロールを補間してなめらかにする、MIT ライセンスの軽量スムーススクロールライブラリ。
- ブラウザ本来のスクロール位置を動かすので、 `position: sticky` や ScrollTrigger がそのまま使える。
- 毎フレーム `raf()` を呼ぶか `autoRaf` を有効にし、推奨 CSS を読み込むのが基本の形。

## 導入

npm

CDN

React

```
npm install lenis      # 旧パッケージ名 @studio-freight/lenis は使わない
```

```
import Lenis from 'lenis'
import 'lenis/dist/lenis.css'      // 推奨 CSS（下の「推奨 CSS」）

const lenis = new Lenis({ autoRaf: true })   // 内部で requestAnimationFrame を回す
lenis.on('scroll', (e) => console.log(e.scroll, e.velocity))
```

```
<link rel="stylesheet" href="https://unpkg.com/lenis@1.3.28/dist/lenis.css">
<script src="https://unpkg.com/lenis@1.3.28/dist/lenis.min.js"></script>
<script>
  const lenis = new Lenis({ autoRaf: true, anchors: true })   // グローバルの Lenis
</script>
```

```
import { ReactLenis, useLenis } from 'lenis/react'
import 'lenis/dist/lenis.css'

export default function App() {
  return (
    <ReactLenis root options={{ lerp: 0.1 }}>   {/* root でページ全体（window）を対象に */}
      <Page />
    </ReactLenis>
  )
}

function Page() {
  const lenis = useLenis(({ scroll }) => { /* スクロールのたびに呼ばれる */ })
  return <button onClick={() => lenis?.scrollTo(0)}>上へ</button>
}
```

Vue・Nuxt 用に `lenis/vue` ・ `lenis/nuxt` もある。

（元のページで操作できます → https://cs.taka1156.site/lenis.html#install）

## 自分でループを回す

```
const lenis = new Lenis()        // autoRaf は既定で false

function raf(time) {
  lenis.raf(time)                // 毎フレーム呼ぶ。time はミリ秒
  requestAnimationFrame(raf)
}
requestAnimationFrame(raf)
```

three.js などほかのループと揃えたいときは、そのループの中で `lenis.raf(time)` を呼ぶ。 `autoRaf: true` と自分のループは、どちらか一方だけにする（時刻の基準が違うループから両方 `raf()` を呼ぶと動きが乱れる）。

## GSAP ScrollTrigger と一緒に使う

```
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'
import Lenis from 'lenis'
gsap.registerPlugin(ScrollTrigger)

const lenis = new Lenis()
lenis.on('scroll', ScrollTrigger.update)          // Lenis が動かした位置を ScrollTrigger に伝える
gsap.ticker.add((time) => lenis.raf(time * 1000)) // GSAP の ticker は秒なのでミリ秒に直す
gsap.ticker.lagSmoothing(0)                       // 処理落ち時の時間補正を切ってずれを防ぐ
```

この場合は `autoRaf` を使わず、ループを GSAP の ticker に任せる。

## オプション

| オプション | 既定 | 意味 |
| --- | --- | --- |
| `lerp` | `0.1` | 補間の強さ（0〜1）。小さいほどゆっくり追いつく |
| `duration` | なし | スクロールの秒数。指定すると `lerp` より優先される（README の表には「既定 1.2・lerp 指定時は無効」とあるが、1.3.28 の実装とは異なる） |
| `easing` | 指数関数の減速 | `duration` と組み合わせる緩急の関数。 `easing` だけ指定すると `duration` は 1 秒になる |
| `smoothWheel` | `true` | ホイールのスクロールをなめらかにする |
| `syncTouch` | `false` | タッチのスクロールも Lenis が動かす（iOS 16 未満では不安定） |
| `wheelMultiplier` / `touchMultiplier` | `1` | ホイール・タッチの移動量の倍率 |
| `orientation` | `"vertical"` | スクロールの向き（ `"horizontal"` で横） |
| `gestureOrientation` | `"vertical"` （横スクロールなら `"both"` ） | 受け付ける操作の向き（ `"horizontal"` ・ `"both"` も可） |
| `infinite` | `false` | 終わりと始まりをつないで無限にスクロール（タッチでは `syncTouch` が必要） |
| `autoRaf` | `false` | requestAnimationFrame のループを自動で回す |
| `anchors` | `false` | `true` でページ内リンク（ `#id` ）を Lenis で動かす。 `{ offset: 80 }` のように `scrollTo` の指定も渡せる |
| `allowNestedScroll` | `false` | 中のスクロールできる要素を自動で見分けて、そこは普通にスクロールさせる（重くなることがある） |
| `prevent` | — | `(node) => boolean` 。true を返した要素の上ではなめらかにしない |
| `autoToggle` | `false` | wrapper の `overflow` に合わせて自動で start / stop（推奨 CSS が必要） |
| `respectReducedMotion` | `true` | OS の「視差効果を減らす」が有効なら補間を切り、 `scrollTo` も即座に移動 |
| `wrapper` / `content` | `window` / `html` | ページ全体ではなく、特定の箱の中をスクロールさせるとき |
| `autoResize` | `true` | 大きさの変化を自動で反映。 `false` なら `resize()` を自分で呼ぶ |

## メソッド・プロパティ・イベント

メソッド

プロパティ

HTML 属性

| メソッド | 意味 |
| --- | --- |
| `scrollTo(target, options)` | 指定した位置へスクロール（下の例） |
| `stop()` / `start()` | スクロールを止める / 再開（モーダルを開いている間など） |
| `raf(time)` | 毎フレーム呼ぶ（ミリ秒） |
| `resize()` | 大きさを計算し直す |
| `on('scroll', fn)` | スクロールのたびに呼ぶ。戻り値の関数で解除 |
| `on('virtual-scroll', fn)` | ホイール・タッチの入力（ `{ deltaX, deltaY, event }` ） |
| `destroy()` | インスタンスを捨ててイベントを外す（SPA で画面を離れるとき） |

```
lenis.scrollTo('#contact', { offset: -80 })     // 要素・セレクター。offset は固定ヘッダーの分
lenis.scrollTo(0, { immediate: true })          // アニメーションなしで一番上へ
lenis.scrollTo('bottom', { duration: 2, lock: true })  // 着くまでユーザーの操作を受け付けない
lenis.scrollTo(500, { force: true, onComplete: () => {} })  // stop() 中でも動かす
```

`target` は数値（px）・CSS セレクター・要素・キーワード（ `top` ・ `bottom` ・ `start` ・ `end` ・ `left` ・ `right` ）。要素へ飛ぶときは、要素の `scroll-margin-top` とスクロールする箱の `scroll-padding-top` も差し引かれる。

| プロパティ | 意味 |
| --- | --- |
| `scroll` | 今のスクロール位置（ `infinite` でも扱える値） |
| `targetScroll` | 向かっている先の位置 |
| `progress` | 進み具合（0〜1）。読了バーなどに |
| `velocity` | 今の速さ |
| `direction` | `1` = 下へ、 `-1` = 上へ |
| `limit` | スクロールできる最大値 |
| `isScrolling` | `"smooth"` / `"native"` / `false` |
| `isStopped` | `stop()` 中か |
| `prefersReducedMotion` | 視差効果を減らす設定に従っているか |

```
<div class="modal-body" data-lenis-prevent>…中だけ普通にスクロール…</div>
```

| 属性 | なめらかにしないもの |
| --- | --- |
| `data-lenis-prevent` | すべて |
| `data-lenis-prevent-wheel` | ホイールだけ |
| `data-lenis-prevent-touch` | タッチだけ |
| `data-lenis-prevent-vertical` / `-horizontal` | 縦 / 横の向きだけ |

（元のページで操作できます → https://cs.taka1156.site/lenis.html#api）

## 推奨 CSS（lenis.css）がしていること

- `html.lenis body { height: auto }` : 高さの計算がずれないようにする
- `.lenis-stopped` のとき `overflow: clip` : `stop()` 中にスクロールできないようにする
- `[data-lenis-prevent]` に `overscroll-behavior: contain` : 中の箱のスクロールが外に伝わらない
- スクロール中は `iframe` の `pointer-events` を切る: iframe の上でホイールが止まらない

Lenis が動いている間は `<html>` に `lenis` ・ `lenis-smooth` ・ `lenis-stopped` などのクラスが付く。自分の CSS の `scroll-behavior: smooth` は外しておく（Lenis 自体は `behavior: "instant"` で動かすので影響しないが、Lenis を通らないページ内リンクなどだけ別の動きになり、挙動がそろわない）。

## よくある誤解

✗ 誤解: Lenis を入れるとスクロールが偽物になり、sticky や IntersectionObserver が使えなくなる

✓ 正しくは: Lenis は `transform` で中身を動かす方式ではなく、ブラウザのスクロール位置そのものを少しずつ動かす。そのため `position: sticky` ・IntersectionObserver・ScrollTrigger はそのまま動く。

✗ 誤解: CSS の scroll-snap と一緒に使える

✓ 正しくは: Lenis は CSS の scroll-snap に対応していない。吸着させたいときは別パッケージの `lenis/snap` を使う。

✗ 誤解: new Lenis() するだけでなめらかになる

✓ 正しくは: 既定では `autoRaf` が `false` なので、 `raf()` を毎フレーム呼ばないと何も起きない。 `autoRaf: true` にするか、自分のループ（GSAP の ticker など）から呼ぶ。

✗ 誤解: スマホのタッチスクロールも既定でなめらかにされる

✓ 正しくは: 既定で補間するのはホイールだけ（ `smoothWheel: true` 、 `syncTouch: false` ）。タッチはブラウザ本来の慣性スクロールのまま。

✗ 誤解: アクセシビリティのため、視差効果を減らす設定は自分で判定して Lenis を切る必要がある

✓ 正しくは: v1.3 では `respectReducedMotion` が既定で `true` で、OS の設定が有効なら Lenis が自動で補間を切る。Lenis 自体は動き続けるので、ScrollTrigger などとの同期は保たれる。

## 関連メモ

- [GSAP 使い方](https://cs.taka1156.site/md/gsap.md) — ScrollTrigger と組み合わせてスクロール演出を作るとき

## 参考

- [darkroomengineering/lenis（README）](https://github.com/darkroomengineering/lenis)
- [Lenis 公式サイト](https://lenis.darkroom.engineering/)
- [lenis（npm）](https://www.npmjs.com/package/lenis)

2026-11-24 に確認（npm `lenis@1.3.28` 。オプションの既定値はパッケージの実装・型定義・README で確認。2.0 は開発版のみ）。
