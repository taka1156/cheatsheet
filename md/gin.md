<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Gin 使い方

> Gin のルーティング・Context・バインディングと検証・ミドルウェア・終了処理・テストをまとめたチートシート

- URL: https://cs.taka1156.site/gin.html
- タグ: Go, API・バックエンド
- 日付: 2026-11-20

---

Go の HTTP Web フレームワーク。ルーティング・ミドルウェア・バインディング（入力の検証）・レンダリングをまとめて提供する（v1.12 時点）

**ざっくり言うと**

- Gin は `net/http` の上で動く Go の Web フレームワークで、 `r.GET(path, handler)` でルートを書く。
- ハンドラーは `*gin.Context` を受け取り、 `c.Param` ・ `ShouldBind…` で入力を読み、 `c.JSON` で返す。
- 入力の検証は構造体タグ `binding:"required"` で書き、失敗時の応答を自分で決めたいなら `ShouldBind` 系を使う。

## 始め方

```
go mod init example.com/app
go get -u github.com/gin-gonic/gin   # v1.12 は Go 1.25 以上
```

```
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()   // Logger と Recovery 入り。何も入れないなら gin.New()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"message": "pong"})
	})
	r.Run(":8080")       // 省略時は :8080（環境変数 PORT があればそれ）
}
```

`gin.H` は `map[string]any` の別名。本番では `GIN_MODE=release` か `gin.SetMode(gin.ReleaseMode)` でデバッグ出力を止める。

## ルーティング

| 書き方 | 意味 |
| --- | --- |
| `r.GET("/users", h)` | `GET` ・ `POST` ・ `PUT` ・ `PATCH` ・ `DELETE` ・ `OPTIONS` ・ `HEAD` |
| `r.Any("/x", h)` ・ `r.Match([]string{"GET","POST"}, "/x", h)` | 全メソッド・指定した複数メソッド |
| `/users/:id` | パスパラメータ → `c.Param("id")` |
| `/files/*path` | 残り全部（ `c.Param("path")` は `/` から始まる） |
| `v1 := r.Group("/v1")` | 共通の接頭辞とミドルウェアをまとめる（ `r.Group("/admin", auth)` ） |
| `r.NoRoute(h)` ・ `r.NoMethod(h)` | 404・405 のときのハンドラー（405 を返すには `r.HandleMethodNotAllowed = true` が必要。既定は無効で 404 になる） |
| `r.Static("/assets", "./assets")` ・ `r.StaticFile("/favicon.ico", "./f.ico")` | 静的ファイル |

```
v1 := r.Group("/v1")
{
	v1.GET("/users", listUsers)
	v1.GET("/users/:id", getUser)
	v1.POST("/users", createUser)
}
```

## Context（c）でよく使うもの

入力

出力

値の受け渡し

| メソッド | 返すもの |
| --- | --- |
| `c.Param("id")` | パスパラメータ |
| `c.Query("q")` ・ `c.DefaultQuery("page", "1")` | クエリ文字列（無ければ空文字・既定値） |
| `c.GetQuery("q")` | 値と「あったか」の `bool` |
| `c.QueryArray("tag")` ・ `c.QueryMap("ids")` | `?tag=a&tag=b` の配列・ `?ids[a]=1` のマップ |
| `c.PostForm("name")` ・ `c.DefaultPostForm(…)` | フォームの値 |
| `c.FormFile("file")` ・ `c.SaveUploadedFile(f, dst)` | アップロードされたファイル |
| `c.GetHeader("Authorization")` ・ `c.Cookie("sid")` | ヘッダー・Cookie |
| `c.ClientIP()` | クライアントの IP（信頼するプロキシの設定に左右される。下の「よくある誤解」） |
| `c.Request` | 元の `*http.Request` （ `c.Request.Context()` など） |

```
c.JSON(http.StatusOK, user)                  // JSON
c.IndentedJSON(200, v)                       // 整形した JSON（開発用）
c.PureJSON(200, v)                           // < > などをエスケープしない
c.String(200, "hello %s", name)
c.XML(200, v); c.YAML(200, v); c.TOML(200, v)
c.HTML(200, "index.tmpl", gin.H{"title": "Top"})   // r.LoadHTMLGlob("templates/*") が必要
c.Redirect(http.StatusFound, "/login")
c.File("./report.pdf"); c.FileAttachment("./a.csv", "data.csv")
c.Data(200, "image/png", bytes)
c.Header("X-Request-Id", id)                 // 返す前にヘッダーを足す
c.Status(http.StatusNoContent)
```

```
// ミドルウェアで入れて、ハンドラーで取り出す（リクエストごとの値）
c.Set("userID", 42)
id := c.GetInt("userID")          // GetString・GetBool・GetTime なども
v, ok := c.Get("userID")          // any と bool
v := c.MustGet("userID")          // 無ければ panic
c.Delete("userID")                // v1.12 で追加
```

（元のページで操作できます → https://cs.taka1156.site/gin.html#context）

## バインディングと検証

構造体タグで「どこから読むか」と「検証ルール」を書き、 `ShouldBind…` で詰める。検証は [go-playground/validator](https://github.com/go-playground/validator) のルール（ `required` ・ `min` ・ `max` ・ `email` ・ `oneof` など）。

```
type CreateUser struct {
	Name  string `json:"name"  binding:"required,max=50"`
	Email string `json:"email" binding:"required,email"`
	Age   int    `json:"age"   binding:"gte=0,lte=150"`
}

func createUser(c *gin.Context) {
	var in CreateUser
	if err := c.ShouldBindJSON(&in); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	c.JSON(http.StatusCreated, in)
}
```

| メソッド | 読む場所（タグ） |
| --- | --- |
| `ShouldBindJSON` | JSON ボディ（ `json:"…"` ） |
| `ShouldBindQuery` | クエリ文字列だけ（ `form:"…"` 。 `form:"page,default=1"` で既定値） |
| `ShouldBind` | Content-Type とメソッドで自動選択（GET はクエリ、POST の JSON・フォームなど） |
| `ShouldBindUri` | パスパラメータ（ `uri:"id"` ） |
| `ShouldBindHeader` | ヘッダー（ `header:"…"` ） |
| `ShouldBindBodyWithJSON` | ボディを保存してから読む。同じボディを別の構造体で 2 回読みたいとき |

`Bind…` （ `Should` なし）は失敗すると自動で 400 を書き込み `Abort` する。エラーの形を自分で決めたいなら `ShouldBind…` を使う。

## ミドルウェア

`c.Next()` の前がハンドラーの前、後がハンドラーの後に動く。登録した順に入り、逆の順に出る。止めるときは `c.Abort…` 。

**Logger 前半** 1 番目に登録

**auth 前半** ここで Abort すると先へ進まない

**handler** c.JSON などで応答

**auth 後半** `c.Next()` の後

**Logger 後半** 所要時間などを記録

```
func Auth() gin.HandlerFunc {
	return func(c *gin.Context) {
		token := c.GetHeader("Authorization")
		if token == "" {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
			return   // Abort しても関数は続くので return も書く
		}
		c.Set("userID", lookup(token))
		c.Next()
	}
}

r := gin.New()
r.Use(gin.Logger(), gin.Recovery())   // 全体に
admin := r.Group("/admin", Auth())    // グループだけに
```

| 組み込み | 用途 |
| --- | --- |
| `gin.Logger()` ・ `gin.LoggerWithConfig(…)` | アクセスログ（ `SkipPaths` で除外） |
| `gin.Recovery()` ・ `gin.CustomRecovery(fn)` | panic を 500 にして落ちないようにする |
| `gin.BasicAuth(gin.Accounts{…})` | Basic 認証 |

CORS・セッション・gzip などは [gin-contrib](https://github.com/gin-contrib) の別モジュール（ `github.com/gin-contrib/cors` など）。

## エラー処理

```
// ハンドラーではエラーを積んで、最後にまとめて返す書き方もできる
func ErrorHandler() gin.HandlerFunc {
	return func(c *gin.Context) {
		c.Next()
		if len(c.Errors) > 0 {
			c.JSON(-1, gin.H{"error": c.Errors.Last().Error()})   // -1 は既に決まったステータスを使う
		}
	}
}

func getUser(c *gin.Context) {
	u, err := repo.Find(c.Request.Context(), c.Param("id"))
	if err != nil {
		c.Status(http.StatusNotFound)
		_ = c.Error(err)   // c.Errors に積む
		return
	}
	c.JSON(200, u)
}
```

## サーバーの設定と終了処理

```
r := gin.Default()
r.SetTrustedProxies([]string{"10.0.0.0/8"})   // プロキシを使わないなら nil

srv := &http.Server{
	Addr:              ":8080",
	Handler:           r,                   // *gin.Engine は http.Handler
	ReadHeaderTimeout: 5 * time.Second,
}
go func() {
	if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
		log.Fatal(err)
	}
}()

ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()
<-ctx.Done()
shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
srv.Shutdown(shutdownCtx)   // 処理中のリクエストを待ってから止める
```

`r.Run()` はタイムアウトも終了処理も設定できないので、本番は `http.Server` に渡す形にする。

## テスト

```
func TestPing(t *testing.T) {
	gin.SetMode(gin.TestMode)
	r := setupRouter()   // ルートを登録した *gin.Engine を返す関数に分けておく

	w := httptest.NewRecorder()
	req := httptest.NewRequest(http.MethodGet, "/ping", nil)
	r.ServeHTTP(w, req)

	if w.Code != 200 || !strings.Contains(w.Body.String(), "pong") {
		t.Fatalf("got %d %s", w.Code, w.Body.String())
	}
}
```

## API サーバーを作る手順

1. **ルーターを関数に分ける** `setupRouter() *gin.Engine` にすると main とテストで同じものを使える
2. **グループで区切る** `/v1` ・ `/admin` などを `Group` にし、認証はグループのミドルウェアにする
3. **入力は構造体で受ける** `binding` タグで検証し、 `ShouldBind…` のエラーを共通の形で返す
4. **DB 層をつなぐ** `c.Request.Context()` を渡す。クエリは [sqlc](https://cs.taka1156.site/md/sqlc.md) や [ent](https://cs.taka1156.site/md/ent.md) で生成すると型が付く
5. **本番の設定** `GIN_MODE=release` ・ `SetTrustedProxies` ・ `http.Server` のタイムアウトと `Shutdown`

## よくある誤解

✗ 誤解: c.ClientIP() は常に本当のクライアントの IP を返す

✓ 正しくは: 何も設定しないと Gin はすべてのプロキシを信頼し、 `X-Forwarded-For` の値をそのまま使う（公式も安全ではないと明記）。前段のプロキシのアドレスを `SetTrustedProxies` で指定するか、プロキシが無いなら `SetTrustedProxies(nil)` にする。CDN 経由なら `r.TrustedPlatform = gin.PlatformCloudflare` なども使える。

✗ 誤解: c.Abort() を呼べばハンドラーの残りの処理も止まる

✓ 正しくは: `Abort` は「この後のハンドラー（チェーンの続き）を呼ばない」印を付けるだけで、今の関数は最後まで実行される。止めたいところで `return` も書く。

✗ 誤解: ハンドラーで起動した goroutine にそのまま c を渡してよい

✓ 正しくは: `*gin.Context` はリクエストが終わると再利用される。goroutine で使うなら `cCp := c.Copy()` した読み取り専用の複製を渡す。

✗ 誤解: ShouldBindJSON でボディを読んだあと、もう一度読める

✓ 正しくは: ボディは一度読むと空になる。別の構造体でもう一度読むなら、最初から `ShouldBindBodyWithJSON` （読んだボディを Context に保存する）を使う。

関連: 同じ「Express 風」の書き方をする TypeScript のフレームワークは [Hono 使い方](https://cs.taka1156.site/md/hono.md) 。

## 出典

- [Gin Web Framework – Documentation](https://gin-gonic.com/en/docs/)
- [gin-gonic/gin docs/doc.md](https://github.com/gin-gonic/gin/blob/master/docs/doc.md) （ルーティング・バインディング・ミドルウェア・テストの例）
- [CHANGELOG](https://github.com/gin-gonic/gin/blob/master/CHANGELOG.md) （v1.12.0: `GetError` ・ `Delete` の追加、Go 1.25 以上）
- [pkg.go.dev – gin](https://pkg.go.dev/github.com/gin-gonic/gin)

確認日: 2026-11-20（gin v1.12.0）
