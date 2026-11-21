<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Cloudflare Radar 使い方

> Cloudflare Radar の画面ごとの見どころ、URL Scanner、Radar API、通知、AI からの使い方をセキュリティ調査の目線でまとめたチートシート

- URL: https://cs.taka1156.site/cloudflare-radar.html
- タグ: Cloudflare, セキュリティ
- 日付: 2026-11-22

---

インターネット全体のトラフィック・攻撃・経路・脆弱性を Cloudflare の観測データで見る無料のハブ。画面・CVE ページ・URL Scanner・API・通知・AI 連携をまとめる（2026-11 時点）

**ざっくり言うと**

- Radar は Cloudflare のネットワークと 1.1.1.1 の集計データで、ネット全体の通信・攻撃・経路を見せる無料サイト。
- 画面で見る・URL Scanner で怪しい URL を調べる・API で取り出す・通知で受け取る、の 4 通りで使う。
- 見えるのは Cloudflare から観測した割合や傾向で、絶対量や自社サイトの状況ではない点に注意する。

## 全体像

**Cloudflare のネットワーク** HTTP リクエスト・NetFlows・攻撃の緩和・メール

**1.1.1.1（DNS）** 匿名化・集計した問い合わせ。ドメインランキングの元

**外部データ** RouteViews・RIPE RIS（BGP）、CT ログ、CVE・CISA KEV など

↓

**Cloudflare Radar** （radar.cloudflare.com）

↓

**画面・URL Scanner** ブラウザで見る・URL を調べる。ログイン不要

**Radar API** 同じデータを JSON / CSV で。トークンが要る

**通知・AI** Notifications、Radar Researcher、MCP サーバー

## 画面の見どころ

上部の検索バー（ `Cmd` / `Ctrl` + `K` ）に国名・ASN・IP・ドメイン・CVE を入れると、その詳細ページへ飛べる。各グラフのメニューから PNG 保存・埋め込み・引用（BibTeX など）・Data Explorer で開く、ができる。

セキュリティ

経路・障害

トラフィック・AI

| 画面 | URL | 見えるもの・使いどころ |
| --- | --- | --- |
| Zero Wire | `/security/zero-wire` | 重大な脆弱性・脅威のブリーフィング。「Needs attention」（今見るべき）と「Reference」（落ち着いたもの）に分かれ、対処手順・Cloudflare での観測・出典が載る。RSS あり |
| CVE ページ | `/zero-wire/cves` | CVE ごとに CVSS・CISA KEV（悪用確認済み）・EPSS（悪用確率）・影響製品・対処・CWE を 1 ページに集約。 `/zero-wire/cves/CVE-2021-44228` のように開く |
| Security | `/security` | L7（HTTP）・L3/4（DDoS）攻撃の傾向、攻撃元・攻撃先の国、緩和に使われた製品、悪性メールの割合など |
| Certificate Transparency | `/certificate-transparency` | CT ログから見た証明書発行の傾向（CA・CT ログごとの割合、有効期間、ワイルドカードの割合など） |
| Post-Quantum | `/post-quantum` | ポスト量子暗号（X25519MLKEM768）の普及率。指定したサイトのポスト量子対応と TLS 実装の不具合を調べるツールもある |
| Adoption & Usage | `/adoption-and-usage` | HTTP/3・IPv6 などの採用状況 |

| 画面 | URL | 見えるもの・使いどころ |
| --- | --- | --- |
| Routing | `/routing` | BGP の統計、IP アドレス空間の推移、Top 100 AS（顧客コーン・アドレス数） |
| RPKI | `/routing/rpki` | RPKI（ROA）の検証状況と ASPA の導入状況。AS ごとの ASPA プロバイダも見られる |
| Anomalies | `/routing/anomalies` | BGP ハイジャック（経路の乗っ取り）と経路リークの検知結果。自社の ASN が巻き込まれていないか確かめる |
| ASPA 検証 | `/routing/aspa-validation` | AS パスを入れると ASPA で Valid / Invalid / Unknown を判定する |
| MRT Explorer | `/routing/mrt-explorer` | BGP のダンプ（MRT ファイル）をブラウザ内だけで解析する。ファイルはアップロードされない |
| Outage Center | `/outage-center` | 国・AS 単位のネット障害と通信の異常。原因（停電・ケーブル切断・政府による遮断など）・範囲・期間の記録 |

| 画面 | URL | 見えるもの・使いどころ |
| --- | --- | --- |
| Traffic | `/traffic` ・ `/traffic/jp` | 国・地域・AS ごとの通信量の推移、人とボットの割合、デバイス・プロトコル、API 通信の割合など |
| Domains | `/domains` | 1.1.1.1 の問い合わせに基づくドメインランキング（上位 100、急上昇、100 万件までの区切り） |
| AI Insights | `/ai-insights` | AI クローラー・AI ボットの通信、robots.txt での扱い、Workers AI のモデル・タスクの人気 |
| Bots Directory | `/bots/directory` | BotBase を元にしたボット・AI エージェントのディレクトリ。運営者・カテゴリ・アクセスの性質・Web Bot Auth での検証状況を調べられる |

URL は `https://radar.cloudflare.com` からの相対パス。国のページは ISO 3166 の 2 文字コード（日本は `jp` ）。

（元のページで操作できます → https://cs.taka1156.site/cloudflare-radar.html#pages）

## やりたいこと別の逆引き

| やりたいこと | どこを見るか |
| --- | --- |
| 届いたリンクが安全か調べたい | URL Scanner（ `/scan` ）。Unlisted で調べる（下の「URL Scanner」） |
| 今どの脆弱性に急いで対処すべきか知りたい | Zero Wire の「Needs attention」と、CVE ページの KEV・EPSS（下の「CVE ページ」） |
| 使っている製品の新しい CVE を追いたい | CVE 一覧を製品名で検索、または CVE フィードを `?severity=critical` 付きで購読 |
| つながらないのが自分だけか、地域全体か知りたい | Outage Center と、その国・AS の Traffic |
| 自社の IP 範囲が乗っ取られていないか見張りたい | Routing の Anomalies ＋ 通知（route hijack / route leak） |
| 知らない証明書が発行されていないか確かめたい | Radar の Certificate Transparency は発行の傾向まで。自分のドメインの証明書を個別に調べるなら crt.sh などの CT 検索サービスを使う |
| 怪しい IP・AS がどこの何者か知りたい | 検索バーに IP / ASN を入れる（所属 AS・国・AS の種類・上流プロバイダ） |
| レポートに使う数字やグラフがほしい | グラフのメニューから PNG・埋め込み・引用。数値は API で（ライセンスは下を参照） |

## CVE ページ（Zero Wire）

公開済みのすべての CVE に 1 ページずつあり、深刻度・悪用状況・影響製品・対処を NVD・CISA・FIRST などの情報からまとめて見られる（2026-11-20 追加）。Zero Wire のブリーフィングが引用した CVE は CVE ページへリンクし、CVE ページからもそのブリーフィングへ戻れる。

開き方・追い方

ページの読み方

優先度の付け方

| やること | URL・操作 |
| --- | --- |
| 1 件を開く | `/zero-wire/cves/CVE-2021-44228` （Log4Shell の例）。検索バー・ `Cmd` / `Ctrl` + `K` に CVE ID や語を入れてもよい |
| 一覧で探す | `/zero-wire/cves` 。日付・CVSS で並べ替え、CVE ID や製品名・ベンダー名で検索、深刻度・年で絞り込み。行には KEV 掲載と修正の有無が印で出る |
| 新しい CVE を購読する | RSS `/zero-wire/cves/feed` ・Atom `?format=atom` ・JSON `?format=json` |
| 重大なものだけ購読する | フィードに `?severity=critical` を付ける（ `?format=json&severity=critical` のように併用） |
| 解説つきのまとめを追う | Zero Wire の RSS `/security/zero-wire/rss` （CVE フィードより件数が少なく、重大なものに絞られる） |

CVE ページ用の API は公開されていない（2026-11 時点）。プログラムで扱うなら JSON フィードを使う。

| ブロック | 載っているもの | 読み方 |
| --- | --- | --- |
| ヘッダー | CVSS スコア・説明・公開日などの日付 | まず何の製品の何の脆弱性かを把握する |
| Exploitation | CISA KEV の掲載（ランサムウェアでの利用も）・CISA SSVC・FIRST EPSS | 「実際に悪用されているか・されそうか」。優先度はここで決まる（次のタブ） |
| Affected products | ベンダー・製品・バージョン（CVE を出した組織と NVD の両方） | 自分の使っている版が範囲に入るかを確かめる。両者が食い違うことがあるので、ベンダーの勧告で最終確認する |
| Recommended actions | 修正版、または当面のリスク低減策。KEV 掲載なら CISA の指示 | すぐ更新できないときの回避策もここで探す |
| Severity | すべての CVSS スコアと各指標の説明 | 攻撃元区分（ネットワーク越しか）・認証の要否・利用者の操作の要否を見ると、自分の環境での危なさが分かる |
| Weaknesses | CWE（弱点の種類）と CAPEC（攻撃パターン） | 同じ種類の弱点がほかにないか探す手がかり |
| References | 勧告・パッチ・その他の出典 | 一次情報（ベンダーの勧告）にたどる |

CVSS は「起きたらどれだけ深刻か」で、「狙われているか」は表さない。Exploitation の 3 つで実際の危なさを足して判断する。

| 指標 | 出しているところ | 意味 |
| --- | --- | --- |
| KEV | CISA | 実際に悪用が確認された脆弱性の一覧。載っていれば「もう狙われている」 |
| SSVC | CISA | 悪用状況・技術的な影響・自動化できるか・組織での使われ方・公共への影響の 5 つから、対応の判断を Track・Track*・Attend・Act（右ほど急ぐ）で出す枠組み |
| EPSS | FIRST | 今後 30 日以内に悪用される確率の推定（0〜1） |
| CVSS | CVE を出した組織・NVD など | 脆弱性そのものの深刻度（0〜10）。出す組織によって値が違うことがある |

1. **使っているか確かめる** Affected products に自分の製品・版が入っていなければ、ほかの指標が高くても対象外。
2. **KEV に載っているか** 載っていれば最優先。ランサムウェアでの利用ありなら、なおさら急ぐ。
3. **SSVC と EPSS を見る** Act / Attend や EPSS が高いものを、KEV 未掲載の中で先にする。
4. **CVSS の指標で自分の環境に当てはめる** ネットワーク越しに認証なしで突けるか、外に公開しているか、で順番を調整する。
5. **Recommended actions で直す** 修正版に上げる。すぐ上げられなければ回避策と、WAF などでの防御（Zero Wire の Cloudflare coverage）を確かめる。

（元のページで操作できます → https://cs.taka1156.site/cloudflare-radar.html#cves）

## URL Scanner

URL をサンドボックスのブラウザで開き、スクリーンショット・通信の記録（HAR）・読み込んだ外部資源・使っている技術・悪性判定（ `verdicts.malicious` ）をレポートにする。Radar の `/scan` 、ダッシュボードの Security Center、API のどれからでも使える。

画面で使う

API で使う

検索の書き方

上限

1. **URL を入れる** `radar.cloudflare.com/scan` に URL を貼る。社内の URL や個人情報を含む URL なら、公開範囲を Unlisted にする。
2. **レポートを読む** Summary（判定・最終的な URL・IP・AS）、Security、Network（リクエスト一覧）、Behavior、Indicators のタブを見る。
3. **保存・共有する** Save as PDF で全タブを 1 つの PDF に、JSON でも保存できる。「Explain with AI」でレポートの読み方を AI に聞ける。

**POST v2/scan** URL を送る。 `uuid` が返る

**GET v2/result/{uuid}** 終わるまで 404。10〜30 秒おきに確かめる

**200 で完了** レポートの JSON。スクリーンショット・HAR は別のエンドポイント

```
# トークン: Account > URL Scanner を Edit で作る
curl "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/urlscanner/v2/scan" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com",
    "visibility": "Unlisted",
    "screenshotsResolutions": ["desktop", "mobile"],
    "customagent": "my-scanner",
    "referer": "https://mail.example.com"
  }'
```

| オプション | 意味 |
| --- | --- |
| `visibility` | `Public` （既定。最近のスキャン一覧と検索に載る）か `Unlisted` （ID を知る人だけが見られる） |
| `screenshotsResolutions` | `desktop` ・ `mobile` ・ `tablet` から複数 |
| `customagent` ・ `referer` | User-Agent と Referer を変える（スマホ向けや、メール経由のときだけ出るページを見る） |
| `customHeaders` | 任意のヘッダー（ `Authorization` など）を付ける |
| `v2/bulk` | 最大 100 件をまとめて送る。優先度が低く、終わるのが遅くなることがある |

`GET v2/search?q=...` （画面の検索も同じ書き方）。フィッシングの同類探しに使える。

| クエリ | 見つかるもの |
| --- | --- |
| `page.domain:"example.com"` | そのドメインのページのスキャン |
| `domain:cdnjs.cloudflare.com` | そのホストに通信したスキャン |
| `page.url:"https://example.com" AND NOT task.url:"https://example.com"` | 別の URL から example.com へリダイレクトしたもの |
| `date:>now-7d AND path:okta-sign-in.min.js` | 直近 7 日で、そのパスの資源を読み込んだもの（偽のログイン画面探し） |
| `hash:<domStructHash>` | HTML の構造が同じページ（同じフィッシングキットの使い回し） |

| プラン | 検索できる履歴 | Public / 月 | Unlisted / 月 | レート |
| --- | --- | --- | --- | --- |
| Free / Radar | 直近 50 件 | 5,000 | なし | 10 秒に 1 回 |
| Self serve（有料） | 30 日 | 5,000 | 500 | 10 秒に 1 回 |
| Enterprise | 12 か月 | 10,000 | 5,000 | 毎秒 12 回 |
| Cloudforce One | 無制限 | 75,000 | 20,000 | 毎秒 12 回 |

成功したスキャンは 12 か月保存、30 日を過ぎた失敗スキャンは削除される。失敗したスキャンや、個人情報らしきものを含むスキャンは自動で Unlisted になる。特定の国からのスキャンは Enterprise のみ。

（元のページで操作できます → https://cs.taka1156.site/cloudflare-radar.html#scanner）

## Radar API

画面と同じデータを取り出せる無料の API。トークンは「Custom Token」で **Account > Radar** を **Read** にして作る。ベース URL は `https://api.cloudflare.com/client/v4/radar/` （アカウント ID は要らない）。

```
curl "https://api.cloudflare.com/client/v4/radar/http/summary/device_type?dateRange=7d&format=json" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

共通パラメータ

よく使うエンドポイント

比較する

| パラメータ | 意味 |
| --- | --- |
| `dateRange` | `7d` など。 `7dControl` はその前の 7 日（比較用） |
| `dateStart` ・ `dateEnd` | UTC の日時（ `2026-11-01T00:00:00Z` ）。URL エンコードする |
| `location` ・ `asn` | 国（ `JP` ）や AS（ `13335` ）で絞る |
| `aggInterval` | 時系列の粒度 `15m` ・ `1h` ・ `1d` ・ `1w` |
| `botClass` | `LIKELY_AUTOMATED` （ボット）・ `LIKELY_HUMAN` （人） |
| `format` | `json` か `csv` （pandas などにそのまま読ませる） |
| `name` | 系列に名前を付ける。返り値のキーになる（下の「比較する」） |

返り値の `meta` に、実際に使われた期間・粒度・信頼度（ `confidenceInfo` ）が入る。 `7d` のような指定は実行時に日付へ変わるので、何の期間だったかは `meta` で確かめる。

| パス（/radar 以下） | 取れるもの |
| --- | --- |
| `/attacks/layer7/timeseries` ・ `/summary/{dimension}` | L7 攻撃の推移・内訳（ `layer3` にすると DDoS） |
| `/annotations/outages` | Outage Center の障害（場所・原因・範囲・期間） |
| `/traffic_anomalies` | 障害の兆候かもしれない通信の異常 |
| `/bgp/hijacks/events` | BGP ハイジャックの検知。 `involvedAsn` ・ `minConfidence=8` （高信頼のみ） |
| `/bgp/leaks/events` | 経路リークの検知（今はプロバイダ→顧客→プロバイダ型のみ） |
| `/bgp/rpki/aspa/snapshot` | ASPA の登録状況 |
| `/ct/timeseries` ・ `/ct/summary/{dimension}` | 証明書発行の推移・CA ごとの内訳 |
| `/post_quantum/tls/support` | 指定したホストのポスト量子 TLS 対応と、TLS 実装の不具合 |
| `/netflows/timeseries` | ルーターで見た通信量の推移（正規化した値） |

ハイジャックの `confidence_score` は 1〜3 が低、4〜7 が中、8 以上が高。全エンドポイントは API リファレンスを参照。

`name` で系列を始め、続くパラメータがその系列に付く。並び順で系列が決まる。

```
# 日本とアメリカの直近 7 日を 1 回で
/radar/netflows/timeseries?name=jp&dateRange=7d&location=JP&name=us&dateRange=7d&location=US&format=json

# 今週と先週
/radar/netflows/timeseries?name=this_week&dateRange=7d&location=JP&name=last_week&dateRange=7dControl&location=JP
```

NetFlows のように最小・最大で正規化する値は、別々のリクエストで取った数字どうしを比べられない。比べたい系列は 1 回のリクエストにまとめる。

（元のページで操作できます → https://cs.taka1156.site/cloudflare-radar.html#api）

## 通知で見張る

ダッシュボードの Notifications で Radar の通知を作ると、指定した国・地域・ASN に変化があったときに知らせてくれる。種類は anomaly（通信の異常）・outage（障害）・route hijack（経路の乗っ取り）・route leak（経路リーク）の 4 つ。送り先はメール・Webhook・PagerDuty。

細かく制御したいときは、Workers で API を定期的に叩く方法が公式ドキュメントに載っている。

**Cron Trigger** 例: 5 分ごと

**/bgp/hijacks/events** `involvedAsn` に自社 ASN、ID の降順

**KV と比べる** 前回処理した ID より新しいものだけ

**通知** Webhook（Discord など）やメール

## AI から使う

| 方法 | 使い方 |
| --- | --- |
| Radar Researcher（ベータ） | Radar のヘッダーから開く AI アシスタント。質問に、API のデータからグラフを作って答える。会話は共有リンクにできる |
| Explain with AI | グラフのメニューや URL Scanner のレポートから、そのデータの意味を聞く |
| MCP サーバー | 公式が勧めるのは統合された Cloudflare API MCP サーバー `https://mcp.cloudflare.com/mcp` （Cloudflare の OAuth で認証）。Radar API もここから呼べる。以前の Radar 専用サーバー `radar.mcp.cloudflare.com` は公式の一覧から外れている |
| WebMCP | ブラウザで動く AI エージェントが、Radar のページ上で検索や URL スキャンを使える |

## データの使い方（ライセンス）

| もの | ライセンス |
| --- | --- |
| 埋め込み・ダウンロードしたグラフや画像 | CC BY 4.0（出典を書けば商用も可） |
| API や直接ダウンロードで得たデータ | CC BY-NC 4.0（非営利のみ。例外は radar@cloudflare.com に相談） |
| Radar のレポートの一部引用 | タイトル・リンク・出典が Cloudflare であることを書く |

## よくある誤解

✗ 誤解: Radar を見れば自分のサイトへの攻撃状況が分かる

✓ 正しくは: Radar はインターネット全体を集計した傾向で、特定のサイトの数字は出ない。自分のゾーンの攻撃は、ダッシュボードの Security Analytics などで見る。

✗ 誤解: グラフの値は通信量や攻撃の件数そのものだ

✓ 正しくは: 多くのグラフは割合や、最小〜最大で 0〜1 に正規化した値で、絶対量は出していない。しかも Cloudflare から見えた範囲だけなので、比較は同じ条件の中で行う。

✗ 誤解: CVSS が高い順に直せば、危ない順に直したことになる

✓ 正しくは: CVSS は脆弱性そのものの深刻度で、実際に狙われているかは表さない。CVSS が中程度でも KEV に載っているものの方が急ぐべきことが多い。CVE ページの Exploitation（KEV・SSVC・EPSS）を合わせて見る。

✗ 誤解: URL Scanner に入れた URL は自分にしか見えない

✓ 正しくは: 既定は Public で、最近のスキャン一覧と検索に載る。トークン入りの URL や社内の URL は、Unlisted にしてから調べる（Free では Unlisted が使えないので、そもそも入れない）。

✗ 誤解: URL Scanner で malicious と出なければ安全だ

✓ 正しくは: 判定はスキャンした時点・その条件での結果にすぎない。相手が UA や地域で中身を切り替えることもあるので、リダイレクト先・読み込んだ資源・似た構造のページも合わせて見る。

✗ 誤解: API のデータは出典を書けば仕事の資料に自由に使える

✓ 正しくは: API のデータは CC BY-NC 4.0 で、営利目的には使えない。グラフの埋め込みや画像は CC BY 4.0 なので、社外向けにはそちらを使うか、Cloudflare に相談する。

## 出典

- [Cloudflare Radar docs](https://developers.cloudflare.com/radar/) （ [最初のリクエスト](https://developers.cloudflare.com/radar/get-started/first-request/) ・ [比較](https://developers.cloudflare.com/radar/get-started/making-comparisons/) ・ [通知](https://developers.cloudflare.com/radar/get-started/configure-alerts/) ・ [BGP の異常](https://developers.cloudflare.com/radar/investigate/bgp-anomalies/) ）
- [URL Scanner](https://developers.cloudflare.com/radar/investigate/url-scanner/) ・ [Scan limits](https://developers.cloudflare.com/security-center/investigate/scan-limits/)
- [Radar API リファレンス](https://developers.cloudflare.com/api/resources/radar/)
- [Radar の changelog](https://developers.cloudflare.com/changelog/product/radar/) （Zero Wire・CVE ページ・Radar Researcher・Routing の各ページ）
- [CISA KEV](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) ・ [CISA SSVC](https://www.cisa.gov/ssvc) ・ [FIRST EPSS](https://www.first.org/epss/)
- [Radar の About（ライセンス）](https://radar.cloudflare.com/about)

確認した日付: 2026-11-22
