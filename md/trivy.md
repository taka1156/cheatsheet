<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Trivy 使い方

> Trivy のターゲットとスキャナー、主要コマンド、CI 向けオプション、設定と除外、DB、2026 年 3 月の侵害を踏まえた安全な使い方をまとめたチートシート

- URL: https://cs.taka1156.site/trivy.html
- タグ: セキュリティ, 開発ツール
- 日付: 2026-11-22

---

Aqua Security の OSS セキュリティスキャナー。イメージ・ファイル・リポジトリ・Kubernetes から脆弱性・設定ミス・シークレット・ライセンスを探す（v0.76 系・2026-11 時点）

**ざっくり言うと**

- Trivy は「何を調べるか（ターゲット）」と「何を探すか（スキャナー）」を組み合わせて使う 1 本の CLI。
- 既定では検出があっても終了コード 0 なので、CI では `--exit-code 1` と `--severity` を付ける。
- 2026 年 3 月に配布物と GitHub Action が侵害されたので、版とコミット SHA を固定して使う。

## ターゲットとスキャナー

書き方は `trivy <ターゲット> [--scanners vuln,secret,...] <対象>` 。ターゲットごとに既定で動くスキャナーが違う。

| ターゲット | 対象 | 既定のスキャナー |
| --- | --- | --- |
| `image` | コンテナイメージ（レジストリ・ローカル・ `--input` で tar） | vuln・secret |
| `fs` | ローカルのディレクトリ・ファイル | vuln・secret |
| `repo` | リモートの git リポジトリ | vuln・secret |
| `rootfs` | 展開済みのルートファイルシステム（VM やコンテナの中） | vuln・secret |
| `config` | IaC（Terraform・CloudFormation・Dockerfile・Kubernetes・Helm） | misconfig |
| `sbom` | CycloneDX・SPDX の SBOM | vuln |
| `k8s` | 稼働中の Kubernetes クラスター | vuln・misconfig・secret・rbac |
| `vm` | 仮想マシンのイメージ（実験的） | vuln・secret |

スキャナーは `vuln` （既知の脆弱性）・ `misconfig` （設定ミス）・ `secret` （埋め込まれた秘密情報）・ `license` （ライセンス）の 4 つ（k8s だけ `rbac` もある）。 `license` はどのターゲットでも既定で無効。

## コマンド早見表

image

fs・repo

config（IaC）

SBOM・k8s

```
trivy image nginx:1.27                         # レジストリのイメージ
trivy image --input app.tar                    # docker save した tar
trivy image --scanners vuln nginx:1.27         # シークレット探しを止めて速くする
trivy image --pkg-types os nginx:1.27          # OS パッケージだけ（library で言語の依存だけ）
trivy image --image-config-scanners secret,misconfig app:1.0   # 環境変数などイメージ設定も見る
```

```
trivy fs .                                     # ロックファイルの脆弱性とシークレット
trivy fs --scanners vuln,secret,misconfig .    # IaC の設定ミスも
trivy fs --scanners license .                  # ライセンス
trivy fs --include-dev-deps .                  # 開発用の依存も（既定では除く言語がある）
trivy repo https://github.com/org/app          # リモートのリポジトリ
```

```
trivy config ./infra                                        # Terraform・Dockerfile・k8s YAML など
trivy config --misconfig-scanners=terraform,dockerfile .    # 種類を絞る
trivy config --helm-values values.prod.yaml ./chart         # Helm の値を差し替えて評価
trivy config --config-check ./policy.rego --check-namespaces user .   # 自作の Rego チェック
```

特定の 1 リソースだけ外すなら、定義のすぐ上に `# trivy:ignore:<ID>` のコメントを書く（ `.trivyignore` はディレクトリ全体に効く）。

```
trivy image --format cyclonedx --output app.cdx.json app:1.0   # SBOM を作る（spdx-json も可）
trivy sbom app.cdx.json                                        # SBOM を脆弱性スキャン
trivy k8s --report summary                                     # 今のコンテキストのクラスターを要約
trivy k8s --include-namespaces prod --report all               # 名前空間を絞って詳細
```

（元のページで操作できます → https://cs.taka1156.site/trivy.html#commands）

## 絞り込みと CI 向けのオプション

| オプション | 意味 |
| --- | --- |
| `--severity CRITICAL,HIGH` | 表示・判定する深刻度（UNKNOWN・LOW・MEDIUM・HIGH・CRITICAL） |
| `--exit-code 1` | 検出があったときの終了コード。 既定は 0 （検出があっても成功扱い） |
| `--ignore-unfixed` | 修正版のないものを隠す（ `--ignore-status affected,will_not_fix,fix_deferred,end_of_life` の省略形） |
| `--exit-on-eol 1` | サポート切れ（EOL）の OS なら失敗にする |
| `--skip-dirs` ・ `--skip-files` | glob で除外（テスト用のデータなど） |
| `--format` ・ `--output` | `table` ・ `json` ・ `sarif` ・ `template` ・ `cyclonedx` ・ `spdx-json` ・ `github` （依存スナップショット） |
| `--dependency-tree` | 脆弱な依存がどの直接依存から来たかを木で表示（table のみ） |
| `--vex ./vex.json` | VEX（「影響しない」などの判定）を読んで検出から外す |

```
# CI の定番: 修正できる HIGH 以上が 1 件でもあれば失敗
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 app:1.0
```

## 設定ファイルと除外

trivy.yaml

.trivyignore.yaml

.trivyignore（旧形式）

カレントの `trivy.yaml` を自動で読む（ `-c` で別ファイル）。フラグと同じ名前のキーで書ける。 `trivy --generate-default-config` で全キーの雛形が出る。

```
severity:
  - HIGH
  - CRITICAL
exit-code: 1
format: table
ignorefile: .trivyignore.yaml
vulnerability:
  ignore-unfixed: true
scan:
  skip-dirs:
    - testdata
```

拡張子が `.yaml` / `.yml` なら構造化した形式として読む。パス・パッケージ・期限・理由を付けられる。

```
vulnerabilities:
  - id: CVE-2023-3817
    purls:
      - "pkg:deb/debian/libssl1.1"
    statement: 該当の関数を使っていない
    expired_at: 2027-03-31
misconfigurations:
  - id: AVD-DS-0002
    paths:
      - "docs/Dockerfile"
    statement: ドキュメント用で本番に出さない
secrets:
  - id: aws-access-key-id
    paths:
      - "testdata/fake.env"
```

使うときは `--ignorefile .trivyignore.yaml` （または trivy.yaml の `ignorefile` ）で指定する。

```
# 1 行 1 ID。exp: で期限を付けられる
CVE-2023-29491 exp:2027-03-31
AVD-DS-0002
```

既定で読まれるのはこの形式の `.trivyignore` 。理由やパスを残せないので、新しく書くなら YAML 形式にする。

（元のページで操作できます → https://cs.taka1156.site/trivy.html#config）

## データベースとキャッシュ

| やりたいこと | 方法 |
| --- | --- |
| DB の取得元 | 初回とその後定期的に、脆弱性 DB を OCI レジストリから取る（既定は `mirror.gcr.io/aquasec/trivy-db` が先で、だめなら `ghcr.io/aquasecurity/trivy-db` ）。社内ミラーは `--db-repository` |
| CI で毎回取りに行かない | `--cache-dir` をキャッシュし、同じジョブ内の 2 回目以降は `--skip-db-update` |
| ネットのない環境 | DB を持ち込み、 `--skip-db-update --offline-scan` |
| 複数のマシンで DB を共有 | `trivy server --listen 0.0.0.0:4954` を立て（既定は localhost のみ）、各マシンは `--server http://host:4954` で問い合わせる。 `--token` で認証を付ける |
| キャッシュを消す | `trivy clean --all` |

## 2026 年 3 月の侵害と、安全な使い方

認証情報を盗まれ、配布物と公式 Action に認証情報を盗むマルウェアが仕込まれた（ `CVE-2026-33634` / `GHSA-69fq-xp46-6x23` ）。2 月末からの攻撃の続き（Aqua などは TeamPCP というグループによるものとしている）。

| 侵害されたもの | 期間（UTC） |
| --- | --- |
| Trivy v0.69.4（GitHub・コンテナ・deb・rpm。その間の `latest` も） | 2026-03-19 18:22 から約 3 時間 |
| `aquasecurity/trivy-action` のタグ 77 個中 76 個（0.0.1〜0.34.2） | 03-19 17:43 頃 から約 12 時間 |
| `aquasecurity/setup-trivy` の全タグ | 03-19 17:43 頃 から約 4 時間 |
| Docker Hub の v0.69.5・v0.69.6 | 03-22 15:43 から約 10 時間 |

1. **Action はコミット SHA で固定する** タグは書き換えられる。今回もタグを参照していた所が被害に遭った。ただし 2025-04-09 より前の trivy-action のコミットは中で setup-trivy をタグで呼んでいたため、SHA で固定していても影響を受けた可能性がある。新しい安全なコミットの SHA で固定する。
2. **Trivy の版も固定する** `version: latest` を使わない。action には版を明示し、イメージはダイジェストで指定する。
3. **署名を確かめる** v0.69.3 からは GitHub の immutable releases が有効で、リリースを後から差し替えられない。バイナリは sigstore の署名で検証できる。
4. **当時使っていたなら調べる** 3 月 19〜20 日（イメージは 22〜23 日）のワークフローのログを見直し、組織に `tpcp-docs` というリポジトリができていないか確かめ、CI の秘密情報をすべて入れ替える。

trivy-action は乗っ取られた 0.0.1〜0.34.2 のタグを削除し、正規のコミットを指す `v` 付きのタグ（例: `v0.34.0` ）として付け直した（v0.0.10・v0.34.1・v0.34.2 は未復元）。以後の新しい版も `v` 付きで出している。

## CI（GitHub Actions）

```
permissions:
  contents: read
  security-events: write      # SARIF のアップロードに必要
jobs:
  trivy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<コミット SHA>
      - name: Build
        run: docker build -t app:${{ github.sha }} .
      - name: Scan
        uses: aquasecurity/trivy-action@<コミット SHA>   # v0.36.0
        with:
          version: v0.76.2            # latest にしない
          image-ref: app:${{ github.sha }}
          severity: CRITICAL,HIGH
          ignore-unfixed: true
          exit-code: "1"
          format: sarif
          output: trivy-results.sarif
          limit-severities-for-sarif: true   # これがないと SARIF では severity が無視され、全件で判定される
          trivyignores: .trivyignore.yaml
      - name: Upload
        if: always()                  # 失敗しても結果は上げる
        uses: github/codeql-action/upload-sarif@<コミット SHA>
        with:
          sarif_file: trivy-results.sarif
```

## OSV-Scanner との使い分け

**Trivy が向く** コンテナイメージ・Kubernetes・IaC・シークレットまで 1 つの道具で見たいとき。OS パッケージの判定がディストリごとに細かい。

**OSV-Scanner が向く** 言語の依存が中心で、コール解析で検出を絞りたい、PR で新しく入った分だけ見たいとき。詳しくは [OSV-Scanner 使い方](https://cs.taka1156.site/md/osv-scanner.md) 。

## よくある誤解

✗ 誤解: Trivy が何か見つければ CI は失敗する

✓ 正しくは: 既定の終了コードは検出があっても 0。 `--exit-code 1` （trivy.yaml なら `exit-code: 1` ）を付けないと、ログに赤字が出ても CI は緑のまま。

✗ 誤解: 古いイメージで 0 件なら問題ない

✓ 正しくは: `--ignore-unfixed` で修正版のないものを全部隠していたり、サポート切れの OS でデータがなかったりすると 0 件になる。 `--exit-on-eol` も併用する。

✗ 誤解: trivy-action@v0.x のようにタグで書けば固定したことになる

✓ 正しくは: タグは持ち主（や乗っ取った攻撃者）が別のコミットに付け替えられる。2026 年 3 月はまさにこの手口だった。固定はコミット SHA で行う。

✗ 誤解: fs スキャンは開発用の依存も含めて調べている

✓ 正しくは: npm などでは開発用の依存は既定で除かれる。ビルドに使う道具の脆弱性も見たいなら `--include-dev-deps` を付ける。

## 出典

- [aquasecurity/trivy](https://github.com/aquasecurity/trivy) ・ [Trivy ドキュメント](https://trivy.dev/) （ [設定ファイル](https://trivy.dev/docs/dev/references/configuration/config-file/) ・ [その他の設定](https://trivy.dev/docs/dev/configuration/others/) ）
- [aquasecurity/trivy-action](https://github.com/aquasecurity/trivy-action)
- [GHSA-69fq-xp46-6x23（侵害の公式アドバイザリ）](https://github.com/aquasecurity/trivy/security/advisories/GHSA-69fq-xp46-6x23) ・ [Aqua のブログ](https://www.aquasec.com/blog/trivy-supply-chain-attack/)

確認した日付: 2026-11-22
