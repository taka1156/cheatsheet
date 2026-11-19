<!-- scripts/memo.py build が自動生成。手で編集しない -->

# Linux よく使うコマンド

> ファイル操作・検索・プロセス・ネットワーク・サービス管理・タイムゾーン確認など、Linux の日常でよく使うコマンドを用途別にまとめたチートシート

- URL: https://cs.taka1156.site/linux-commands.html
- タグ: Linux
- 日付: 2026-11-19

---

ファイル・検索・プロセス・ネットワーク・サービス管理を用途別に引くチートシート

**ざっくり言うと**

- 日常の作業は「移動・ファイル操作・閲覧・検索・権限・プロセス・ネットワーク・圧縮・システム」の 9 種類でほぼ足りる。
- 小さなコマンドを `|` （パイプ）でつなぎ、 `>` でファイルに書き出すのが Linux の基本の組み立て方。
- `rm` はゴミ箱を通らず即削除、 `kill -9` は後片付けなしで止めるので、使う前に一度確認する。

## コマンド チートシート

オプションは GNU coreutils など一般的な Linux ディストリビューションのもの。macOS（BSD 系）では一部違う。 危険 は取り消せない・影響が大きい操作。

移動・確認

ファイル操作

閲覧

検索・テキスト処理

権限・ユーザー

プロセス

ネットワーク

圧縮・アーカイブ

システム・サービス

| コマンド | やること |
| --- | --- |
| pwd / cd |  |
| `pwd` | 今いるディレクトリ（カレントディレクトリ）の絶対パスを表示 |
| `cd /var/log` | 指定したディレクトリへ移動 |
| `cd` / `cd ~` | ホームディレクトリへ戻る |
| `cd ..` | 1 つ上の階層へ |
| `cd -` | 直前にいたディレクトリへ戻る（2 か所を行き来するとき便利） |
| ls |  |
| `ls` | ファイル・ディレクトリの一覧 |
| `ls -l` | 詳細表示（権限・所有者・サイズ・更新日時） |
| `ls -la` | `.` で始まる隠しファイルも含めて詳細表示 |
| `ls -lh` | サイズを K / M / G の読みやすい単位で |
| `ls -lt` | 更新日時の新しい順（ `-ltr` で古い順＝最新が一番下） |
| `ls -lS` | サイズの大きい順 |
| `ls -d */` | ディレクトリだけを表示 |
| その他 |  |
| `tree -L 2` | 2 階層までをツリー表示（別途インストールが必要なことが多い） |
| `clear` | 画面をクリア（ `Ctrl + L` でも同じ） |
| `history` | 実行したコマンドの履歴。 `!番号` で再実行、 `Ctrl + R` で履歴を検索 |
| `!!` | 直前のコマンドを再実行（ `sudo !!` で直前のコマンドを sudo 付きで） |

| コマンド | やること |
| --- | --- |
| 作る：mkdir / touch |  |
| `mkdir dir` | ディレクトリを作る |
| `mkdir -p a/b/c` | 途中の階層もまとめて作る。すでにあってもエラーにしない |
| `touch file.txt` | 空のファイルを作る。既にあれば更新日時だけ今にする |
| コピー・移動：cp / mv |  |
| `cp a.txt b.txt` | ファイルをコピー |
| `cp -r src/ dst/` | ディレクトリごと再帰的にコピー |
| `cp -a src/ dst/` | 権限・所有者・日時も保ったままコピー（バックアップ向き） |
| `cp -i a b` | 上書きする前に確認する |
| `mv old.txt new.txt` | 名前を変える（同じディレクトリ内での移動） |
| `mv file dir/` | ディレクトリへ移動 |
| `mv -i a b` | 上書きする前に確認する（ `-n` は上書きしない） |
| 消す：rm |  |
| `rm file` | ファイルを削除（ゴミ箱を通らない） |
| `rm -i file` | 1 つずつ確認してから削除 |
| `rm -r dir` | ディレクトリを中身ごと削除 |
| `rm -rf dir` 危険 | 確認なしで中身ごと削除（存在しないファイルのエラーは出さない）。 `rm -rf "$DIR"/*` のように書くと、変数が空のとき `/` 以下が消える事故になるので注意（GNU rm は `rm -rf /` 自体は既定で拒否する） |
| `rmdir dir` | 空のディレクトリだけを削除（中身があると失敗するので安全） |
| リンク |  |
| `ln -s 実体 リンク名` | シンボリックリンク（ショートカット）を作る。引数の順番に注意 |
| `readlink -f リンク` | リンクの最終的な実体のパスを表示 |

| コマンド | やること |
| --- | --- |
| cat / less |  |
| `cat file` | ファイルの中身を全部表示 |
| `cat -n file` | 行番号を付けて表示 |
| `cat a b > c` | 複数のファイルをつなげて 1 つにする（本来の用途は concatenate） |
| `less file` | 長いファイルをページ送りで読む。 `/語` で検索、 `n` で次、 `G` で末尾、 `q` で終了 |
| head / tail |  |
| `head file` | 先頭 10 行を表示 |
| `head -n 50 file` | 先頭 50 行 |
| `tail file` | 末尾 10 行を表示 |
| `tail -n 100 file` | 末尾 100 行 |
| `tail -f app.log` | 追記をリアルタイムで表示し続ける（ログ監視の定番。 `Ctrl + C` で終了） |
| `tail -F app.log` | `-f` に加え、ログがローテーションで作り直されても追いかける |
| 数える・比べる |  |
| `wc -l file` | 行数を数える（ `-w` 単語数、 `-c` バイト数） |
| `diff a b` | 2 つのファイルの差分（ `diff -u` で git と同じ形式） |

| コマンド | やること |
| --- | --- |
| grep（中身を探す） |  |
| `grep 'error' app.log` | 文字列を含む行を表示 |
| `grep -i 'error' file` | 大文字・小文字を区別しない |
| `grep -r 'TODO' src/` | ディレクトリ以下を再帰的に検索 |
| `grep -n 'word' file` | 行番号付きで表示 |
| `grep -v 'debug' file` | 含ま ない 行を表示（除外） |
| `grep -l 'word' *.txt` | 一致したファイル名だけ表示 |
| `grep -c 'word' file` | 一致した行数を表示 |
| `grep -w 'id' file` | 単語として一致するものだけ（ `uid` などは除く） |
| `grep -E 'warn\|error' file` | 拡張正規表現（ `\|` で OR など） |
| `grep -A 3 -B 1 'error' file` | 一致行の後 3 行・前 1 行も表示（ `-C 2` で前後 2 行） |
| `ps aux \| grep nginx` | 他のコマンドの出力から探す（パイプと組み合わせる定番） |
| find（ファイルを探す） |  |
| `find . -name '*.log'` | 名前で探す（ `-iname` で大文字小文字を無視）。パターンは引用符で囲む |
| `find . -type f` / `-type d` | ファイルだけ／ディレクトリだけ |
| `find . -mtime -7` | 7 日以内に更新されたもの（ `+7` は端数を切り捨てて数えるため 8 日以上前。分単位なら `-mmin` ） |
| `find . -size +100M` | 100MB より大きいもの |
| `find . -maxdepth 2 -name x` | 2 階層までに限って探す |
| `find . -name '*.tmp' -delete` 危険 | 見つけたものを削除。まず `-delete` なしで対象を確かめる |
| `find . -name '*.sh' -exec chmod +x {} +` | 見つけたもの全部にコマンドを実行 |
| コマンドの場所 |  |
| `which python3` | PATH 上でどの実行ファイルが使われるかを表示 |
| `type ll` | エイリアス・シェル組み込み・関数も含めて正体を表示 |
| テキストの加工 |  |
| `sort file` | 並べ替え（ `-n` 数値順、 `-r` 逆順、 `-h` 1K・2M などの単位付き） |
| `sort \| uniq -c \| sort -rn` | 重複を数えて多い順に（集計の定番） |
| `cut -d ',' -f 1,3 file.csv` | 区切り文字で列を取り出す |
| `awk '{print $1}' file` | 空白区切りの 1 列目を取り出す |
| `sed 's/old/new/g' file` | 置換して表示（ `-i` でファイルを直接書き換え） |
| `... \| xargs rm` | 前のコマンドの出力を引数にして実行 |

| コマンド | やること |
| --- | --- |
| chmod（権限を変える） |  |
| `chmod +x script.sh` | 実行権限を付ける |
| `chmod 755 file` | 所有者は全部、グループ・他人は読み＋実行（スクリプト・ディレクトリ向き） |
| `chmod 644 file` | 所有者は読み書き、他は読みだけ（普通のファイル） |
| `chmod 600 ~/.ssh/id_ed25519` | 所有者だけ読み書き（グループや他人が読める秘密鍵は ssh が無視する。400 でもよい） |
| `chmod u+w,g-w file` | 記号で指定（u 所有者・g グループ・o 他人・a 全員） |
| `chmod -R 755 dir` | ディレクトリ以下にまとめて適用 |
| `chmod 777 file` 危険 | 誰でも書き換え可能になる。動かないときの回避策として使わない |
| 所有者・ユーザー |  |
| `chown user:group file` | 所有者とグループを変える（ `-R` で再帰） |
| `sudo コマンド` | 管理者（root）権限で実行 |
| `whoami` / `id` | 今のユーザー名／UID・所属グループ |

| コマンド | やること |
| --- | --- |
| 見る |  |
| `ps aux` | 全プロセスの一覧（PID・CPU・メモリ・コマンド） |
| `ps aux \| grep node` | 名前でプロセスを探す（ `pgrep -a node` でも同じ） |
| `top` / `htop` | CPU・メモリを使っている順にリアルタイム表示（ `q` で終了） |
| lsof（開いているファイル・ポート） |  |
| `lsof -i :3000` | ポート 3000 を使っているプロセスを表示（「ポートが使用中」の調査の定番） |
| `lsof -i -P -n` | 通信中の全プロセス（ `-P -n` でポート番号・IP を名前に変換せず速く） |
| `lsof -p 1234` | PID 1234 が開いているファイル |
| `lsof +D /path` | そのディレクトリ以下を開いているプロセス（アンマウントできないときの調査） |
| `lsof -t -i :3000` | PID だけを出す（ `kill $(lsof -t -i :3000)` のように組み合わせる） |
| kill（止める） |  |
| `kill 1234` | PID 1234 に終了を依頼（SIGTERM。プロセスは後片付けしてから終われる） |
| `kill -9 1234` 危険 | 強制終了（SIGKILL）。後片付けされないので、普通の kill で止まらないときだけ |
| `kill -HUP 1234` | SIGHUP を送る（設定の再読み込みに使うデーモンが多い） |
| `pkill -f 'node server'` | コマンドラインの一部で一致したプロセスを止める |
| `killall nginx` | 名前が一致するプロセスを全部止める |
| `kill -l` | シグナルの一覧 |
| ジョブ（今のシェルで動かしているもの） |  |
| `コマンド &` | バックグラウンドで実行 |
| `Ctrl + Z` → `bg` | 実行中のものを一時停止 → バックグラウンドで再開 |
| `jobs` / `fg %1` | ジョブの一覧／1 番をフォアグラウンドに戻す |
| `nohup コマンド &` | ログアウトしても止まらないように実行（出力は `nohup.out` ） |

| コマンド | やること |
| --- | --- |
| ping（届くか） |  |
| `ping example.com` | 相手に届くか・応答時間を確かめる（Linux は止めるまで続く。 `Ctrl + C` ） |
| `ping -c 4 example.com` | 4 回だけ送って終わる |
| curl（HTTP） |  |
| `curl https://example.com` | GET してレスポンスの本文を表示 |
| `curl -I URL` | ヘッダーだけ取得（HEAD リクエスト） |
| `curl -i URL` | ヘッダーと本文の両方を表示 |
| `curl -L URL` | リダイレクトを追いかける |
| `curl -o out.zip URL` | ファイルに保存（ `-O` で URL のファイル名のまま保存） |
| `curl -X POST -H 'Content-Type: application/json' -d '{"a":1}' URL` | JSON を POST |
| `curl -H 'Authorization: Bearer TOKEN' URL` | ヘッダーを付けて送る |
| `curl -v URL` | 通信の詳細（TLS・送受信ヘッダー）を表示してデバッグ |
| `curl -fsSL URL` | エラー時は失敗扱い・進捗を出さない・リダイレクト追従（スクリプトでの定番） |
| その他 |  |
| `ss -tlnp` | 待ち受け（LISTEN）中の TCP ポートとプロセス（古い `netstat -tlnp` の後継） |
| `ip a` | 自分の IP アドレス・ネットワークインターフェース（古い `ifconfig` の後継） |
| `dig example.com` | DNS の名前解決を確かめる（ `+short` で結果だけ） |
| `ssh user@host` | リモートにログイン（ `-i 鍵` 、 `-p ポート` ） |
| `scp file user@host:/path` | ssh 経由でファイルをコピー（ `rsync -avz` は差分だけ送れる） |
| `wget URL` | ファイルをダウンロード |

| コマンド | やること |
| --- | --- |
| tar（c 作る・x 展開・t 一覧、z gzip・f ファイル指定・v 詳細） |  |
| `tar -czvf out.tar.gz dir/` | ディレクトリを gzip 圧縮でまとめる |
| `tar -xzvf in.tar.gz` | tar.gz を今のディレクトリに展開 |
| `tar -xzf in.tar.gz -C /opt` | 展開先を指定 |
| `tar -tzf in.tar.gz` | 展開せずに中身の一覧を見る |
| `tar -xf in.tar.xz` | GNU tar は展開時に圧縮形式を自動で判定するので `-xf` だけでよい |
| `tar -cJf out.tar.xz dir/` | xz で圧縮（gzip より小さく、遅い） |
| zip / gzip |  |
| `zip -r out.zip dir/` | zip で圧縮（Windows と受け渡すとき） |
| `unzip in.zip -d dir/` | zip を展開（ `-l` で一覧だけ） |
| `gzip file` / `gunzip file.gz` | 単体ファイルを圧縮／展開（元ファイルは置き換わる） |

| コマンド | やること |
| --- | --- |
| systemctl（サービス管理・systemd） |  |
| `systemctl status nginx` | 状態（動いているか・直近のログ）を表示 |
| `sudo systemctl start nginx` | 起動（ `stop` で停止） |
| `sudo systemctl restart nginx` | 再起動 |
| `sudo systemctl reload nginx` | 止めずに設定だけ再読み込み（対応しているサービスのみ） |
| `sudo systemctl enable --now nginx` | OS 起動時に自動で起動する設定＋今すぐ起動（ `disable` で自動起動をやめる） |
| `systemctl is-enabled nginx` | 自動起動が有効か |
| `systemctl list-units --type=service` | 読み込まれているサービスの一覧（ `--failed` で失敗したものだけ） |
| `sudo systemctl daemon-reload` | unit ファイルを書き換えたあとに読み直す |
| `journalctl -u nginx -f` | サービスのログを追いかける（ `-n 100` 直近 100 行、 `--since "1 hour ago"` ） |
| crontab（定期実行） |  |
| `crontab -l` | 自分の定期実行の設定を表示 |
| `crontab -e` | 設定を編集（書式は下の図） |
| `crontab -r` 危険 | 確認なしで設定を全部消す。 `-e` と打ち間違えやすい |
| `sudo crontab -u user -l` | 他のユーザーの設定を見る |
| 日時・タイムゾーン |  |
| `timedatectl` | 現在時刻（ローカル・UTC）・タイムゾーン・NTP 同期の状態をまとめて表示（systemd） |
| `date` | 現在の日時をタイムゾーンの略号付きで表示（例: JST、UTC） |
| `date +%Z%z` | タイムゾーンの略号と UTC からのずれだけを表示（例: `JST+0900` ） |
| `date -u` | UTC で現在時刻を表示 |
| `TZ=Asia/Tokyo date` | 設定を変えずに、指定したタイムゾーンで時刻を表示 |
| `ls -l /etc/localtime` | システムのタイムゾーン設定の実体（リンク先が `/usr/share/zoneinfo/Asia/Tokyo` など） |
| `cat /etc/timezone` | タイムゾーン名を表示（古い Debian / Ubuntu 系のみ。Debian の tzdata 2024b-5 以降は新しく作られないので、 `timedatectl` か `/etc/localtime` のリンク先で確かめる） |
| `timedatectl list-timezones \| grep Tokyo` | 指定できるタイムゾーン名を探す |
| `sudo timedatectl set-timezone Asia/Tokyo` | システムのタイムゾーンを変更する |
| ディスク・メモリ・OS |  |
| `df -h` | ディスクの空き容量（ファイルシステムごと） |
| `du -sh dir` | ディレクトリの合計サイズ |
| `du -h --max-depth=1 \| sort -h` | 直下のどれが大きいかを小さい順に（容量不足の調査） |
| `free -h` | メモリの使用量 |
| `uname -a` / `cat /etc/os-release` | カーネルの版／ディストリビューションと版 |
| `uptime` | 起動からの時間と負荷（load average） |
| 環境変数 |  |
| `env` / `echo $PATH` | 環境変数の一覧／PATH の中身 |
| `export KEY=value` | 環境変数を設定（そのシェルと子プロセスだけ。永続化は `~/.bashrc` などに書く） |
| `source ~/.bashrc` | 設定ファイルを今のシェルに読み込み直す |

（元のページで操作できます → https://cs.taka1156.site/linux-commands.html#cheatsheet）

## つなぐ：パイプとリダイレクト

コマンドの出力は、標準出力（stdout・番号 1）と標準エラー出力（stderr・番号 2）の 2 本に分かれている。 `|` は stdout を次のコマンドの入力へ、 `>` はファイルへ流す。

cat access.log | grep 404 | wc -l

アクセスログから 404 の行を取り出して数える。左の出力が右の入力になる。

| cmd > out.txt | stdout をファイルに書き出す（上書き） |
| --- | --- |
| cmd >> out.txt | stdout をファイルの末尾に追記 |
| cmd 2> err.txt | stderr だけをファイルへ |
| cmd > all.log 2>&1 | stdout と stderr をまとめて同じファイルへ（順番が大事。 `2>&1` は後ろ） |
| cmd > /dev/null 2>&1 | 出力を全部捨てる（cron でよく使う） |
| cmd \| tee out.txt | 画面に出しつつファイルにも保存 |
| cmd1 && cmd2 | cmd1 が成功したときだけ cmd2 を実行（ `\|\|` は失敗したときだけ） |

## chmod の数字の読み方

`ls -l` の `-rwxr-xr-x` は、先頭 1 文字が種類（ `-` ファイル・ `d` ディレクトリ・ `l` リンク）、残り 9 文字が 3 文字ずつ「所有者・グループ・他人」の権限。r=4・w=2・x=1 を足した数字 3 桁で表す。

所有者 (u)

r 4 w 2 x 1

7

グループ (g)

r 4 - 0 x 1

5

他人 (o)

r 4 - 0 x 1

5

例: `rwxr-xr-x` = 755。ディレクトリでは x が「中に入れる（cd できる）」、r が「中身を一覧できる」の意味になる。

## crontab の書式

1 行に「いつ（5 つの欄）」と「何を」を書く。 `*` は毎回、 `*/5` は 5 ごと、 `1-5` は範囲、 `1,15` は列挙。

**0** 分

0-59

**3** 時

0-23

* 日

1-31

* 月

1-12

**1-5** 曜日

0-7 (0,7=日)

**/opt/backup.sh** 実行するコマンド

```
# 平日の 3:00 にバックアップ（出力はログへ）
0 3 * * 1-5 /opt/backup.sh >> /var/log/backup.log 2>&1

# 5 分ごと
*/5 * * * * /usr/bin/curl -fsS https://example.com/health > /dev/null

# 毎月 1 日の 0:00
0 0 1 * * /opt/monthly.sh
```

cron は最小限の環境変数（PATH も短い）で動くので、コマンドはフルパスで書く。 `%` は改行の意味になるので `\%` とエスケープする。

### 時刻はどのタイムゾーン基準か

crontab の時刻はサーバーのシステムのタイムゾーンで解釈される。クラウドの VM やコンテナは UTC のことが多く、そのまま `0 3 * * *` と書くと日本時間の 12:00 に動く。書く前に `timedatectl` か `date` で確かめる。

- タイムゾーンを変えたら、cron を再起動しないと古い設定のまま動くことがある（ `sudo systemctl restart cron` 。RHEL 系は `crond` ）
- cronie（RHEL・Fedora 系の cron）は crontab の先頭に `CRON_TZ=Asia/Tokyo` と書くと、その crontab だけ別のタイムゾーンで解釈できる。Debian / Ubuntu の cron では使えない
- systemd のタイマー（ `OnCalendar=` ）なら、 `OnCalendar=*-*-* 03:00 Asia/Tokyo` のように時刻にタイムゾーンを書ける

## よくある誤解

✗ 誤解: rm で消したファイルはゴミ箱から戻せる

✓ 正しくは: `rm` はゴミ箱を通さずにすぐ削除する。復元ツールで戻せることもあるが期待できない。不安なら `rm -i` を使うか、 `trash-cli` の `trash` コマンドでゴミ箱に送る。

✗ 誤解: kill はプロセスを強制的に止めるコマンドだ

✓ 正しくは: シグナルを指定しない `kill` は SIGTERM（終了してほしいという依頼）を送るだけで、プロセスは後片付けしてから終わる。無視されることもあり、そのときに初めて `kill -9` （SIGKILL）を使う。

✗ 誤解: ターミナルで動いたコマンドは cron でもそのまま動く

✓ 正しくは: cron はログインシェルの設定（ `~/.bashrc` など）を読まず、PATH も短い。コマンドはフルパスで書き、必要な環境変数は crontab の先頭か、スクリプトの中で設定する。

✗ 誤解: systemctl enable するとサービスがすぐ起動する

✓ 正しくは: `enable` は「OS の起動時に自動で起動する」設定だけで、今は起動しない。今すぐ起動もするなら `enable --now` か、別に `start` を実行する。

✗ 誤解: Permission denied が出たら chmod 777 にすればよい

✓ 正しくは: 誰でも書き換えられる状態になり危険。原因は所有者の違いであることが多いので、 `ls -l` で所有者と権限を確かめ、 `chown` や必要な権限だけの付与（ `chmod u+x` など）で直す。

## 出典と確認日

- [Linux man pages online (man7.org)](https://man7.org/linux/man-pages/)
- [GNU Coreutils マニュアル](https://www.gnu.org/software/coreutils/manual/) （ls・cp・mv・rm・chmod・head・tail など）
- [GNU grep マニュアル](https://www.gnu.org/software/grep/manual/) / [GNU findutils](https://www.gnu.org/software/findutils/manual/html_mono/find.html) / [GNU tar](https://www.gnu.org/software/tar/manual/)
- [systemctl (freedesktop.org)](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html)
- [timedatectl (freedesktop.org)](https://www.freedesktop.org/software/systemd/man/latest/timedatectl.html) / [systemd.time（OnCalendar の書式）](https://www.freedesktop.org/software/systemd/man/latest/systemd.time.html)
- [curl man page](https://curl.se/docs/manpage.html)
- [crontab(5)](https://man7.org/linux/man-pages/man5/crontab.5.html) / [lsof(8)](https://man7.org/linux/man-pages/man8/lsof.8.html) / [signal(7)](https://man7.org/linux/man-pages/man7/signal.7.html)

確認日: 2026-11-19
