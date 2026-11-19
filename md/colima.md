<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Colima 使い方

> macOS で Docker Desktop の代わりに Colima を使う方法。仕組み、コマンド、start のオプション、設定ファイル、つまずきどころをまとめたチートシート。

- URL: https://cs.taka1156.site/colima.html
- タグ: 開発ツール
- 日付: 2026-11-20

---

macOS（と Linux）で Docker / containerd / Kubernetes を手軽に動かす CLI のチートシート（v0.10.3 時点）

**ざっくり言うと**

- Colima は Lima の Linux VM の中で Docker などを動かし、Mac の `docker` コマンドから使えるようにする CLI。
- `colima start` で VM を作って起動し、Docker コンテキスト `colima` に自動で切り替える。
- VM の種類やマウント方式は作成後に変えられないので、変えたいときは `colima delete` して作り直す。

## 仕組み

名前の由来は Containers on Lima。Mac 上の `docker` CLI は、Colima が作った Docker コンテキストを通して VM の中の Docker Engine に話しかける。

**Mac（ホスト）** `docker` / `kubectl` / `nerdctl` などのクライアント

**Docker コンテキスト** 名前は `colima` （別プロファイルは `colima-名前` ）。

ソケット `~/.colima/default/docker.sock`

**Lima VM（Ubuntu）** VM の種類: vz（macOS 13 以降の既定）/ qemu / krunkit

dockerd containerd k3s incus

**マウント** 既定で `$HOME` を書き込み可でマウント。ほかの場所は `mounts` に足す。

**ポート転送** コンテナが公開したポートは自動で Mac の `localhost` に転送される（既定は ssh 方式、TCP のみ）。

**ホストへの接続** コンテナから Mac へは `host.docker.internal` で届く。

## はじめかた

1. **インストール** `brew install colima docker` 。Colima は VM を用意するだけなので、 `docker` クライアント（必要なら `docker-compose` ・ `docker-buildx` ・ `kubectl` ）は別に入れる。
2. **起動** `colima start` 。初回は VM イメージをダウンロードして作成する。既定は 2 CPU・メモリ 2GiB・ディスク 100GiB。
3. **使う** `docker run hello-world` 。追加の設定なしで Mac の `docker` がそのまま使える。
4. **止める** `colima stop` 。コンテナやイメージは残り、次の `colima start` で戻る。

ログイン時に自動で起動したいときは、Homebrew で入れていれば `brew services start colima` 。

## コマンド チートシート

プロファイル名を省くと `default` 。 `-p 名前` （ `--profile` ）でも指定できる。 危険 はデータが消える操作。

起動・停止

状態・プロファイル

VM の操作

ランタイム

設定・保守

| コマンド | やること |
| --- | --- |
| `colima start` | VM を作成（初回）または起動する。すでに起動中なら何もしない |
| `colima start --cpus 4 --memory 8` | CPU・メモリ（GiB）を指定して起動。止めてから実行すると既存の VM にも反映される |
| `colima start --edit` | 設定ファイル（ `colima.yaml` ）をエディタで開いてから起動する |
| `colima start --foreground` | フォアグラウンドで動かし続ける（launchd などで管理するとき） |
| `colima stop` | VM を止める。コンテナ・イメージ・ボリュームは残る |
| `colima stop --force` | 正常終了を待たずに止める。 `colima list` が `Broken` のときに使う |
| `colima restart` | 止めてから起動し直す（設定ファイルを書き換えたあとなど） |
| `colima delete` 危険 | VM を削除する。v0.9.0 以降、コンテナのデータは別ディスクに残り、次の `start` で戻る |
| `colima delete --data` 危険 | コンテナ・イメージ・ボリュームのデータも含めてすべて消す |

| コマンド | やること |
| --- | --- |
| `colima status` | 起動状態・ランタイム・ソケットのパスなどを表示（ `-e` で詳細、 `-j` で JSON） |
| `colima list` | 全プロファイルの状態・CPU・メモリ・ディスク・ランタイム・IP を一覧（ `ls` でも可） |
| `colima start work` | プロファイル `work` で別の VM を作って起動する（同時に複数動かせる） |
| `colima stop work` | プロファイル `work` を止める |
| `colima clone default work` | 既存のプロファイルを複製して新しいプロファイルを作る |
| `colima version` | Colima と、中で動くランタイムのバージョンを表示 |
| `docker context ls` | Docker コンテキストの一覧。 `*` が今使っているもの |
| `docker context use colima` | Colima の Docker に切り替える（Docker Desktop に戻すなら `desktop-linux` など） |

| コマンド | やること |
| --- | --- |
| `colima ssh` | VM にログインする |
| `colima ssh -- uname -a` | VM の中で 1 コマンドだけ実行する。 `--` で Colima のフラグと区切る |
| `colima ssh-config` | VM に接続するための SSH 設定を表示（エディタのリモート接続などに使う） |
| `colima ssh -- sudo fstrim -a` | VM の中で空いたディスク領域を Mac 側に返す（v0.5.0 以降は起動時にも自動で行う） |
| `colima start --network-address` | Mac から直接届く IP アドレスを VM に割り当てる（macOS のみ。既定はオフ） |
| `colima start --dns 1.1.1.1` | VM の DNS サーバーを指定する。VM からネットにつながらないときに試す |
| `colima start --mount ~/work:w` | マウントするディレクトリを指定（ `:w` で書き込み可）。指定すると既定の `$HOME` の代わりになる |

| コマンド | やること |
| --- | --- |
| containerd |  |
| `colima start --runtime containerd` | Docker の代わりに containerd で起動する |
| `colima nerdctl -- ps` | VM の中の nerdctl を実行する |
| `colima nerdctl install` | Mac の `$PATH` に `nerdctl` のラッパーを置き、 `nerdctl` だけで呼べるようにする |
| Kubernetes（k3s） |  |
| `colima start --kubernetes` | Kubernetes（k3s）も一緒に起動し、 `kubectl` のコンテキストを切り替える |
| `colima kubernetes stop` | Kubernetes だけ止める（ `start` ・ `delete` ・ `reset` もある） |
| Incus |  |
| `colima start --runtime incus` | Incus（コンテナと VM を扱う）で起動する。Mac の `incus` クライアントから使う |
| AI モデル（Apple Silicon・macOS 13 以降） |  |
| `colima start --vm-type krunkit` | GPU を使える krunkit の VM で起動する（krunkit を別に入れておく） |
| `colima model run gemma3` | AI モデルを取得して実行する（Docker ランタイムと krunkit が必要） |

| コマンド | やること |
| --- | --- |
| `colima start --edit` | そのプロファイルの `colima.yaml` を編集してから起動 |
| `colima template` | 新しく作る VM の既定値（テンプレート）を編集する |
| `colima template --print` | テンプレートのファイルのパスを表示するだけ |
| `colima update` | 中の Docker / containerd だけを最新にする（v0.7.6 以降） |
| `brew upgrade colima` | Colima 本体を更新する。新しい VM イメージを使うには `colima delete` → `colima start` |
| `colima prune` | ダウンロード済みのイメージなどのキャッシュを消す（ `-a` で Lima の分も） |
| `colima start --verbose` | 詳しいログを出して起動する。起動しない原因を調べるとき |

（元のページで操作できます → https://cs.taka1156.site/colima.html#commands）

## colima start のよく使うオプション

| オプション | 既定値 | 意味 |
| --- | --- | --- |
| `-c` `--cpus` | 2 | CPU 数 |
| `-m` `--memory` | 2 | メモリ（GiB） |
| `-d` `--disk` | 100 | コンテナ用ディスク（GiB）。あとから増やせるが減らせない |
| `-r` `--runtime` | docker | `docker` / `containerd` / `incus` |
| `-t` `--vm-type` | vz | `vz` （Apple の仮想化。macOS 13 以降の既定）/ `qemu` / `krunkit` 。作成後は変えられない |
| `--vz-rosetta` | オフ | Apple Silicon で amd64 のイメージを Rosetta 2 で速く動かす（vz のみ） |
| `-a` `--arch` | Mac と同じ | `aarch64` / `x86_64` 。違う方を選ぶと qemu のエミュレーションになり遅い |
| `--mount-type` | virtiofs（vz） | `virtiofs` / `9p` / `sshfs` 。qemu の既定は sshfs。作成後は変えられない |
| `-V` `--mount` | `$HOME` | マウントするディレクトリ。 `:w` で書き込み可、 `none` でマウントなし |
| `-k` `--kubernetes` | オフ | k3s を一緒に起動する |
| `--network-address` | オフ | Mac から届く IP を VM に割り当てる |
| `-n` `--dns` | なし | VM の DNS サーバー（複数回指定できる） |
| `--env` | なし | VM に渡す環境変数（ `--env KEY=value` ） |
| `--port-forwarder` | ssh | `ssh` （TCP のみ・安定）/ `grpc` （UDP も可・試験的）/ `none` |
| `-p` `--profile` | default | プロファイル名（ `colima start 名前` と同じ） |

CPU・メモリ・ディスクは、 `colima stop` してから新しい値で `colima start` すれば変えられる。VM の種類・マウント方式・アーキテクチャは作り直しが必要。

## 設定ファイル

フラグの代わりに YAML でも設定できる。 `colima start --edit` で開くのが早い。

| ファイル | 中身 |
| --- | --- |
| `~/.colima/default/colima.yaml` | プロファイル `default` の設定。別プロファイルは `~/.colima/名前/colima.yaml` |
| `~/.colima/_templates/default.yaml` | 新しく作る VM の既定値（ `colima template` で編集） |
| `~/.colima/default/docker.sock` | Docker のソケット。 `colima status` でも確かめられる |
| `~/.colima/_lima/_config/override.yaml` | Lima の設定を上書きする（上級者向け） |

置き場所は `$COLIMA_HOME` で変えられる。設定していなければ `~/.colima` が使われる（ `~/.colima` が無く `$XDG_CONFIG_HOME` が設定されているときは `$XDG_CONFIG_HOME/colima` 。Linux で両方とも無いときは `~/.config/colima` ）。

### colima.yaml の例

```
cpu: 4
memory: 8           # GiB
disk: 100           # GiB。増やすのは値を変えて再起動するだけ
runtime: docker
vmType: vz          # 作成後は変えられない
rosetta: true       # Apple Silicon で amd64 イメージを速く
mountType: virtiofs # 作成後は変えられない
mounts:             # 書くと既定の $HOME マウントの代わりになる
  - location: ~/work
    writable: true
  - location: /Volumes/data
    writable: true
network:
  dns: [1.1.1.1]
docker:             # Docker の daemon.json に入る内容
  insecure-registries:
    - myregistry.local:5000
env:
  TZ: Asia/Tokyo
```

## Docker Desktop との共存・ソケット

Colima は起動時に Docker コンテキスト `colima` を作って既定にするので、Docker Desktop を入れたままでも使える。どちらを使うかは `docker context use` で切り替える。

コンテキストを見ないツール（一部の IDE やライブラリ）が `Cannot connect to the Docker daemon at unix:///var/run/docker.sock` と言うときは、どちらかで Colima のソケットを教える。

```
# 1. 環境変数で指す（シェルの設定ファイルに書く）
export DOCKER_HOST="unix://$HOME/.colima/default/docker.sock"

# 2. 既定のパスにリンクを張る（ほかの Docker と衝突することがある）
sudo ln -sf $HOME/.colima/default/docker.sock /var/run/docker.sock
```

## よくある誤解

✗ 誤解: 起動中に colima start --cpus 4 を実行すれば CPU 数が変わる

✓ 正しくは: 起動中の `colima start` は「already running」と出して何もしない。先に `colima stop` してから新しい値で `colima start` する。

✗ 誤解: colima start --vm-type や --mount-type を付け直せば既存の VM も切り替わる

✓ 正しくは: VM の種類とマウント方式は作成時に決まり、あとから変えられない。 `colima delete` してから作り直す（v0.9.0 以降はコンテナのデータは残る）。

✗ 誤解: colima delete をするとイメージやボリュームもすべて消える

✓ 正しくは: v0.9.0 以降はコンテナのデータを別ディスクに置くので、 `colima delete` では消えず、次の `colima start` で戻る。すべて消したいときは `colima delete --data` 。

✗ 誤解: Mac のどのディレクトリでも docker run -v でマウントできる

✓ 正しくは: 既定で VM にマウントされるのは `$HOME` だけ。それ以外（ `/Volumes/…` など）を `-v` で指定するとエラーにならず空のディレクトリになる。 `colima.yaml` の `mounts` に足して `colima restart` する。

✗ 誤解: Colima を使うには Docker Desktop をアンインストールする必要がある

✓ 正しくは: Docker コンテキストで切り替えるので共存できる。Colima は起動時に自分のコンテキストを既定にする。

✗ 誤解: docker system prune で消せば Mac のディスクもすぐ空く

✓ 正しくは: VM の中で空いた領域が Mac 側に返るのは次の起動時（v0.5.0 以降）。すぐ返したいときは `colima ssh -- sudo fstrim -a` 。

## うまく動かないとき

1. ** `colima list` が Broken** Mac の再起動のあとなどに起きる。 `colima stop --force` で Stopped に戻してから `colima start` 。
2. **起動に失敗する** `colima start --verbose` でログを見る。Apple Silicon で x86_64 版の Homebrew（と Colima）を使っていないかも確かめる。
3. **更新したらおかしくなった** `colima start debug` のように別プロファイルで試し、そちらが動くなら `colima delete` → `colima start` で作り直す。
4. **VM からネットにつながらない** 多くは DNS。 `colima start --dns 8.8.8.8 --dns 1.1.1.1` を試し、 `colima ssh -- ping -c4 google.com` で確かめる。

## 出典

- [abiosoft/colima（README）](https://github.com/abiosoft/colima)
- [Colima FAQ](https://github.com/abiosoft/colima/blob/main/docs/FAQ.md)
- [colima.run/docs](https://colima.run/docs/)

v0.10.3 のソース（ `cmd/` ・ `embedded/defaults/colima.yaml` ）と照らして 2026-11-20 に確認。
