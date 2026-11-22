<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Skills CLI（npx skills）使い方

> エージェントスキルを入れる・更新する・消す Vercel Labs の CLI、npx skills のコマンド・オプション・置き場所・ロックファイルをまとめた。

- URL: https://cs.taka1156.site/skills-cli.html
- タグ: 開発ツール
- 日付: 2026-11-22

---

Vercel Labs のエージェントスキル管理 CLI。GitHub などからスキルを入れ、Claude Code・Codex・Cursor など 79 のエージェントに配る（skills v1.9.0・2026-11 時点）

**ざっくり言うと**

- `npx skills add owner/repo` で、リポジトリの SKILL.md を各エージェントのスキル置き場に入れる CLI。
- 実体を `.agents/skills/` に 1 つ置き、各エージェントのフォルダからはシンボリックリンクで指す。
- プロジェクトに入れたものは `skills-lock.json` に記録され、CI で更新や改変を検知できる。

## 仕組み

**ソース** GitHub・GitLab・Git URL・ローカル・ダウンロード URL

**.agents/skills/<name>** 実体（canonical）。グローバルなら `~/.agents/skills/`

**.claude/skills/ など** 各エージェントのフォルダ。実体へのシンボリックリンク

Node.js 22.20 以降が要る。インストールせずに `npx` で使うのがふつう。 `add` は対話式で、入れるスキルとエージェントを選ぶ（入っているエージェントは自動で検出される）。シンボリックリンクが使えない環境では `--copy` で各フォルダにコピーする。

## コマンド早見表

入れる

確認・更新・削除

探す・試す・作る

実験的

```
# 対話式（スキルとエージェントを選ぶ）
npx skills add vercel-labs/agent-skills

# 中身を一覧するだけ（入れない）
npx skills add vercel-labs/agent-skills --list

# スキルとエージェントを指定
npx skills add vercel-labs/agent-skills --skill web-design-guidelines -a claude-code

# ユーザー単位（全プロジェクト）に、確認なしで（CI 向け）
npx skills add vercel-labs/agent-skills --skill web-design-guidelines -g -a claude-code -y

# リポジトリの全スキルを全エージェントへ
npx skills add vercel-labs/agent-skills --all

# 同名スキルの衝突を避けて別名で入れる
npx skills add vercel-labs/agent-skills --skill web-design-guidelines --rename vercel-web-design
```

```
# 入っているスキルの一覧（ls でもよい）
npx skills list
npx skills ls -g
npx skills ls -a claude-code -a cursor

# 更新があるか・手で書き換えられていないか（あれば終了コード 1）
npx skills check

# 更新
npx skills update                 # 全部（スコープを聞かれる）
npx skills update my-skill        # 名前を指定
npx skills update -y              # プロジェクト内ならプロジェクト、外ならグローバル

# 削除（rm でもよい）
npx skills remove my-skill
npx skills remove owner/repo      # そのソースから入れたものを全部
npx skills remove --global my-skill
npx skills remove --all           # ロックに記録されたものを確認なしで全部
```

```
# skills.sh のスキルを探す
npx skills find                   # 対話式
npx skills find typescript
npx skills find react --owner vercel

# 入れずに 1 回だけ使う（プロンプトを出力 / エージェントを起動）
npx skills use vercel-labs/agent-skills@web-design-guidelines | claude
npx skills use vercel-labs/agent-skills@web-design-guidelines --agent claude-code

# スキルのひな形を作る
npx skills init                   # 今のフォルダに SKILL.md
npx skills init my-skill          # my-skill/SKILL.md
```

`experimental_` の付くコマンドは予告なく変わる・なくなることがある。

```
# npm の依存が同梱しているスキルをプロジェクトに取り込む
npm install -D skills agent-browser
npx skills experimental_sync -y

# skills-lock.json からプロジェクトのスキルを復元（skills install でもよい）
npx skills experimental_install
```

`package.json` の `"prepare": "skills experimental_sync -y"` にすると、 `npm install` のたびに同期される。 `package.json` の `"skills"` 配列（ `"npm:パッケージ名"` や `"owner/repo@skill"` ）でも取り込むスキルを宣言できる。

（元のページで操作できます → https://cs.taka1156.site/skills-cli.html#commands）

## add に渡せるソース

| 形式 | 例 |
| --- | --- |
| GitHub の省略形 | `vercel-labs/agent-skills` |
| タグ・コミットで固定 | `owner/repo@v1.2.3 --skill my-skill` |
| GitHub の URL | `https://github.com/vercel-labs/agent-skills` |
| リポジトリ内の 1 スキル | `https://github.com/owner/repo/tree/main/skills/name` |
| GitLab・Azure Repos | `https://gitlab.com/org/repo` 、 `https://dev.azure.com/org/project/_git/repo` |
| 任意の Git URL | `git@github.com:owner/repo.git` |
| ローカル | `./my-local-skills` |
| ダウンロード URL | SKILL.md 単体か .zip・.tar・.tgz（既定の上限 10 MiB） |

非公開リポジトリも同じコマンドで入る。Git の認証情報 → GitHub CLI（ `gh` ）→ SSH の順に試す。GitHub API 用に `GITHUB_TOKEN` / `GH_TOKEN` を渡すこともできる（必須ではない）。

## add のオプション

| オプション | 意味 |
| --- | --- |
| `-g, --global` | プロジェクトではなくユーザーのホームに入れる |
| `-a, --agent <agents...>` | 入れるエージェント（ `claude-code` ・ `codex` など。 `'*'` で全部） |
| `-s, --skill <skills...>` | 入れるスキル名（ `'*'` で全部。空白を含む名前は引用符で囲む） |
| `-l, --list` | 入れずに一覧だけ出す |
| `--copy` | シンボリックリンクではなくコピーする |
| `--rename <name>` | 選んだ 1 つのスキルを別名で入れる（フォルダ名・ロック・SKILL.md の `name` が変わる） |
| `-y, --yes` | 確認をすべて飛ばす |
| `--all` | 全スキルを全エージェントに、確認なしで |

## 主なエージェントの置き場所

| エージェント | `-a` の値 | プロジェクト | グローバル（ `-g` ） |
| --- | --- | --- | --- |
| Claude Code | `claude-code` | `.claude/skills/` | `~/.claude/skills/` |
| Codex | `codex` | `.agents/skills/` | `~/.agents/skills/` |
| Cursor | `cursor` | `.agents/skills/` | `~/.cursor/skills/` |
| OpenCode | `opencode` | `.agents/skills/` | `~/.config/opencode/skills/` |
| Gemini CLI | `gemini-cli` | `.agents/skills/` | `~/.gemini/skills/` |
| GitHub Copilot | `github-copilot` | `.agents/skills/` | `~/.copilot/skills/` |

プロジェクトの置き場所が `.agents/skills/` のエージェントは、実体をそのまま読む。ほかのエージェントの一覧は公式の [docs/agents.md](https://github.com/vercel-labs/skills/blob/main/docs/agents.md) 。

## skills-lock.json と CI

プロジェクトに入れたスキルは `skills-lock.json` （version 2）に、ソース・Git のコミット・中身のハッシュ（ `computedHash` ）と一緒に記録される。コミットしてチームで共有する。グローバルのロックは `~/.agents/.skill-lock.json` （ `XDG_STATE_HOME` があれば `$XDG_STATE_HOME/skills/.skill-lock.json` ）。

| 使い方 | コマンド | 動き |
| --- | --- | --- |
| 状態を見る | `skills list` | ロックと比べて `modified` （手で変えた）・ `untracked` （ロックにない）・ `Missing` （ロックにあるが無い）を表示。 `--json` で機械向け |
| CI で止める | `skills check` | 更新か手元の改変があれば終了コード 1 |
| ロックどおりに戻す | `skills install --frozen-lockfile` | 記録したコミットを取り直し、ハッシュが合わなければ失敗。CI では `--frozen-lockfile` が既定 |

## スキルを作る

`npx skills init my-skill` でひな形ができる。必須は frontmatter の `name` （英小文字とハイフン）と `description` の 2 つ。どちらかが欠けると「No skills found」になる。

```
---
name: my-skill
description: What this skill does and when to use it
---

# My Skill

Instructions for the agent to follow when this skill is activated.
```

CLI がリポジトリで探す場所は、ルートの `SKILL.md` 、 `skills/` （ `skills/<分類>/<name>/` のように 3 段まで）、 `.claude/skills/` などの各エージェントのフォルダ。それ以外の場所も探すなら `--full-depth` 。 `metadata.internal: true` を付けたスキルは、 `INSTALL_INTERNAL_SKILLS=1` のときだけ表示・インストールされる。

## 環境変数とテレメトリ

| 変数 | 意味 |
| --- | --- |
| `DISABLE_TELEMETRY` / `DO_NOT_TRACK` | 匿名の利用統計と、セキュリティ監査の問い合わせを止める |
| `GITHUB_TOKEN` / `GH_TOKEN` | GitHub API に使うトークン（任意） |
| `INSTALL_INTERNAL_SKILLS` | `1` か `true` で internal のスキルも出す |
| `SKILLS_DOWNLOAD_MAX_BYTES` など | ダウンロード URL の上限（展開後のサイズ・ファイル数も同様に変えられる） |

公開と確認できた GitHub リポジトリから入れるときは、 `add` が skills.sh の監査結果（Gen・Socket・Snyk のリスク評価）を表に出す。参考にはなるが、入れる前に SKILL.md を自分で読むのが基本。

## よくある誤解

✗ 誤解: npx skills add は Vercel のスキルしか入れられない

✓ 正しくは: GitHub・GitLab・Azure Repos・任意の Git URL・ローカルのフォルダ・ダウンロード URL から入れられる。非公開リポジトリも、手元の Git や GitHub CLI の認証で入る。Cloudflare の [Security Audit Skill](https://cs.taka1156.site/md/security-audit-skill.md) もこの CLI で入れる。

✗ 誤解: skills.sh にあるスキルや、npx skills add で入れたスキルは安全だと保証されている

✓ 正しくは: CLI はソースのファイルをそのままコピー・リンクするだけで、中身の安全性は保証しない。 `add` で出る Gen・Socket・Snyk の監査結果も第三者による参考情報で、公開と確認できた GitHub リポジトリにしか出ない。

スキルの SKILL.md はエージェントへの指示としてそのまま読み込まれるので、悪意のある指示を仕込めばプロンプトインジェクションになる。同梱のスクリプトもエージェントが実行しうる。入れる前に SKILL.md と同梱ファイルを読み、信頼できるソースをタグやコミットで固定して入れる。

✗ 誤解: remove --all を使うと、手で書いたスキルまで消える

✓ 正しくは: `--all` （と `--skill '*'` ）が消すのは、ロックファイルに記録されたものだけ。手で置いたスキルは「skipped」と表示されて残る。消したいなら名前を指定する。

✗ 誤解: skills check は更新を表示するだけで、CI では使えない

✓ 正しくは: 更新があるか、入れた中身が手で変えられていると終了コード 1 で終わる。そのまま CI のゲートに使える。

✗ 誤解: グローバルに入れると、全プロジェクトのロックファイルに記録される

✓ 正しくは: `-g` で入れたものはユーザー単位のロック（既定は `~/.agents/.skill-lock.json` ）に入り、プロジェクトの `skills-lock.json` には載らない。チームで揃えたいスキルはプロジェクトに入れる。

## 出典

- [vercel-labs/skills](https://github.com/vercel-labs/skills) （README.md・docs/commands.md・docs/agents.md・docs/creating-skills.md・docs/troubleshooting.md。v1.9.0、コミット 57217cf）
- [skills.sh](https://skills.sh) （スキルのディレクトリ）

確認日: 2026-11-22
