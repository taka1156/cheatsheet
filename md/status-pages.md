<!-- scripts/memo.py build が自動生成。手で編集しない -->

# クラウド・SaaS ステータスページ早見表

> AWS・GCP・Azure・Cloudflare・GitHub・Slack・M365・Google Workspace・npm などの公式ステータスページと通知の受け方をまとめた障害時の早見表

- URL: https://cs.taka1156.site/status-pages.html
- タグ: セキュリティ
- 日付: 2026-11-22

---

障害かも？と思ったときに開く公式ステータスページ・自分の環境向けのヘルス画面・通知の受け方

**ざっくり言うと**

- 各社の公式ステータスページと、ログインして見る「自分の環境向け」のヘルス画面のリンク集。
- Azure・M365・GCP の公開ページは大規模障害が中心で、個別の影響はアカウント向けの画面に出る。
- 障害が起きてから探すのでは遅いので、RSS・アラート・Slack 通知を平時に設定しておく。

## まず開くページ

「公開」はログイン不要。「自分の環境向け」はアカウント・テナントへのログインと権限が必要。

| サービス | 公開ステータス | 自分の環境向け |
| --- | --- | --- |
| AWS | [AWS Health Dashboard – Service health](https://health.aws.amazon.com/health/status) | AWS Health Dashboard – Your account health（コンソール。公開ページから「Open your account health」） |
| Google Cloud | [Google Cloud Service Health](https://status.cloud.google.com/) | Personalized Service Health（コンソールの Service Health ダッシュボード） |
| Azure | [Azure status](https://azure.status.microsoft/) | Azure portal の Service Health（と Resource Health） |
| Cloudflare | [Cloudflare Status](https://www.cloudflarestatus.com/) （拠点ごとは [/locations](https://www.cloudflarestatus.com/locations) ） | なし（ダッシュボードの Notifications で個別のアラートは設定できる） |
| GitHub | [GitHub Status](https://www.githubstatus.com/) | なし |
| Slack | [Slack System Status](https://slack-status.com/) | なし |
| Microsoft 365 | [status.cloud.microsoft](https://status.cloud.microsoft/) （と X の @MSFT365Status） | [Microsoft 365 管理センター](https://admin.microsoft.com/) → 正常性 (Health) → サービス正常性 (Service health) |
| Google Workspace | [Google Workspace Status Dashboard](https://www.google.com/appsstatus/dashboard/) | 管理コンソール → ルール → Apps outage alert（アラートセンター・メール） |
| npm | [npm Status](https://status.npmjs.org/) | なし |
| Docker Hub | [Docker System Status](https://www.dockerstatus.com/) | なし |
| Vercel | [Vercel Status](https://www.vercel-status.com/) | なし |

## 障害かも？の確かめ方

1. **自分の側を切り分ける** 別の回線・別の端末・別のリージョンで再現するか。自分のデプロイや設定変更の直後なら、まずそちらを疑う。
2. **自分の環境向けの画面を見る** AWS・GCP・Azure・M365 は、影響を受けた顧客だけに出る通知がある。公開ページが緑でも、ここに出ていることがある。
3. **公開ステータスページを見る** 上の表から。アカウント向けの画面やコンソール自体が開けないときの「予備の窓口」でもある。
4. **依存先も見る** 自社サービスの多くは Cloudflare・AWS（特に us-east-1）・GitHub などに乗っている。直接使っていないつもりのサービスが原因のこともある。
5. **公式発表がまだなら外部の情報と突き合わせる** [Downdetector](https://cs.taka1156.site/md/downdetector.md) （利用者の報告）や [Cloudflare Radar](https://cs.taka1156.site/md/cloudflare-radar.md) （通信の観測）で、自分だけか全体かを確かめる。

## 通知を受ける・機械で読む

AWS・GCP・Azure

Cloudflare・GitHub・Slack

M365・Workspace

（元のページで操作できます → https://cs.taka1156.site/status-pages.html#notify）

### AWS

- 公開ページの各イベントの横の RSS で、サービス×リージョンごとに購読できる（例: `https://status.aws.amazon.com/rss/ec2-us-east-1.rss` ）。
- RSS の形式は変わることがあると公式が注意している。機械で取り込むなら Amazon EventBridge で AWS Health のイベントを受ける。
- 公開イベント（アカウントに依らないもの）は AWS User Notifications の管理通知には来ない。EventBridge のルールか、自分で設定する通知で受ける。

### Google Cloud

- 公開ページの RSS: `https://status.cloud.google.com/en/feed.atom` 。ページ下部から JSON の履歴も取れる。プログラムで扱うときは表示名ではなく製品 ID で判定する。
- Personalized Service Health は Service Health API を有効にして使う。アラートは Cloud Monitoring の通知チャネル（メール・Slack・PagerDuty・Pub/Sub・Webhook など）に送れる。

### Azure

- 公開ページはページ上部の RSS で購読できる。履歴ページでは過去の障害の事後レビュー (PIR) が読める。
- 本命は Service Health アラート（Azure Monitor のアクショングループでメール・SMS・Webhook など）。公開ページにはメールなどで知らせる機能がない（RSS を自分で購読するだけ）。

### Cloudflare

- 2026-08 にステータスページが作り直され、障害とメンテナンスが別のフィードになった（RSS・Atom）。
- ステータスページの通知はメール・Webhook・Slack・Discord・Google Chat で受けられる。Cloudflare 本体とは別の基盤から送られるので、Cloudflare が落ちていても届く。
- どのページも `Accept: text/markdown` を付けると Markdown で返る（AI エージェント向け）。

```
https://www.cloudflarestatus.com/api/v3/incidents.rss
https://www.cloudflarestatus.com/api/v3/maintenance.rss
curl -H "Accept: text/markdown" https://www.cloudflarestatus.com/locations
```

### GitHub

- Atlassian Statuspage で動いていて、Statuspage 共通の JSON API と履歴フィードがある。ページの「Subscribe to updates」でメールなども受けられる。

```
curl https://www.githubstatus.com/api/v2/summary.json   # 全体と各コンポーネント
curl https://www.githubstatus.com/api/v2/status.json    # 全体の状態だけ
https://www.githubstatus.com/history.rss
```

### Slack

- 認証なしの Status API と RSS・Atom がある。ページの「Subscribe to updates」でメール通知も受けられる。

```
curl https://slack-status.com/api/v2.0.0/current   # 進行中の障害
curl https://slack-status.com/api/v2.0.0/history   # 過去の障害
https://slack-status.com/feed/rss
```

Slack の障害時は Slack に流す通知も届かないことがある。Slack の状態だけはメールなど別の経路でも受けておく。

### Microsoft 365

- 個別の障害は管理センターのサービス正常性に出る。計画メンテナンスやセキュリティ関連はメッセージセンターに出る。
- Microsoft 365 Admin モバイルアプリでプッシュ通知を受けられる。
- 公開ページ `status.cloud.microsoft` は、管理センターやサービス正常性そのものが開けない障害のときに使われる。サインインの障害（Entra ID）は Azure status 側も見る。

### Google Workspace

- ダッシュボード下部の「RSS Feed」「JSON History」から購読・取り込みができる。Google は Gmail 自体が止まっても気づけるよう RSS の購読を勧めている。
- 管理コンソールの Apps outage alert（システム定義のルール）でアラートセンターやメールに通知できるが、RSS より遅れることがある。
- 履歴は過去 5 年分まで見られる（「View history」）。

## よくある誤解

✗ 誤解: 公開ステータスページが緑なら、クラウド側に問題はない

✓ 正しくは: Azure・Microsoft 365 は、影響を受けた顧客にだけ届く通知を主な連絡手段にしていて、公開ページに載るのは広い範囲の障害などに限られる。AWS・GCP も、個別の影響はアカウント向けの画面に出る。公開ページだけで判断しない。

✗ 誤解: 障害が起きたらステータスページを見に行けば十分

✓ 正しくは: 反映まで時間がかかることがあり、ページやコンソール自体が重くなることもある。平時に RSS・アラート・Webhook を設定し、障害を起こしているサービスとは別の経路（Slack の障害ならメールなど）で受けておく。

✗ 誤解: AWS の状態は RSS をスクレイピングすれば確実に取れる

✓ 正しくは: AWS は RSS の形式が変わりうると明記している。システムに組み込むなら EventBridge で AWS Health のイベントを受ける。

## 出典

- [AWS Health Dashboard（AWS Health ユーザーガイド）](https://docs.aws.amazon.com/health/latest/ug/aws-health-dashboard-status.html)
- [Use Google Cloud Service Health as a fallback](https://docs.cloud.google.com/support/docs/dashboard) ・ [Personalized Service Health overview](https://docs.cloud.google.com/service-health/docs/overview) ・ [Manage Personalized Service Health access](https://docs.cloud.google.com/service-health/docs/manage-access) ・ [Configure alerts in the Service Health dashboard](https://docs.cloud.google.com/service-health/docs/configure-alerts-dashboard)
- [Azure status overview](https://learn.microsoft.com/azure/service-health/azure-status-overview) ・ [When to use Azure Service Health versus the status page](https://azure.microsoft.com/en-us/blog/when-to-use-azure-service-health-versus-the-status-page/)
- [New Cloudflare Status page（Cloudflare changelog）](https://developers.cloudflare.com/changelog/post/2026-08-11-new-status-page/)
- [Slack Status API](https://docs.slack.dev/reference/slack-status-api)
- [How to check Microsoft 365 service health](https://learn.microsoft.com/microsoft-365/enterprise/view-service-health)
- [Check the status of a Google Workspace service](https://knowledge.workspace.google.com/admin/reports/check-the-status-of-a-google-workspace-service)

確認した日付: 2026-11-22。GitHub の API は Statuspage の共通仕様と第三者の資料による。npm・Docker Hub・Vercel の URL は第三者の監視サービスの記載で確かめた。URL は変わることがあるので、開けないときは各社のドキュメントで確かめる。
