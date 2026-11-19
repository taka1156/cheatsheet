<!-- scripts/memo.py build が自動生成。手で編集しない -->

# zap 使い方

> Go の構造化ロガー zap の Logger と SugaredLogger、フィールド、Config、zapcore、レベル変更、テストをまとめたチートシート

- URL: https://cs.taka1156.site/zap.html
- タグ: Go, API・バックエンド
- 日付: 2026-11-20

---

Uber 製の Go 向け構造化ロガー（structured logger）。速さとメモリ確保の少なさが売り（v1.28 時点）

**ざっくり言うと**

- zap は JSON などの構造化ログを速く書く Go のロガーで、 `zap.NewProduction()` で始められる。
- 型付きフィールドで書く `*zap.Logger` と、 `printf` 風に書ける `*zap.SugaredLogger` の 2 種類がある。
- 出力先・形式・レベルは `zap.Config` か `zapcore` で組み、終了前に `Sync()` で書き出す。

## 始め方

```
go get -u go.uber.org/zap   # 対応する Go は最新の 2 つのマイナー版
```

```
package main

import (
	"time"

	"go.uber.org/zap"
)

func main() {
	logger, err := zap.NewProduction()   // 開発中は zap.NewDevelopment()
	if err != nil {
		panic(err)
	}
	defer logger.Sync()   // バッファに残ったログを書き出す

	logger.Info("server started",
		zap.String("addr", ":8080"),
		zap.Int("workers", 4),
		zap.Duration("boot", 120*time.Millisecond),
	)
}
```

出力（1 行の JSON。 `ts` は UNIX 秒）:

```
{"level":"info","ts":1763600000.123,"caller":"app/main.go:16","msg":"server started","addr":":8080","workers":4,"boot":0.12}
```

## Logger と SugaredLogger

Logger

SugaredLogger

使い分け

メッセージ＋型付きフィールド。リフレクションを使わないので最も速く、メモリ確保もほぼ無い。

```
logger.Info("user login",
	zap.String("user", "alice"),
	zap.Int64("id", 42),
	zap.Error(err),          // キーは "error"
)
logger.Warn("slow query", zap.Duration("took", d))
```

`logger.Sugar()` で作る。キーと値を交互に並べる・ `printf` 風に書ける。Logger より 2〜4 割ほど遅く、メモリ確保も多いが、それでも多くのロガーより速い。

```
sugar := logger.Sugar()
sugar.Infow("user login", "user", "alice", "id", 42)   // w: キーと値の組
sugar.Infof("user %s logged in", "alice")             // f: printf 風
sugar.Info("user ", "alice", " logged in")            // fmt.Sprint 風
sugar.Infoln("user", "alice")                          // fmt.Sprintln 風（間に空白）

plain := sugar.Desugar()   // Logger に戻す（コストはほぼ無い）
```

| 観点 | Logger | SugaredLogger |
| --- | --- | --- |
| 書き方 | `Info(msg, zap.String(k, v))` | `Infow(msg, k, v)` ・ `Infof(…)` |
| 型の確認 | コンパイル時 | 実行時（キーが文字列でないと開発モードで panic） |
| 速さ | 最速・確保ほぼ 0 | 2〜4 割遅い・確保が多い |
| 向く所 | リクエスト処理などよく通る所・ライブラリ | 起動処理・CLI・速さが要らない所 |

1 つのアプリで両方を使ってよい。 `Sugar()` と `Desugar()` はどちらも安い。

（元のページで操作できます → https://cs.taka1156.site/zap.html#kinds）

## 用意されたロガー

| 関数 | 形式 | 最低レベル | スタックトレース | そのほか |
| --- | --- | --- | --- | --- |
| `zap.NewProduction()` | JSON・ `ts` は UNIX 秒 | Info | Error 以上 | サンプリングあり（1 秒ごと、同じレベル・メッセージは最初の 100 件、以降 100 件に 1 件）。出力は stderr |
| `zap.NewDevelopment()` | console（タブ区切り）・ISO 8601 の時刻 | Debug | Warn 以上 | 開発モード: `DPanic` で panic する。出力は stderr |
| `zap.NewExample()` | JSON・時刻と caller なし | Debug | なし | 出力が毎回同じになるのでテストやドキュメントの例向け。stdout |
| `zap.NewNop()` | 何も出さない | — | — | ロガーを受け取る関数に「ログ不要」を渡すとき |

開発用の出力（ `NewDevelopment` ）:

```
2026-11-20T10:00:00.000+0900	INFO	app/main.go:16	server started	{"addr": ":8080", "workers": 4, "boot": "120ms"}
```

どれもオプションを渡せる: `zap.NewProduction(zap.AddCaller(), zap.Fields(zap.String("app", "api")))` 。エラーを返さない形にするなら `zap.Must(zap.NewProduction())` （失敗すると panic）。

## レベル

**Debug** -1

**Info** 0（既定）

**Warn** 1

**Error** 2

**DPanic** 開発モードだけ panic

**Panic** 書いてから panic

**Fatal** 書いてから os.Exit(1)

赤い枠の 2 つは、ログを書いたあとに必ず処理を止める（レベルで出力が無効になっていても止める）。 `DPanic` は「起きてはいけないが回復できる」ことに使い、本番では Error のように書くだけで続行する。

```
logger.Log(zap.WarnLevel, "msg")              // レベルを変数で渡す
if ce := logger.Check(zap.DebugLevel, "dump"); ce != nil {
	ce.Write(zap.Any("state", expensive()))   // 出すときだけ重い値を作る
}
lvl, _ := zapcore.ParseLevel("warn")         // 文字列から（"debug" "info" "warn" "error" "dpanic" "panic" "fatal"）
```

## フィールド

| 関数 | 用途 |
| --- | --- |
| `zap.String` ・ `Int` ・ `Int64` ・ `Uint` ・ `Float64` ・ `Bool` | 基本の型。ポインタ版（ `Stringp` など。nil なら `null` ）と、スライス版（ `Strings` ・ `Ints` など）もある |
| `zap.Duration` ・ `zap.Time` | 時間（形式は EncoderConfig の `EncodeDuration` ・ `EncodeTime` で決まる） |
| `zap.Error(err)` ・ `zap.NamedError("cause", err)` | エラー。キーは `error` ／指定した名前。 `Errors("errs", list)` で複数 |
| `zap.Any("k", v)` | 型を見て適切なフィールドを選ぶ。分からない型はリフレクション（ `zap.Reflect` ）になり遅い |
| `zap.Stringer("k", v)` | `String()` を持つ値（書くときに呼ぶ） |
| `zap.Object("k", v)` ・ `zap.Array("k", v)` | `zapcore.ObjectMarshaler` ・ `ArrayMarshaler` を実装した型（下の「独自の型」） |
| `zap.Inline(v)` | ObjectMarshaler の中身を入れ子にせず同じ階層に展開 |
| `zap.Dict("k", fields...)` | その場でフィールドをまとめて入れ子のオブジェクトにする |
| `zap.Namespace("k")` | これ以降のフィールドを `k` の中に入れる |
| `zap.Stack("stack")` ・ `zap.Skip()` | 今のスタックトレース／何もしないフィールド（条件付きで足すとき） |
| `zap.Binary` ・ `zap.ByteString` | バイト列（base64）／UTF-8 の文字列として |

```
logger.Info("order",
	zap.Dict("user", zap.String("name", "alice"), zap.Int("id", 42)),
	zap.Namespace("payment"),
	zap.String("method", "card"),
)
// {…,"msg":"order","user":{"name":"alice","id":42},"payment":{"method":"card"}}（level・ts などは省略）
```

## 子ロガー（With・Named）

```
reqLog := logger.With(zap.String("request_id", id))   // 以後のログに毎回付く
reqLog.Info("start")                                  // {"msg":"start","request_id":"…"}

db := logger.Named("db")          // "logger":"db"。重ねると "db.pool"
db.Info("connected")

lazy := logger.WithLazy(zap.Object("cfg", cfg))   // 実際に書くときまでエンコードを遅らせる
quiet := logger.WithOptions(zap.IncreaseLevel(zap.WarnLevel))   // この子だけ Warn 以上
```

`With` したフィールドはその場で一度だけエンコードされ、毎回のログでは使い回される。リクエストごとの値は `With` で付けた子ロガーを `context.Context` などで渡すとよい（zap 自体には context から取り出す仕組みは無い）。

## オプション（zap.Option）

| オプション | 効果 |
| --- | --- |
| `zap.AddCaller()` ・ `zap.WithCaller(false)` | 呼び出し元のファイル:行を付ける／外す |
| `zap.AddCallerSkip(1)` | ロガーを包む関数を自分で書いたとき、caller をその分だけ上にずらす |
| `zap.AddStacktrace(zap.ErrorLevel)` | このレベル以上にスタックトレースを付ける |
| `zap.Fields(fs...)` | 全ログに付けるフィールド（ `With` と同じ） |
| `zap.IncreaseLevel(lvl)` | 最低レベルを上げる（下げることはできない） |
| `zap.Development()` | 開発モード（ `DPanic` で panic、Sugar の不正なキーで panic） |
| `zap.Hooks(func(zapcore.Entry) error)` | 書くたびに呼ばれる関数（件数の計測など） |
| `zap.WrapCore(func(zapcore.Core) zapcore.Core)` | Core を差し替える・包む（出力先の追加など） |
| `zap.ErrorOutput(ws)` | zap 自身の内部エラーの書き先 |
| `zap.WithFatalHook(…)` ・ `zap.WithPanicHook(…)` | Fatal・Panic の後の動作を変える（テストで `os.Exit` させない、など） |
| `zap.WithClock(clock)` | 時刻の取得元を差し替える（テスト用） |

## Config で組み立てる

プリセットを元に一部だけ変えるのが簡単。 `zap.Config` は JSON・YAML のタグを持つので、設定ファイルから読むこともできる。

```
cfg := zap.NewProductionConfig()
cfg.Level = zap.NewAtomicLevelAt(zap.DebugLevel)
cfg.OutputPaths = []string{"stdout", "/var/log/app.log"}
cfg.EncoderConfig.TimeKey = "time"
cfg.EncoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder   // ts を読める形に
cfg.Sampling = nil                                          // サンプリングを止める
cfg.InitialFields = map[string]any{"service": "api"}

logger, err := cfg.Build()   // ここにも zap.Option を渡せる
```

| フィールド（JSON キー） | 意味 |
| --- | --- |
| `Level` （ `level` ） | 最低レベル。 `AtomicLevel` なので後から変えられる |
| `Development` （ `development` ） | 開発モード。スタックトレースの既定も Warn 以上になる |
| `DisableCaller` ・ `DisableStacktrace` | caller・スタックトレースを付けない |
| `Sampling` （ `sampling` ） | `{Initial, Thereafter}` 。 `nil` で無効 |
| `Encoding` （ `encoding` ） | `"json"` か `"console"` |
| `EncoderConfig` （ `encoderConfig` ） | キー名（ `MessageKey` ・ `LevelKey` ・ `TimeKey` など）と値の形式（ `EncodeLevel` ・ `EncodeTime` ・ `EncodeCaller` ） |
| `OutputPaths` （ `outputPaths` ） | 書き先。 `"stdout"` ・ `"stderr"` ・ファイルパス |
| `ErrorOutputPaths` | zap 自身の内部エラーの書き先 |
| `InitialFields` （ `initialFields` ） | 全ログに付けるフィールド |

### EncoderConfig でよく変えるもの

| 項目 | 選べる値の例 |
| --- | --- |
| `EncodeTime` | `zapcore.EpochTimeEncoder` （本番の既定）・ `ISO8601TimeEncoder` ・ `RFC3339TimeEncoder` ・ `RFC3339NanoTimeEncoder` ・ `zapcore.TimeEncoderOfLayout("2006-01-02 15:04:05")` |
| `EncodeLevel` | `LowercaseLevelEncoder` （info）・ `CapitalLevelEncoder` （INFO）・ `CapitalColorLevelEncoder` （色付き。console 向け） |
| `EncodeDuration` | `SecondsDurationEncoder` （0.12）・ `StringDurationEncoder` （"120ms"）・ `MillisDurationEncoder` |
| `EncodeCaller` | `ShortCallerEncoder` （pkg/file.go:12）・ `FullCallerEncoder` （フルパス） |
| キー名 | `zapcore.OmitKey` （空文字）を入れるとその項目を出さない |

## zapcore で組み立てる

Config で足りないとき（出力先ごとにレベルや形式を変える、ローテーションするファイルに書く、など）は Core を自分で作る。Core は「どう書くか」「どこへ書くか」「どのレベルを書くか」の 3 つでできている。

***zap.Logger** Info などを呼ぶ所

zapcore.Core = NewCore(…)

**Encoder** どう書くか（JSON・console）

**WriteSyncer** どこへ書くか（stdout・ファイル）

**LevelEnabler** どのレベルを書くか

**書き先** 端末・ファイル・収集基盤

```
encCfg := zap.NewProductionEncoderConfig()
encCfg.EncodeTime = zapcore.ISO8601TimeEncoder

// Error 以上はファイルへ JSON、全部を stdout へ console 形式で
file := zapcore.AddSync(&lumberjack.Logger{   // gopkg.in/natefinch/lumberjack.v2
	Filename: "/var/log/app/error.log", MaxSize: 100, MaxBackups: 3, MaxAge: 28,
})
core := zapcore.NewTee(
	zapcore.NewCore(zapcore.NewJSONEncoder(encCfg), file, zap.ErrorLevel),
	zapcore.NewCore(zapcore.NewConsoleEncoder(encCfg), zapcore.Lock(os.Stdout), zap.DebugLevel),
)
logger := zap.New(core, zap.AddCaller(), zap.AddStacktrace(zap.ErrorLevel))
```

| 関数 | 用途 |
| --- | --- |
| `zapcore.NewCore(enc, ws, lvl)` | Core を作る |
| `zapcore.NewTee(cores...)` | 複数の Core に同じログを流す |
| `zapcore.AddSync(w)` | `io.Writer` を WriteSyncer にする |
| `zapcore.Lock(ws)` | 排他制御を付ける（ `*os.File` を複数の goroutine から書くとき） |
| `zapcore.NewMultiWriteSyncer(ws...)` | 同じ形式のまま複数の書き先へ |
| `&zapcore.BufferedWriteSyncer{WS: ws}` | バッファしてまとめて書く（ `Sync` ・ `Stop` を忘れない） |
| `zapcore.NewSamplerWithOptions(core, time.Second, 100, 100)` | サンプリングを付ける |
| `zap.LevelEnablerFunc(func(l zapcore.Level) bool {…})` | 「Warn だけ」など任意の条件でレベルを選ぶ |

zap 自体にはファイルのローテーション機能が無い。公式 FAQ は `logrotate` などの外部の仕組みか、 `lumberjack` を WriteSyncer として使う方法を案内している。

## 実行中にレベルを変える

```
atom := zap.NewAtomicLevelAt(zap.InfoLevel)
cfg := zap.NewProductionConfig()
cfg.Level = atom
logger := zap.Must(cfg.Build())

atom.SetLevel(zap.DebugLevel)        // 再起動せずに変わる（goroutine 安全）

http.Handle("/log/level", atom)      // AtomicLevel は http.Handler
// GET  → {"level":"info"}
// PUT  → curl -X PUT localhost:8080/log/level -d level=debug
//        curl -X PUT localhost:8080/log/level -H "Content-Type: application/json" -d '{"level":"debug"}'
```

レベルを変えるエンドポイントは管理用のポートに置くなど、外から触れないようにする。

## グローバルロガー・標準 log・slog

グローバル

標準 log

slog

```
undo := zap.ReplaceGlobals(logger)   // 戻す関数が返る
defer undo()

zap.L().Info("via global")           // *Logger
zap.S().Infof("via %s", "sugar")     // *SugaredLogger
```

`ReplaceGlobals` を呼ぶまで `zap.L()` ・ `zap.S()` は `NewNop()` で、何も出さない。公式はグローバルよりも引数で渡すことを勧めている。

```
undo := zap.RedirectStdLog(logger)   // log.Print などを Info で zap へ
defer undo()

std := zap.NewStdLog(logger)                      // *log.Logger を作る（http.Server.ErrorLog など）
std, _ = zap.NewStdLogAt(logger, zap.ErrorLevel)  // レベルを指定
```

`log/slog` の API で書いて、出力は zap の Core に任せる。 `go.uber.org/zap/exp` は実験的なモジュールで、API が変わることがある（v0.3.0 で `HandlerOptions` から関数オプションへ変わった）。

```
import "go.uber.org/zap/exp/zapslog"   // go get go.uber.org/zap/exp（Go 1.21 以上）

h := zapslog.NewHandler(logger.Core(),
	zapslog.WithCaller(true),
	zapslog.WithName("api"),
)
slog.SetDefault(slog.New(h))
slog.Info("hello", "user", "alice")
```

（元のページで操作できます → https://cs.taka1156.site/zap.html#global）

## 独自の型をログに出す

`zapcore.ObjectMarshaler` を実装すると、 `zap.Object` でリフレクション無しに書ける。出したくない項目（パスワードなど）をここで省ける。

```
type User struct {
	Name     string
	Email    string
	Password string
}

func (u User) MarshalLogObject(enc zapcore.ObjectEncoder) error {
	enc.AddString("name", u.Name)
	enc.AddString("email", u.Email)
	// Password は書かない
	return nil
}

logger.Info("signup", zap.Object("user", u))   // {"user":{"name":"…","email":"…"}}
logger.Info("signup", zap.Inline(u))           // {"name":"…","email":"…"}

// スライスは ObjectValues / Objects（ジェネリクス）で
logger.Info("batch", zap.ObjectValues("users", users))   // []User（メソッドがポインタ受け取りでも可）
```

## テスト

zaptest

observer

ログを `t.Log` に流す。失敗したテストのときだけ表示されるので見通しがよい。

```
import "go.uber.org/zap/zaptest"

func TestService(t *testing.T) {
	logger := zaptest.NewLogger(t, zaptest.Level(zap.WarnLevel))
	svc := NewService(logger)
	// …
}
```

書かれたログをメモリに貯めて、内容を確かめる。

```
import "go.uber.org/zap/zaptest/observer"

core, logs := observer.New(zap.InfoLevel)
logger := zap.New(core)

DoSomething(logger)

if logs.FilterMessage("retrying").Len() != 3 {
	t.Errorf("got %v", logs.All())
}
entry := logs.All()[0]
_ = entry.ContextMap()   // フィールドを map[string]any で
```

（元のページで操作できます → https://cs.taka1156.site/zap.html#test）

## Gin とつなぐ

[Gin](https://cs.taka1156.site/md/gin.md) の組み込みの Logger・Recovery の代わりに、 [gin-contrib/zap](https://github.com/gin-contrib/zap) を使うとアクセスログと panic を zap で書ける。

```
import ginzap "github.com/gin-contrib/zap"

r := gin.New()
r.Use(ginzap.Ginzap(logger, time.RFC3339, true))   // 第 3 引数: 時刻を UTC にする
r.Use(ginzap.RecoveryWithZap(logger, true))        // 第 2 引数: スタックトレースを付ける
```

## アプリに入れる手順

1. **main で 1 つ作る** 環境変数などで `NewProductionConfig` と `NewDevelopmentConfig` を切り替え、 `defer logger.Sync()`
2. **引数で渡す** 構造体のフィールドやコンストラクタの引数に `*zap.Logger` を持たせる。使わない所には `zap.NewNop()`
3. **文脈を With で付ける** リクエスト ID・ユーザー ID などは子ロガーに付けて渡す。部品ごとに `Named`
4. **よく通る所は Logger** 型付きフィールドで書き、重い値は `Check` で出すときだけ作る
5. **本番の設定** 時刻の形式・サンプリング・出力先を決め、 `AtomicLevel` で後からレベルを変えられるようにする

## よくある誤解

✗ 誤解: Sync() がエラーを返したらログが失われている

✓ 正しくは: 出力先が端末の stderr・stdout のとき、OS によっては `fsync` に対応しておらず `sync /dev/stderr: invalid argument` などのエラーが返る。書き込み自体は済んでいるので、 `defer logger.Sync()` の戻り値は無視するのが一般的（ `_ = logger.Sync()` ）。ファイルに書いているときのエラーは確かめる。

✗ 誤解: logger.Fatal を呼んでも defer の後始末は実行される

✓ 正しくは: `Fatal` はログを書いたあと `os.Exit(1)` を呼ぶので、 `defer` は実行されない（DB の切断なども行われない）。後始末が要るなら、エラーを返して main で終わらせるか、 `zap.WithFatalHook` で動作を変える。

✗ 誤解: zap.Any を使えばどんな型でも速く書ける

✓ 正しくは: `Any` は知っている型なら専用のフィールドを選ぶが、それ以外は `zap.Reflect` （ `encoding/json` 相当のリフレクション）になり遅い。よく出す独自の型は `ObjectMarshaler` を実装して `zap.Object` で書く。

✗ 誤解: NewProduction のロガーは渡したログを全部出す

✓ 正しくは: 本番の設定はサンプリングが有効で、1 秒の間に同じレベル・同じメッセージが 100 件を超えると、以降は 100 件に 1 件しか出さない。全部要るなら `cfg.Sampling = nil` にして `cfg.Build()` する。

✗ 誤解: ReplaceGlobals を呼ばなくても zap.L() でログが出る

✓ 正しくは: グローバルロガーの初期値は `NewNop()` なので、置き換えるまで何も出ない。

関連: HTTP サーバーのフレームワークは [Gin 使い方](https://cs.taka1156.site/md/gin.md) 。

## 出典

- [pkg.go.dev – go.uber.org/zap](https://pkg.go.dev/go.uber.org/zap) ・ [zapcore](https://pkg.go.dev/go.uber.org/zap/zapcore)
- [uber-go/zap](https://github.com/uber-go/zap) （README・ [FAQ](https://github.com/uber-go/zap/blob/master/FAQ.md) ・ [CHANGELOG](https://github.com/uber-go/zap/blob/master/CHANGELOG.md) 。既定値は `config.go` のソースで確認）
- [zap/exp CHANGELOG](https://github.com/uber-go/zap/blob/master/exp/CHANGELOG.md) （zapslog v0.3.0）
- [gin-contrib/zap](https://github.com/gin-contrib/zap)

確認日: 2026-11-20（zap v1.28.0・zap/exp v0.3.0）
