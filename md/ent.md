<!-- scripts/memo.py build が自動生成。手で編集しない -->

# ent 使い方

> ent のスキーマ定義（Fields・Edges）・CRUD・トランザクション・Atlas でのマイグレーションをまとめたチートシート

- URL: https://cs.taka1156.site/ent.html
- タグ: Go, データベース
- 日付: 2026-11-20

---

Go のコードでスキーマ（エンティティと関係）を書き、型付きのクライアントを生成する Go の ORM（v0.14 時点）

**ざっくり言うと**

- ent は `ent/schema/*.go` に Fields と Edges を書くと、型付きのクエリビルダーを生成する ORM。
- スキーマを変えたら `go generate ./ent` で作り直し、 `client.User.Query().Where(…)` のように呼ぶ。
- 開発中は自動マイグレーションで足りるが、本番は Atlas でマイグレーションのファイルを作って当てる。

## 生成の流れ

**ent/schema/user.go** Fields・Edges・Indexes

**go generate ./ent** `entc` がコードを生成

**ent パッケージ** `client.go` ・ `user_query.go` ・ `user/where.go` …

**DB** 自動マイグレーション or Atlas

## 始め方

```
go mod init example.com/app
go run -mod=mod entgo.io/ent/cmd/ent new User Car   # ent/schema/user.go・car.go の雛形
go generate ./ent                                   # ent/generate.go の go:generate を実行
```

```
// ent/generate.go（new で作られる）
package ent

//go:generate go run -mod=mod entgo.io/ent/cmd/ent generate ./schema
```

機能フラグは `generate --feature sql/upsert,sql/versioned-migration ./schema` のように足す（upsert・バージョン付きマイグレーションなど）。

## スキーマを書く

```
package schema

type User struct{ ent.Schema }

func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").NotEmpty().MaxLen(50),
		field.String("email").Unique(),
		field.Int("age").Positive().Optional(),
		field.Enum("role").Values("admin", "member").Default("member"),
		field.String("password").Sensitive(),        // String() や JSON に出さない
		field.UUID("uid", uuid.UUID{}).Default(uuid.New).Immutable(),
		field.Time("deleted_at").Optional().Nillable(),   // *time.Time
	}
}

func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("cars", Car.Type),                    // User 1 — n Car
	}
}

func (User) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("name", "age"),
	}
}

func (User) Mixin() []ent.Mixin {
	return []ent.Mixin{mixin.Time{}}                  // create_time・update_time を足す
}
```

### フィールドのオプション

| メソッド | 意味 |
| --- | --- |
| `Optional()` | 作成時に省略できる。DB の列は NULL 可になる（フィールドは既定で必須） |
| `Nillable()` | Go の型を `*T` にして、ゼロ値と NULL を区別する（ `Optional` と一緒に使う） |
| `Default(v)` ・ `UpdateDefault(fn)` | 作成時・更新時の既定値（ `time.Now` や `uuid.New` を関数で渡せる） |
| `Unique()` ・ `Immutable()` | 一意制約・作成後に変更できない（Update に setter が生えない） |
| `Sensitive()` | 出力やログに値を出さない |
| `NotEmpty()` ・ `MaxLen(n)` ・ `Match(re)` ・ `Positive()` ・ `Range(a,b)` ・ `Validate(fn)` | 検証（保存時に走る） |
| `StorageKey("col")` ・ `SchemaType(map…)` ・ `GoType(T)` | 列名・DB の型・Go の型を変える |
| `Comment("…")` | 生成コードのコメント（DB の列コメントにするには `entsql.WithComments(true)` のアノテーションも付ける） |

### エッジ（関係）

| 関係 | 書き方 |
| --- | --- |
| 1 対多 | User: `edge.To("cars", Car.Type)` ／ Car: `edge.From("owner", User.Type).Ref("cars").Unique()` |
| 1 対 1 | User: `edge.To("card", Card.Type).Unique()` ／ Card: `edge.From("owner", User.Type).Ref("card").Unique().Required()` |
| 多対多 | Group: `edge.To("users", User.Type)` ／ User: `edge.From("groups", Group.Type).Ref("users")` |
| 自己参照 | `edge.To("friends", User.Type)` （双方向）・ `edge.To("following", User.Type).From("followers")` |
| 外部キーを列として持つ | Car に `field.Int("owner_id").Optional()` を書き、 `edge.From("owner", …).Ref("cars").Unique().Field("owner_id")` |

`edge.To` が持つ側（外部キーや結合テーブルの持ち主）、 `edge.From(…).Ref(…)` は逆向きの参照（DB には新しい列を作らない）。

## CRUD

接続

作成

取得

更新・削除

```
client, err := ent.Open("postgres", "host=localhost user=app dbname=app sslmode=disable")
// ent.Open("mysql", "user:pass@tcp(localhost:3306)/app?parseTime=True")
// ent.Open("sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
if err != nil { log.Fatal(err) }
defer client.Close()

// 開発用: スキーマに合わせてテーブルを作る・足す
if err := client.Schema.Create(ctx); err != nil { log.Fatal(err) }
```

ドライバーは別に import する（ `_ "github.com/lib/pq"` ・ `_ "github.com/go-sql-driver/mysql"` ・ `_ "github.com/mattn/go-sqlite3"` など）。

```
u, err := client.User.Create().
	SetName("a8m").
	SetEmail("a8m@example.com").
	AddCars(tesla, ford).          // エッジもつなげる
	Save(ctx)

// まとめて作成
users, err := client.User.CreateBulk(
	client.User.Create().SetName("a").SetEmail("a@x"),
	client.User.Create().SetName("b").SetEmail("b@x"),
).Save(ctx)

// upsert（--feature sql/upsert が必要）
err = client.User.Create().SetName("a").SetEmail("a@x").
	OnConflictColumns(user.FieldEmail).UpdateNewValues().Exec(ctx)
```

```
u, err := client.User.Get(ctx, id)                 // ID で 1 件
u, err := client.User.Query().
	Where(user.Email("a8m@example.com")).
	Only(ctx)                                       // 0 件か 2 件以上ならエラー

users, err := client.User.Query().
	Where(user.AgeGT(20), user.NameHasPrefix("a")).   // AND。OR は user.Or(…)
	Order(user.ByCreateTime(sql.OrderDesc())).  // mixin.Time の create_time。sql は entgo.io/ent/dialect/sql
	Limit(10).Offset(20).
	All(ctx)

n, err := client.User.Query().Where(user.HasCars()).Count(ctx)
ok, err := client.User.Query().Where(user.Email(e)).Exist(ctx)

// エッジをたどる・まとめて読む（N+1 を避ける）
cars, err := u.QueryCars().All(ctx)
users, err = client.User.Query().WithCars().All(ctx)  // users[i].Edges.Cars

if ent.IsNotFound(err) { /* 見つからない */ }
```

```
u, err = u.Update().SetAge(31).AddCarIDs(carID).Save(ctx)   // 取得済みの 1 件
n, err := client.User.Update().
	Where(user.AgeLT(18)).
	SetRole(user.RoleMember).
	Save(ctx)                                  // 条件に合う全件。戻り値は件数

err = client.User.DeleteOne(u).Exec(ctx)
n, err = client.User.Delete().Where(user.NameEQ("x")).Exec(ctx)
```

（元のページで操作できます → https://cs.taka1156.site/ent.html#crud）

## トランザクション

```
func WithTx(ctx context.Context, client *ent.Client, fn func(tx *ent.Tx) error) error {
	tx, err := client.Tx(ctx)
	if err != nil { return err }
	defer func() {
		if v := recover(); v != nil {
			tx.Rollback()
			panic(v)
		}
	}()
	if err := fn(tx); err != nil {
		if rerr := tx.Rollback(); rerr != nil {
			err = fmt.Errorf("%w: rolling back transaction: %v", err, rerr)
		}
		return err
	}
	return tx.Commit()
}

// tx.User.Create()… のように、tx から同じ API を使う
```

## マイグレーション

自動（開発向け）

Atlas でファイルを作る（本番向け）

```
err := client.Schema.Create(ctx,
	migrate.WithDropIndex(true),    // 既定では消さない
	migrate.WithDropColumn(true),
)
```

起動のたびに今の DB との差分を当てる。消す変更は既定で行わない。複数台の同時起動やレビューのしにくさがあるので、本番ではファイル方式にする。

```
# 差分からマイグレーションのファイルを作る（--dev-url は差分計算用の空の DB）
atlas migrate diff add_users \
  --dir "file://ent/migrate/migrations" \
  --to "ent://ent/schema" \
  --dev-url "docker://postgres/15/test?search_path=public"

# 当てる
atlas migrate apply \
  --dir "file://ent/migrate/migrations" \
  --url "postgres://app:pass@localhost:5432/app?sslmode=disable"
```

`atlas.sum` がフォルダの中身の改ざんや順番の入れ替えを検出する。手でファイルを直したら `atlas migrate hash` 。

（元のページで操作できます → https://cs.taka1156.site/ent.html#migrate）

## 導入の手順

1. **雛形を作る** `ent new User` でスキーマを作り、Fields と Edges を書く
2. **生成する** `go generate ./ent` 。生成物（ `ent/` の下）はコミットする
3. **開発中は自動で** `client.Schema.Create` でテーブルを作って試す
4. **本番前にファイル方式へ** `atlas migrate diff` で SQL のファイルを作り、レビューしてから `apply`
5. **API に載せる** ハンドラー（ [Gin](https://cs.taka1156.site/md/gin.md) など）から `c.Request.Context()` を渡して呼ぶ

## よくある誤解

✗ 誤解: Optional を付ければ、Go の値でも NULL と 0 を区別できる

✓ 正しくは: `Optional` は「作成時に省略できる・列が NULL 可」というだけで、Go の型は `T` のまま（NULL はゼロ値で読まれる）。区別したいなら `Nillable()` も付けて `*T` にする。

✗ 誤解: edge.From を書くと、DB に別の外部キー列ができる

✓ 正しくは: `edge.From(…).Ref(…)` は `edge.To` の逆向きの参照で、新しい列やテーブルは作らない。関係の実体は `edge.To` の側で 1 つだけ。

✗ 誤解: client.Schema.Create を本番で毎回走らせれば、マイグレーションは要らない

✓ 正しくは: 自動マイグレーションは列やインデックスを消さず、変更内容をレビューする場所もない。本番ではバージョン付きのマイグレーション（Atlas）でファイルを作ってから当てる。

関連: SQL を先に書いてコードを生成する方式は [sqlc 使い方](https://cs.taka1156.site/md/sqlc.md) 。

## 出典

- [ent – Quick Introduction](https://entgo.io/docs/getting-started)
- [Fields](https://entgo.io/docs/schema-fields) ・ [Edges](https://entgo.io/docs/schema-edges) ・ [Indexes](https://entgo.io/docs/schema-indexes) ・ [Mixin](https://entgo.io/docs/schema-mixin)
- [CRUD API](https://entgo.io/docs/crud) ・ [Eager Loading](https://entgo.io/docs/eager-load) ・ [Transactions](https://entgo.io/docs/transactions)
- [Automatic Migration](https://entgo.io/docs/migrate) ・ [Versioned Migrations](https://entgo.io/docs/versioned-migrations)

確認日: 2026-11-20（ent v0.14.6）
