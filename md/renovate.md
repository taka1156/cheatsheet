<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Renovate 使い方

> 依存関係の更新 PR を自動で作る Renovate の導入方法・renovate.json の書き方・packageRules・automerge・スケジュールをまとめたチートシート

- URL: https://cs.taka1156.site/renovate.html
- タグ: 開発ツール, セキュリティ
- 日付: 2026-11-22

---

依存関係の更新 PR を自動で作るボット。導入・renovate.json・packageRules・automerge の早見表（v44 時点）

**ざっくり言うと**

- リポジトリの依存（npm・Go・Docker・GitHub Actions など）の新しい版を見つけ、更新 PR を自動で作るボット。
- `renovate.json` の `extends` でプリセットを読み、 `packageRules` で依存ごとに振る舞いを変える。
- まず `config:recommended` から始め、PR が多すぎたらグループ化・スケジュール・automerge で減らす。

## 動く仕組み

Renovate は実行されるたびに（App なら定期的に）リポジトリを読み、次の流れで PR を作る。常駐してファイルを監視するのではなく、毎回はじめから調べ直す。

**設定を読む** `renovate.json` と `extends` のプリセットを合わせる

**依存を見つける** manager（ `npm` ・ `gomod` ・ `dockerfile` …）が各ファイルから依存と今の版を取り出す

**新しい版を調べる** datasource（npm レジストリ・Docker Hub・GitHub Releases…）に問い合わせる

**振り分ける** `packageRules` でグループ化・無効化・automerge などを決める

**ブランチと PR** `renovate/…` ブランチを作って PR を出す。条件を満たせば automerge

Dependency Dashboard（Issue 1 つ）に、保留中・作成待ち・エラーの更新がまとまる。チェックボックスで「今すぐ PR を作る」「作り直す」を指示できる。

## 導入

Mend Renovate App

GitHub Action

CLI・Docker

GitHub.com ならいちばん手軽。Mend がホストするボットが動くので、トークンの管理も実行環境も要らない。

1. **App を入れる** `github.com/apps/renovate` から Install し、対象を「All repositories」か「Select repositories」で選ぶ
2. **Onboarding PR を確認する** 「Configure Renovate」という PR が来る。中身は `renovate.json` （既定は `config:recommended` ）と、これから来る PR の予告
3. **マージする** マージした時点から更新 PR が作られ始める。マージ前にこの PR のブランチで設定を直すと、予告の内容も作り直される

自分の CI でボットを動かす（セルフホスト）。社内レジストリに届く・実行タイミングを自分で決められるのが利点。

```
# .github/workflows/renovate.yml
on:
  schedule:
    - cron: '0 18 * * *'      # 毎日 3:00 JST
  workflow_dispatch:
jobs:
  renovate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/create-github-app-token@v3   # GitHub App で動かす場合
        id: app-token
        with:
          client-id: ${{ vars.RENOVATE_APP_CLIENT_ID }}
          private-key: ${{ secrets.RENOVATE_APP_KEY }}
      - uses: renovatebot/github-action@v46.3.14   # 版を固定する（Renovate に更新させる）
        with:
          token: ${{ steps.app-token.outputs.token }}
        env:
          RENOVATE_REPOSITORIES: 'owner/repo'
          LOG_LEVEL: debug
```

個人アクセストークン（PAT）でも動くが、PR の作成者がその人になる。GitHub App にするとボット名（ `xxx[bot]` ）で PR が作られる。

GitLab・Gitea/Forgejo などや自社ホストの環境では、CLI か Docker イメージで動かす（Bitbucket Cloud には Mend の App もある）。

```
# npm で
npx renovate --platform=gitlab --token=$TOKEN group/project

# Docker で（公式イメージ。必要なツールは実行時に入る。ツール入りは 44-full）
docker run --rm -e RENOVATE_TOKEN -e RENOVATE_PLATFORM=gitlab \
  ghcr.io/renovatebot/renovate:44 group/project
```

ボット側の設定（ `platform` ・ `token` ・ `repositories` ・ `onboardingConfig` など）は「セルフホスト用の設定」で、 `config.js` ・環境変数（ `RENOVATE_*` ）・CLI 引数に書く。リポジトリの `renovate.json` に書いても無視される。

（元のページで操作できます → https://cs.taka1156.site/renovate.html#install）

## 設定ファイル

置き場所は次のどれか（上から探して最初に見つかったもの 1 つだけを使う）。

| 場所 | メモ |
| --- | --- |
| `renovate.json` / `renovate.jsonc` / `renovate.json5` | いちばん普通。コメントを書くなら `.jsonc` が推奨（ `.json` も JSONC として読まれる） |
| `.github/renovate.json(5)` ・ `.gitlab/renovate.json(5)` | ルートを散らかしたくないとき |
| `.renovaterc` ・ `.renovaterc.json(5)` | dotfile 派向け |
| `package.json` の `"renovate"` | 非推奨。使わない |

```
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "timezone": "Asia/Tokyo",
  "schedule": ["* 0-6 * * 1"],
  "labels": ["dependencies"]
}
```

`$schema` を書くとエディタで補完と検証が効く。

### プリセット（extends）

| 書き方 | 意味 |
| --- | --- |
| `config:recommended` | 標準の推奨セット。Dependency Dashboard・monorepo のグループ化（ `group:monorepos` ・ `group:recommended` ）・ `node_modules` とテスト用フォルダの除外・Age/Confidence バッジ・置き換え／回避策ルールなど |
| `config:best-practices` | recommended ＋ Docker と GitHub Actions のダイジェスト固定・devDependencies の固定・設定の自動移行・npm は公開 3 日後から提案・週 1 のロックファイル更新・放棄パッケージの検出 |
| `:dependencyDashboard` | 先頭が `:` は `default:` の略 |
| `schedule:weekly` ・ `schedule:nonOfficeHours` | スケジュールの組み込みプリセット |
| `github>owner/repo` | 別リポジトリの `default.json` を読む（組織共通の設定）。 `github>owner/repo:name` なら `name.json` |
| `local>owner/repo` | 同じプラットフォーム上のリポジトリから読む（GitLab・セルフホストでも使える） |

プリセットの一部だけ外したいときは `"ignorePresets": [":ignoreModulesAndTests"]` 。

## よく使う設定

PR の量

automerge

版の扱い

対象

セキュリティ

| キー | 既定 | 意味 |
| --- | --- | --- |
| `schedule` | `["at any time"]` | ブランチを作ってよい時間帯。cron の 5 項目で書く（分は `*` にする）。 `"before 4am on monday"` のような英文の書き方（later）は非推奨 |
| `timezone` | なし（UTC） | `schedule` を解釈するタイムゾーン（ `Asia/Tokyo` など） |
| `prHourlyLimit` | `2` | 1 時間に新しく作る PR の数。 `0` で無制限 |
| `prConcurrentLimit` | `10` | 同時に開いておく PR の数。 `0` で無制限 |
| `groupName` | — | 同じ名前の更新を 1 つのブランチ・PR にまとめる（ `packageRules` の中で使う） |
| `separateMajorMinor` | `true` | メジャーとマイナー以下を別の PR にする |
| `separateMinorPatch` | `false` | マイナーとパッチも分ける |
| `dependencyDashboardApproval` | `false` | Dashboard でチェックを入れるまで PR を作らない（承認制） |
| `prCreation` | `"immediate"` | `"not-pending"` ・ `"status-success"` にすると、ブランチの CI が終わる／通るまで PR を作らない |

| キー | 既定 | 意味 |
| --- | --- | --- |
| `automerge` | `false` | CI が通ったら人の手を介さずマージする |
| `automergeType` | `"pr"` | `"branch"` にすると PR を作らずブランチを直接マージする（通知が減る）。ほかに `"pr-comment"` |
| `platformAutomerge` | `true` | GitHub の auto-merge など、プラットフォームの機能でマージする（Renovate の次の実行を待たない） |
| `automergeSchedule` | `["at any time"]` | automerge してよい時間帯。 `schedule` とは別に決める。ただし既定の `platformAutomerge: true` では守られないので、守らせるなら `platformAutomerge: false` も書く |
| `automergeStrategy` | `"auto"` | `squash` ・ `rebase` ・ `merge-commit` などのマージ方法 |

ブランチ保護で必須のチェックがある前提で使う。必須レビューを設定していると、レビューが済むまで automerge されない（Renovate をバイパス対象に入れることもできる）。

| キー | 既定 | 意味 |
| --- | --- | --- |
| `rangeStrategy` | `"auto"` | `^1.2.0` のような範囲の直し方。 `pin` （固定版にする）・ `bump` （範囲の下限を上げる）・ `replace` ・ `widen` ・ `update-lockfile` （範囲内ならロックファイルだけ）・ `in-range-only` |
| `lockFileMaintenance` | 無効 | ロックファイルを作り直して間接依存もまとめて更新する。 `{"enabled": true}` で有効。既定のスケジュールは月曜の早朝 |
| `minimumReleaseAge` | なし | 公開から指定の期間（ `"3 days"` など）が経つまで提案しない。取り下げ・乗っ取られた版を避ける |
| `pinDigests` | `false` | Docker イメージや GitHub Actions をタグでなく SHA256 ダイジェストに固定する |
| `rebaseWhen` | `"auto"` | 既存ブランチを作り直す条件。 `conflicted` （衝突時だけ）・ `behind-base-branch` ・ `never` など |
| `recreateWhen` | `"auto"` | 閉じた PR を作り直すか。 `always` ・ `never` |

| キー | 既定 | 意味 |
| --- | --- | --- |
| `enabledManagers` | すべて | 書いた manager だけを動かす（例: `["npm", "github-actions"]` ） |
| `ignorePaths` | `node_modules` ・ `bower_components` | 調べないパス（glob）。 `config:recommended` はテスト用フォルダなども足す |
| `ignoreDeps` | `[]` | 無視する依存の名前（完全一致）。パターンを使うなら `packageRules` ＋ `enabled: false` |
| `baseBranchPatterns` | `[]` （既定ブランチ） | 更新を当てるブランチ。 `"$default"` ・ `/^release\//` のように書ける |
| `customManagers` | `[]` | 対応していないファイル（Makefile・任意の YAML など）から正規表現で版を拾う |
| `labels` ・ `addLabels` | `[]` | PR のラベル。 `labels` は置き換え、 `addLabels` は追加 |
| `semanticCommits` | `"auto"` | コミットに `chore(deps):` などを付けるか。 `auto` は履歴を見て決める |

| キー | 既定 | 意味 |
| --- | --- | --- |
| `vulnerabilityAlerts` | 有効 | GitHub の Dependabot アラートを読み、修正版への PR をスケジュールに関係なく作る（GitHub で、アラートの読み取り権限があるとき） |
| `osvVulnerabilityAlerts` | `false` | osv.dev の脆弱性情報から修正 PR を作る。GitHub 以外でも使える |
| `minimumReleaseAge` | なし | サプライチェーン攻撃対策。 `config:best-practices` は npm に 3 日を設定する |
| `configMigration` | `false` | 古い書き方のキーを新しい書き方に直す PR を出す |

（元のページで操作できます → https://cs.taka1156.site/renovate.html#options）

## packageRules

`packageRules` は「 **match* で対象を選び、残りのキーで振る舞いを上書きする** 」ルールの配列。上から順に当てはめ、後のルールが前のルールを上書きする。

| match キー | 選ぶもの |
| --- | --- |
| `matchPackageNames` | パッケージ名（ `"react"` ・ `"@types/**"` ・ `"/^eslint/"` ・ `"!foo"` ） |
| `matchDepNames` | ファイルに書かれた依存名（パッケージ名と違うことがある Docker・Go など） |
| `matchUpdateTypes` | 更新の種類: `major` ・ `minor` ・ `patch` ・ `pin` ・ `digest` ・ `pinDigest` ・ `lockFileMaintenance` ・ `rollback` ・ `bump` ・ `replacement` |
| `matchDepTypes` | `dependencies` ・ `devDependencies` ・ `peerDependencies` など |
| `matchManagers` | `npm` ・ `dockerfile` ・ `github-actions` ・ `gomod` など |
| `matchDatasources` | 版の取得元: `npm` ・ `docker` ・ `go` ・ `github-tags` など |
| `matchFileNames` | 依存が書かれたファイルのパス（ `"apps/web/**"` ）。monorepo で場所ごとに変えるとき |
| `matchCurrentVersion` | 今の版（ `"< 1.0.0"` など） |

名前などの指定は共通の書き方: 普通の文字列は glob（大文字小文字を区別しない）、 `/…/` は正規表現（RE2、 `/…/i` で区別しない）、先頭の `!` は否定。肯定のパターンが 1 つ以上あればどれかに合うこと、否定のパターンにはどれにも合わないことが条件になる。

v38 から `matchPackagePatterns` ・ `matchPackagePrefixes` ・ `excludePackageNames` は `matchPackageNames` に統合された（古い書き方は自動で読み替えられ、 `configMigration` で書き直せる）。

### よく使う書き方

まとめる

自動マージ

メジャーは承認制

止める・固定する

任意のファイル

```
"packageRules": [
  {
    "description": "メジャー以外は 1 本の PR にまとめる",
    "matchUpdateTypes": ["minor", "patch"],
    "groupName": "all non-major dependencies",
    "groupSlug": "all-minor-patch"
  },
  {
    "matchPackageNames": ["@types/**"],
    "groupName": "type definitions"
  }
]
```

プリセット `group:allNonMajor` を `extends` に足すのと同じ効果（1 つ目のルール）。

```
"packageRules": [
  {
    "description": "devDependencies のマイナー・パッチは CI が通れば自動マージ",
    "matchDepTypes": ["devDependencies"],
    "matchUpdateTypes": ["minor", "patch"],
    "automerge": true
  },
  {
    "description": "GitHub Actions はブランチのまま自動マージ（PR を作らない）",
    "matchManagers": ["github-actions"],
    "matchUpdateTypes": ["minor", "patch", "digest"],
    "automerge": true,
    "automergeType": "branch"
  }
]
```

```
"packageRules": [
  {
    "matchUpdateTypes": ["major"],
    "dependencyDashboardApproval": true,
    "addLabels": ["major"]
  }
]
```

メジャー更新は Dashboard でチェックを入れたときだけ PR になる。

```
"packageRules": [
  {
    "description": "特定のパッケージを更新しない",
    "matchPackageNames": ["legacy-lib", "/^@internal\\//"],
    "enabled": false
  },
  {
    "description": "Node 本体は 22 系に留める",
    "matchPackageNames": ["node"],
    "allowedVersions": "<23"
  }
]
```

`allowedVersions` は範囲（その版の versioning の書き方）か `/正規表現/` で書く。

```
// Dockerfile の ENV などに書いた版を拾う
"customManagers": [
  {
    "customType": "regex",
    "managerFilePatterns": ["/(^|/)Dockerfile$/"],
    "matchStrings": [
      "# renovate: datasource=(?<datasource>\\S+) depName=(?<depName>\\S+)\\s+ENV \\S+_VERSION=(?<currentValue>\\S+)"
    ]
  }
]
```

```
# Dockerfile 側
# renovate: datasource=github-releases depName=cli/cli
ENV GH_VERSION=2.60.0
```

名前付きグループ `depName` ・ `currentValue` ・ `datasource` を取れれば更新できる。ファイルの指定は `managerFilePatterns` （旧 `fileMatch` ）。

（元のページで操作できます → https://cs.taka1156.site/renovate.html#rules）

## 確認・デバッグ

| やること | 方法 |
| --- | --- |
| 設定の検証 | `npx --yes --package renovate -- renovate-config-validator` （ `--strict` で古い書き方も警告）。pre-commit のフックもある |
| 手元で何が見つかるか見る | `LOG_LEVEL=debug npx renovate --platform=local` （今いるフォルダを読み、PR は作らない） |
| 実際のリポジトリで試す | `npx renovate --dry-run=full --token=$TOKEN owner/repo` 。 `extract` ・ `lookup` ・ `full` の順に深く調べる |
| PR を作り直す | PR 本文の「rebase/retry」のチェックボックスに印を付ける |
| スケジュール外に今すぐ作る | Dependency Dashboard の該当行にチェックを入れる |
| App の実行ログを見る | Mend の Developer Portal（ `developer.mend.io` ）でリポジトリごとのジョブログが見られる |

## Renovate の PR 同士がコンフリクトしたら

同じ `package.json` やロックファイルを触る PR が並ぶと、1 本マージした時点で残りがコンフリクトする。 **手で直さず Renovate に任せる** のが基本。Renovate はコンフリクトした PR を、ベースブランチの最新から作り直してくれる（Git の rebase ではなく、更新をやり直した新しいコミットにする）。

1. **1 本ずつマージする** コンフリクトした残りの PR はそのままにしておく
2. **次の実行を待つ** Renovate が次に動いたとき、コンフリクトした PR を最新のベースから作り直す
3. **急ぐなら rebase/retry にチェック** PR 本文のチェックボックスに印を付けると、その PR を優先して作り直す

手でコンフリクトを解消してコミットすると（GitHub の「Resolve conflicts」ボタンも同じ）、Renovate は誰かが編集したとみなし、そのブランチを自動で更新しなくなる。元に戻すには rebase/retry にチェックを入れる（手で入れた変更は消える）。コンフリクトが頻繁なら、 `groupName` でまとめるか automerge にすると減る。

## Dependabot との使い分け

**Renovate** GitHub・GitLab・Bitbucket・Azure DevOps・Gitea/Forgejo などで動く。対応 manager が多く、プリセットで組織共通の設定を配れる。packageRules・Dashboard・customManagers で細かく決められる。

**Dependabot** GitHub に組み込みで、 `.github/dependabot.yml` だけで始められる。Security updates が GitHub のアラートと直結。細かい振り分けや対応する形式は Renovate より少ない。

両方を同じエコシステムに有効にすると、同じ更新の PR が二重に来る。Renovate に寄せるなら Dependabot の version updates は止める（security alerts は残して Renovate の `vulnerabilityAlerts` に読ませてよい）。

## よくある誤解

✗ 誤解: config:recommended に入れればロックファイルの更新もされる

✓ 正しくは: `lockFileMaintenance` は既定で無効で、 `config:recommended` にも入っていない。間接依存も上げたいなら `{"lockFileMaintenance": {"enabled": true}}` か、 `config:best-practices` （ `:maintainLockFilesWeekly` を含む）を使う。

✗ 誤解: timezone を書かなくても schedule は日本時間で解釈される

✓ 正しくは: `timezone` が無いと UTC で解釈される。 `"timezone": "Asia/Tokyo"` を一緒に書く。

✗ 誤解: schedule を設定すれば automerge もその時間帯だけになる

✓ 正しくは: `schedule` はブランチ・PR を作る時間帯。マージの時間帯は `automergeSchedule` で別に決める（既定は「いつでも」）。既定の `platformAutomerge: true` では PR 作成時にマージが予約されるので、時間帯を守るには `platformAutomerge: false` が要る。また、スケジュール外でも既存の PR の更新や脆弱性の修正 PR は動くことがある。

✗ 誤解: パターンで選ぶには matchPackagePatterns を使う

✓ 正しくは: v38 以降は `matchPackageNames` に glob や `/正規表現/` をそのまま書く。 `matchPackagePatterns` は古い書き方で、 `renovate-config-validator --strict` が移行を促す。

✗ 誤解: Renovate の PR を閉じれば、その更新は二度と来ない

✓ 正しくは: 閉じた PR は「その版を無視する」意味になるが、さらに新しい版が出れば新しい PR が来る。メジャー更新の PR を閉じると、そのメジャー系列全体を無視する。恒久的に止めたいなら `packageRules` で `enabled: false` や `allowedVersions` を書く。

## 出典

- [Renovate Docs](https://docs.renovatebot.com/) ・ [Configuration Options](https://docs.renovatebot.com/configuration-options/) ・ [String Pattern Matching](https://docs.renovatebot.com/string-pattern-matching/)
- [Shareable Config Presets](https://docs.renovatebot.com/config-presets/) ・ [Full Config Presets](https://docs.renovatebot.com/presets-config/)
- [Running Renovate](https://docs.renovatebot.com/getting-started/running/) ・ [Installing & Onboarding](https://docs.renovatebot.com/getting-started/installing-onboarding) ・ [renovatebot/github-action](https://github.com/renovatebot/github-action)
- [Config Validation](https://docs.renovatebot.com/config-validation) ・ [Release notes for major versions](https://docs.renovatebot.com/release-notes-for-major-versions)

確認した日付: 2026-11-22（Renovate v44 系）
