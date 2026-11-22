<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Security Audit Skill 使い方

> Cloudflare のコーディングエージェント用スキル security-audit の入れ方・頼み方・6 フェーズ・出力ファイル・判定と重大度をまとめた。

- URL: https://cs.taka1156.site/security-audit-skill.html
- タグ: セキュリティ, 開発ツール
- 日付: 2026-11-22

---

Cloudflare が公開した、コーディングエージェントをセキュリティ監査人にするスキル `security-audit` （MIT・2026-11 時点）

**ざっくり言うと**

- Markdown の手順書と JSON スキーマ・検証ツールの一式で、エージェントにコードベースの脆弱性監査をさせる。
- 見つける役と確かめる役を別のサブエージェントに分け、6 フェーズで候補を反証してから報告する。
- 1 回で全部は見つからないので繰り返し実行し、対象コードは必ずネットから切ったサンドボックスで動かす。

## インストールと頼み方

Skills CLI（ `npx skills` ）で入れる（使い方は [Skills CLI 使い方](https://cs.taka1156.site/md/skills-cli.md) ）。Claude Code・Cursor・Codex など、Agent Skills を読めて並列のサブエージェントを使えるエージェントで使える。

```
# このプロジェクトに入れる
npx skills add https://github.com/cloudflare/security-audit-skill \
  --skill security-audit

# ユーザー単位で入れる（全プロジェクトで使う）
npx skills add https://github.com/cloudflare/security-audit-skill \
  --skill security-audit \
  --global

# エージェントの選択や非対話のオプション
npx skills --help
```

監査したいリポジトリでエージェントを起動し、ふつうに頼む。「security audit」「find vulnerabilities」「pen-test」などの言葉で自動的に読み込まれる。

```
security audit this codebase
find security vulnerabilities in ./src
do a security review, output to ~/audits/my-project
```

## 2 つのモード

スキルを読み込んだだけでは監査は始まらない。 **既定はガイダンスモード** で、フル監査は明示的に頼んだときだけ。どちらとも取れる依頼なら、エージェントがファイルを作る前に 1 つ質問する。

| モード | 始まる頼み方 | やること |
| --- | --- | --- |
| ガイダンス | セキュリティの質問、特定箇所のレビュー、見つけた問題の調査・トリアージ | 必要な部分だけ使う。6 フェーズは回さず、出力フォルダも作らない |
| フル監査 | 「コードベースを監査して」「pen-test して」、full / comprehensive / end-to-end のレビュー、レポートのファイルが欲しいとき | 6 フェーズを全部回し、決まったファイルを書き出す |

## フル監査の 6 フェーズ

親エージェント（parent）が全体を仕切り、共有ファイルを書くのは親だけ。調べる作業はサブエージェントに渡す。

1. **偵察（Reconnaissance）** 構成・信頼境界（trust boundary）・入力面・過去の結果を調べ、 `architecture.md` と、何をどこまで見たかを記録する `coverage-ledger.json` を作る。
2. **カバレッジに沿った探索（Hunting）** 台帳の単位ごとに独立した hunter を割り当てる。coverage critic が見落としを探し、足りなければ次の波を出す。
3. **候補の検証（Candidate validation）** 候補 1 つずつを、見つけた本人ではない新しい verifier に渡して反証（disprove）させる。
4. **構造化出力（Structured output）** 検証で判定が決まった記録（confirmed・needs_validation・rejected）を `findings.json` に書き、スキーマと検証ツールで確かめる。
5. **記録の独立検証（Record verification）** 新しいエージェントが最終的なソースの主張を確かめ直す。中身が大きく変わったら、さらに別の verifier を付ける。
6. **レポート（Reporting）** 検証済みの記録と台帳から `REPORT.md` などを作る。本番環境を叩く手順は書かない。

終わり方は 2 つだけ。(a) レポートを全部書き、検証ツールが 2 つとも通る。(b) `run_status: "incomplete"` と理由を記録し、レポートに抜けを書く。フェーズの途中で止めない。

## 判定（verdict）と重大度

| 判定 | 意味 | 重大度 |
| --- | --- | --- |
| confirmed | 入口から到達点（sink）までのソースの追跡がそろい、サンドボックスの中で結果を観測できた脆弱性 | 付ける |
| needs_validation | ソースに根拠はあるが、確かめられない事実が 1 つ残っている（デプロイ設定やプロキシの挙動など、リポジトリに無いもの）。その事実を正確に書く | 付けない |
| rejected | 検証で反証された候補。次の実行で同じ主張を蒸し返さないために残す | 付けない |

重大度は confirmed だけに付け、実際に示せた影響を超えない。

| 重大度 | 目安 |
| --- | --- |
| critical | 認証なしでコード実行・データストア全体へのアクセス・任意アカウントの乗っ取り |
| high | 明示的なセキュリティ制御を完全に破る（認証回避、テナントをまたぐ読み書き、他人に効く stored XSS、認証ありのコード実行、認証なしで共有サービスを外から止める） |
| medium | 本物の境界違反だが、影響範囲が狭い・前提条件がまれ |
| low | 秘密ではない内部情報の露出、手間のわりに得るものが小さい |
| informational | 影響は最小。大きな問題の前提条件として役立つもの |

high と medium の分かれ目は「制御を完全に破ったか、弱めただけか」。具体的な被害を言えないなら、感覚より低い。

## 出力ファイル

出力先を指定しなければ、対象の外の `~/security-audit-skill/<repo-name>/run-<N>` に書く。対象リポジトリの中に書くのは、Git に無視されるフォルダを自分で指定したときだけ。

| ファイル | 中身 |
| --- | --- |
| `run-metadata.json` | 対象・コミット・プロファイル・範囲・予算・実行の状態 |
| `architecture.md` | 偵察でまとめた構成と信頼境界 |
| `coverage-ledger.json` | カバレッジ台帳。どの単位を誰がどこまで見たか |
| `findings.json` | 3 種類の判定の記録（ `report-schema.json` に沿う） |
| `REPORT.md` | 全体のレポート |
| `FINDINGS-DETAIL.md` | 重大度 medium 以上の confirmed の詳細（追跡・再現手順・ソースの修正と回帰テスト） |
| `NEEDS-VALIDATION.md` | needs_validation の一覧と、確かめ方の案 |
| `agents/<id>/` | サブエージェントごとの `scratch/` （作業用）と `artifacts/` （親が選んで残す証拠） |

## 検証ツール

どちらも依存パッケージなしの Node.js スクリプト。親が決まったタイミングで自動で流すが、手でも実行できる。 `<skill-dir>` は `SKILL.md` のあるフォルダ。

```
# 台帳を作った後と、更新するたびに
node <skill-dir>/validate-coverage-ledger.cjs <output-dir>/coverage-ledger.json

# フェーズ 4 と、フェーズ 5 で記録を差し替えるたびに
node <skill-dir>/validate-findings.cjs <output-dir>/findings.json
```

台帳の検証ツールは、5 MiB・ネスト 64 段・10,000 単位を超える入力を拒否する。

## プロファイル・範囲・予算

プロファイル

範囲を絞る

予算

| プロファイル | 向く場面 | 違い |
| --- | --- | --- |
| `quick` | 小さい対象、再実行、最初のざっと見 | 台帳の単位を粗くし、hunter の波 1 回と critic 1 回だけ。候補ごとの verifier は 1 つ |
| `standard` | 既定 | 書かれたとおりの手順 |
| `deep` | 大きい対象、重要度の高い対象 | 単位をサブシステムごとに細かく分け、critic が何も見つけなくなるまで回す。候補の検証と記録の検証を別のエージェントにする |

プロファイルで変わるのは広さと重ね方だけ。証拠の基準（独立検証・サンドボックス・スキーマ検証）は変わらない。

パス・サブシステム・ドメイン・2 つのコミットの差分（diff）に絞った scoped run もできる。範囲外は台帳に `out_of_scope` と書かれ、 `covered` （確認済み）にはならない。scoped run と quick は、レポートで「部分的なカバレッジ」と名乗る。

予算は「全フェーズでのエージェント呼び出しの最大回数」で指定する（ `run-metadata.json` の `budget` ）。偵察・critic・検証の分を先に取っておき、残りで hunter を割り当てる。最低限も賄えない予算なら、エージェントを 1 つも起動せず、予算を増やすか範囲を絞るよう求める。

検証の途中で予算が尽きたら `run_status: "incomplete"` になる。未検証の候補を `needs_validation` に言い換えて済ませることはしない。

（元のページで操作できます → https://cs.taka1156.site/security-audit-skill.html#profiles）

## 攻撃クラスの補助ファイル

偵察の結果に合わせて、対象の種類ごとの補助ファイルを選んで hunter に渡す。共通は `ATTACK-CLASSES.md` 。

| ファイル | 対象 |
| --- | --- |
| `WEB-PROTOCOL-AND-AUTH.md` | HTTP のリクエスト解釈のずれ・キャッシュ・認証プロトコル |
| `CLIENT-SIDE.md` | DOM への注入・メッセージの送り元の信頼・クリックジャッキング・プロトタイプ汚染 |
| `AI-AND-LLM.md` | プロンプトインジェクション・エージェントやツールの権限・出力の扱い |
| `SUPPLY-CHAIN-AND-RELEASE.md` | 依存・CI・リリース・署名・更新・プラグイン |
| `CLOUD-AND-DEPLOYMENT.md` | IAM・IaC・コンテナ・サーバーレス・実行時の設定 |
| `PROTOCOLS-RPC-AND-MESSAGING.md` | RPC・シリアライズ・キュー・Webhook・ストリーミング |
| `DATA-ISOLATION-AND-LIFECYCLE.md` | テナント分離・エクスポート・バックアップ・削除と復元 |
| `RESOURCE-EXHAUSTION-AND-AVAILABILITY.md` | 共有リソース・クォータ・ワーカーの枯渇、運営側の課金 |
| `MEMORY-SAFETY-AND-BINARY.md` | メモリ安全性・バイナリ・カーネル（ネイティブコード） |
| `DESKTOP-MOBILE-AND-LOCAL-IPC.md` | ネイティブアプリ・ディープリンク・WebView・ローカル IPC |

## 必要なもの（特にサンドボックス）

- ツール呼び出しと **並列のサブエージェント** が使えるモデルとエージェント
- Node.js（検証ツール用）
- 対象のビルド・テスト・ブラウザ・ファザーなどを動かすための、 **OS が強制するサンドボックス**

サンドボックスには次の 4 つがすべて要る。1 つでも無ければ対象のコードは実行せず、その候補は `needs_validation` のまま残す。

1. **外部ネットワークなし** 必要ならループバックだけの分離された名前空間を使う。
2. **空の環境変数から始める** 許可リストの安全な値だけを入れる。 `HOME` ・一時フォルダ・キャッシュも作業用フォルダの中にする。
3. **書き込みは作業用フォルダだけ** 対象とツールチェーンは読み取り専用にする。
4. **リソースの上限** CPU・メモリ・プロセス数・ファイルサイズ・ディスク・実行時間に低い上限をかける。

依存のインストールや、ビルド中のダウンロードもさせない。デプロイ済みのエンドポイント・外部サービス・本番の ID には触らない。

## Trivy・OSV-Scanner との使い分け

**security-audit が向く** 自分たちのコードに固有のロジックの穴（認可漏れ、テナント越え、プロンプトインジェクションなど）を、ソースの追跡つきで探したいとき。時間とトークンがかかる。

**スキャナーが向く** 既知の CVE・設定ミス・シークレットを、速く同じ結果で CI に組み込みたいとき。 [Trivy 使い方](https://cs.taka1156.site/md/trivy.md) ・ [OSV-Scanner 使い方](https://cs.taka1156.site/md/osv-scanner.md) 。

## よくある誤解

✗ 誤解: 1 回フル監査すれば脆弱性は出尽くす

✓ 正しくは: Cloudflare のテストでは、1 回の実行で見つかったのは、何度も実行して見つかった脆弱性全体のおよそ半分だった。実行は積み上がる仕組みで、2 回目以降は前回の台帳と結果を読み、見ていない所や変わったソースを狙う。

✗ 誤解: スキルを入れると、セキュリティの質問のたびに重い監査が走る

✓ 正しくは: 既定はガイダンスモード。6 フェーズとファイルの書き出しは、コードベースの監査・pen-test・フルレビュー・レポートのファイルを明示的に頼んだときだけ。

✗ 誤解: needs_validation は確度の低い脆弱性なので、重大度を低くして扱えばよい

✓ 正しくは: needs_validation は「脆弱性かどうかを決める事実が 1 つ確かめられていない」状態で、重大度は付かない。書かれた事実（本番の設定など）を持ち主が確かめれば、confirmed にも rejected にもなりうる。

✗ 誤解: ベストプラクティスから外れている所は、すべて脆弱性として報告される

✓ 正しくは: 報告するのは、実際の信頼境界を破れる場合だけ。チェックリストからの逸脱や、別の層ですでに攻撃が防がれている多層防御の欠けは、脆弱性ではなく強化のメモ（hardening）として扱う。

✗ 誤解: 監査のついでに、見つけた脆弱性を直してくれる

✓ 正しくは: 対象のソースは書き換えない。confirmed ごとに、守るべき不変条件と、最後に信頼できる判断の場所での最小の修正案、回帰テストを提案する。

## 出典

- [cloudflare/security-audit-skill](https://github.com/cloudflare/security-audit-skill) （README.md・skills/security-audit/SKILL.md ほか。コミット c1c8a8c、2026-09-14）
- [Build your own vulnerability harness](https://blog.cloudflare.com/build-your-own-vulnerability-harness) （Cloudflare ブログ。このスキルから発展した社内の脆弱性探索の仕組み）

確認日: 2026-11-22
