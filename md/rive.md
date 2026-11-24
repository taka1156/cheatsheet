<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Rive 使い方

> Rive の Web ランタイム（v2.55）と React での再生、データバインディングでの操作、非推奨になった入力・イベントの置き換え先。

- URL: https://cs.taka1156.site/rive.html
- タグ: フロントエンド
- 日付: 2026-11-24

---

Rive エディタで作ったインタラクティブなアニメーション（.riv）を、Web で再生・操作するためのチートシート（Web ランタイム v2.55・React 用 v4.41・2026-11 時点）

**ざっくり言うと**

- Rive エディタで作った `.riv` を、 `<canvas>` の上で再生する MIT ライセンスのランタイム。
- 動きはファイルの中のステートマシンが決め、コードからはビューモデルの値を読み書きして操作する。
- ステートマシンの入力（input）や Rive イベントは非推奨。新しく作るならデータバインディングで書く。

## 全体像

1. **エディタで作る** アートボード（画面）に絵を描き、タイムラインで動きを付け、ステートマシンで「どの状態のときにどの動きか」をつなぐ
2. **ビューモデルを用意する** コードと受け渡ししたい値（数値・文字列・真偽値・色・列挙・トリガー・リストなど）をビューモデルのプロパティとして定義し、図形やステートマシンの遷移に結び付ける（データバインディング）
3. **書き出す** `.riv` ファイルとしてエクスポートする。無料プランの書き出しでは短い Rive のスプラッシュが入り、有料プランでは入らない
4. **ランタイムで再生する** `new Rive({ src, canvas, stateMachine, autoBind: true })` で読み込み、 `rive.viewModelInstance` の値を読み書きする

## パッケージの選び方

| パッケージ | 描画 | 向いている場面 |
| --- | --- | --- |
| `@rive-app/webgl2` | WebGL 2（Rive Renderer） | 公式の推奨。エディタと同じレンダラーで描き、Vector Feathering なども含めて全機能が使える |
| `@rive-app/canvas` | Canvas 2D | WebGL のコンテキスト数の上限を気にせず、たくさん並べたいとき。Vector Feathering はまだ描けない |
| `@rive-app/canvas-lite` | Canvas 2D | いちばん小さい版。API は同じだが、テキスト・レイアウト・音声・スクリプトのエンジンと Vector Feathering が入っていない |
| `@rive-app/react-webgl2` ほか | 上と同じ | React 用。 `react-webgl2` ・ `react-canvas` ・ `react-canvas-lite` があり、中身はそれぞれ上のパッケージ |

低レベル API（自分で描画ループを書く）として `@rive-app/canvas-advanced` などもある。普通は上の高レベル API で足りる。

## 読み込んで再生する

JavaScript

CDN

React

```
npm install @rive-app/webgl2
```

```
import { Rive, Layout, Fit, Alignment } from '@rive-app/webgl2'

const canvas = document.getElementById('hero')
const r = new Rive({
  src: '/anim/hero.riv',              // URL。ArrayBuffer なら buffer: で渡す
  canvas,
  artboard: 'Main',                   // 省略で既定のアートボード
  stateMachine: 'State Machine 1',    // 再生するステートマシン
  autoplay: true,
  autoBind: true,                     // 既定のビューモデルのインスタンスを自動で結び付ける
  layout: new Layout({ fit: Fit.Contain, alignment: Alignment.Center }),
  onLoad: () => {
    r.resizeDrawingSurfaceToCanvas()  // 解像度を canvas の CSS サイズ × devicePixelRatio に合わせる
  },
})

addEventListener('resize', () => r.resizeDrawingSurfaceToCanvas())
```

パッケージを `@rive-app/canvas` に替えても、コードはそのまま動く（API は共通）。 `<canvas>` の大きさは CSS で決めておく（ `width: 100%; aspect-ratio: 16 / 9` など）。 `resizeDrawingSurfaceToCanvas()` を呼ばないと、高 DPI の画面でぼやける。

```
<canvas id="hero" width="600" height="400"></canvas>
<script src="https://cdn.jsdelivr.net/npm/@rive-app/webgl2@2.55.1/rive.js"></script>
<script>
  const r = new rive.Rive({         // グローバル変数は小文字の rive
    src: '/anim/hero.riv',
    canvas: document.getElementById('hero'),
    stateMachine: 'State Machine 1',
    autoplay: true,
    autoBind: true,
    onLoad: () => r.resizeDrawingSurfaceToCanvas(),
  })
</script>
```

```
npm install @rive-app/react-webgl2
```

```
import { useRive, useViewModelInstanceNumber, useViewModelInstanceTrigger } from '@rive-app/react-webgl2'

export function Hero() {
  const { rive, RiveComponent } = useRive({
    src: '/anim/hero.riv',
    stateMachine: 'State Machine 1',
    autoplay: true,
    autoBind: true,
  })
  const vmi = rive?.viewModelInstance
  const { value: progress, setValue: setProgress } = useViewModelInstanceNumber('progress', vmi)
  const { trigger: celebrate } = useViewModelInstanceTrigger('celebrate', vmi, {
    onTrigger: () => console.log('ファイル側から発火した'),
  })

  return (
    <>
      <RiveComponent style={{ width: 400, height: 300 }} />  {/* 親の大きさに合わせて描画面も自動で調整 */}
      <button onClick={() => setProgress((progress ?? 0) + 10)}>+10</button>
      <button onClick={celebrate}>お祝い</button>
    </>
  )
}
```

ほかに `useViewModelInstanceString` ・ `Boolean` ・ `Color` ・ `Enum` があり、どれも値と、値を変える関数を返す（ `List` は件数と追加・削除の関数、 `Image` は差し替える関数だけ）。 `autoBind` を使わないときは `useViewModel(rive)` → `useViewModelInstance(viewModel, { rive })` でインスタンスを作って結び付ける。

（元のページで操作できます → https://cs.taka1156.site/rive.html#setup）

## データバインディング（コードから操作する）

```
const vmi = r.viewModelInstance          // autoBind: true のとき、読み込み後に使える

vmi.number('progress').value = 75        // 数値
vmi.string('title').value = 'こんにちは'  // 文字列
vmi.boolean('isDark').value = true       // 真偽値
vmi.color('accent').rgb(255, 80, 0)      // 色（rgba() / argb() / value も）
vmi.enum('mode').value = 'compact'       // 列挙（定義した値の名前）
vmi.trigger('celebrate').trigger()       // トリガーを発火
vmi.number('card/price').value = 980     // 入れ子のビューモデルは / でたどる

// ファイル側（アニメーションやリスナー）で値が変わったら知る
vmi.number('score').on((value) => console.log(value))   // コールバックには新しい値が渡る
vmi.trigger('done').on(() => console.log('完了の合図'))
```

| 書き方 | 意味 |
| --- | --- |
| `number` / `string` / `boolean` / `color` / `enum` / `trigger` | 型ごとにプロパティを取り出す。名前が違うと `null` |
| `list(path)` | リスト。項目（ビューモデルのインスタンス）を足す・消す・入れ替える |
| `image(path)` / `artboard(path)` | 画像やアートボードを差し替える |
| `viewModel(path)` | 入れ子のビューモデルのインスタンス |
| `prop.on(cb)` / `prop.off(cb)` | 値の変化を受け取る / やめる |
| `r.viewModelByName('Card')` | 名前でビューモデルを取る（ `defaultViewModel()` も） |
| `vm.instanceByName('Dark')` / `vm.defaultInstance()` / `vm.instance()` | エディタで作ったインスタンス / 既定のインスタンス / 空のインスタンス |
| `r.bindViewModelInstance(vmi)` | 作ったインスタンスを結び付ける（ `autoBind` を使わないとき） |

値の読み取りや `on` の通知は、ステートマシンが次のフレームを進めたときの値になる。

## 再生の操作・後片付け

| メソッド・プロパティ | 意味 |
| --- | --- |
| `play()` / `pause()` / `stop()` | 再生・一時停止・停止 |
| `reset({ stateMachine, autoplay })` | 最初の状態に戻す |
| `load({ src, ... })` | 別のファイル・アートボードに読み替える |
| `layout = new Layout({...})` | 収め方を変える（下の表） |
| `resizeDrawingSurfaceToCanvas()` | 描画面の解像度を canvas の表示サイズに合わせる |
| `stateMachineNames` / `isPlaying` | ステートマシンの名前一覧 / 再生中か |
| `on(EventType.Load, fn)` | 読み込み・再生などの通知（ `Load` ・ `LoadError` ・ `Play` ・ `Pause` ・ `Stop` など） |
| `volume` | 音声の音量（0〜1） |
| `cleanup()` | WASM 側のメモリを解放する。SPA で画面を離れるときに必ず呼ぶ |

### Fit（収め方）

| 値 | 収め方 |
| --- | --- |
| `Fit.Contain` | 全体が入るように縮める（既定） |
| `Fit.Cover` | 隙間なく埋める（はみ出しは切れる） |
| `Fit.Fill` | 縦横比を無視して引き伸ばす |
| `Fit.FitWidth` / `Fit.FitHeight` | 幅 / 高さを合わせる |
| `Fit.ScaleDown` / `Fit.None` | 大きいときだけ縮める / 拡大縮小しない |
| `Fit.Layout` | アートボードを canvas の大きさに作り直す（エディタのレイアウト機能で作ったレスポンシブな画面向け。 `layoutScaleFactor` で倍率） |

## 非推奨になった書き方と置き換え先

| 古い書き方 | 置き換え先 |
| --- | --- |
| `r.stateMachineInputs('SM')` で入力を取り、 `input.value = …` / `input.fire()` | ビューモデルのプロパティ（ `number` ・ `boolean` ・ `trigger` ） |
| `useStateMachineInput()` （React） | `useViewModelInstanceNumber()` などのフック |
| `r.on(EventType.RiveEvent, …)` （Rive イベント） | プロパティの `on()` （ファイル側でトリガーや値を変える） |
| `EventType.StateChange` | プロパティの `on()` か、ステートの Actions |
| `setTextRunValue()` | 文字列プロパティ（ `vmi.string()` ） |
| `animations: 'idle'` （タイムラインだけの再生）・ `scrub()` | ステートマシンで再生を制御する |
| `stateMachines: ['A', 'B']` （複数を同時に再生） | 1 つのステートマシンにまとめる（ `stateMachine` ） |

非推奨になっても今のファイル・コードは動く。公式は、既存のファイルを作り直す必要はなく、新しい作業ではデータバインディングを使うよう勧めている。

## よくある誤解

✗ 誤解: コードからアニメーションを操作するなら、ステートマシンの入力（input）を使う

✓ 正しくは: 入力はすべてのランタイムで非推奨になり、データバインディング（ビューモデルのプロパティ）に置き換わった。プロパティは遷移だけでなく、色・文字・位置など結び付けたものなら何でも動かせ、ファイル側で変わった値もコードで受け取れる。

✗ 誤解: Rive はランタイムも書き出しも全部無料

✓ 正しくは: ランタイムはオープンソース（MIT）で、ロイヤリティもかからない。ただし無料プランで書き出した `.riv` は再生の最初に短い Rive のスプラッシュが入り、入れたくなければ有料プランが要る。

✗ 誤解: たくさん並べるときも、webgl2 版で 1 つずつ Rive を作ればよい

✓ 正しくは: 普段は webgl2 版でよい（公式の推奨）が、WebGL にはブラウザごとにコンテキスト数の上限がある。たくさん並べるなら `useOffscreenRenderer: true` で 1 つのコンテキストを共有するか（React の `useRive` は既定で有効）、上限のない `@rive-app/canvas` を使う。

✗ 誤解: canvas 要素を DOM から外せばメモリも解放される

✓ 正しくは: Rive は WASM 側にメモリを確保している。使い終わったら `r.cleanup()` を呼ぶ（React の `useRive` はアンマウント時に自動で片付ける）。

## 関連メモ

- [GSAP 使い方](https://cs.taka1156.site/md/gsap.md) — DOM 側の演出と組み合わせるとき（Rive の数値プロパティを GSAP のトゥイーンで動かすなど）

## 参考

- [Rive Docs: Web (JS)](https://rive.app/docs/runtimes/web/web-js)
- [Rive Docs: Web のデータバインディング](https://rive.app/docs/runtimes/web/data-binding) ・ [React のデータバインディング](https://rive.app/docs/runtimes/react/data-binding)
- [Rive Docs: 入力・イベントからデータバインディングへの移行](https://rive.app/docs/editor/data-binding/migration-guide)
- [Rive Docs: Canvas vs WebGL2](https://rive.app/docs/runtimes/web/canvas-vs-webgl) （パッケージの選び方）

2026-11-24 に確認（npm `@rive-app/webgl2@2.55.1` ・ `@rive-app/canvas@2.55.1` ・ `@rive-app/react-webgl2@4.41.0` 。非推奨の扱いはパッケージの型定義の注記で確認）。
