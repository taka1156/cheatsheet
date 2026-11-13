<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Void のデプロイ先：直接 Cloudflare vs Void Platform

> Void アプリを自分の Cloudflare に直接出す場合と Void Platform に出す場合の違い、メリット・デメリット、選び方。

- URL: https://cs.taka1156.site/void-deploy-targets.html
- タグ: Void, Cloudflare
- 日付: 2026-11-13

---

同じ Void アプリを、自分の Cloudflare に出すか、チームの Void Platform に出すか

**ざっくり言うと**

- アプリのコードと API はどちらでも **同じ** 。変わるのは「誰の Cloudflare に、誰の権限で出すか」と、出したあとの管理のしかた
- **直接 Cloudflare** ：準備ゼロですぐ出せる。1 人で使う前提で、ログは流しっぱなし（保存しない）、利用者や上限の管理はない
- **Void Platform** ：管理者が基盤を入れる手間と費用がかかる代わりに、社員は Cloudflare の権限なしで出せて、ログ履歴・役割・上限・SSO を管理できる

Void 自体は → [Void チートシート](https://cs.taka1156.site/md/void.md) 、Platform の入れ方と管理コマンドは → [Void Platform チートシート](https://cs.taka1156.site/md/void-platform.md)

## 図でわかる：誰が何を持っているか

### 直接 Cloudflare

**開発者**

自分の Cloudflare にログイン（またはトークン）

↓ `void deploy --platform cloudflare`

**開発者の Cloudflare アカウント**

Worker ＋ D1・KV・R2 など。料金も上限もこのアカウント

### Void Platform

**開発者**

会社のログイン（GitHub・Google・OIDC・Access）だけ

↓ `void deploy --platform void`

**Void Platform** （管理者が運用）

利用者・プランの上限・ログ履歴・デプロイ履歴

↓

**会社の Cloudflare アカウント**

Workers for Platforms の dispatch namespace の中で、各アプリが Worker として動く

## どちらを選ぶか

1. **自分（か少人数）のアプリで、全員が Cloudflare の権限を持っていてよい？** → はい： 直接 Cloudflare
2. **社員や顧客に、Cloudflare の鍵を渡さずにアプリを出させたい？** → はい： Void Platform
3. **1 つのアプリを複数人で触る（閲覧だけの人・デプロイできる人を分けたい）？** → はい： Void Platform （直接デプロイには共同作業の機能がない）
4. **ログを後から遡りたい・人ごとに使用量の上限をかけたい？** → はい： Void Platform
5. 迷ったら 直接 Cloudflare で始める。コードは同じなので、あとから `void connect` で Platform に出し直せる（データやシークレットは移らないので入れ直す）

## 項目ごとの違い

準備・費用

開発者の操作

機能

運用・管理

| 項目 | 直接 Cloudflare | Void Platform |
| --- | --- | --- |
| 最初にやること | `void init` でデプロイ先に Cloudflare を選ぶだけ | 管理者が `void platform install` （ドメイン・ログイン・DB・トークンを設定） |
| 必要な Cloudflare の契約 | Workers Free の範囲で動く（Sandbox などは有料） | Workers for Platforms の有料プラン ＋ R2・Analytics Engine を有効化 |
| ほかに要るもの | なし | 基盤専用の空の PostgreSQL、ログインの提供元、アプリ用ドメイン（試すだけなら workers.dev） |
| 料金を払うのは | デプロイした人のアカウント | 基盤を入れた会社のアカウント（＋ DB の料金） |
| Workers AI の使用量 | 自分のアカウントの枠を使う | 基盤のアカウントの枠。人ごとの上限がかかることがある |

| 項目 | 直接 Cloudflare | Void Platform |
| --- | --- | --- |
| デプロイする人に必要なもの | Cloudflare アカウント（またはそのトークン） | Platform のアカウントだけ |
| つなぐ | `void connect --platform cloudflare` | `void connect platform.example.com` |
| デプロイ | `void deploy --platform cloudflare` | `void deploy --platform void --project my-app` |
| CI から | `void init --github` でワークフローを生成。 `CLOUDFLARE_API_TOKEN` を登録 | 自分で書いた CI ＋ プロジェクト専用トークン（ `void project token create` ）。 `VOID_TOKEN` ・ `VOID_API_URL` |
| 既存の Worker | `wrangler.jsonc` がある Worker を紐付けて引き継げる | — |
| 共同作業 | なし（1 人） | Reader・Collaborator・Project administrator・Owner の役割で招待 |

D1・KV・R2・Queues・cron・ISR・WebSocket・Durable Objects・静的サイト・SSR は **どちらも使える** 。違いがあるのは次のところ。

| 項目 | 直接 Cloudflare | Void Platform |
| --- | --- | --- |
| 実行ログ | ライブ tail（その場で流れるだけ） | 基盤に保存されたログ履歴を後から見られる |
| ロールバック | Worker のバージョンに戻す | 基盤に残したデプロイ履歴に戻す |
| カスタムドメイン | 使える | `--custom-domains` で有効にしたとき（インストール時か後から `void platform upgrade` 。アプリ専用ゾーンが必要） |
| Sandbox | Workers Paid と Docker が必要 | 基盤のアカウントが Workers Paid なら使える |
| アプリへのアクセス制限 | 自分で Cloudflare Access を設定 | インストール時に「公開／基盤と同じポリシー／別ポリシー」を選び、新しいアプリに自動で適用 |
| 使いすぎたとき | 自分のアカウントの上限・課金どおり | プランの上限を超えると 429 と制限ページ（ `public/usage-limit.html` で変更可） |
| キャッシュの一括削除 | — | `void project purge-cache` |
| ほかのアプリからの承認付きデプロイ | — | Connected apps（OAuth）とプライベートプレビュー |

| 項目 | 直接 Cloudflare | Void Platform |
| --- | --- | --- |
| 誰が管理するか | 各自 | 基盤の管理者（ `void platform …` ） |
| 利用者の管理 | — | サインアップの許可リスト・招待・停止 |
| 上限 | — | プランごとのリソース上限（課金はしない） |
| 監視 | Cloudflare のダッシュボード | `void platform system health` ・操作の記録・デプロイのログ |
| 基盤自体の保守 | 不要 | アップグレード・修復・トークン更新・停止／削除を管理者が行う |
| 画面 | Cloudflare のダッシュボード | 管理者用の画面あり。利用者用ダッシュボードと GitHub 自動ビルドは標準に無い（基盤をカスタムビルド） |

（元のページで操作できます → https://cs.taka1156.site/void-deploy-targets.html#compare）

## メリット・デメリット

### 直接 Cloudflare

メリット

- 準備がいらず、 `void deploy` ですぐ出せる
- 基盤のための追加の契約・DB・運用がない（Workers Free でも動く）
- 既存の Worker と `wrangler.jsonc` を引き継げる
- GitHub Actions のワークフローを自動で作れる
- Cloudflare のアカウントを自分で直接触れる

デメリット

- デプロイする人全員に Cloudflare の権限が要る
- メンバー招待・役割などの共同作業ができない
- ログは流れるだけで保存されない
- 人ごとの上限や利用状況の管理がない

### Void Platform

メリット

- 社員に Cloudflare の鍵を渡さずにデプロイさせられる
- 会社の SSO（Google・OIDC・Cloudflare Access）でログイン
- プロジェクトごとに役割を分けて共同作業できる
- ログ履歴とデプロイ履歴が残り、後から調べられる
- プランで人ごとの上限をかけ、使いすぎを止められる
- 新しいアプリに自動でアクセス制限をかけられる

デメリット

- Workers for Platforms の有料プランと PostgreSQL が必要
- インストールの手順が多い（ドメイン・ログイン・トークン）
- 基盤のアップグレード・修復を管理者が続ける必要がある
- 利用者用ダッシュボードと GitHub 自動ビルドは標準に無い
- カスタムドメインや Sandbox は設定・契約しだい

## 変わらないこと・注意点

- アプリのコード・ `void/db` などの API・ファイル規約は同じ。 `void project status` ・ `logs` ・ `rollback` 、 `void secret` などのコマンドも同じで、つないでいる先に対して動く
- デプロイ先ごとにプロジェクト・シークレット・データは別々。乗り換えるときは自動では移らない
- どちらでも、ロールバックで DB のマイグレーションは戻らない
- 保存先を変えずに 1 回だけ別の Platform に出すなら `VOID_API_URL=https://… void deploy --platform void --project my-app`

## 出典

- [Install a Void Platform](https://void.cloud/guide/self-hosted-platform) （Deployment features の比較表）・ [Prerequisites](https://void.cloud/guide/platform/installation/prerequisites)
- [Deployment](https://void.cloud/guide/deployment) ・ [Cloudflare](https://void.cloud/integrations/cloudflare) （直接デプロイ・CI・Scope and limitations）
- [Project Collaboration](https://void.cloud/guide/project-collaboration) ・ [AI](https://void.cloud/guide/ai) （Usage Limits）
- [voidzero-dev/void](https://github.com/voidzero-dev/void) （ `docs/` 、 `void` 1.8.0 時点）

確認日: 2026-11-13
