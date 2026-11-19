<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Skeema 使い方

> Skeema のフォルダ構成と .skeema・diff/push/pull・安全装置・lint・workspace をまとめたチートシート

- URL: https://cs.taka1156.site/skeema.html
- タグ: データベース
- 日付: 2026-11-20

---

MySQL・MariaDB のスキーマを CREATE 文のファイルで管理し、DB との差分から ALTER を自動で作る宣言型のツール（v1.14 時点）

**ざっくり言うと**

- Skeema は、テーブルごとの `CREATE TABLE` の .sql ファイルを「あるべき姿」として Git で管理する CLI。
- `skeema diff` がファイルと DB を比べて ALTER を作り、 `skeema push` がそれを実行する。
- 列やテーブルを消す変更は既定で止まるので、意図したときだけ `--allow-unsafe` を付ける。

## 考え方: マイグレーションのファイルを書かない

**users.sql を編集** CREATE TABLE に列を足す

**skeema diff** DB と比べて `ALTER TABLE …` を表示

**PR でレビュー** CI で `skeema lint`

**skeema push** 生成した DDL を実行

逆向き（DB を手で変えた・別のツールで変えた）は `skeema pull` でファイルを DB に合わせる。

## インストール

```
brew install skeema/tap/skeema     # macOS / Linux（Homebrew）
# Linux は skeema.io・GitHub Releases の .deb / .rpm / tarball も使える
```

このリポジトリの無料版（Community）が扱うのはテーブルとルーチン（プロシージャ・関数）。ビュー・トリガー・イベント・シードデータ・Windows 版は有料の Premium 版。

## フォルダの構成と .skeema

schemas/ ホストごとのフォルダ（ `init -d` で名前を決める）

├─ .skeema `[production]` などの節に host・port・user・flavor

├─ app/ スキーマ（データベース）ごとのフォルダ

│ ├─ .skeema `schema=app` ・既定の文字コードと照合順序

│ ├─ users.sql 1 ファイル 1 テーブルの `CREATE TABLE`

│ └─ orders.sql

└─ analytics/ 別のスキーマ

```
# schemas/.skeema（ホストの設定。節が無い行は全環境に効く）
user=app
ignore-table=^_

[production]
host=db.internal
port=3306
flavor=mysql:8.0

[development]
host=127.0.0.1
port=3306

# schemas/app/.skeema
schema=app
default-character-set=utf8mb4
default-collation=utf8mb4_0900_ai_ci
```

設定は親フォルダの `.skeema` から子へ引き継がれる。環境名を省くと `production` の節が使われる。パスワードはファイルに書かず、環境変数 `MYSQL_PWD` か、 `-p` （値なしで入力を求める）で渡す。

## コマンド

| コマンド | 用途 |
| --- | --- |
| `skeema init -h HOST -u USER -p -d schemas` | DB の今のスキーマを .sql ファイルに書き出してフォルダを作る（ `--schema app` で 1 つだけ） |
| `skeema add-environment development -h 127.0.0.1 -d schemas` | `.skeema` に別の環境の節を足す（手で書いてもよい） |
| `skeema diff [環境]` | ファイルに合わせるための DDL を表示するだけ（ `push --dry-run` と同じ） |
| `skeema push [環境]` | その DDL を実行する |
| `skeema pull [環境]` | DB 側の変更をファイルに取り込む |
| `skeema lint [環境]` | ファイルの CREATE 文をルールで検査する（既定で整形もする。止めるなら `--skip-format` ） |
| `skeema format` | CREATE 文を `SHOW CREATE` と同じ正規の形に整える |

終了コード: `diff` は差分なし 0・差分あり 1・エラー 2 以上。 `lint` は警告か整形があると 1・エラーがあると 2 以上、 `format` は整形したファイルがあると 1。CI ではこれで判定する。

## よく使うオプション

安全装置

作業場所（workspace）

lint のルール

オンライン変更

| オプション | 意味 |
| --- | --- |
| `--allow-unsafe` | DROP や列の型を狭めるなど、データが消えうる変更を許す（既定 false） |
| `--safe-below-size=10m` | このサイズ未満のテーブルなら消す変更も許す（開発環境向け） |
| `--dry-run` | `push` で実行せずに表示だけ |
| `--ignore-table='^_'` ・ `--ignore-schema` | 正規表現に合うテーブル・スキーマを対象から外す（gh-ost の作業用テーブルなど） |
| `--first-only` | 同じフォルダに複数のホスト・スキーマがあるとき最初の 1 つだけ |

Skeema はファイルの CREATE 文を一度どこかの DB で実際に流し、その結果と比べる。その場所が workspace。

| 設定 | 動き |
| --- | --- |
| `workspace=temp-schema` （既定） | 対象の DB サーバーに一時スキーマ `_skeema_tmp` を作って流し、終わったら消す（ `temp-schema` で名前を変更） |
| `workspace=docker` | ローカルの Docker で `flavor` （例: `mysql:8.0` ）のコンテナを立てて使う。本番に一時スキーマを作りたくないとき・CI 向け |
| `docker-cleanup=stop` ／ `destroy` | 終わったときにコンテナを止める・消す（v1.14 の既定は `none` ） |

各ルールは `lint-ルール名=ignore|warning|error` で強さを決める（ `.skeema` に書ける）。 `error` のルールに当たると `push` も止まる。

| ルール | 見るもの（既定） |
| --- | --- |
| `lint-pk` | 主キーが無いテーブル（warning） |
| `lint-dupe-index` | 重複したセカンダリインデックス（warning） |
| `lint-charset` + `allow-charset=utf8mb4` | 許可した文字コード以外（warning） |
| `lint-engine` + `allow-engine=innodb` | 許可したストレージエンジン以外（warning） |
| `lint-auto-inc` ・ `lint-pk-type` | AUTO_INCREMENT・主キーの型（warning・ignore） |
| `lint-zero-date` ・ `lint-reserved-word` ・ `lint-display-width` | ゼロ日付の既定値・予約語の名前・ `int(11)` のような表示幅（warning） |
| `lint-definer` + `allow-definer` | ルーチンの DEFINER（error） |
| `lint-has-fk` ・ `lint-has-enum` ・ `lint-has-float` ・ `lint-name-case` | 外部キー・ENUM・FLOAT・大文字の名前を禁止したいとき（ignore） |

```
# .skeema — 大きいテーブルだけ gh-ost で ALTER する
[production]
alter-wrapper="gh-ost --execute --alter {CLAUSES} --database={SCHEMA} --table={TABLE} --host={HOST} --user={USER} --password={PASSWORD}"
alter-wrapper-min-size=1g     # これ未満は通常の ALTER
alter-algorithm=inplace       # 通常の ALTER に ALGORITHM= を付ける
alter-lock=none
```

`{CLAUSES}` は ALTER TABLE の後ろの部分、 `{DDL}` は文全体。pt-online-schema-change も同じように指定できる。

（元のページで操作できます → https://cs.taka1156.site/skeema.html#options）

## 導入の手順

1. **今の DB から書き出す** `skeema init -h HOST -u USER -p -d schemas` でフォルダを作り、Git に入れる
2. **環境を足す** 開発用の DB を `[development]` 節に書き、 `skeema push development` で同じ形にする
3. **変更はファイルで** .sql の CREATE 文を直し、 `skeema diff` で出る ALTER を確かめる
4. **CI で検査** PR ごとに `skeema lint` （ `workspace=docker` にすると本番 DB を使わない）
5. **本番へ** マージ後に `skeema diff` → `skeema push` 。消す変更なら `--allow-unsafe` を明示する

## よくある誤解

✗ 誤解: 列名を変えるには CREATE 文の列名を書き換えればよい

✓ 正しくは: Skeema はファイルと DB の「差」しか見ないので、名前の変更は「古い列を消して新しい列を足す」と解釈され、データが消える（unsafe として止まる）。名前の変更は `ALTER TABLE … RENAME COLUMN` を DB で直接実行し、 `skeema pull` でファイルを合わせる。

✗ 誤解: diff はファイルを読むだけなので、DB には何もしない

✓ 正しくは: 既定の `workspace=temp-schema` では、対象の DB サーバーに一時スキーマを作って CREATE 文を流してから比べる。権限や負荷が気になる環境では `workspace=docker` にする。

✗ 誤解: Skeema は PostgreSQL にも使える

✓ 正しくは: 対応するのは MySQL と MariaDB（とその互換の Percona Server・Aurora など）だけ。PostgreSQL で宣言型の管理をするなら Atlas などを使う。

関連: Skeema のフォルダを [sqlc](https://cs.taka1156.site/md/sqlc.md) の `schema` に指定すれば、同じ CREATE 文から Go のコードを生成できる。

## 出典

- [Skeema Documentation](https://www.skeema.io/docs/) （ [Commands](https://www.skeema.io/docs/commands/) ・ [Options](https://www.skeema.io/docs/options/) ・ [Configuration](https://www.skeema.io/docs/config/) ）
- [skeema/skeema](https://github.com/skeema/skeema) （README・各コマンドとオプションの定義、v1.14.2）
- [Safety features](https://www.skeema.io/docs/features/safety/) ・ [Online schema change tools](https://www.skeema.io/docs/features/osc/)

確認日: 2026-11-20（skeema v1.14.2。2.0 の開発が進んでおり、一部のオプションは v2 で変わる予定）
