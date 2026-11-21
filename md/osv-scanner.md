<!-- scripts/memo.py build が自動生成。手で編集しない -->

# OSV-Scanner 使い方

> OSV-Scanner v2 のソース・イメージのスキャン、出力形式と終了コード、osv-scanner.toml、CI、fix をまとめたチートシート

- URL: https://cs.taka1156.site/osv-scanner.html
- タグ: セキュリティ, 開発ツール
- 日付: 2026-11-22

---

Google の OSS 脆弱性スキャナー。ロックファイル・SBOM・コンテナイメージの依存を OSV.dev のデータベースと照らし合わせる（v2 系・2026-11 時点）

**ざっくり言うと**

- OSV-Scanner は依存の一覧を取り出し、OSV.dev の脆弱性データと照合する無料の CLI（Apache-2.0）。
- ロックファイルの正確な版で照合し、Go などではコール解析で「呼んでいない脆弱な関数」も見分ける。
- 終了コード 1 は検出あり、128 は依存が 1 つも見つからない。CI では両者を分けて扱う。

## 動き方

**取り出す** OSV-SCALIBR がロックファイル・SBOM・イメージの層から、パッケージ名と版を集める

**照合する** OSV.dev（GitHub Advisory・PyPI・Go・RustSec・各 Linux ディストリ…）と突き合わせる

**絞る** コール解析・ `osv-scanner.toml` の除外を当てる

**出力する** 表・JSON・SARIF・HTML など。終了コードで結果を返す

OSV は「どの版からどの版までが影響を受けるか」を機械で読める形で書く共通形式。各エコシステムの一次情報を同じ形にそろえてあるので、版の範囲での照合が正確になる。

## インストール

バイナリ

Homebrew

go install

Docker

GitHub Releases のビルド済みバイナリは SLSA3 準拠で、来歴（provenance）ファイルが付く。 `slsa-verifier` で取得元を確かめてから使える。

```
brew install osv-scanner
```

```
go install github.com/google/osv-scanner/v2/cmd/osv-scanner@latest   # v2 はパスに /v2 が入る
```

```
docker run --rm -v "$PWD":/src ghcr.io/google/osv-scanner scan source -r /src
```

CI では `latest` ではなく版を固定したタグ（できればダイジェスト）を使う。

（元のページで操作できます → https://cs.taka1156.site/osv-scanner.html#install）

## コマンド早見表

ソース

イメージ

出力・その他

| コマンド | 意味 |
| --- | --- |
| `osv-scanner scan source -r .` | ディレクトリを再帰的にたどり、ロックファイルなどを見つけて照合（ `scan source` は省略でき、 `osv-scanner -r .` と書いても同じ。 `-r` がないとサブディレクトリに入らない） |
| `-L package-lock.json -L go.mod` | ファイルを直接指定（複数可）。SBOM（ `*.spdx.json` ・ `*.cdx.json` ）もこれで渡す |
| `-L 'requirements.txt:./deps/extra.txt'` | 決まった名前でないファイルは `形式:パス` で渡す |
| `--no-ignore` | `.gitignore` されたファイルも調べる（既定では除く） |
| `--include-git-root` | ルートの git リポジトリも対象にする（既定では飛ばす） |
| `--x-exclude=test` | ディレクトリを除く（実験的）。 `g:**/test/**` （glob）や `r:` （正規表現）も使える。旧名 `--experimental-exclude` は非推奨 |
| `--call-analysis=all` | 全言語でコール解析（Go は既定で有効、Rust・JAR は実験的）。止めるなら `--no-call-analysis=go` |
| `--licenses="MIT,Apache-2.0"` | 許可したライセンス以外を検出にする。値なしの `--licenses` は一覧だけ（deps.dev のデータ） |
| `--offline --download-offline-databases` | DB を手元に落として照合。以後はネットなしで動く |

| コマンド | 意味 |
| --- | --- |
| `osv-scanner scan image my-app:1.2.3` | ローカルになければ pull し、 `docker save` した中身を調べる。コンテナは起動しない |
| `osv-scanner scan image --archive ./app.tar` | 保存済みの Docker 形式の tar を調べる。Docker が入っていなくてよい |
| `--format html --serve` | 層・ベースイメージごとに絞り込める HTML レポートをローカルで開く（ポート 8000） |

層を意識して調べ、OS パッケージ（Alpine・Debian・Ubuntu など）と言語の依存（Go・Java・Node・Python など）を見る。ソースのディレクトリと同時には指定できない。

| コマンド | 意味 |
| --- | --- |
| `--format json --output-file result.json` | 形式を選んでファイルへ（旧名 `--output` は非推奨）。JSON のときは結果だけが標準出力に出る |
| `--all-packages` | 検出のないパッケージも含め、取り出した全部を JSON に出す（取りこぼしの確認に） |
| `--config ./osv.toml` | 各ディレクトリの `osv-scanner.toml` の代わりに、このファイルを全体に当てる |
| `osv-scanner fix ...` | ガイド付き修正（実験的）。下の「fix」を参照 |

（元のページで操作できます → https://cs.taka1156.site/osv-scanner.html#commands）

## 出力形式と終了コード

| `--format` | 使いどころ |
| --- | --- |
| `table` （既定）・ `vertical` | 端末で読む |
| `markdown` | PR のコメントや issue に貼る |
| `json` | スクリプトで処理する |
| `sarif` ・ `gh-annotations` | GitHub の Code Scanning、Actions の注釈 |
| `html` | ブラウザで絞り込みながら読む |
| `cyclonedx-1-4` 〜 `1-7` ・ `spdx-2-3` | SBOM として出す |

| 終了コード | 意味 | CI での扱い |
| --- | --- | --- |
| `0` | 依存は見つかり、検出なし | 通す |
| `1` | 脆弱性かライセンス違反を検出 | 止めて、依存の担当がトリアージ |
| `127` | 一般的なエラー | 止めて、スキャンの仕組みを直す |
| `128` | パッケージが 1 つも見つからない | 止める。対象のパスやロックファイルの有無を確かめる |

「1 以外なら成功」と書くと、128（何も調べていない）が緑になる。129〜255 は結果以外のエラー用に予約されている。

## osv-scanner.toml（除外の設定）

調べるファイルと同じディレクトリに置く。子ディレクトリには効かない（子にも置くか、 `--config` で 1 つを全体に当てる）。

```
# 1 件の脆弱性を期限付きで除外（別名の CVE / GHSA もまとめて外れる）
[[IgnoredVulns]]
id = "GO-2022-0968"
effectiveUntil = 2027-03-31
reason = "SSH を使わないサービス。2027Q1 の移行で解消。SEC-1842"

# 開発用の依存グループを丸ごと対象外に
[[PackageOverrides]]
group = "dev"
ignore = true

# 特定パッケージ・版だけ対象外に
[[PackageOverrides]]
name = "internal-example"
version = "1.2.3"
ecosystem = "Go"
ignore = true
```

`effectiveUntil` を過ぎると再び検出される（旧名 `ignoreUntil` は非推奨）。期限と理由を必ず書き、git の履歴で誰がいつ認めたか追えるようにする。

## CI（GitHub Actions）

公式の再利用ワークフローが 2 つある。PR 用は「その PR で新しく入った脆弱性」だけを報告し、定期用は全体を調べて SARIF を Security タブに上げる。

```
# .github/workflows/osv-scanner.yml
on:
  pull_request:
  schedule:
    - cron: "30 12 * * 1"   # 毎週。データは毎日増えるので、コードが変わらなくても回す
permissions:
  actions: read
  contents: read
  security-events: write    # SARIF のアップロードに必要
jobs:
  scan-pr:
    if: github.event_name == 'pull_request'
    uses: google/osv-scanner-action/.github/workflows/osv-scanner-reusable-pr.yml@<コミット SHA>  # v2.x
  scan-scheduled:
    if: github.event_name == 'schedule'
    uses: google/osv-scanner-action/.github/workflows/osv-scanner-reusable.yml@<コミット SHA>
    with:
      scan-args: |-
        -r
        ./
        --config=osv-scanner.toml
```

タグは書き換えられることがある（2026 年 3 月の Trivy の侵害はこの手口。 [Trivy 使い方](https://cs.taka1156.site/md/trivy.md) を参照）ので、外部の Action はコミット SHA で固定し、Dependabot などで更新する。

## fix（ガイド付き修正・実験的）

脆弱性を消すための版上げを、依存の深さ・深刻度・修正方法で選んで提案・適用する。対応は npm の `package-lock.json` （その場で書き換え）・ `package.json` （再ロック）と Maven の `pom.xml` （override）。

```
# 対話式（npm のみ）
osv-scanner fix --interactive -M package.json -L package-lock.json

# 非対話（既定。CI 向け）: 深さ 3 まで、CVSS 5 以上、dev 依存は除く
osv-scanner fix --max-depth=3 --min-severity=5 --ignore-dev --strategy=in-place -L package-lock.json
```

パッケージマネージャーのスクリプトを実行することがあるので、信頼できないプロジェクトでは使わない。

## Trivy との使い分け

**OSV-Scanner が向く** 言語の依存（ロックファイル）が中心のリポジトリ。コール解析で検出を絞りたい、PR で「新しく入った分」だけ見たい、npm / Maven の版上げを提案させたいとき。

**Trivy が向く** コンテナイメージ・Kubernetes・IaC（Terraform・Dockerfile）・シークレットまで 1 つの道具でまとめて見たいとき。詳しくは [Trivy 使い方](https://cs.taka1156.site/md/trivy.md) 。

データの出どころが違うので、同じ依存でも検出がずれることがある。重要なリポジトリでは両方を回して突き合わせるのも手。

## よくある誤解

✗ 誤解: package.json だけあれば正確に調べられる

✓ 正しくは: 範囲指定（ `^1.2.0` など）だけでは実際に入る版が決まらない。 `package-lock.json` などのロックファイルで、推移的な依存まで版を固定した状態を調べる。

✗ 誤解: 終了コードが 0 なら依存は安全だ

✓ 正しくは: 0 は「今の OSV.dev のデータで一致がない」だけ。対象のファイルを取りこぼしていれば何も調べていないし、データは毎日増える。 `--all-packages` で取り出した中身を確かめ、定期スキャンも回す。

✗ 誤解: コール解析で unexecuted になった脆弱性は無視してよい

✓ 正しくは: 解析は勧告に書かれた関数名と静的な呼び出しの経路に頼るので、リフレクションや動的な呼び出しは見落とす。優先度を下げる材料にとどめ、除外するなら理由と期限を書く。

✗ 誤解: scan image はコンテナを起動して中を調べる

✓ 正しくは: v2 の `scan image` はイメージを tar に書き出して静的に調べ、コンテナは起動しない（v1 の `--docker` は起動していた）。

## 出典

- [google/osv-scanner](https://github.com/google/osv-scanner) ・ [Releases](https://github.com/google/osv-scanner/releases)
- [Usage](https://google.github.io/osv-scanner/usage/) ・ [Source scanning](https://google.github.io/osv-scanner/usage/scan-source) ・ [Container image scanning](https://google.github.io/osv-scanner/usage/scan-image)
- [Configuration](https://google.github.io/osv-scanner/configuration/) ・ [Output](https://google.github.io/osv-scanner/output) ・ [GitHub Action](https://google.github.io/osv-scanner/github-action) ・ [v1→v2 移行ガイド](https://google.github.io/osv-scanner/migration-guide.html)

確認した日付: 2026-11-22
