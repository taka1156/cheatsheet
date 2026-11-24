<!-- scripts/memo.py build が自動生成。手で編集しない -->

# GSAP 使い方

> GSAP（v3.15）のトゥイーン・タイムライン・イージング・ScrollTrigger・主なプラグイン・React での使い方。

- URL: https://cs.taka1156.site/gsap.html
- タグ: フロントエンド
- 日付: 2026-11-24

---

JavaScript のアニメーションライブラリ GSAP（GreenSock Animation Platform）のチートシート（v3.15 系・2026-11 時点）

**ざっくり言うと**

- CSS・SVG・JS のオブジェクトの値を、時間に沿って滑らかに変える JavaScript のアニメーションライブラリ。
- `gsap.to()` で 1 つ動かし、 `timeline` で順番を組み、ScrollTrigger でスクロールに結び付ける。
- v3.13 から SplitText などのプラグインも含めて全部無料。ただし MIT ではなく独自の無料ライセンス。

## 導入

npm

CDN

React

```
npm install gsap        # プラグインも全部入っている（型定義も同梱）
```

```
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'
import { SplitText } from 'gsap/SplitText'

gsap.registerPlugin(ScrollTrigger, SplitText)   // プラグインは使う前に 1 回登録する
```

```
<script src="https://cdn.jsdelivr.net/npm/gsap@3.15/dist/gsap.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/gsap@3.15/dist/ScrollTrigger.min.js"></script>
<script>
  gsap.registerPlugin(ScrollTrigger)   // グローバルの gsap・ScrollTrigger を使う
</script>
```

```
npm install gsap @gsap/react
```

```
import { useRef } from 'react'
import gsap from 'gsap'
import { useGSAP } from '@gsap/react'
gsap.registerPlugin(useGSAP)

function Box() {
  const container = useRef(null)
  useGSAP(() => {
    gsap.to('.box', { x: 200 })          // セレクターは container の中だけを探す
  }, { scope: container })               // アンマウント時に自動で revert される
  return <div ref={container}><div className="box" /></div>
}
```

クリックなどで後から作るアニメーションは `const { contextSafe } = useGSAP({ scope })` で包むと、同じく片付けの対象になる。

（元のページで操作できます → https://cs.taka1156.site/gsap.html#install）

## トゥイーン（1 つ動かす）

```
gsap.to('.box', { x: 200, duration: 1 })              // 今の値 → 指定の値
gsap.from('.box', { opacity: 0, y: 40 })               // 指定の値 → 今の値（登場の演出）
gsap.fromTo('.box', { scale: 0 }, { scale: 1 })        // 開始と終了を両方指定
gsap.set('.box', { xPercent: -50 })                    // 時間 0 ですぐ設定

const tween = gsap.to('.dot', { y: -20, repeat: -1, yoyo: true, stagger: 0.1 })
```

| 指定 | 既定 | 意味 |
| --- | --- | --- |
| `duration` | `0.5` | 秒 |
| `ease` | `"power1.out"` | 緩急（下の「イージング」） |
| `delay` | `0` | 始まるまでの秒数 |
| `repeat` | `0` | 繰り返す回数。 `-1` で無限 |
| `yoyo` | `false` | 繰り返すたびに向きを反転 |
| `stagger` | — | 複数の対象を秒ずつずらす。 `{ each: 0.1, from: 'center' }` も可 |
| `easeReverse` | — | 逆再生のときの緩急。 `true` で順方向と同じ形（3.15〜。 `yoyoEase` は非推奨） |
| `paused` | `false` | 作るだけで再生しない |
| `overwrite` | `false` | `"auto"` で、同じ対象・同じ値を動かす先行トゥイーンの該当部分を止める |
| `onStart` / `onUpdate` / `onComplete` | — | 開始・毎フレーム・完了で呼ぶ関数 |

### 動かせる値

| 書き方 | 意味 |
| --- | --- |
| `x` / `y` | `translateX/Y` （px）。 `left` ・ `top` より軽い |
| `xPercent` / `yPercent` | 自分の幅・高さに対する % |
| `rotation` / `rotationX` / `rotationY` | 回転（度）。 `"+=90"` で今の値から相対 |
| `scale` / `scaleX` / `scaleY` | 拡大・縮小 |
| `autoAlpha` | `opacity` ＋ 0 のとき `visibility: hidden` |
| `transformOrigin` | 回転・拡大の中心（ `"left top"` など） |
| `'--my-var'` | CSS 変数そのもの |
| `color: 'var(--c)'` | CSS 変数の値に向かって動かす（3.13〜） |
| `attr: { r: 20 }` | SVG などの属性 |
| 普通のオブジェクト | `gsap.to(obj, { value: 100 })` 。three.js の `mesh.position` なども動かせる |

## タイムライン（順番を組む）

```
const tl = gsap.timeline({ defaults: { duration: 0.6, ease: 'power2.out' }, repeat: 0 })
tl.from('.title', { y: 40, opacity: 0 })
  .from('.text', { opacity: 0 }, '-=0.3')    // 前の終わりの 0.3 秒前から
  .from('.btn', { scale: 0 }, '<')          // 前と同時に
  .addLabel('end')
  .to('.bg', { opacity: 0.5 }, 'end+=0.2')
```

第 3 引数（位置パラメーター）で、タイムラインのどこに置くかを決める。図は、3 つ目のトゥイーンを置いた位置。

前の 2 つ

省略 / `">"`

`"<"`

`"-=0.5"`

`"<0.5"`

`0.5`

前のトゥイーン（各 1 秒。横幅 30% が 1 秒） 3 つ目のトゥイーンが入る位置

| 位置 | 置かれる場所 |
| --- | --- |
| 省略 / `">"` | 直前のトゥイーンの終わり（順番に並ぶ） |
| `"<"` | 直前のトゥイーンの始まり（同時に動く） |
| `"+=0.5"` / `"-=0.5"` | タイムライン全体の終わりから 0.5 秒後 / 前 |
| `"<0.5"` / `">-0.2"` | 直前の始まりの 0.5 秒後 / 直前の終わりの 0.2 秒前 |
| `1` | タイムラインの 1 秒の位置（絶対） |
| `"label"` / `"label+=1"` | ラベルの位置 / その 1 秒後 |
| `"<25%"` | 直前のトゥイーンの 25% の位置 |

## 再生の操作

| メソッド | 意味 |
| --- | --- |
| `play()` / `pause()` / `resume()` | 再生・一時停止・再開 |
| `reverse()` / `restart()` | 逆再生・最初から |
| `seek(1.5)` / `progress(0.5)` | 秒・割合（0〜1）の位置へ飛ぶ |
| `timeScale(2)` | 再生速度（2 で倍速） |
| `kill()` | 止めて捨てる（値はそのまま） |
| `revert()` | 止めて、GSAP が書き込んだインラインスタイルを元に戻す |
| `then()` | 完了で解決する Promise（ `await tween` ） |
| `gsap.killTweensOf('.box')` | その対象のトゥイーンを全部止める |

## イージング

| 名前 | 動き |
| --- | --- |
| `"none"` | 一定の速さ |
| `"power1"` 〜 `"power4"` | 数字が大きいほど緩急が強い。 `power1` = quad、 `power2` = cubic、 `power3` = quart、 `power4` = quint 相当 |
| `"sine"` / `"expo"` / `"circ"` | 弱い / とても強い / 円弧の緩急 |
| `"back(1.7)"` | 一度行き過ぎて戻る |
| `"elastic(1, 0.3)"` | ばねのように揺れる |
| `"bounce"` | 跳ねる |
| `"steps(5)"` | 5 段階でカクカク |

それぞれ `.in` （ゆっくり始まる）・ `.out` （ゆっくり終わる。既定）・ `.inOut` を付ける（例: `"power3.inOut"` ）。自分で曲線を描くなら CustomEase。

## ScrollTrigger

```
gsap.registerPlugin(ScrollTrigger)

// 1. 画面に入ったら再生
gsap.from('.card', {
  y: 60, opacity: 0, stagger: 0.1,
  scrollTrigger: { trigger: '.cards', start: 'top 80%' },
})

// 2. スクロール量に合わせて進める・固定する
const tl = gsap.timeline({
  scrollTrigger: {
    trigger: '.section',
    start: 'top top',     // trigger の上端が画面の上端に来たら
    end: '+=2000',        // そこから 2000px スクロールする間
    scrub: 1,             // スクロールに追従（数値は追いつくまでの秒数）
    pin: true,            // その間 .section を画面に固定
    markers: true,        // 開発中だけ: 開始・終了の位置を表示
  },
})
tl.to('.a', { x: 500 }).to('.b', { rotation: 360 })
```

| 指定 | 既定 | 意味 |
| --- | --- | --- |
| `trigger` | — | 位置の基準にする要素 |
| `start` | `"top bottom"` | 「要素のどこ 画面のどこ」が重なったら開始。 `pin` があると `"top top"` |
| `end` | `"bottom top"` | 終了位置。 `"+=500"` で開始から 500px |
| `scrub` | `false` | `true` でスクロール位置に直結、数値で遅れて追従 |
| `toggleActions` | `"play none none none"` | 入る・出る・戻って入る・戻って出るときの動作（play / pause / resume / reverse / restart / reset / complete / none） |
| `pin` | `false` | 区間の間、要素を固定。 `pinSpacing` で後ろの余白を調整 |
| `once` | `false` | 1 回だけ動かす |
| `snap` | — | 止まったときに区切り（ `1 / 4` 、 `"labels"` など）へ吸着 |
| `toggleClass` | — | 区間内だけクラスを付ける |
| `onEnter` / `onLeave` / `onUpdate` | — | 区間に入る・出る・進むたびに呼ぶ（ `self.progress` で進み具合） |
| `containerAnimation` | — | 横スクロールのアニメーションの中の要素を基準にする |

| 静的メソッド | 意味 |
| --- | --- |
| `ScrollTrigger.create({...})` | アニメーションなしでスクロール位置だけ使う |
| `ScrollTrigger.batch('.item', {...})` | 画面に入った要素をまとめて処理 |
| `ScrollTrigger.refresh()` | 位置を計算し直す（画像の読み込みやレイアウト変更のあと） |
| `ScrollTrigger.getAll()` / `killAll()` | 全部取得 / 全部止める |
| `ScrollTrigger.normalizeScroll(true)` | モバイルのアドレスバーの伸び縮みによるずれ・ガタつきを抑える |

## 主なプラグイン

すべて `gsap` パッケージに入っていて、 `gsap/名前` から import して `registerPlugin` する。

| プラグイン | 用途 |
| --- | --- |
| `ScrollTrigger` | スクロールに連動（上の節） |
| `ScrollSmoother` | ScrollTrigger の上に作るスムーススクロール（ネイティブのスクロールを使う） |
| `ScrollToPlugin` | `gsap.to(window, { scrollTo: '#sec' })` でスクロール位置を動かす |
| `SplitText` | 文字・単語・行に分けて 1 つずつ動かす |
| `Flip` | レイアウトが変わる前後をなめらかにつなぐ（FLIP） |
| `Draggable` ＋ `InertiaPlugin` | ドラッグ・回転操作と、離したあとの慣性 |
| `Observer` | ホイール・タッチ・ポインターの向きを統一して受け取る |
| `MotionPathPlugin` | SVG のパスに沿って動かす |
| `DrawSVGPlugin` | SVG の線を描くように見せる |
| `MorphSVGPlugin` | SVG の形を別の形に変形（3.14 で `smooth` が追加） |
| `CustomEase` | 自分で描いた曲線をイージングにする |
| `TextPlugin` / `ScrambleTextPlugin` | 文字を差し替える / ランダムな文字を経由して差し替える |

### SplitText の例

```
gsap.registerPlugin(SplitText)
SplitText.create('.headline', {
  type: 'lines, words',
  mask: 'lines',           // 行ごとに overflow: clip の箱で包む（下から出てくる演出）
  autoSplit: true,         // フォントの読み込み・幅の変化で分け直す
  onSplit: (self) =>       // 分け直すたびに呼ばれる。返したアニメーションは自動で引き継がれる
    gsap.from(self.lines, { yPercent: 100, stagger: 0.08 }),
})
```

## レスポンシブ・片付け

```
// 画面幅や設定ごとにアニメーションを分ける。条件が外れると自動で revert される
const mm = gsap.matchMedia()
mm.add({
  isDesktop: '(min-width: 800px)',
  reduceMotion: '(prefers-reduced-motion: reduce)',
}, (ctx) => {
  const { isDesktop, reduceMotion } = ctx.conditions
  gsap.to('.box', { x: isDesktop ? 400 : 100, duration: reduceMotion ? 0 : 1 })
})

// 一式をまとめて片付ける（React 以外のコンポーネントなど）
const ctx = gsap.context(() => { gsap.to('.box', { x: 100 }) }, rootElement)
ctx.revert()   // 中で作ったトゥイーン・ScrollTrigger・SplitText を全部元に戻す
```

## よくある誤解

✗ 誤解: SplitText や MorphSVG は有料会員（Club GSAP）だけのプラグイン

✓ 正しくは: 2025-04-30 の v3.13 から、旧会員限定のプラグインも含めて全部無料で、商用利用もできる。すべて npm の `gsap` パッケージに入っていて、専用のレジストリやトークンは要らない。

✗ 誤解: 無料になったので GSAP は MIT ライセンスのオープンソースになった

✓ 正しくは: ライセンスは Webflow の独自ライセンス（Standard "No Charge" GSAP License）。普通のサイトやアプリには無料で使えるが、Webflow と競合する「コードを書かずにアニメーションを作るビジュアルツール」に組み込むことは禁止されている。

✗ 誤解: React では useEffect の中で gsap.to を呼べばよい

✓ 正しくは: StrictMode では effect が 2 回走り、片付けないとアニメーションが二重になる。 `@gsap/react` の `useGSAP()` を使うと、中で作ったものがアンマウント時に自動で revert される。

✗ 誤解: タイムラインの中の各トゥイーンに scrollTrigger を付ければ、順番にスクロールで動く

✓ 正しくは: タイムラインの子トゥイーンに付けた `scrollTrigger` は、親のタイムラインと再生位置の取り合いになり正しく動かない。 `scrollTrigger` はタイムライン自体か、独立したトゥイーンに付ける。

✗ 誤解: 位置を動かすときは left や top を動かす

✓ 正しくは: `left` ・ `top` ・ `width` はレイアウトの再計算が起きて重い。位置は `x` ・ `y` 、大きさは `scale` を使う（transform なので GPU で合成できる）。

## 関連メモ

- [Lenis 使い方](https://cs.taka1156.site/md/lenis.md) — スムーススクロールを Lenis にして、ScrollTrigger と同期させるとき
- [three.js 使い方](https://cs.taka1156.site/md/threejs.md) — 3D の物の位置やカメラを GSAP で動かすとき

## 参考

- [GSAP Docs](https://gsap.com/docs/v3/)
- [ScrollTrigger](https://gsap.com/docs/v3/Plugins/ScrollTrigger/)
- [GSAP & React（useGSAP）](https://gsap.com/resources/React/)
- [3.13 リリース（全プラグインの無料化）](https://gsap.com/blog/3-13/) ・ [3.15 リリース（easeReverse）](https://gsap.com/blog/3-15/)
- [Standard "No Charge" GSAP License](https://gsap.com/community/standard-license/)

2026-11-24 に確認（npm `gsap@3.15.1` 。npm には `4.0.0-beta` も出ているが、このページは v3 の内容）。
