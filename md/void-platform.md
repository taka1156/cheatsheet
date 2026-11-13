<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Void Platform チートシート

> 会社の Cloudflare に入れる Void の共有デプロイ基盤 Void Platform の仕組み、インストール、開発者と管理者の使い方。

- URL: https://cs.taka1156.site/void-platform.html
- タグ: Void, Cloudflare
- 日付: 2026-11-13

---

会社の Cloudflare アカウントに入れる、Void アプリ用の共有デプロイ基盤

**ざっくり言うと**

- 管理者が自分たちの Cloudflare アカウントに 1 度インストールすると、社員は Cloudflare の権限なしで `void connect` → `void deploy` だけでアプリを公開できる
- ログイン方法（GitHub・Google・OIDC・Cloudflare Access）、誰が使えるか、プランごとの上限、ログ・ロールバック・利用状況を管理者がまとめて管理する
- フレームワーク（Void SDK）とは別の、 **任意の** 部品。1 人で使うなら不要で、自分の Cloudflare に直接デプロイすればよい

フレームワーク側（ルート・ページ・DB・ `void` CLI の基本）は [Void チートシート](https://cs.taka1156.site/md/void.md) 。

## 何のためのものか

AI でサッと作った社内アプリ（バイブコーディング）を社員に公開させたいが、全員に Cloudflare アカウントの鍵は渡したくない、というときのための基盤。当初は全ユーザーが使う 1 つの共有プラットフォームとして作る予定だったが、Cloudflare 合流後に方針を変え、誰でも自分の Cloudflare に入れられるオープンソース（MIT）にした。

**管理者** ： `void platform install` で会社の Cloudflare アカウントに基盤を入れる。ログイン方法・利用できる人・プランを決める

↓ API の URL を共有

**開発者（社員）** ： `void connect <API URL>` → 会社のログインでサインイン → `void deploy --platform void`

↓

各アプリ

Workers for Platforms の dispatch namespace の中で、各アプリが Worker として動く。 `my-app.example.app` のような URL

基盤

API・プロキシ・ログ収集などの Worker、ユーザー・プロジェクト・デプロイ履歴を持つ PostgreSQL、利用量の記録

## 直接 Cloudflare と Platform の違い

アプリのコード（API）はどちらでも同じ。違うのはデプロイ先の管理のしかた。メリット・デメリットと選び方は → [直接 Cloudflare vs Void Platform](https://cs.taka1156.site/md/void-deploy-targets.md)

| 項目 | 直接 Cloudflare ／ Void Platform |
| --- | --- |
| 向いている人 | 個人のアプリ ／ チーム・会社・顧客に使わせる |
| デプロイする人に必要なもの | Cloudflare アカウント ／ Platform のログインだけ |
| D1・KV・R2・Queues・cron・ISR・WebSocket | どちらも使える |
| Workers AI | 自分のアカウントと Gateway ／ 基盤の Gateway 経由（使用量を計測） |
| 実行ログ | Cloudflare のライブ tail ／ 基盤に保存されたログ |
| ロールバック | Worker のバージョン ／ 基盤に残したデプロイ履歴 |
| プロジェクトの共同作業（メンバー招待） | なし（1 人） ／ あり |
| カスタムドメイン | 使える ／ `--custom-domains` で有効化（インストール時か、後から `void platform upgrade` で。アプリ専用ゾーンと Cloudflare for SaaS の設定が必要） |
| GitHub からのデプロイ | `void init --github` のワークフロー ／ 自分の CI ＋ プロジェクト用トークン |

ユーザー用ダッシュボードと GitHub の自動ビルドは標準のインストールに含まれず、基盤を改造してビルドする（カスタムビルド）必要がある。管理者用の画面は標準で付く。

## インストールに必要なもの

| もの | メモ |
| --- | --- |
| Cloudflare アカウント | Workers for Platforms（有料プラン）・R2・Analytics Engine を有効にしておく |
| 空の PostgreSQL | 基盤専用（ユーザー・プロジェクト・デプロイを保存）。Neon・PlanetScale Postgres など。 直接接続かセッションモード （トランザクションプーリングは不可）。各アプリは D1 を使ってよい |
| ログインの提供元 | GitHub・Google・汎用 OIDC・Cloudflare Access から選ぶ（複数可） |
| アプリ用のドメイン | 例 `example.app` → アプリは `my-app.example.app` 。試すだけなら `workers.dev` で可 |
| Node.js | 24.22 以上 または 26.12 以上 |

Cloudflare と DB の利用料はそれぞれ別にかかる。Platform の「プラン」は上限を決めるだけで、課金はしない。

## インストール（管理者）

```
pnpm add --global void

void platform install --plan            # 作るものを確認するだけ（何も変えない）
void platform install                   # 保存したプランを選んで実行
#   --application-domain example.app    # 独自ドメインで
#   --workers-dev                       # 試すだけ
#   --runtime-logs minimal              # URL やメッセージをログに残さない
```

1. インストール名（例 `team` → `void-team-api` などの名前になる）、ドメイン、表示名を答える
2. ログイン方法と「デプロイしたアプリを誰が見られるか」（公開／基盤と同じ Access ポリシー／別のポリシー）を選ぶ
3. Cloudflare のトークン・PostgreSQL の URL・ログインの認証情報を入力（秘密の値はキーチェーンに暗号化して保存）
4. 表示された `/setup` の URL とワンタイムコードで最初の管理者を確定
5. DNS 待ちなどで止まったら `void platform install --resume --name <id>`

### 作られる主なリソース

| リソース | 数・用途 |
| --- | --- |
| Workers | 5（API・プロキシ・ログ取り込み・ディスパッチ・メール）＋ workers.dev のアプリごとに 1 |
| Workers for Platforms 名前空間 | 1（ユーザーのアプリが入る） |
| KV ・ R2 ・ Queues | 3（ルーティング・ISR・静的ファイル） ・ 1（アセット） ・ 3（使用量・cron・デプロイ） |
| Hyperdrive ・ AI Gateway | PostgreSQL への接続 ・ AI の振り分けと計測 |
| DNS ・ ゾーン | 独自ドメインのときだけワイルドカードレコード（ゾーンが無ければ作る） |

## 使う（開発者）

```
# 管理者から聞いた API の URL につなぐ（初回はブラウザでサインイン）
void connect platform.example.com
void deploy --platform void --project my-app    # 初回はプロジェクト作成を確認
void project link                               # 既存のプロジェクトに紐付けるとき

void project status ・ logs ・ rollback ・ cancel
```

デプロイは基盤が受け付けたあとは CLI を閉じても続く。認証情報はプラットフォームの URL ごとにキーチェーンへ保存。 `void platform list` ・ `void platform use [id]` で接続先を切り替え。

### チームで使う（プロジェクトのメンバー）

| 役割 | できること |
| --- | --- |
| Reader | プロジェクト・使用量・デプロイ・ログ・メンバーを見る（デプロイ不可） |
| Collaborator | ＋ デプロイ・ロールバック・マイグレーション・シークレット・DB 設定 |
| Project administrator | ＋ ドメイン・メール宛先・GitHub 設定・招待・役割の変更 |
| Owner | ＋ プロジェクト削除・CI 用トークンの発行。上限はオーナーのプランで決まる |

```
void project team invite teammate@example.com --role collaborator   # 登録済みのユーザーだけ
void project team pending  ・  accept <id>  ・  decline <id>          # 招待された側
void project team list  ・  role <user-id> reader  ・  remove <user-id>
```

### CI からデプロイ

```
# オーナーがプロジェクト専用・期限付きのトークンを作る
void project token create --name ci --expires-in 30
# → VOID_TOKEN と VOID_API_URL を CI のシークレットに

# CI の中
void connect "$VOID_API_URL" --no-login
void deploy --platform void --project "$VOID_PROJECT"
```

人のログイントークンは CI に使わない。更新は `void project token renew <id>` 、失効は `revoke` 。Access で守っている基盤では `VOID_ACCESS_CREDENTIALS` も要る。

## 管理者コマンド

管理者のログインはデプロイ用とは別（ `void platform auth login` 、1 時間）。変更系は `--plan` で確認、 `--yes` で確認なしに適用。全コマンドに `--json` 。ブラウザなら `<API URL>/admin/login` 。

利用者

プラン

運用・ログ

保守・停止

| コマンド | やること |
| --- | --- |
| `void platform signup allow github teammate` | GitHub ユーザーのサインアップを許可 |
| `void platform signup allow email '*@example.com'` | メールアドレス・ドメインで許可 |
| `void platform signup show` | 許可リストを見る（ `open` ・ `restrict` で開放／制限） |
| `void platform invitation send alex@example.org` | 招待を送る |
| `void platform user list` ・ `show <id>` | ユーザー一覧・詳細 |
| `void platform user suspend <id> --reason "…"` | ユーザーを停止（ `restore` で戻す、 `delete` で削除） |
| `void platform project list` ・ `owner` | プロジェクト一覧・オーナーの移譲 |
| `void platform zero-trust configure` | プロジェクトのホスト名を Cloudflare Access で守る設定（ `--protect-new-projects` で新しいアプリに既定でかける。各プロジェクトで公開に戻せる） |

プラン ＝ アカウントごとのリソース上限（課金ではない）。変更に再デプロイは不要。

| コマンド | やること |
| --- | --- |
| `void platform config plans` | プランを見る |
| `void platform config plans add team --copy pro --name "Team"` | 既存プランをコピーして追加 |
| `void platform config plans set team --file team-plan.json` | 上限を編集 |
| `void platform config plans default team` | 新規アカウントの既定にする |
| `void platform user plan <user-id> team` | ユーザーにプランを割り当て |
| `void platform config plans archive` ・ `remove --replacement pro` | アーカイブ ・ 代わりを決めて削除 |

| コマンド | やること |
| --- | --- |
| `void platform system health` ・ `overview` | 基盤の状態（異常なら終了コードが 0 以外） |
| `void platform system events` | 操作の記録（変更が途中で切れたら、繰り返す前にここを見る） |
| `void platform deployment list --project <id>` | デプロイ一覧 |
| `void platform deployment logs <id> --since 10m --follow` | アプリのログを追う |
| `void platform deployment cancel <id>` | デプロイを取り消す |
| `void platform project ai-logging` | プロジェクトごとの AI Gateway のログ記録 |

| コマンド | やること |
| --- | --- |
| `void platform status [id]` | インストールの状態 |
| `void platform upgrade [id]` | 新しい版に上げる（ `--runtime-logs minimal\|full` でログ設定も変更） |
| `void platform repair [id]` | 消えたリソースを作り直す |
| `void platform rollback [id] --runtime <path>` | 基盤自体を前の版に戻す |
| `void platform discover --account <id> --installation <name>` | 別のマシンから管理する |
| `void platform disable [id]` ・ `enable [id]` | データを残したまま止める ・ 再開 |
| `void platform uninstall [id]` | トラフィックを止めて削除（既定では多くのリソースと PostgreSQL・ゾーンは残る。 `--purge-resources` で DNS・Access なども消す） |

（元のページで操作できます → https://cs.taka1156.site/void-platform.html#admin）

## 発展：Connected apps

別のアプリ（例：社内のアプリ生成ツール）を OAuth クライアントとして登録すると、ユーザーは Void アカウントでそのアプリにサインインでき、そのアプリが作ったコードを **ユーザーが Void 上で承認して** 自分のプロジェクトにデプロイできる。管理画面の **Admin → Connected apps** で登録。許可すれば本番前のプライベートプレビューも作れる。

## よくある誤解

✗ 誤解: Void を使うには Platform が必須

✓ 正しくは: `void deploy --platform cloudflare` で自分のアカウントに直接デプロイできる。Platform はチームで共有したいときの任意の部品。

✗ 誤解: Platform のプランで利用者に課金できる

✓ 正しくは: できない。プランはリソースの上限を決めるだけ。Cloudflare と DB の料金は基盤を入れた側にかかる。

✗ 誤解: アンインストールすれば全部消える

✓ 正しくは: 既定では Workers・KV・R2・Hyperdrive などは確認用に残り、外部の PostgreSQL とゾーンは常に残る。入れ直すときは新しい空の PostgreSQL を使う。一時的に止めるだけなら `disable` 。

✗ 誤解: 公開前に IT 部門が審査する仕組みがある

✓ 正しくは: ドキュメント上、デプロイ前の必須の審査ステップは無い。サインアップの許可リスト、アプリへの Access ポリシー、プランの上限で絞る。必要なら基盤をフォークして足す。

## 出典

- [Install a Void Platform](https://void.cloud/guide/self-hosted-platform) （Prerequisites・Install and Configure Login・First Deployment・Domains and Resources・Maintenance・Disable and Uninstall）
- [Platform Administration](https://void.cloud/guide/platform-administration) （Access・Plans and Limits・Operations・Connected apps）と [Project Collaboration](https://void.cloud/guide/project-collaboration)
- [CLI Reference](https://void.cloud/reference/cli) （ `void platform` ）、 [voidzero-dev/void](https://github.com/voidzero-dev/void) （ `void` 1.8.0 時点）
- [Deploy with Void](https://blog.cloudflare.com/deploy-with-void/) （Cloudflare Blog, 2026-10-15）
- [The New Stack](https://thenewstack.io/?p=22840734) （2026-10-16。セルフホスト化の経緯と、デプロイ前の審査について）

確認日: 2026-11-13
