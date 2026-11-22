<!-- scripts/memo.py build が自動生成。手で編集しない -->

# ESLint・Prettier → Oxlint・Oxfmt 移行（Vite+）

> ESLint・Prettier を Oxlint・Oxfmt に、husky を vp staged に移す手順と変換スクリプト

- URL: https://cs.taka1156.site/oxc-migration.html
- タグ: 開発ツール
- 日付: 2026-11-23

---

# ESLint・Prettier → Oxlint・Oxfmt 移行

Vite+ 環境を前提に、lint・format の移行と、husky・lint-staged から `vp staged` への移行をまとめる

**ざっくり言うと**

- Vite+ なら `vp migrate` （既存の Vite+ は `--full` ）で ESLint・Prettier を Oxlint・Oxfmt に変換できる
- flat config と JSON・JS の Prettier 設定は自動、旧 `.eslintrc` と husky は自動では変換されない
- husky は手で `.vite-hooks/` ・ `staged` ・ `vp config` に移す（下の変換スクリプトで自動化できる）

## 何が何に置き換わるか

点線の箱は `vp migrate` が自動で変換しないもの（手で移すか、下のスクリプトを使う）。

移行前 移行後（Vite+）

`eslint.config.js` （flat config） `eslint .`

`vite.config.ts` の `lint` `vp lint` （Oxlint）

`.eslintrc.*` （v8 以前の形式） そのままでは変換されない

先に flat config へ変換してから移行 `npx @eslint/migrate-config .eslintrc.json`

`.prettierrc` ・ `prettier.config.js` `prettier --write .`

`vite.config.ts` の `fmt` `vp fmt` （Oxfmt）

`.eslintignore` ・ `.prettierignore`

`lint.ignorePatterns` ・ `fmt.ignorePatterns` `.gitignore` は両方とも自動で読む

lint-staged（JSON の設定） `package.json` の `lint-staged` など

`vite.config.ts` の `staged` `vp staged` （中身は lint-staged）

`.husky/pre-commit` など `npx lint-staged`

`.vite-hooks/pre-commit` `vp staged`

`"prepare": "husky"` `core.hooksPath=.husky/_`

`"prepare": "vp config"` `core.hooksPath=.vite-hooks/_`

## 移行の手順

Vite+（vp migrate）

単体で移す

注意の色の番号（6）は、 `vp migrate` では終わらず手作業が要る手順。

1. **ブランチを切り、旧 .eslintrc を flat config にする** `vp migrate` は旧形式の `.eslintrc` を見つけると ESLint とその lint スクリプトを残す。先に `npx @eslint/migrate-config .eslintrc.json` で `eslint.config.mjs` にしておく。
2. **vp migrate を実行する** Vite+ を入れていないプロジェクトは `vp migrate` だけで全部やる。既に Vite+ のプロジェクトは、ESLint・Prettier・フックの移行が既定では飛ばされるので `vp migrate --full` 。モノレポはワークスペースのルートで 1 回だけ。
3. **表示される警告とレポートを読む** 先に質問にまとめて答え、その後は質問なしで実行される。変換できなかった ESLint ルールの一覧と husky を検出したときの警告が出て、最後にファイルごとのレポートが出る。コミットする前に読む。
4. **確かめる** `vp install` → `vp check` → `vp test` → `vp build` （ライブラリは `vp pack` ）。 `vp check` で新しく出た指摘と、整形の差分を見る。
5. **差分を分けてレビューする** 移行は変更したファイルを整形する。整形だけの差分と、依存・設定・import の変更は分けて見ると、ロジックの変更を見落としにくい。
6. **husky を使っていたら手で移す** husky のフック・スクリプト・依存はそのまま残る（警告だけ）。下の「husky の移行」の手順かスクリプトで移す。

Vite+ を使わず、oxlint・oxfmt を単体で入れるときの手順。設定は `.oxlintrc.json` ・ `.oxfmtrc.json` に出る（Vite+ に移すときは `vp migrate` が `vite.config.ts` にまとめる）。

| やること | コマンド |
| --- | --- |
| ESLint の flat config を変換 | `npx @oxlint/migrate` （パスを渡すなら `npx @oxlint/migrate ./eslint.config.mjs` ） |
| 変換できなかったルールを見る | `npx @oxlint/migrate --details` |
| 型情報を使うルールも移す | `npx @oxlint/migrate --type-aware` （ `tsrs` か `oxlint-tsgolint` が要る） |
| ESLint プラグインを JS プラグインにしない | `npx @oxlint/migrate --js-plugins=false` （既定では未対応のプラグインを `jsPlugins` に移す） |
| `// eslint-disable` を書き換える | `--replace-eslint-comments` （ `// oxlint-disable` にする。そのままでも効く） |
| Prettier の設定を変換して整形 | `pnpm add -D oxfmt@latest && pnpm oxfmt --migrate=prettier && pnpm oxfmt` |
| 整形のチェックだけ（CI） | `oxfmt --check` |
| lint-staged で使う | `oxfmt --no-error-on-unmatched-pattern` |

`@oxlint/migrate` は 1 回きり使うもので、 `package.json` に入れない。使うときは `oxlint` と同じ版にそろえる。

（元のページで操作できます → https://cs.taka1156.site/oxc-migration.html#flow）

## vp migrate が変換するもの・しないもの

| 対象 | 扱い |
| --- | --- |
| ESLint の flat config | Oxlint の設定（ `lint` ）に変換。ESLint の依存・設定・スクリプトは置き換え |
| 旧形式の `.eslintrc` | ESLint とその lint スクリプトを残す |
| Prettier の設定 | Oxfmt の設定（ `fmt` ）に変換。Prettier を使っている間は、移行後の整形を飛ばす |
| `.oxlintrc*` ・ `.oxfmtrc*` | `vite.config.ts` にまとめる |
| package.json の scripts | `eslint` → `vp lint` 、 `prettier` → `vp fmt` 、 `lint-staged` → `vp staged` |
| lint-staged（JSON の設定） | フックツールを残さないなら `staged` に移す。husky などを残すなら lint-staged も残る |
| JSON でない `.lintstagedrc` ・ `lint-staged.config.*` | 移さない |
| husky | フック・スクリプト・設定・依存を変えずに残し、警告を出す |
| lefthook・simple-git-hooks・yorkie | `--hooks` を付けても残し、警告を出す |

フラグ: `--full` （フック・ESLint・Prettier の移行も含める）、 `--hooks` / `--no-hooks` （フックの設定だけ入れる／入れない）、 `--no-interactive` （質問なし）。

## ESLint 側で気をつけること

- 変換できないルールは一覧で出る。どうしても要るルールは ESLint を残して `"lint:eslint": "eslint ."` を作り、 `vp run lint:eslint` で動かせる（ESLint のスクリプトが残っている場合だけ。flat config を変換すると消えるので、手で戻す）
- 併用中は `eslint-plugin-oxlint` で、Oxlint が持つルールを ESLint 側で切る。実行は速い方から `oxlint && eslint`
- 同じリポジトリ内のローカルプラグイン（ `../eslint-plugin-myplugin` など）は移らない。 `jsPlugins` に手で足す
- `@oxlint/migrate` は、ESLint で有効にしていなかったルールが増えないよう `correctness` カテゴリを切った状態で出す。必要なら後で有効にする

## Prettier 側で気をつけること

- Oxfmt の `printWidth` の既定は 100（Prettier は 80）。 `--migrate=prettier` は設定に無ければ 80 を書いて、Prettier の見た目を保つ
- 読める設定: `package.json` の `prettier` 、JSON の `.prettierrc` 、 `.prettierrc.json` 、JS・TS の設定。YAML・JSON5・TOML は先に JSON か JS にする
- JS・TS の設定は評価した値に置き換わる。環境での分岐などは手で `oxfmt.config.ts` に移す。入れ子の設定は探さないので、ディレクトリごとに実行する
- Prettier プラグインは使えない。tailwindcss・svelte・astro・packagejson のプラグインの設定は組み込みの機能（ `sortTailwindcss` ・ `sortPackageJson` など）に移る。ほかは警告を出して飛ばす
- `// prettier-ignore` はそのまま効く（ `// oxfmt-ignore` も使えるが、効くのは JS・TS のファイルだけ）
- `eslint-plugin-prettier` は外す。ESLint を残すなら `eslint-config-prettier` は残す
- 全体を整形したコミットのハッシュは `.git-blame-ignore-revs` に書いておく

## husky の移行（手で行う）

`vp migrate` は husky を変換しない。公式ガイドが lefthook などに示している手順と Commit Hooks ガイドをもとにすると、次のとおり。

1. **staged に移す** lint-staged のコマンドを `vite.config.ts` の `staged` に移す（ `vp staged` はこのブロックしか読まない）。
2. **prepare を vp config にする** `"prepare": "husky"` を `"prepare": "vp config"` にする。
3. **.vite-hooks/pre-commit を作る** 中身は `vp staged` の 1 行。 `commit-msg` などほかのフックも同じフォルダに置く。
4. **ディスパッチャーを入れる** `vp hooks enable` か `vp config` 。 `core.hooksPath` が別のフォルダを指していると `vp config` はディスパッチャーを入れない。husky のファイルが残っているうちに切り替えるなら `vp hooks enable --replace-husky` を 1 回だけ実行する（ `prepare` には書かない。古い版にこのオプションが無ければ `git config --unset core.hooksPath` してから `vp config` ）。
5. **動いたら husky を消す** コミットしてフックが動くのを確かめてから、 `.husky/` と husky・lint-staged の依存を消す。

```
// vite.config.ts
import { defineConfig } from 'vite-plus';

export default defineConfig({
  staged: {
    '*': 'vp check --fix',          // 公式の既定。glob ごとに分けてもよい
  },
});
```

| やりたいこと | コマンド・設定 |
| --- | --- |
| フックの状態を見る | `vp hooks status` |
| この clone だけフックを止める | `vp hooks disable` （戻すときは `vp hooks enable` ） |
| フックのフォルダを変える | `vp config --hooks-dir .githooks` |
| CI などでフックを動かさない | 環境変数 `VP_GIT_HOOKS=0` （ `HUSKY=0` も効く。 `prepare` での再インストールも止まる） |
| 変更が出たら失敗させる | `vp staged --fail-on-changes` |

`.vite-hooks/pre-commit` などはコミットする。 `.vite-hooks/_` （ディスパッチャー）は自動で無視され、 `vp config` で作り直される。 `vp staged` は Node.js 22.22.1 以上（22 系）か 24.11.0 以上、Git 2.32.0 以上が要る。

## husky → Vite+ 変換スクリプト

上の手順をまとめて行うシェルスクリプト（公式のものではない）。 `vp migrate` の後、リポジトリのルートで動かす。まず `--dry-run` で何が変わるかを見る。

```
bash migrate-husky-to-vp.sh --dry-run          # 変わる内容を表示するだけ
bash migrate-husky-to-vp.sh                    # 実行（未コミットの変更があると止まる）
bash migrate-husky-to-vp.sh --default-staged   # lint-staged を変換せず '*': 'vp check --fix' にする
```

| やること | 中身 |
| --- | --- |
| lint-staged → `staged` | `package.json` の `lint-staged` ・ `.lintstagedrc(.json)` を読み、 `eslint` ・ `oxlint` → `vp lint --fix` 、 `prettier` ・ `oxfmt` → `vp fmt` 、両方あれば `vp check --fix` にまとめる。それ以外のコマンドは残す。 `defineConfig({` の直後に挿入 |
| `.husky/*` → `.vite-hooks/*` | husky v8 以前の shebang と `husky.sh` の読み込みを消し、 `npx lint-staged` などを `vp staged` に置き換える |
| package.json | `prepare` （・ `postinstall` ）の `husky` / `husky install` を `vp config` にし、 `lint-staged` の欄を消す |
| Git の設定 | husky の `core.hooksPath` を外す（どの版の `vp config` でもディスパッチャーが入るように） |
| 後片付け | `vp remove husky lint-staged` 、 `.husky/` を消す、 `vp config --no-agent` と `vp hooks status` |
| やらないこと | JS・YAML の lint-staged 設定の変換、 `staged` が既にある設定への追記、 `defineConfig(() => …)` のような関数形式への挿入（警告を出して手で直す） |

**migrate-husky-to-vp.sh**

```
#!/usr/bin/env bash
# husky + lint-staged を Vite+（vp config / .vite-hooks / vp staged）へ移す
# 使い方: bash migrate-husky-to-vp.sh [--dry-run] [--default-staged] [--force]
#   --dry-run         何をするかを表示するだけ（ファイルは変えない）
#   --default-staged  lint-staged の中身を変換せず staged: { '*': 'vp check --fix' } にする
#   --force           作業ツリーに未コミットの変更があっても進める
# 前提: リポジトリのルートで実行 / vp migrate 済み（vite.config.* が vite-plus の defineConfig）
set -euo pipefail

DRY=0; DEFAULT=0; FORCE=0
for a in "$@"; do
  case "$a" in
    --dry-run) DRY=1 ;;
    --default-staged) DEFAULT=1 ;;
    --force) FORCE=1 ;;
    *) echo "不明なオプション: $a" >&2; exit 2 ;;
  esac
done

say()  { printf '\033[1m==>\033[0m %s\n' "$*"; }
warn() { printf '\033[33m[警告]\033[0m %s\n' "$*" >&2; }
die()  { printf '\033[31m[中止]\033[0m %s\n' "$*" >&2; exit 1; }
run()  { if [ "$DRY" = 1 ]; then echo "  (dry-run) $*"; else "$@"; fi; }

# ---- 0. 事前チェック --------------------------------------------------------
[ -f package.json ] || die "package.json がない。リポジトリのルートで実行する"
git rev-parse --is-inside-work-tree >/dev/null 2>&1 || die "Git リポジトリではない"
[ -d .husky ] || die ".husky/ がない（husky を使っていない）"
command -v vp >/dev/null 2>&1 || die "vp が見つからない（Vite+ の CLI を入れる）"
NODE="node"; command -v node >/dev/null 2>&1 || NODE="vp node"

CONFIG=""
for f in vite.config.ts vite.config.mts vite.config.js vite.config.mjs; do
  [ -f "$f" ] && { CONFIG="$f"; break; }
done
[ -n "$CONFIG" ] || die "vite.config.* がない。先に vp migrate を実行する"

if [ "$FORCE" = 0 ] && [ -n "$(git status --porcelain)" ]; then
  die "未コミットの変更がある。コミットするか --force を付ける（戻すときは git で戻せるようにしておく）"
fi

# ---- 1. lint-staged の設定を読んで staged ブロックを作る ---------------------
LS_SRC=""
if [ "$DEFAULT" = 0 ]; then
  if $NODE -e 'process.exit(require("./package.json")["lint-staged"]?0:1)' 2>/dev/null; then
    LS_SRC="package.json"
  elif [ -f .lintstagedrc.json ]; then LS_SRC=".lintstagedrc.json"
  elif [ -f .lintstagedrc ] && $NODE -e 'JSON.parse(require("fs").readFileSync(".lintstagedrc","utf8"))' 2>/dev/null; then
    LS_SRC=".lintstagedrc"
  fi
  for f in .lintstagedrc.js .lintstagedrc.cjs .lintstagedrc.mjs .lintstagedrc.yaml .lintstagedrc.yml \
           lint-staged.config.js lint-staged.config.cjs lint-staged.config.mjs lint-staged.config.ts; do
    [ -f "$f" ] && warn "$f は JSON ではないので変換しない。staged の中身を手で見直す"
  done
  [ -f .lintstagedrc ] && [ -z "$LS_SRC" ] && warn ".lintstagedrc が JSON ではない（YAML など）。変換しない"
fi
[ -z "$LS_SRC" ] && say "lint-staged の JSON 設定なし → staged: { '*': 'vp check --fix' } を使う" \
                 || say "lint-staged の設定を $LS_SRC から変換する"

# node で: staged ブロックの生成 → vite.config への挿入（既にあれば挿入しない）
STAGED_JS=$(LS_SRC="$LS_SRC" $NODE <<'EOF'
const fs = require('fs');
const src = process.env.LS_SRC;
let rules = { '*': 'vp check --fix' };
if (src) {
  const raw = src === 'package.json'
    ? require(process.cwd() + '/package.json')['lint-staged']
    : JSON.parse(fs.readFileSync(src, 'utf8'));
  // npx / pnpm exec などの前置きを外して、ESLint・Prettier・oxlint・oxfmt を vp に置き換える
  const pre = String.raw`^(?:npx(?:\s+--no(?:-install)?)?(?:\s+--)?\s+|pnpm(?:\s+exec)?\s+|yarn\s+|bunx\s+|npm\s+exec\s+(?:--\s+)?)?`;
  const conv = (c) => {
    const s = c.trim();
    if (new RegExp(pre + String.raw`(?:eslint|oxlint)\b`).test(s)) return 'vp lint --fix';
    if (new RegExp(pre + String.raw`(?:prettier|oxfmt)\b`).test(s)) return 'vp fmt';
    if (new RegExp(pre + String.raw`vp\s`).test(s)) return s.replace(new RegExp(pre), '');
    return s; // それ以外（tsc・stylelint など）はそのまま残す
  };
  rules = {};
  for (const [glob, v] of Object.entries(raw)) {
    let cmds = (Array.isArray(v) ? v : [v]).map(conv);
    cmds = [...new Set(cmds)];
    if (cmds.includes('vp lint --fix') && cmds.includes('vp fmt')) {
      // fmt と lint が両方あるなら 1 つにまとめる
      cmds = ['vp check --fix', ...cmds.filter(c => c !== 'vp lint --fix' && c !== 'vp fmt')];
    }
    rules[glob] = cmds.length === 1 ? cmds[0] : cmds;
  }
}
const q = (s) => `'${s.replace(/\\/g, '\\\\').replace(/'/g, "\\'")}'`;
const val = (v) => Array.isArray(v) ? `[${v.map(q).join(', ')}]` : q(v);
const body = Object.entries(rules).map(([k, v]) => `    ${q(k)}: ${val(v)},`).join('\n');
process.stdout.write(`  staged: {\n${body}\n  },`);
EOF
)
echo "$STAGED_JS" | sed 's/^/    /'

if grep -Eq '(^|[^A-Za-z_])staged[[:space:]]*:' "$CONFIG"; then
  warn "$CONFIG に staged が既にある。上の内容と見比べて手で直す"
elif [ "$DRY" = 1 ]; then
  echo "  (dry-run) $CONFIG の defineConfig({ の直後に staged を挿入"
else
  STAGED_JS="$STAGED_JS" CONFIG="$CONFIG" $NODE <<'EOF' || { warn "自動で挿入できなかった。上の staged を $CONFIG の defineConfig({ ... }) に手で貼る"; }
const fs = require('fs');
const file = process.env.CONFIG;
const s = fs.readFileSync(file, 'utf8');
const m = /defineConfig\(\s*\{/.exec(s);           // defineConfig({ ... }) の形だけ扱う
if (!m) process.exit(1);                             // 関数形式などは手で
const at = m.index + m[0].length;
fs.writeFileSync(file, s.slice(0, at) + '\n' + process.env.STAGED_JS + s.slice(at));
EOF
fi

# ---- 2. .husky/<hook> → .vite-hooks/<hook> ----------------------------------
HOOKS_DIR=".vite-hooks"
run mkdir -p "$HOOKS_DIR"
for f in .husky/*; do
  [ -f "$f" ] || continue
  name=$(basename "$f")
  case "$name" in _*|*.sh|*.md) continue ;; esac   # .husky/_ と補助ファイルは移さない
  [ -e "$HOOKS_DIR/$name" ] && { warn "$HOOKS_DIR/$name が既にあるので上書きしない"; continue; }
  say ".husky/$name → $HOOKS_DIR/$name"
  # husky v4〜v8 の shebang と husky.sh の読み込みを消し、lint-staged を vp staged に置き換える
  conv=$(sed -E \
    -e '/^#!\/usr\/bin\/env (ba)?sh/d' \
    -e '/husky\.sh/d' \
    -e 's#(^|[;&|[:space:]])(npx( --no(-install)?)?( --)? |pnpm( exec)? |yarn |bunx |npm exec (-- )?)?lint-staged( [^;&|]*)?#\1vp staged#' \
    "$f" | sed -e '/./,$!d')                          # 先頭の空行を詰める
  if [ "$DRY" = 1 ]; then printf '%s\n' "$conv" | sed 's/^/    | /'
  else printf '%s\n' "$conv" > "$HOOKS_DIR/$name"; chmod +x "$HOOKS_DIR/$name"; fi
done
[ -f "$HOOKS_DIR/pre-commit" ] || [ "$DRY" = 1 ] || { say "pre-commit が無いので vp staged だけの pre-commit を作る"; echo "vp staged" > "$HOOKS_DIR/pre-commit"; chmod +x "$HOOKS_DIR/pre-commit"; }

# ---- 3. package.json: prepare を vp config に、lint-staged の欄を消す -------
say "package.json の prepare を vp config に置き換え、lint-staged の欄を消す"
if [ "$DRY" = 0 ]; then
  $NODE <<'EOF'
const fs = require('fs');
const txt = fs.readFileSync('package.json', 'utf8');
const pkg = JSON.parse(txt);
const indent = (txt.match(/^[ \t]+(?=")/m) || ['  '])[0];
pkg.scripts ??= {};
for (const k of ['prepare', 'postinstall']) {
  const v = pkg.scripts[k];
  if (typeof v === 'string' && /\bhusky\b/.test(v)) {
    const next = v.replace(/\bhusky(\s+install)?(\s+\S*\.husky\S*)?/g, 'vp config');
    pkg.scripts[k] = next;
  }
}
if (!Object.values(pkg.scripts).some(v => /\bvp config\b/.test(v))) {
  pkg.scripts.prepare = pkg.scripts.prepare ? `${pkg.scripts.prepare} && vp config` : 'vp config';
}
delete pkg['lint-staged'];
fs.writeFileSync('package.json', JSON.stringify(pkg, null, indent) + (txt.endsWith('\n') ? '\n' : ''));
EOF
fi

# ---- 4. husky の core.hooksPath を外す（新しい vp は --replace-husky でも切り替えられるが、版によらず入るように）
HP=$(git config --get core.hooksPath || true)
if [ -n "$HP" ]; then
  case "$HP" in
    *.husky*) say "core.hooksPath=$HP を外す"; run git config --unset core.hooksPath ;;
    *) warn "core.hooksPath=$HP（husky 以外）が設定されている。vp config はこのままだとディスパッチャーを入れない" ;;
  esac
fi

# ---- 5. husky・lint-staged を消す ------------------------------------------
DEPS=$($NODE -e 'const p=require("./package.json");const d={...p.dependencies,...p.devDependencies};console.log(["husky","lint-staged"].filter(n=>n in d).join(" "))')
[ -n "$DEPS" ] && { say "依存から外す: $DEPS"; run vp remove $DEPS; }
run rm -rf .husky
[ "$LS_SRC" = ".lintstagedrc.json" ] || [ "$LS_SRC" = ".lintstagedrc" ] && run rm -f "$LS_SRC"

# ---- 6. Vite+ のフックを入れて確かめる -------------------------------------
say "vp config でディスパッチャーを入れる"
run vp config --no-agent
run vp hooks status

cat <<'MSG'

次にやること:
  1. vite.config.* の staged と .vite-hooks/ の中身を読んで確かめる
  2. 何か 1 ファイルを git add して git commit し、フックが動くか試す
  3. .vite-hooks/ をコミットする（.vite-hooks/_ は自動で無視される）
  4. CI では VP_GIT_HOOKS=0 でフックを止められる（HUSKY=0 も効く）
MSG
```

変換の例: `"*.{ts,tsx}": ["eslint --fix", "prettier --write"]` → `'*.{ts,tsx}': 'vp check --fix'` 、 `"*.md": "prettier --write"` → `'*.md': 'vp fmt'` 。 `vp` をスタブにした仮のプロジェクトで動作を確かめた（実際の `vp config` の結果は `vp hooks status` で確かめる）。

## よくある誤解

✗ 誤解: vp migrate を流せば husky も Vite+ のフックに変わる

✓ 正しくは: husky・lefthook・simple-git-hooks・yorkie は変換されず、警告が出るだけ。husky を残している間は lint-staged も残る。手で移すか、上のスクリプトを使う。

✗ 誤解: 既に Vite+ のプロジェクトでも vp migrate だけで ESLint・Prettier が移る

✓ 正しくは: 既存の Vite+ プロジェクトのアップグレードでは、フック・ESLint・Prettier の移行は飛ばされ、 `--full` を付けるよう案内が出る。Vite+ を入れていないプロジェクトは常に全部を移す。

✗ 誤解: Oxfmt は Prettier と同じく 80 文字で折り返す

✓ 正しくは: Oxfmt の `printWidth` の既定は 100。 `--migrate=prettier` や `vp migrate` を使えば 80 が書かれるが、手で設定を作るときは自分で書く。

✗ 誤解: vp staged は .lintstagedrc や lint-staged.config.js も読む

✓ 正しくは: `vp staged` が読むのは `vite.config.ts` の `staged` だけ。中で lint-staged 17 を使っているが、単体の設定ファイルは見ない。

✗ 誤解: 移行したら // eslint-disable のコメントを全部書き換える必要がある

✓ 正しくは: Oxlint は `// eslint-disable` ・ `// eslint-disable-next-line` をそのまま解釈する。書き換えたいときだけ `@oxlint/migrate --replace-eslint-comments` を使う（ `vp migrate` は ESLint を移すときにこれも実行する）。

関連: [oxlint・oxfmt 使い方](https://cs.taka1156.site/md/oxlint-oxfmt.md) ・ [Vite+ と vp コマンド チートシート](https://cs.taka1156.site/md/vite-plus.md)

## 出典と確認日

- [Oxc: Migrate from ESLint](https://oxc.rs/docs/guide/usage/linter/migrate-from-eslint)
- [Oxc: Migrate from Prettier](https://oxc.rs/docs/guide/usage/formatter/migrate-from-prettier)
- [Vite+: Migrate](https://viteplus.dev/guide/migrate) ・ [Migrate rules](https://main.viteplus.dev/guide/migrate-rules)
- [Vite+: Commit Hooks](https://www.viteplus.dev/guide/commit-hooks)
- [oxc-project/oxlint-migrate](https://github.com/oxc-project/oxlint-migrate)

確認日: 2026-11-23。Vite+ 1.x、Oxlint・Oxfmt は 2026 年 11 月時点の版。 `vp migrate` の挙動は版で変わるので、実行前に `vp help migrate` で確かめる。
