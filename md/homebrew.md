<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Homebrew 使い方

> macOS・Linux のパッケージマネージャー Homebrew（brew コマンド）の仕組み・用語、用途別コマンド、Brewfile、環境変数、5.0〜7.0 の変更点をまとめたチートシート。

- URL: https://cs.taka1156.site/homebrew.html
- タグ: 開発ツール
- 日付: 2026-11-23

---

macOS・Linux のパッケージマネージャー `brew` のチートシート（Homebrew 7.0 系、7.0.15 時点）

**ざっくり言うと**

- Homebrew は CLI ツール（formula）と Mac アプリ（cask）を `brew` コマンド 1 つで入れて更新するパッケージマネージャー。
- ビルド済みの bottle を Cellar に展開し、prefix の `bin` などへシンボリックリンクを張るので `sudo` は要らない。
- 6.0 から公式以外の tap は `brew trust` で信頼するまで読み込まれず、7.0 で `brew vulns` が加わった。

## 仕組みと用語

パッケージ定義（Ruby のファイル）は tap というリポジトリにまとまっている。公式の `homebrew/core` （formula）と `homebrew/cask` （cask）は、普段は Git で取り込まずに JSON の API から読む。

**tap** formula・cask・外部コマンドの定義を置くリポジトリ homebrew/core, homebrew/cask, user/repo

**bottle** ビルド済みのバイナリ。無い環境ではソースからビルドする

**Cellar** バージョンごとのフォルダ（keg）に展開する $(brew --prefix)/Cellar/wget/1.25.0

**prefix** keg の中身をリンクして PATH から使えるようにする $(brew --prefix)/bin/wget

**formula** ソースからビルドするパッケージの定義。CLI ツールやライブラリ（例: `wget` ）。

**cask** ビルド済みの macOS アプリなどを入れる定義（例: `firefox` ）。 `Caskroom` に置かれる。

**keg** 1 つの formula の 1 バージョンを入れたフォルダ。 `opt/` には今のバージョンへのリンクがある。

### prefix（インストール先）の既定

| 環境 | prefix | 補足 |
| --- | --- | --- |
| macOS Apple Silicon | `/opt/homebrew` | macOS 15 Sequoia 以降で Tier 1（公式サポート） |
| macOS Intel | `/usr/local` | 7.0 で Tier 3（サポート外・新しい bottle なし）。2027 年 9 月に動かなくなる予定 |
| Linux WSL | `/home/linuxbrew/.linuxbrew` | インストール後は sudo なしで使える既定の場所 |

## はじめかた

1. **インストール** 下の公式スクリプトを実行する（Apple Silicon の Mac なら `.pkg` のインストーラーもある）。
2. **PATH を通す** インストーラーが最後に表示する行（下の 2 行目のような形）を `~/.zprofile` （Linux なら `~/.bashrc` など）に足す。
3. **確かめる** `brew doctor` で問題がないかを見る。警告が出ても困っていなければ無視してよい、と公式に書かれている。
4. **入れる** `brew install wget` （CLI）、 `brew install --cask firefox` （アプリ）。名前が formula と重ならなければ `--cask` は省ける。

```
# 1. インストール
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
# 2. PATH を通す（Apple Silicon の Mac の例）
eval "$(/opt/homebrew/bin/brew shellenv zsh)"
```

## コマンド チートシート

危険 は、元に戻しにくい操作（データや設定まで消す、変更を捨てる）。

探す・調べる

入れる・消す

更新

依存・お掃除

tap・trust

services

診断・場所

| コマンド | やること |
| --- | --- |
| `brew search jq` | formula 名と cask 名を部分一致で探す。 `/^jq$/` のように `/` で囲むと正規表現 |
| `brew search --desc json` | 説明文（cask は名前も）から探す |
| `brew info jq` | バージョン・依存・インストール状況・注意書き（caveats）を表示 |
| `brew info --json=v2 --installed` | 入っている formula と cask を JSON で出す（スクリプト向け） |
| `brew desc jq` | 1 行の説明だけを表示 |
| `brew home jq` | 公式サイトをブラウザで開く |
| `brew list` | 入っている formula と cask の一覧（ `ls` でも可）。 `--formula` ・ `--cask` で絞る |
| `brew list --versions` | 入っている formula をバージョンつきで表示 |
| `brew list jq` | その formula が入れたファイルの場所を表示 |
| `brew leaves -r` | 自分で入れた（依存ではない）formula だけを一覧 |
| `brew which-formula rg` | そのコマンドを提供する formula を調べる |

| コマンド | やること |
| --- | --- |
| `brew install jq` | formula を入れる。古い版が入っていれば更新する。依存などほかのパッケージも入るときは計画を表示して確認を求める |
| `brew install -y jq` | 確認なしで入れる（ `--no-ask` ・ `--yes` と同じ） |
| `brew install --dry-run jq` | 何が入るかを表示するだけ（ `-n` ） |
| `brew install --cask firefox` | cask（アプリ）を入れる |
| `brew install -s jq` | bottle を使わずソースからビルドする（ `--build-from-source` ） |
| `brew install --HEAD foo` | 開発中の最新版（HEAD）をビルドして入れる（formula が対応している場合） |
| `brew install python@3.12` | `@` つきの名前で、Homebrew が用意している特定のメジャー版を入れる |
| `brew version-install automake@1.12` | 過去のバージョンを自分用の tap に取り出して入れる（5.1 で追加）。以後の更新は自己責任 |
| `brew reinstall jq` | 入れ直す |
| `brew uninstall jq` | 削除する（ `remove` ・ `rm` でも可）。依存で入ったものは残る |
| `brew uninstall --zap --cask foo` 危険 | アプリの設定・キャッシュなど関連ファイルもまとめて消す。ほかのアプリと共有するファイルも消えることがある |
| `brew link jq` / `brew unlink jq` | prefix へのシンボリックリンクを張る／外す（keg はそのまま） |

| コマンド | やること |
| --- | --- |
| `brew update` | Homebrew 本体とパッケージ定義を最新にする（入っているパッケージは更新しない） |
| `brew outdated` | 新しい版がある formula と cask を一覧 |
| `brew upgrade` | 古くなったパッケージをすべて更新する（pin したものは除く） |
| `brew upgrade jq` | 指定したものだけ更新する |
| `brew upgrade --greedy` | 自動更新するアプリ（ `auto_updates true` ）や `version :latest` の cask も更新対象に入れる（ `-g` ） |
| `brew pin node` | バージョンを固定し、 `brew upgrade` で上げないようにする |
| `brew unpin node` | 固定を外す |
| `brew list --pinned` | 固定しているパッケージの一覧 |
| `brew update-reset` 危険 | Homebrew と tap の Git リポジトリを `origin/HEAD` に戻す。手元の変更は消える |

| コマンド | やること |
| --- | --- |
| `brew deps --tree jq` | 依存をツリーで表示 |
| `brew deps --installed` | 入っている formula ごとの依存を一覧 |
| `brew uses --installed openssl@3` | その formula に依存している、入っている formula を表示（消してよいかの確認に） |
| `brew autoremove` | 依存として入り、もう誰にも使われていない formula を消す。 `-n` で一覧だけ |
| `brew cleanup` | 古いバージョンの keg と古いダウンロードを消す |
| `brew cleanup -n` | 何が消えるかを表示するだけ |
| `brew cleanup --prune=all` | キャッシュのファイルを日数に関係なく消す |
| `brew vulns` | 入っている formula の既知の脆弱性を OSV.dev で調べる（7.0 で追加）。cask は対象外 |
| `brew vulns --severity=high --deps` | high 以上だけを、依存も含めて表示 |
| `brew vulns --fix-available` | 修正版が出ているものだけを表示 |

| コマンド | やること |
| --- | --- |
| `brew tap` | 取り込んでいる tap の一覧 |
| `brew tap user/repo` | `https://github.com/user/homebrew-repo` を tap として取り込む |
| `brew tap user/repo URL` | GitHub 以外（SSH など）の URL から取り込む |
| `brew install user/repo/foo` | tap 名つきの完全な名前で入れる。その formula だけを信頼したことになる |
| `brew trust --formula user/repo/foo` | 公式以外の tap の formula を 1 つだけ信頼する（ `--cask` ・ `--command` も同様） |
| `brew trust user/repo` | tap 全体を信頼する。今後追加される定義もすべて信頼することになる |
| `brew trust` | 信頼している項目の一覧 |
| `brew untrust user/repo` | 信頼を取り消す |
| `brew untap user/repo` | tap を外す（7.0 から、その tap のパッケージを先に消すかを聞く） |

| コマンド | やること |
| --- | --- |
| `brew services` | 管理しているサービスの一覧（ `list` と同じ） |
| `brew services start postgresql@17` | すぐ起動し、ログイン時に自動起動するよう登録する |
| `brew services run postgresql@17` | 起動するが、自動起動には登録しない |
| `brew services stop postgresql@17` | 止めて、自動起動の登録も外す（ `--keep` で登録は残す） |
| `brew services restart postgresql@17` | 止めてから起動し直す（登録もする） |
| `brew services info postgresql@17` | そのサービスの状態を表示 |
| `sudo brew services start foo` | ログイン時ではなく起動時に root で動かす。prefix の持ち主を信頼できるときだけ使う |

| コマンド | やること |
| --- | --- |
| `brew doctor` | 環境の問題を調べる（ `dr` でも可）。7.0 で `--json` 出力が加わった |
| `brew config` | バージョン・prefix・OS などの設定を表示（問題を報告するときに貼る） |
| `brew --version` | Homebrew のバージョン |
| `brew --prefix` | prefix の場所。 `brew --prefix jq` でその formula の場所（ `opt/jq` ） |
| `brew --cellar` / `brew --caskroom` | Cellar・Caskroom の場所 |
| `brew --cache` | ダウンロードのキャッシュの場所 |
| `brew shellenv` | PATH などを設定する `export` 文を出力する |
| `brew commands` | 使えるコマンドの一覧 |
| `brew help install` | コマンドの使い方を表示（ `brew install --help` と同じ） |

該当するコマンドがありません。

（元のページで操作できます → https://cs.taka1156.site/homebrew.html#commands）

## Brewfile（brew bundle）

入れるものを `Brewfile` に並べ、 `brew bundle` でまとめて入れる。新しい Mac の準備や、チームで同じ道具をそろえるときに使う。formula・cask・tap のほか、Mac App Store（ `mas` ）、VS Code 拡張、Go・Cargo・npm・uv のツールなども書ける。

```
# Brewfile
# 公式以外の tap は trusted: true で信頼を宣言できる
tap "user/repo", trusted: true
brew "jq"
brew "postgresql@17", restart_service: true
cask "firefox"
mas "Xcode", id: 497799835
vscode "editorconfig.editorconfig"
# Ruby なので条件も書ける
brew "glibc" if OS.linux?
```

| コマンド | やること |
| --- | --- |
| `brew bundle` | 今のフォルダの `Brewfile` にあるものを入れる（ `brew bundle install` と同じ。入っていて古ければ更新する） |
| `brew bundle dump` | 今入っているものを `Brewfile` に書き出す。既にあるファイルを上書きするなら `--force` |
| `brew bundle check` | `Brewfile` のものが全部入っているかを確かめる（足りなければ終了コードが 0 以外） |
| `brew bundle cleanup` | `Brewfile` に無いものを消す。既定は消す前に確認し、 `--force` で確認なしに消す |
| `brew bundle list` | `Brewfile` の中身を一覧（既定は formula だけ） |
| `brew bundle add jq` | `Brewfile` に項目を足す（ `--cask` などで種類を指定） |
| `brew bundle --file=path/Brewfile` | 別の場所の `Brewfile` を使う |
| `brew bundle --global` | `~/.Brewfile` など、ユーザー共通の `Brewfile` を使う |

## よく使う環境変数

| 変数 | 効果 |
| --- | --- |
| `HOMEBREW_NO_AUTO_UPDATE=1` | `install` ・ `upgrade` ・ `tap` の前の自動 `brew update` をしない |
| `HOMEBREW_AUTO_UPDATE_SECS` | 自動 update の間隔（秒）。既定は 86400（24 時間）。止めるより間隔を延ばすのが推奨 |
| `HOMEBREW_NO_INSTALL_CLEANUP=1` | `install` ・ `upgrade` のあとの自動 `cleanup` をしない |
| `HOMEBREW_NO_ASK=1` | インストール前の確認（ask モード）をしない |
| `HOMEBREW_UPGRADE_GREEDY=1` | `brew upgrade` を常に `--greedy` で動かす |
| `HOMEBREW_NO_ANALYTICS=1` | 利用統計を送らない（ `brew analytics off` でも止められる） |
| `HOMEBREW_NO_ENV_HINTS=1` | 環境変数についてのヒントを表示しない |

## 最近の主な変更（5.0〜7.0）

| 版（公開日） | 変わったこと |
| --- | --- |
| 5.0.0 2025-11-12 | ダウンロードの並列化が既定に。 `brew mcp-server` （4.6.0 で内蔵）で使える開発用コマンドが増えた。cask の `--no-quarantine` を非推奨に |
| 5.1.0 2026-03-10 | `brew version-install` を追加。 `brew bundle` の対応を拡大 |
| 6.0.0 2026-06-11 | tap の信頼（tap trust）を導入し、公式以外の tap は信頼するまで読み込まないように。インストール前に計画を出して確認する ask モードを既定に（当初は開発者向け） |
| 7.0.0 2026-09-13 | `brew vulns` を追加。公式 GUI アプリ BrewUI（ `brew install homebrew-app` 、macOS 26 以降）。Intel Mac と macOS 14 を Tier 3 に、macOS 10.15 以前のサポートを終了 |

## よくある誤解

✗ 誤解: 権限エラーが出たら sudo brew で実行すればよい

✓ 正しくは: `brew` は prefix の持ち主のユーザーで動かす前提で、 `sudo` を付けると prefix の持ち主が書き換えたコードを root で動かすことになる。権限エラーは prefix の持ち主を直す。例外は、起動時に root で動かしたいサービス（ `sudo brew services` ）など、ドキュメントが `sudo` を示している場合だけ。

✗ 誤解: brew update で入っているパッケージも新しくなる

✓ 正しくは: `brew update` は Homebrew 本体とパッケージ定義を更新するだけ。パッケージを新しくするのは `brew upgrade` （ `brew outdated` で対象を確認できる）。

✗ 誤解: brew upgrade ですべての cask が更新される

✓ 正しくは: 自分で更新するアプリ（ `auto_updates true` ）と `version :latest` の cask は既定で飛ばされる。まとめて上げるなら `brew upgrade --greedy` 。

✗ 誤解: brew uninstall で依存パッケージも一緒に消える

✓ 正しくは: 消えるのは指定したものだけ。使われなくなった依存は `brew autoremove` で消す（ `-n` で事前に確認できる）。

✗ 誤解: brew tap した tap の formula はそのまま入れられる

✓ 正しくは: 6.0 からは公式以外の tap は信頼が要る。 `brew install user/repo/foo` と完全な名前で入れるか、 `brew trust` で信頼してから短い名前で入れる。

## 出典

- [brew(1) – Manpage](https://docs.brew.sh/Manpage)
- [Installation](https://docs.brew.sh/Installation)
- [Homebrew Bundle and Brewfile](https://docs.brew.sh/Brew-Bundle-and-Brewfile)
- [Tap Trust](https://docs.brew.sh/Tap-Trust)
- [Versions](https://docs.brew.sh/Versions)
- [Homebrew 7.0.0（ブログ）](https://brew.sh/2026/09/13/homebrew-7.0.0/) ・ [6.0.0](https://brew.sh/2026/06/11/homebrew-6.0.0/) ・ [5.1.0](https://brew.sh/2026/03/10/homebrew-5.1.0/) ・ [5.0.0](https://brew.sh/2025/11/12/homebrew-5.0.0/) ・ [4.6.0](https://brew.sh/2025/08/05/homebrew-4.6.0/)

Homebrew/brew 7.0.15 の `docs/` とソース（ `Library/Homebrew/cmd/` ）と照らして 2026-11-23 に確認。
