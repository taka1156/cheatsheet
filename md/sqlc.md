<!-- scripts/memo.py build が自動生成。手で編集しない -->

# sqlc 使い方

> sqlc の sqlc.yaml・クエリの注釈・マクロ・生成コードの使い方・コマンドをまとめたチートシート

- URL: https://cs.taka1156.site/sqlc.html
- タグ: Go, データベース
- 日付: 2026-11-20

---

書いた SQL から型付きの Go コードを生成するツール。PostgreSQL・MySQL・SQLite に対応（v1.31 時点）

**ざっくり言うと**

- sqlc は、スキーマ（DDL）とクエリの SQL を読んで、型付きの Go の関数と構造体を生成するコード生成ツール。
- クエリに `-- name: GetUser :one` と名前と戻り方を書くと、 `q.GetUser(ctx, id)` が生える。
- ORM ではないので SQL はそのまま書く。動的な条件の組み立ては苦手で、マイグレーションも別のツールで行う。

## 生成の流れ

**schema.sql** CREATE TABLE（マイグレーションのフォルダでもよい）

**query.sql** `-- name:` 付きのクエリ

**sqlc generate** sqlc.yaml に従って解析

**生成コード** `db.go` ・ `models.go` ・ `query.sql.go`

## インストール

```
brew install sqlc
go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest
docker run --rm -v $(pwd):/src -w /src sqlc/sqlc generate
```

## sqlc.yaml

```
version: "2"
sql:
  - engine: "postgresql"        # postgresql / mysql / sqlite
    schema: "db/migrations"     # ファイルでもフォルダでもよい
    queries: "db/queries"
    gen:
      go:
        package: "db"
        out: "internal/db"
        sql_package: "pgx/v5"   # 既定は database/sql
        emit_json_tags: true
        emit_interface: true    # Querier インターフェースも出す（モック用）
        overrides:
          - db_type: "uuid"
            go_type:
              import: "github.com/google/uuid"
              type: "UUID"
```

| キー（ `gen.go` の下） | 意味 |
| --- | --- |
| `sql_package` | `database/sql` （既定）・ `pgx/v5` ・ `pgx/v4` |
| `emit_json_tags` ・ `json_tags_case_style` | 構造体に `json:"…"` を付ける・その表記（ `camel` ・ `snake` など） |
| `emit_interface` | `Querier` インターフェースを出す |
| `emit_empty_slices` | `:many` が 0 件のとき `nil` ではなく空のスライスを返す |
| `emit_pointers_for_null_types` | NULL になる列を `*string` などにする（PostgreSQL + pgx と SQLite のみ。既定は `sql.NullString` ・ `pgtype.Text` ） |
| `emit_prepared_queries` | プリペアドステートメントを使う `Prepare` を出す |
| `query_parameter_limit` | 引数がこの数を超えたら Params 構造体にまとめる（既定 1） |
| `overrides` | DB の型（ `db_type` ）か列（ `column: "users.id"` ）ごとに Go の型を差し替える |
| `rename` | 生成される名前を変える（ `rename: { "uid": "UserID" }` ） |

`overrides` の `db_type` は NULL 可かどうかで別扱い。両方を差し替えるなら `nullable: true` 付きともう 1 つの 2 行を書く。

## クエリの書き方（注釈）

```
-- name: GetAuthor :one
SELECT * FROM authors WHERE id = $1 LIMIT 1;

-- name: ListAuthors :many
SELECT * FROM authors ORDER BY name;

-- name: CreateAuthor :one
INSERT INTO authors (name, bio) VALUES ($1, $2) RETURNING *;

-- name: DeleteAuthor :exec
DELETE FROM authors WHERE id = $1;
```

| 注釈 | 生成される関数の戻り値 |
| --- | --- |
| `:one` | 1 行（ `(Author, error)` ）。0 件なら `sql.ErrNoRows` （pgx は `pgx.ErrNoRows` ） |
| `:many` | 複数行（ `([]Author, error)` ） |
| `:exec` | `error` だけ |
| `:execrows` | 影響した行数（ `(int64, error)` ） |
| `:execresult` | `sql.Result` （pgx では `pgconn.CommandTag` ） |
| `:execlastid` | 最後に挿入した ID（ `LastInsertId` 。MySQL・SQLite 向け） |
| `:copyfrom` | 大量 INSERT（PostgreSQL の COPY・MySQL の LOAD DATA）。PostgreSQL は `sql_package: pgx/v5` （か v4）、MySQL は `sql_driver: github.com/go-sql-driver/mysql` も必要 |
| `:batchexec` ・ `:batchmany` ・ `:batchone` | pgx のバッチ送信（PostgreSQL + `pgx` のみ） |

## マクロ

| マクロ | 用途 |
| --- | --- |
| `sqlc.arg(name)` ・ `@name` | 引数に名前を付ける（ `$1` の代わり。略記の `@name` は MySQL では使えない） |
| `sqlc.narg(name)` | NULL を渡せる引数にする（任意の検索条件など） |
| `sqlc.slice(ids)` | `IN (…)` にスライスを渡す（MySQL・SQLite。PostgreSQL は `= ANY($1::int[])` ） |
| `sqlc.embed(users)` | JOIN の結果に既存のモデル構造体を丸ごと埋め込む |

```
-- name: SearchAuthors :many
SELECT * FROM authors
WHERE (sqlc.narg(name)::text IS NULL OR name ILIKE sqlc.narg(name))
  AND created_at >= @since
ORDER BY id;
```

## 生成したコードを使う

pgx/v5

database/sql

トランザクション

```
pool, err := pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
if err != nil { return err }
defer pool.Close()

q := db.New(pool)
a, err := q.CreateAuthor(ctx, db.CreateAuthorParams{
	Name: "Ada",
	Bio:  pgtype.Text{String: "math", Valid: true},
})
authors, err := q.ListAuthors(ctx)
```

```
conn, err := sql.Open("mysql", "user:pass@/app?parseTime=true")
if err != nil { return err }

q := db.New(conn)
a, err := q.GetAuthor(ctx, 1)
if errors.Is(err, sql.ErrNoRows) {
	// 見つからない
}
```

```
tx, err := pool.Begin(ctx)
if err != nil { return err }
defer tx.Rollback(ctx)        // Commit 後に呼んでも害はない

qtx := q.WithTx(tx)           // 同じクエリをトランザクションの中で
if _, err := qtx.CreateAuthor(ctx, p); err != nil { return err }
if err := qtx.DeleteAuthor(ctx, oldID); err != nil { return err }
return tx.Commit(ctx)
```

（元のページで操作できます → https://cs.taka1156.site/sqlc.html#use）

## コマンド

| コマンド | 用途 |
| --- | --- |
| `sqlc init` | 空の `sqlc.yaml` を作る |
| `sqlc generate` | コードを生成する |
| `sqlc compile` | 生成せずに SQL の文法と型だけを確かめる |
| `sqlc diff` | 生成結果と今のファイルが違えば失敗（CI で「generate し忘れ」を検出） |
| `sqlc vet` | `rules` に CEL で書いた独自ルールでクエリを検査 |
| `sqlc parse` | SQL の AST を JSON で出力する（v1.31 で追加。 `sqlc fmt` ・ `sqlc analyze` は main ブランチにあるが v1.31.1 時点では未リリース） |
| `-f path/sqlc.yaml` | 設定ファイルを指定する（CLI のドキュメントに載っている `--no-database` は今の版には実装されていない） |

## マイグレーションとの組み合わせ

sqlc はスキーマを変更しない。 `schema` にマイグレーションのフォルダを指定すると、ファイル名の順に読んで最終的な形を組み立てる。対応しているツールの形式: atlas・dbmate・golang-migrate・goose・sql-migrate・tern。ダウン側（ `-- +goose Down` など）は無視される。

MySQL で「CREATE 文を正として差分を当てる」宣言型の管理をするなら [Skeema](https://cs.taka1156.site/md/skeema.md) と組み合わせ、 `schema` に Skeema のフォルダを指定する、という使い方もできる。

## 導入の手順

1. **設定を作る** `sqlc init` のあと、エンジン・スキーマ・クエリ・出力先を書く
2. **クエリを書く** `-- name: 名前 :注釈` を付けて、1 ファイルに機能ごとのクエリをまとめる
3. **生成する** `sqlc generate` 。生成物はコミットし、 `go generate` から呼んでもよい
4. **CI で確かめる** `sqlc diff` と `sqlc vet` を流し、生成し忘れと危ないクエリを止める

## よくある誤解

✗ 誤解: sqlc は ORM の一種で、マイグレーションも面倒を見てくれる

✓ 正しくは: sqlc はコードを生成するだけで、実行時には何もしない。スキーマの変更は golang-migrate・goose・Atlas・Skeema など別のツールで行い、sqlc はその結果のスキーマを読むだけ。

✗ 誤解: 条件を足し引きする動的なクエリも sqlc で自由に組み立てられる

✓ 正しくは: 生成されるのは書いたとおりの固定の SQL。任意の条件は `sqlc.narg` と `IS NULL OR …` で表すのが基本で、並び替えの列を変えるなどの大きな組み立ては、その部分だけ手書きやクエリビルダーにする。

✗ 誤解: SELECT * と書くと、実行時に列が増えても自動で追従する

✓ 正しくは: `*` は生成時に列名へ展開されて固定される。スキーマを変えたら `sqlc generate` をやり直す（CI の `sqlc diff` で検出できる）。

関連: Go のコードでスキーマを書く方式は [ent 使い方](https://cs.taka1156.site/md/ent.md) 。

## 出典

- [sqlc Documentation](https://docs.sqlc.dev/en/latest/)
- [Configuration](https://docs.sqlc.dev/en/latest/reference/config.html) ・ [Query annotations](https://docs.sqlc.dev/en/latest/reference/query-annotations.html) ・ [Macros](https://docs.sqlc.dev/en/latest/reference/macros.html) ・ [CLI](https://docs.sqlc.dev/en/latest/reference/cli.html)
- [Overriding types](https://docs.sqlc.dev/en/latest/howto/overrides.html) ・ [Using transactions](https://docs.sqlc.dev/en/latest/howto/transactions.html) ・ [Modifying the database schema](https://docs.sqlc.dev/en/latest/howto/ddl.html)
- [Changelog](https://docs.sqlc.dev/en/latest/reference/changelog.html) （v1.31.1、2026-04-22）

確認日: 2026-11-20（sqlc v1.31.1）
