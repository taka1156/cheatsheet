<!-- scripts/memo.py build が自動生成。手で編集しない -->

# three.js 使い方

> three.js（r187）の導入、シーン・カメラ・レンダラーの基本形、よく使う API とアドオン、glTF 読み込みなどの定番手順。

- URL: https://cs.taka1156.site/threejs.html
- タグ: フロントエンド
- 日付: 2026-11-24

---

ブラウザで 3D を描く JavaScript ライブラリ three.js のチートシート（r187・2026-11 時点）

**ざっくり言うと**

- WebGL・WebGPU を直接書かずに、シーン・カメラ・メッシュ・ライトを組み立てて 3D を描くライブラリ。
- シーンに物を置き、カメラから見た絵をレンダラーが `<canvas>` に毎フレーム描く、が基本の形。
- 視点操作や glTF 読み込みは本体ではなく `three/addons/` から読み込む。版ごとに API が変わるので版は固定する。

## 全体像

**Scene**

- `Mesh` = `Geometry` （形）＋ `Material` （見た目）
- `Light` （光源）
- `Group` （まとめて動かす）
- `scene.background` ・ `environment`

**Camera**

- `PerspectiveCamera` （遠近あり）
- `OrthographicCamera` （遠近なし）

**Renderer**

- `renderer.render(scene, camera)`
- `renderer.domElement` が `<canvas>`
- `setAnimationLoop()` で毎フレーム呼ぶ

物はすべて `Object3D` の子孫で、 `position` ・ `rotation` ・ `scale` と親子関係（ `add()` ）を持つ。単位は自由だが、glTF やライトの物理単位に合わせて「1 = 1 m」にしておくと楽。

## 導入

npm + Vite

import map（CDN）

```
npm install three
npm install -D vite @types/three   # TypeScript なら @types/three も（本体に型は入っていない）
npx vite                           # 開発サーバー
```

```
// main.js
import * as THREE from 'three'
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js'
```

公式マニュアルもビルドツール（Vite など）を使う形を勧めている。 `@types/three` は本体と同じ版（ `0.187.x` ）にそろえる。

```
<script type="importmap">
{
  "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.187.0/build/three.module.js",
    "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.187.0/examples/jsm/"
  }
}
</script>
<script type="module">
  import * as THREE from 'three'
  import { OrbitControls } from 'three/addons/controls/OrbitControls.js'
</script>
```

本体とアドオンは **同じ版・同じ CDN** から読む（混ぜると three.js が 2 重に読み込まれて壊れる）。URL の版は必ず固定し、 `latest` にしない。 `file://` では動かないので、ローカルでも `npx serve` などのサーバーで開く。

（元のページで操作できます → https://cs.taka1156.site/threejs.html#install）

## 最小のコード

```
import * as THREE from 'three'
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'

// 1. シーン・カメラ・レンダラー
const scene = new THREE.Scene()
scene.background = new THREE.Color(0x202530)

const camera = new THREE.PerspectiveCamera(50, innerWidth / innerHeight, 0.1, 100)
camera.position.set(3, 2, 5)

const renderer = new THREE.WebGLRenderer({ antialias: true })
renderer.setPixelRatio(Math.min(devicePixelRatio, 2))   // 高 DPI で重くなりすぎないよう上限
renderer.setSize(innerWidth, innerHeight)
document.body.appendChild(renderer.domElement)

// 2. 物とライト
const cube = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.MeshStandardMaterial({ color: 0x44aa88 }),
)
scene.add(cube)
scene.add(new THREE.AmbientLight(0xffffff, 0.4))
const sun = new THREE.DirectionalLight(0xffffff, 2)
sun.position.set(5, 10, 5)
scene.add(sun)

// 3. 視点操作
const controls = new OrbitControls(camera, renderer.domElement)
controls.enableDamping = true

// 4. 毎フレーム描く
const timer = new THREE.Timer()
timer.connect(document)   // タブが裏にある間は時間を進めない
renderer.setAnimationLoop((time) => {
  timer.update(time)
  cube.rotation.y += timer.getDelta()   // 1 秒で 1 ラジアン（フレームレートに依存しない）
  controls.update()                     // enableDamping のときは毎フレーム必要
  renderer.render(scene, camera)
})

// 5. 画面サイズに追従
addEventListener('resize', () => {
  camera.aspect = innerWidth / innerHeight
  camera.updateProjectionMatrix()       // カメラの値を変えたら必ず呼ぶ
  renderer.setSize(innerWidth, innerHeight)
})
```

角度はラジアン。度で書きたいときは `THREE.MathUtils.degToRad(45)` 。

## よく使う API

ジオメトリ

マテリアル

ライト

カメラ

Object3D

レンダラー

| クラス | 引数（主なもの） | 用途 |
| --- | --- | --- |
| `BoxGeometry` | `(w, h, d)` | 箱 |
| `SphereGeometry` | `(radius, widthSeg, heightSeg)` | 球。分割数を上げると滑らか |
| `PlaneGeometry` | `(w, h)` | 板。床にするなら `rotation.x = -Math.PI / 2` |
| `CylinderGeometry` | `(rTop, rBottom, h, radialSeg)` | 円柱・円錐 |
| `CapsuleGeometry` | `(radius, height, capSeg, radialSeg, heightSeg)` | カプセル（キャラの当たり判定の見た目など） |
| `TorusGeometry` | `(radius, tube)` | ドーナツ |
| `BufferGeometry` | `setAttribute('position', new BufferAttribute(arr, 3))` | 頂点を自分で並べる |

同じ形をたくさん並べるときは、Mesh を大量に作らず `InstancedMesh(geometry, material, count)` と `setMatrixAt(i, matrix)` を使う（描画が 1 回で済む）。

| クラス | ライト | 用途 |
| --- | --- | --- |
| `MeshBasicMaterial` | 不要 | 単色・テクスチャをそのまま。陰影なし |
| `MeshLambertMaterial` | 必要 | つやなしの軽い陰影 |
| `MeshPhongMaterial` | 必要 | ハイライトつき（古典的） |
| `MeshStandardMaterial` | 必要 | PBR。 `metalness` ・ `roughness` 。迷ったらこれ |
| `MeshPhysicalMaterial` | 必要 | Standard ＋ ガラス・クリアコート・布など |
| `MeshNormalMaterial` | 不要 | 法線を色で表示（デバッグ用） |
| `LineBasicMaterial` / `PointsMaterial` | 不要 | `Line` ・ `Points` 用 |
| `ShaderMaterial` | — | GLSL を自分で書く（WebGLRenderer 専用） |

```
new THREE.MeshStandardMaterial({
  color: 0xffffff, map: texture,          // 色・カラーテクスチャ
  metalness: 0, roughness: 0.5,
  transparent: true, opacity: 0.8,        // 半透明は transparent も必要
  side: THREE.DoubleSide,                 // 裏面も描く（板の裏が消えるとき）
  wireframe: true,
})
```

| クラス | 引数 | どんな光か |
| --- | --- | --- |
| `AmbientLight` | `(color, intensity)` | 全体を均一に明るく。影・陰影はできない |
| `HemisphereLight` | `(skyColor, groundColor, intensity)` | 空と地面の 2 色の環境光。屋外向け |
| `DirectionalLight` | `(color, intensity)` | 太陽のような平行光。 `position` から `target` へ向く |
| `PointLight` | `(color, intensity, distance, decay)` | 電球。距離で減衰（ `decay` 既定 2） |
| `SpotLight` | `(color, intensity, distance, angle, penumbra)` | スポットライト |
| `RectAreaLight` | `(color, intensity, w, h)` | 面光源。Standard / Physical のみ、影なし |

ライトは物理単位（ `PointLight` ・ `SpotLight` はカンデラ）。距離 1 m を 1 にしていないと、点光源は暗すぎたり明るすぎたりする。見た目を手早く整えるなら、環境マップ（下の「定番の手順」）を `scene.environment` に入れるのが早い。

```
// 遠近あり: (縦の画角[度], 縦横比, near, far)
const camera = new THREE.PerspectiveCamera(50, w / h, 0.1, 1000)

// 遠近なし: (left, right, top, bottom, near, far)
const s = 5, a = w / h
const ortho = new THREE.OrthographicCamera(-s * a, s * a, s, -s, 0.1, 100)

camera.position.set(0, 2, 5)
camera.lookAt(0, 0, 0)            // OrbitControls を使うなら controls.target で決める
camera.updateProjectionMatrix()   // fov・aspect・near・far などを変えたら呼ぶ
```

`near` を小さく・ `far` を大きくしすぎると、奥で面がちらつく（Z ファイティング）。範囲は必要な分だけにする。

| 書き方 | 意味 |
| --- | --- |
| `obj.position.set(x, y, z)` | 位置（親からの相対）。Y が上 |
| `obj.rotation.set(x, y, z)` | 回転（ラジアン、オイラー角） |
| `obj.scale.setScalar(2)` | 拡大・縮小 |
| `obj.lookAt(v)` | 指定した点の方を向く |
| `parent.add(child)` / `remove(child)` | 親子関係。親を動かすと子もついてくる |
| `obj.traverse(fn)` | 子孫を全部たどる（glTF の中のメッシュに設定するときなど） |
| `obj.getObjectByName('name')` | 名前で探す |
| `obj.getWorldPosition(new Vector3())` | ワールド座標での位置 |
| `obj.visible = false` | 非表示 |
| `obj.userData` | 自由に使えるデータ置き場 |

| 書き方 | 意味 |
| --- | --- |
| `new WebGLRenderer({ antialias, alpha, canvas })` | `alpha: true` で背景を透明に。 `canvas` で既存の要素に描く |
| `setSize(w, h)` / `setPixelRatio(r)` | 描画サイズ・解像度の倍率 |
| `setAnimationLoop(fn)` | 毎フレーム呼ぶ（WebXR でもこれ）。 `null` で止める |
| `shadowMap.enabled = true` | 影を有効にする |
| `toneMapping = THREE.ACESFilmicToneMapping` | トーンマッピング（HDR の環境マップと組み合わせる） |
| `outputColorSpace` | 出力の色空間。既定の `SRGBColorSpace` のままでよい |
| `renderer.info` | 描画回数・三角形数・メモリ上のジオメトリ・テクスチャ数（重さの確認） |
| `dispose()` | レンダラーを捨てる（SPA で画面を離れるとき） |

（元のページで操作できます → https://cs.taka1156.site/threejs.html#api）

## よく使うアドオン

すべて `three/addons/` から名前つきで import する（ `THREE.OrbitControls` のように本体の名前空間からは使えない）。

| import 元 | クラス | 用途 |
| --- | --- | --- |
| `controls/OrbitControls.js` | `OrbitControls` | ドラッグで注視点の周りを回る・ズーム・パン |
| `controls/MapControls.js` | `MapControls` | 地図のような操作（左ドラッグでパン） |
| `controls/TransformControls.js` | `TransformControls` | 物を掴んで移動・回転・拡大するギズモ |
| `controls/PointerLockControls.js` | `PointerLockControls` | FPS 風の視点 |
| `loaders/GLTFLoader.js` | `GLTFLoader` | glTF / GLB モデルの読み込み（推奨形式） |
| `loaders/DRACOLoader.js` | `DRACOLoader` | Draco 圧縮された glTF を解く |
| `loaders/KTX2Loader.js` | `KTX2Loader` | KTX2（GPU 圧縮テクスチャ） |
| `loaders/HDRLoader.js` | `HDRLoader` | `.hdr` の環境マップ（旧 `RGBELoader` ） |
| `environments/RoomEnvironment.js` | `RoomEnvironment` | ファイルなしで使える室内の環境マップ |
| `postprocessing/EffectComposer.js` | `EffectComposer` | ポストエフェクト（ `RenderPass` ・ `UnrealBloomPass` など） |
| `libs/stats.module.js` | `Stats` （default） | FPS 表示 |
| `libs/lil-gui.module.min.js` | `GUI` | パラメーターを触る GUI |

## 定番の手順

glTF を読む

環境マップ

テクスチャ

影

クリック判定

アニメーション

片付け

```
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js'
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js'

const draco = new DRACOLoader()
draco.setDecoderPath('/draco/')   // node_modules/three/examples/jsm/libs/draco/ を public/draco/ に置く

const loader = new GLTFLoader()
loader.setDRACOLoader(draco)      // Draco 圧縮のモデルを読むときだけ必要

const gltf = await loader.loadAsync('/models/robot.glb')
scene.add(gltf.scene)
gltf.scene.traverse((o) => {
  if (o.isMesh) { o.castShadow = true; o.receiveShadow = true }
})
// gltf.animations: AnimationClip の配列 / gltf.cameras: カメラ
```

モデルが大きすぎる・見えないときは、 `new THREE.Box3().setFromObject(gltf.scene)` で大きさと中心を測って、カメラの位置や `scale` を合わせる。

```
// ファイルなし: 室内っぽい環境光で PBR マテリアルを照らす
import { RoomEnvironment } from 'three/addons/environments/RoomEnvironment.js'
const pmrem = new THREE.PMREMGenerator(renderer)
scene.environment = pmrem.fromScene(new RoomEnvironment(), 0.04).texture

// .hdr ファイルから
import { HDRLoader } from 'three/addons/loaders/HDRLoader.js'
const hdr = await new HDRLoader().loadAsync('/env/studio.hdr')
hdr.mapping = THREE.EquirectangularReflectionMapping
scene.environment = hdr        // 照明・映り込み
scene.background = hdr         // 背景にも出すなら
renderer.toneMapping = THREE.ACESFilmicToneMapping
```

```
const tex = await new THREE.TextureLoader().loadAsync('/img/wood.jpg')
tex.colorSpace = THREE.SRGBColorSpace        // 色のテクスチャ（map）は必ず指定。無いと白っぽくなる
tex.wrapS = tex.wrapT = THREE.RepeatWrapping
tex.repeat.set(4, 4)                         // 4×4 で繰り返す
tex.anisotropy = renderer.capabilities.getMaxAnisotropy()   // 斜めから見たときのぼやけを減らす

new THREE.MeshStandardMaterial({ map: tex })
// normalMap・roughnessMap などデータのテクスチャは colorSpace を変えない
```

1. **レンダラーで有効にする** `renderer.shadowMap.enabled = true` 。既定の `PCFShadowMap` で影の縁は柔らかい。ぼかしの強さは `sun.shadow.radius` で調整する（ `PCFSoftShadowMap` は r182 で非推奨）
2. **ライトに影を落とさせる** `sun.castShadow = true` 。 `AmbientLight` ・ `HemisphereLight` は影を作れない
3. **物に設定する** 影を落とす側に `mesh.castShadow = true` 、受ける側（床など）に `receiveShadow = true`
4. **範囲と解像度を調整する** `DirectionalLight` の影は `sun.shadow.camera` （left / right / top / bottom）の範囲内だけ。粗いときは `sun.shadow.mapSize.set(2048, 2048)` 。範囲は `new THREE.CameraHelper(sun.shadow.camera)` で見える

```
const raycaster = new THREE.Raycaster()
const pointer = new THREE.Vector2()

renderer.domElement.addEventListener('pointerdown', (e) => {
  const rect = renderer.domElement.getBoundingClientRect()
  pointer.x = ((e.clientX - rect.left) / rect.width) * 2 - 1     // -1〜1 に変換
  pointer.y = -((e.clientY - rect.top) / rect.height) * 2 + 1    // 上が +1
  raycaster.setFromCamera(pointer, camera)
  const hits = raycaster.intersectObjects(clickables)   // 子孫も調べる（既定で recursive）
  if (hits.length) console.log(hits[0].object, hits[0].point)   // 近い順
})
```

対象を `scene.children` 全体にすると重いので、判定したい物だけの配列を渡す。

```
// glTF に入っているアニメーションを再生
const mixer = new THREE.AnimationMixer(gltf.scene)
const action = mixer.clipAction(THREE.AnimationClip.findByName(gltf.animations, 'Walk'))
action.play()
// 切り替え: next.reset().play(); action.crossFadeTo(next, 0.3)

const timer = new THREE.Timer()
timer.connect(document)
renderer.setAnimationLoop((time) => {
  timer.update(time)
  mixer.update(timer.getDelta())   // 経過秒を渡す
  renderer.render(scene, camera)
})
```

```
// シーンから外しても GPU のメモリは解放されない。明示的に dispose する
scene.remove(mesh)
mesh.geometry.dispose()
mesh.material.dispose()
mesh.material.map?.dispose()       // テクスチャも別に解放

// まとめて片付ける（SPA で画面を離れるときなど）
renderer.setAnimationLoop(null)
controls.dispose()
renderer.dispose()
```

漏れていないかは `renderer.info.memory` （geometries・textures の数）が増え続けないかで確かめる。

（元のページで操作できます → https://cs.taka1156.site/threejs.html#recipes）

## WebGPURenderer

```
import * as THREE from 'three/webgpu'    // WebGPU 版はこちらから（'three' と混ぜない）

const renderer = new THREE.WebGPURenderer({ antialias: true })
await renderer.init()                    // 最初に 1 回 await してから render する
renderer.setAnimationLoop(() => renderer.render(scene, camera))
```

| 項目 | WebGLRenderer | WebGPURenderer |
| --- | --- | --- |
| import 元 | `three` | `three/webgpu` |
| WebGPU が無いブラウザ | — | WebGL 2 のバックエンドに自動で切り替わる |
| 独自シェーダー | `ShaderMaterial` （GLSL） | TSL（ `three/tsl` ）でノードを組む。GLSL の `ShaderMaterial` は使えない |
| ポストエフェクト | `EffectComposer` | `RenderPipeline` （r183 で `PostProcessing` から改名）と TSL のノード |
| `renderAsync()` | — | 非推奨（r181 から）。 `init()` を待ってから `render()` |

既存のコードや情報が多いのは WebGLRenderer。 `ShaderMaterial` や `EffectComposer` に頼るなら WebGL のまま、計算シェーダーや新しい機能が欲しいなら WebGPU、で選ぶ。

## よくある誤解

✗ 誤解: MeshStandardMaterial を使えば、ライトを置かなくても見える

✓ 正しくは: `MeshStandardMaterial` ・ `Lambert` ・ `Phong` ・ `Physical` はライトか `scene.environment` が無いと真っ黒になる。まず確かめたいだけなら `MeshBasicMaterial` か `MeshNormalMaterial` を使う。

✗ 誤解: script タグで three.min.js を読めば THREE が使える

✓ 正しくは: `build/three.min.js` （グローバルの `THREE` ）は r161 で配布が終わり、r186 では minify 済みの `three.module.min.js` も npm の配布物から消えた。今は ES モジュール（npm か import map）だけ。 `THREE.GLTFLoader` のような古い書き方も動かない。

✗ 誤解: アニメーションの時間は THREE.Clock で測る

✓ 正しくは: `Clock` は r183 で非推奨になり、使うと警告が出る。代わりに本体の `THREE.Timer` を使う。 `timer.update()` をフレームの最初に 1 回呼び、そのフレームの中では `getDelta()` を何回呼んでも同じ値が返る。

✗ 誤解: scene.remove() すればメモリも解放される

✓ 正しくは: シーンから外すだけでは、GPU 上のジオメトリ・マテリアル・テクスチャは残る。使わなくなったら `dispose()` を呼ぶ（上の「片付け」）。

✗ 誤解: 位置やカメラの設定を変えれば、すぐ画面に反映される

✓ 正しくは: 描画は `render()` を呼んだときだけ。アニメーションループを回していないなら、変えたあとに `render()` を呼ぶ。カメラの `fov` ・ `aspect` ・ `near` ・ `far` は、さらに `updateProjectionMatrix()` が要る。

✗ 誤解: three.js はセマンティックバージョニングなので、マイナー版の更新なら壊れない

✓ 正しくは: npm の版は `0.187.0` のように 0 始まりで、毎回のリリース（r187）が破壊的変更を含みうる。非推奨の API は約 10 リリース残ってから消える。上げるときは公式の Migration Guide を版ごとに読み、本体と `@types/three` を一緒に上げる。

## 関連メモ

- [Vite+ と vp コマンド チートシート](https://cs.taka1156.site/md/vite-plus.md) — three.js のプロジェクトを Vite で動かすとき

## 参考

- [three.js manual: Installation](https://threejs.org/manual/#en/installation)
- [three.js docs](https://threejs.org/docs/) （API リファレンス）
- [three.js Migration Guide](https://github.com/mrdoob/three.js/wiki/Migration-Guide) （版ごとの破壊的変更）
- [three.js Releases](https://github.com/mrdoob/three.js/releases)

2026-11-24 に確認（three r187 / npm `three@0.187.0` 。 `Clock` の非推奨・ `HDRLoader` ・ `renderAsync` の非推奨はパッケージのソースで確認）。
