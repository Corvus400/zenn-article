---
title: "Claude Code の sandbox で Gradle ビルドが必ず落ちる理由と回避方法"
emoji: "🐘"
type: "tech"
topics:
  - "claudecode"
  - "gradle"
  - "kotlin"
  - "sandbox"
published: true
---

# 初めに

フリーランスのAndroid開発者の[とだやま.R](https://github.com/Corvus400)です。普段はモバイルアプリを書いていますが、最近は[Claude Code](https://code.claude.com/docs)のsandboxを有効にして作業しています。sandboxはコマンドのファイル書き込みやネットワーク接続を既定で制限してくれるので、エージェントに任せる範囲を広げても安心感があります。

ところが、Android/Kotlinの開発で必ずぶつかる壁がありました。`./gradlew` を走らせると、ファイルの書き込みとネットワークを許可しているのに、毎回 `Operation not permitted` で落ちるのです。最初は許可リストの設定が足りないのだろうと考えて、`~/.gradle` を書き込み許可に足し、依存先のホストも許可リストに入れました。それでも落ちます。`--offline` を付けても `--no-daemon` を付けても落ちます。

結局、許可リストをどう調整しても直りませんでした。これはGradle側の事情で、設定では回避できない問題だったからです。なぜそうなるのか、どう回避するのかを順に見ていきます。

# 先に結論

要点だけ先に置いておきます。原因はGradleが起動時にloopbackへsocketをbindしようとすることです。sandboxのネットワーク境界はloopbackへのbindを許可していないので、ここで `Operation not permitted` になります。

おもしろいことに、落ちるsocketはGradleのバージョンで変わりました。現行の9.x (手元では9.4.1) では、`FileLockCommunicator` がfile lockの通知用のUDP `DatagramSocket` を開く場面で落ちます。これは[gradle#25762](https://github.com/gradle/gradle/issues/25762) と同じ経路です。少し前の8.x (8.10) では、daemonの通信用のTCP server socketのbindで落ちます。どちらもloopbackへのbindなので、sandboxは同じように弾きます。

このsocketはGradleの起動に必須で、`--no-daemon` でも単発のdaemonが起動するため直りません。loopback内の通信なので `--offline` も効きません。さらに、このsocketを止めるGradle側のフラグもありません。だから設定では回避できず、実務で落ち着くのは、そのコマンドのときだけsandboxを外すやり方です。具体的にはClaude Codeに `dangerouslyDisableSandbox: true` を付けて実行させるか、`! ./gradlew ...` でターミナルから直接実行します。詳しくは後の節で書きます。

# 再現条件と症状

再現は簡単です。sandboxを有効にしたまま、Gradleプロジェクトで `./gradlew help` のような軽いタスクを走らせるだけです。自分の環境では、依存とツールチェインが揃っているのに、毎回これで落ちました。現行の9.4.1で出るのは次のような例外です。

```text
* What went wrong:
Gradle could not start your build.
> Could not create service of type FileLockContentionHandler using BasicGlobalScopeServices.createFileLockContentionHandler().
   > java.net.SocketException: Operation not permitted
...
Caused by: java.net.SocketException: Operation not permitted
    at java.base/sun.nio.ch.Net.bind0(Native Method)
    at java.base/sun.nio.ch.DatagramSocketAdaptor.bind(DatagramSocketAdaptor.java:108)
    at java.base/java.net.DatagramSocket.<init>(DatagramSocket.java:387)
    at org.gradle.cache.internal.locklistener.DefaultFileLockCommunicator.<init>(DefaultFileLockCommunicator.java:45)
```

底を見ると、`DefaultFileLockCommunicator` がUDPの `DatagramSocket` をbindしようとして `Operation not permitted` になっています。依存の取得でもツールチェインのダウンロードでもなく、Gradleが内部の仕組みを初期化する段階です。だから依存を全てキャッシュ済みにしても、`--offline` を付けても症状は変わりません。

ちなみに少し前の8.10で同じことをすると、トップは `Unable to start the daemon process` になります。底は `TcpIncomingConnector` でのTCP server socketのbind失敗でした。落ちるsocketは版で変わりますが、どちらもloopbackへのbindという点は同じです。

# よくある誤解を順に潰す

このエラーは `Operation not permitted` としか出ないので、原因の見当を付けにくいです。自分も最初はsandboxの許可リスト設定を疑いました。ありがちな誤解から順に潰していきます。

## 誤解1: ファイルの書き込み許可が足りない

最初に疑うのはこれだと思います。Gradleは `~/.gradle` 配下にキャッシュやロックファイルを作るので、そこを書き込み許可に入れ忘れているのではないか、と。実際、`~/.gradle` を許可していないと、Gradleのwrapperがファイル書き込みのエラーで落ちることはあります。`gradle-x.x-bin.zip.lck (Operation not permitted)` のような形です。これは[claude-code#19380](https://github.com/anthropics/claude-code/issues/19380)で報告されている現象で、確かにファイルの書き込み許可の話です。

ただ、これは今回の原因とは別物です。自分のsandbox設定では `~/.gradle` を既に書き込み許可に入れています。それでも `./gradlew` は落ちます。落ちる場所もファイル書き込みではなく、先ほどのsocketのbindです。つまりファイル許可の問題は別の入口の話で、socketの問題はそれとは無関係に残ります。

## 誤解2: ネットワークの許可ホストが足りない

次に疑うのはネットワークです。Gradleは依存やツールチェインを取りに行くので、その接続先を許可リストに入れていないのではないか、と。これも筋は通っていますが、今回の原因ではありません。依存とツールチェインを全てキャッシュ済みにして `--offline` で走らせても落ちるからです。#25762にも次のように書かれています。

> Gradle cannot run without being able to listen on a port, even when all project dependencies are available locally.

外向きの接続が一つも要らない状況でも落ちる、ということです。だから許可ホストを足すのは今回の解になりません。

# 真因: 起動時に loopback へ socket を bind する

ではどこで落ちているのか。Gradleは起動時にいくつかのloopback socketをbindします。一つはdaemonとクライアントの通信用のTCP server socketです。もう一つは、file lockの競合を他プロセスへ知らせるために `FileLockCommunicator` が使うUDPの `DatagramSocket` です。手元で実測すると、現行の9.4.1では後者のUDP socketで、8.10では前者のTCP socketで落ちました。どちらが先に出るかは版で変わりますが、いずれもloopbackへのbindです。

大事なのは、これがdaemonを切っても起きることです。Gradleは `--no-daemon` を付けても単発のdaemonプロセスを起動するので、結局これらのsocketを作ろうとして落ちます。そしてloopback内の通信であって外のサーバーへ出ていくものではないので、`--offline` でネットワークを切っても関係がありません。

流れにすると、`Gradle起動` → `loopback に socket を bind` → `sandbox のネットワーク境界が拒否` → `Operation not permitted` です。sandboxのネットワーク制限は許可リスト方式で、許可リストにloopback (127.0.0.1) を入れない限りloopbackへのbindは通りません。実際、自分のsandbox設定の許可ホストにはHTTPSの取得先しか並んでおらず、loopbackは入っていません。ここで弾かれます。

[gradle#25762](https://github.com/gradle/gradle/issues/25762)は、まさにこの `FileLockCommunicator` のUDP `DatagramSocket` がsandboxで開けない件を報告しています。`--offline` でも直らないことも、このissueに書かれています。なお、後の節で出てくるUnix domain socket (AF_UNIX) とは別物なので、そこは区別します。

# 3つの失敗モードを切り分ける

`Operation not permitted` は色々な原因で出るので混同しやすいです。Gradleとsandboxの組み合わせで遭遇しうる失敗を、socketの種類で3つに切り分けておきます。混同すると、効かない対策に時間を使ってしまいます。

| 失敗モード                              | 何が起きるか                                                                                              | socketの種類           | 解決の方向                          |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------- | ---------------------- | ----------------------------------- |
| (A) 外向きの接続が弾かれる              | 依存取得・ツールチェイン自動DL・build scanが許可外ホストへ出ようとして失敗                                | INETの外向き           | 許可ホストを足す / 取得自体を止める |
| (B) loopbackへのbindが弾かれる (本記事) | 起動時にdaemonがTCP server socketを (または `FileLockCommunicator` がUDP socketを) loopbackにbindして失敗 | INET loopbackのTCP/UDP | そのコマンドだけsandboxを外す       |
| (C) Unix domain socketが弾かれる        | `listen EPERM` などでAF_UNIXのbindやlistenが通らない                                                      | AF_UNIX                | Gradleとは別問題 (後述)             |

(A)は素直なネットワークの話です。例えばツールチェインの自動ダウンロードが許可外のホストへ出て落ちるなら、許可ホストを足すか、自動ダウンロードそのものを止めれば直ります。後者は `org.gradle.java.installations.auto-download=false` を指定します ([Gradle Toolchains](https://docs.gradle.org/current/userguide/toolchains.html))。これは設定で直せる種類の失敗です。

(B)が本記事の主役です。これは設定では直せません。前の節のとおり、loopbackへのsocket bindをsandboxが許可しないからです。実測では9.4.1がUDPのfile lock socket、8.10がTCPのdaemon socketで落ちました (#25762 は前者を報告)。

(C)は別物として置いておきます。macOSの[claude-code#41254](https://github.com/anthropics/claude-code/issues/41254)やLinuxの[claude-code#44180](https://github.com/anthropics/claude-code/issues/44180)で報告されている現象です。Unix domain socket (AF_UNIX) のbindやlistenが通らない、という話です。`tsx` のIPCやPlaywrightなどで起きます。ここで紛らわしいのは、sandboxに `allowUnixSockets` や `allowAllUnixSockets` のような設定があることです。これらは(C)のAF_UNIX向けであって、(B)のINET loopbackには効きません。socketの種類が違うので、(C)向けの設定で(B)は直らないのです。

# 回避策

ここからは実務での回避です。(B)が相手なら、結論はもう出ています。そのコマンドのときだけsandboxを外すことです。

一番手軽なのは、Claude CodeのBashツールに `dangerouslyDisableSandbox: true` を付けて実行することです。これはsandboxが原因でコマンドが失敗したときの逃げ道として用意されている、コマンド単位の指定です ([sandboxing](https://code.claude.com/docs/en/sandboxing))。Gradleのビルドだけこれを付ければ、他のコマンドはsandboxの内側のまま守られます。実際に同じ `gradle help` を `dangerouslyDisableSandbox: true` で走らせてみました。すると、socketのエラーは消えて `BUILD SUCCESSFUL` になりました。止めていたのはsandboxだけだった、という確認です。

注意したいのは、これはBashツールのパラメータで、Monitorツールにはこのパラメータが無いことです。なので、Claude CodeにMonitorツールでGradleのビルドを進捗監視させようとすると、sandboxを外せずに落ちます。Claude Codeに長時間のビルドやログを追わせたいときは、`! ./gradlew ...` でターミナルから直接実行する形にしています。`!` はClaude Codeのshellモードのプレフィックスで、Claudeの解釈や承認を挟まずにそのままshellで実行し、出力を会話へ取り込めます ([interactive-mode](https://code.claude.com/docs/en/interactive-mode))。

毎回 `dangerouslyDisableSandbox` を付けるのが面倒なら、`excludedCommands` に `gradle` や `gradlew` を入れる手もあります。これを使うと、常にsandboxの外で実行できます ([settings](https://code.claude.com/docs/en/settings))。なお、この設定が効かないという報告 ([claude-code#10524](https://github.com/anthropics/claude-code/issues/10524)) もあるので、入れたら一度実際に効いているか確かめると安心です。

(A)の外向きだけが原因なら、sandboxを外す必要はありません。許可ホストを足すか、`org.gradle.java.installations.auto-download=false` のように自動取得を止めれば済みます。(A)と(B)の両方が絡む場合は、(B)が残るのでsandboxを外す方へ倒します。

sandboxを丸ごと外すのが嫌なら、`allowLocalBinding` を試す価値があります。これはloopbackのポートへのbindを許可する設定です ([settings](https://code.claude.com/docs/en/settings))。今回の真因はまさにloopbackへのsocket bindなので、sandboxを保ったまま本件を直せる可能性があります。ただし自分はまだ実測していないので、効くかどうかは手元で確かめてからにしてください。

> `dangerouslyDisableSandbox` はその名のとおりsandboxを外す指定です。全コマンドに常用するとsandboxの意味が薄れるので、Gradleのように原因が分かっているコマンドだけに絞るのがおすすめです。

# (任意) hook で運用に固定する

ここまでの回避を毎回手で思い出すのは面倒です。自分はClaude CodeのPreToolUse hookを入れています。Claude Codeが `gradle` や `gradlew` をMonitorツールで実行しようとしたら止めて、Bash + `dangerouslyDisableSandbox` へ誘導します。本質は前の節の回避策で、hookは再発防止のおまけです。

中身は20行ほどです。Monitorツールのコマンドを見て、`gradle` か `gradlew` を含むなら `permissionDecision: "deny"` を返します。理由として「Bashツールに `dangerouslyDisableSandbox: true` を付けて」と案内します。

```bash
#!/bin/bash
# PreToolUse hook (matcher: "Monitor")。gradle 実行を止めて Bash + dangerouslyDisableSandbox へ誘導する
set -uo pipefail

PAYLOAD="$(cat)"
COMMAND=$(printf '%s' "$PAYLOAD" | jq -r '.tool_input.command // empty')

# Monitor で gradle/gradlew を回そうとしたときだけ介入する
if ! echo "$COMMAND" | grep -qE '(gradlew|gradle)\s'; then
    exit 0
fi

MSG="Gradle は sandbox の loopback socket 制限 (起動時の loopback への socket bind が EPERM) で失敗します。Monitor には dangerouslyDisableSandbox パラメータが無いため、Bash ツールに dangerouslyDisableSandbox: true を付けて実行してください。長時間ビルドはユーザーに ! でターミナル直実行を依頼する手もあります。"
jq -n --arg msg "$MSG" '{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": $msg
  }
}'
```

PreToolUseのhookは、`permissionDecision: "deny"` と `permissionDecisionReason` を返すと、その理由がClaudeに渡ります。なのでhookで止めるだけでなく、「ではBashで `dangerouslyDisableSandbox` を付けよう」とClaude自身が次の手に進めます。この出力の決まりは[公式ドキュメント](https://code.claude.com/docs/en/hooks)で確認できます。

# 最後に

`Operation not permitted` という素っ気ないエラーの正体は、Gradleが起動時にloopbackへsocketをbindする、という地味な仕組みでした。外向きの通信でもファイルの書き込みでもありません。だから許可リストをいくら足しても直りません。実務では、Gradleのコマンドのときだけsandboxを外すのが今のところの落とし所です。

念のため書いておくと、Gradleがloopbackのsocketを使うこと自体は前からある挙動です。UDP側が開けない件は[gradle#25762](https://github.com/gradle/gradle/issues/25762)に記録されています。この記事で足したのは、それをClaude Codeのsandboxという文脈に置き直した部分です。実測で落ちるsocketが版で変わる (9.x はUDP / 8.x はTCP) ことを確かめました。さらに、ファイル許可の話 (#19380) やUnix domain socketの話 (#41254 / #44180) と切り分けました。

同じところで詰まっている方の手がかりになれば嬉しいです。もっと良い回避策をご存知の方がいたら、コメントで教えてもらえると助かります。
