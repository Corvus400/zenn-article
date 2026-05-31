---
title: "Claude Code と Codex の両方に、機密情報と個人情報を漏らさせない hook を作った話"
emoji: "🔒"
type: "tech"
topics:
  - "claudecode"
  - "codex"
  - "gitleaks"
  - "hooks"
  - "security"
published: true
---

# 初めに

フリーランスのAndroid開発者の[とだやま.R](https://github.com/Corvus400)です。最近は[Claude Code](https://code.claude.com/docs)と[Codex](https://developers.openai.com/codex/cli)を両方使っています。

きっかけは少し恥ずかしい失敗でした。公開予定の個人リポジトリ2本で、コミット履歴に機密情報と個人情報が混ざっていたのです。公開前に気付き[git-filter-repo](https://github.com/newren/git-filter-repo)で消しましたが、後から消すのは完全な対策になりません(理由は最後に触れます)。それなら入口で止めようと考えたのがこの仕組みです。

> 1本の共有 hook で両方(と手動)のコミットをまとめて止めつつ、各エージェントの hook で gitleaks が見ない個人情報も塞ぐ。

「gitleaks を pre-commit に入れましょう」という基本導入の記事ではなく、複数のAIエージェントをまたいで漏洩を止める設計の話です。

# この仕組みが向いている人・要らない人

判定の軸は一つです。AIエージェントに外向きの成果物(コミット / PR・Issue の本文 / 公開リポジトリへのファイル)を作らせているか。

向いている人は次のとおりです。

- Claude Code や Codex のような複数のAIエージェントを併用し、コミットや PR をある程度任せている。
- git と GitHub で開発していて、手元に個人情報(実名の絶対パス、クラウドドライブ名、個人メールアドレス)が散らばっている。

要らない人は次のとおりです。

- 使っているエージェントが1つだけ。hook を1セット用意すれば済み、「2つ分」の工夫は要りません。
- 機密情報の管理を組織側のスキャンや CI で完結させ、ローカルには何も持たない。
- AIに push 権限を渡していない、または公開リポジトリを扱わない。ただし後から公開する可能性があるなら、履歴に紛れた機密情報・個人情報の git-filter-repo 後始末が要るので対象になり得ます。

万能ではありません。「複数のAIに外向きの作業を任せていて、ローカルに出したくないものがある」人向けの、わりと狭い話です。

# 全体設計

難しいことはしていません。層は2つで、どのツールからでも必ず通る共有の git 関門(L1)と、各エージェント独自の hook(L2)です。経路も2つで、コミットに混じる機密情報(a)と、gitleaks が見ない個人情報(b、PR・Issue の本文やファイル編集)です。

| 層                           | 実体                                                                       | Claude Code | Codex | 性質                                      |
| ---------------------------- | -------------------------------------------------------------------------- | :---------: | :---: | ----------------------------------------- |
| L1: git 関門(機密情報)       | pre-commit の gitleaks(`core.hooksPath` をマシン全体に)                    |     ✅      |  ✅   | ツールに依存しない・共有(1本で両方を守る) |
| L2: すり抜けの阻止(機密情報) | Claude=`cc-guard-gitleaks` / Codex=hooksPath の保護 + リポジトリ側チェック |     ✅      |   △   | エージェント別。形が違う(後述)            |
| L2: 個人情報(gitleaks の外)  | Claude=`cc-block-local-info-in-issue-pr` / Codex=`local_info` 検出         |     ✅      |  ✅   | エージェント別。こちらは強く揃っている    |

設計の肝は、「1実装で両方を守れる層」と「各エージェントに複製する層」を分けたことです。機密情報の入口であるコミットは全エージェント共通なので1本で済みますが、PR の本文やファイル編集はエージェントごとに通り道が違うので、それぞれの hook に置くしかありません。

# 全コミットに gitleaks を1本で効かせる

`core.hooksPath` をマシン全体(`git config --global`)に設定します。すると、このマシンの git コミットは Claude Code・Codex・手動のいずれでも同じ pre-commit を踏みます。その pre-commit が gitleaks を呼んでステージ差分をスキャンし、機密情報を見つけたら `exit 1` で止めます。

```bash
# 配線: このマシンの全コミットを1つの pre-commit に集約する
git config --global core.hooksPath "$HOME/dotfiles/git-hooks"

# pre-commit の肝: ステージ差分を gitleaks でスキャンし、見つけたら止める
gitleaks git --pre-commit --staged --redact --verbose .
```

専用の `gitleaks.toml` は置かず、内蔵ルールにそのまま乗っています。例外は、機密情報ではない中身が API 鍵の形を模した文字列を含み、gitleaks が誤検知する場面です。そのときだけ `git -c core.hooksPath= commit` でそのコマンドに限って関門を外します(全部を止める `--no-verify` とは違い、1コマンドだけです)。

導入方法はもう語り尽くされているので省きます。言いたいのは、マシン全体に設定しておけば後からエージェントが1つ増えても勝手に守られる、という一点です。

# AIが検査をすり抜けるのを止める(機密情報)

L1 があっても穴は残ります。「とりあえずコミットして」と言われたAIが、`git commit --no-verify` で L1 ごと飛ばすことがあるからです。特に Sonnet や低い Reasoning Level だと、中身を確認せず雑なコミットをしがちです。これは思い込みではなく、[`--no-verify` を使わせたいという要望が issue にも上がっている](https://github.com/anthropics/claude-code/issues/40117)ほど現実的な動きです。AI自身が脅威になります。ここの守り方は Claude Code と Codex で形が違います。

Claude Code では、コミット前に走る `cc-guard-gitleaks` が gitleaks をもう一度走らせます。さらに `--no-verify` や `-n`、くっつけた短縮形(`-nm` のように `n` を紛れ込ませた形)まで正規表現で見て止めます。単純な文字列一致だと `-nm` を取りこぼすので、正規表現で塞いでいます。実体はこの数行です。

```bash
# git commit を含むコマンドだけを対象に、--no-verify と 短縮形の -n を検出して止める
# 「-nm」のように -n が他のフラグと結合した形も拾う
if echo "$COMMAND" | grep -qE '(^|\s)--no-verify(\s|=|$)|\s-[a-zA-Z]*n[a-zA-Z]*(\s|$)'; then
  echo '[blocked] git commit に --no-verify / -n(短縮形を含む) は使用禁止です。' >&2
  exit 2
fi
```

Codex は `--no-verify` を名指しで止める仕組みを持たず、別の角度で守ります。コミットや push を見つけたとき、リポジトリ側が `verify-public-hygiene.sh` のようなチェックを置いていれば走らせ、失敗したら止めます(リポジトリ側が選んで入れる方式)。加えて `lefthook install --reset-hooks-path` のように共有している `core.hooksPath` を書き換えるコマンド自体を止め、土台を引き剥がされないようにします。コミット前に注意を促すメッセージも出します。

| 観点                                 | Claude Code |    Codex     |
| ------------------------------------ | :---------: | :----------: |
| `--no-verify`/`-nm` を名指しで止める |     ✅      | ❌(持たない) |
| コミット前の再スキャン               |     ✅      |      —       |
| 共有 hooksPath の書き換えを阻止      |      —      |      ✅      |
| リポジトリ側チェックへの委譲         |      —      |   ✅(任意)   |

要するに、Claude 側はすり抜けに使うオプションを正規化して止め、Codex 側は土台を守りつつリポジトリのチェックに委ねます。同じ L1 の補強でも、複製の仕方は揃っていません。

# gitleaks がカバーしない流出経路を塞ぐ

ここがいちばん効いている実感があります。gitleaks は機密情報を守ってくれますが、次の2つは見ません。

- 機密情報ではない個人情報。絶対パス(`/Users/<実名>/`)や `~/Desktop` のようなホーム配下、Windows のパス。クラウドドライブ名(`マイドライブ` / `My Drive` / `iCloud Drive`)も対象です。実名やメールアドレス、`/private/var/folders/` のような macOS のシステムパスも含みます。
- git の外の経路。PR や Issue の本文です。コミットを通らないので gitleaks のスキャンにかかりません。

しかもAIは作業メモを下敷きに本文を書くので、ここが盲点になります。だから各エージェントの hook で塞ぎました。

Claude Code では、`cc-block-local-info-in-issue-pr` という hook を使います。これは `gh` コマンドや `mcp__github__*` 系のツールの本文から個人情報のパターンを見つけると、AIに「書き直して」と拒否を返します。見ているのは値そのものではなく「形」です。検出パターンはこれだけです。

```bash
# 検出パターン (名前|正規表現)。個人情報の実値ではなく「形」を並べている
PATTERNS=(
  'email|[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}'
  'macos-home-path|/Users/[A-Za-z0-9_.-]+/'
  'tilde-path|~/(Desktop|Documents|Downloads|Library)/'
  'windows-path|[A-Z]:\\Users\\'
  'drive-name|マイドライブ|My Drive|iCloud Drive'
  'apple-system-path|/private/var/folders/|/Library/Developer/'
)

# 自動生成・bot のメールだけは許可リストで除外する
ALLOWLIST_EMAIL='noreply@anthropic\.com|noreply@github\.com|.*\[bot\]@users\.noreply\.github\.com'
```

個人のメールアドレスは、先頭の汎用メール正規表現1本で拾います。特定のアドレスを名指しで書いてはいません。許可リストで除外しているのは bot や noreply のアドレスだけなので、自分の `xxx@gmail.com` のような個人メールアドレスはそのまま引っかかります。拒否のときは `マイドライブ` を `共有資料` に、のような置き換え候補まで添えます。

Codex では、`codex-hook-dispatch` の中に `local_info` という同じ考え方の検出があります。該当部分の抜粋はこれだけです。

```bash
# Codex 側(codex-hook-dispatch の local_info)。Claude とほぼ同じパターンに UUID を足したもの
local patterns=(
  'email|[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}'
  'macos-home-path|/'"Users"'/[A-Za-z0-9_.-]+/'
  'tilde-path|~/(Desktop|Documents|Downloads|Library)/'
  'drive-name|マイ''ドライブ|My'' Drive|iCloud Drive'
  'uuid-or-device-id|[0-9A-Fa-f]{8}-[0-9A-Fa-f]{4}-[0-9A-Fa-f]{4}-[0-9A-Fa-f]{4}-[0-9A-Fa-f]{12}'
)
```

当たる範囲は Codex の方が広く、PR 本文だけでなくファイル編集(`apply_patch` / `Edit` / `Write`)に混じった個人情報まで止めます。ただし検出パターンを書いたファイル自体の編集は除外します。上の `/'"Users"'/` や `マイ''ドライブ` の妙な区切りは、`Users` や `マイドライブ` のリテラルを文字列連結で割って、検出器が自分のルールで自分を弾かないようにする仕掛けです。

両方とも、もう一段の逃げ道も塞いでいます。`--body-file`・`-F`・`--body @ファイル`・`$(cat ...)` のようにファイル経由やコマンド展開で本文を渡すと、hook は中身を覗けません。だから本文を検査する前に、この渡し方そのものを無条件で弾きます。Claude 側と Codex 側のどちらも、別々の実装で同じ穴を塞いでいます。

```bash
# 本文が hook に渡らない経路(ファイル参照・コマンド置換)を、本文検査より前に拒否する
ESCAPE_HATCH_REGEX='(--body-file|(^|[[:space:]])-F[[:space:]]|--body[[:space:]]+@|--body[[:space:]]+-([[:space:]]|$)|[$]\([[:space:]]*cat[[:space:]]|[$]\([[:space:]]*<|<\([[:space:]]*cat[[:space:]])'
```

個人情報の経路に関しては、機密情報のすり抜け対策と違って、両エージェントでかなり揃っています。同じパターンを別々の hook の仕組みで実装した形です。なお、上のとおり検出に使っているのは正規表現のパターンであって個人情報そのものではないので、`マイドライブ|My Drive|iCloud Drive` のような行をこの記事に載せても問題はありません。

# 実際に効いているか ― セッション記録で確かめる

仕組みの説明だけで「効いています」と言っても信用ならないので、ドキュメントの読み解きではなく、実際に hook が発火したセッション記録で確かめます。

まず機密情報の方です。Codex のセッション記録には、コミットのたびに gitleaks を走らせた跡が数百件残っていました。大半は素通りでしたが、中には実際に機密情報を検知してコミットを止めたものもあります。止めたときのログはこうです(パスと SHA は伏せています)。

```text
gitleaks  WRN leaks found: 1
[pre-commit] gitleaks が秘匿情報を検知しました。commit を阻止します。
```

これが「Codex のコミットでも、共有している gitleaks が止めている」ことの直接の証拠です。ツールに関係なく効く、という設計どおりの動きが、Claude ではなく Codex 側のログに出ている点が大事だと思っています。

次に個人情報の方です。`gh pr create` や `mcp__github__create_pull_request` の本文に `/Users/<ユーザー名>/` が混じったとします。このとき Codex の `local_info` が、実際に拒否を返していました。

```text
permissionDecisionReason:
  "Refusing local paths, email addresses, or personal environment details.
   Detected patterns:
     - macos-home-path: /Users/<ユーザー名>/"
```

これが「個人情報の混入もちゃんと止まる」直接の証拠です。一点だけ区別しておきます。本文をファイル経由で渡そうとすると出る `Refusing GitHub body input that hooks cannot inspect ...` は、上の個人情報の検出とは別の仕組みです。「中身を覗けない渡し方」を拒否しているだけで、個人情報を検出したわけではありません。証拠としてこの2つは混ぜないようにしています。

効いていない・揃っていないところは以下の通りです。

- Codex は `--no-verify` を名指しで止める仕組みを持っていません(L1 + リポジトリ側チェック + 注意メッセージに頼っています)。
- リポジトリ側チェック(`verify-public-hygiene.sh`)は、リポジトリがそれを置いているときだけ動きます。
- 専用の `gitleaks.toml` が置かれていない場合、検知は内蔵ルール頼みです。
- Codex の hook の仕様は公式ドキュメントが薄く、動きはセッションの実際の発火で裏どりしています。「公式が保証している」ではなく「実際にこう動いていた」という確認の仕方です(Claude 側の hook の出力の決まりは[公式ドキュメント](https://code.claude.com/docs/en/hooks)で確認できます)。

| 守れているか | 機密情報(コミット) | 個人情報(PR/編集) | `--no-verify` を名指しで止める | リポジトリ側チェック |
| ------------ | :----------------: | :---------------: | :----------------------------: | :------------------: |
| Claude Code  |         ✅         |        ✅         |               ✅               |          —           |
| Codex        |         ✅         |   ✅(編集まで)    |               ❌               |       △(任意)        |

# 自分はこう置いている

核になる部分は、この記事にそのまま載せています。どれも `$HOME` 相対のパスと正規表現だけで、実際の個人情報は入っていないからです。土台の pre-commit は、先ほどの gitleaks を呼んで見つかったら止めるだけの数行です。

```bash
# $HOME/dotfiles/git-hooks/pre-commit
gitleaks git --pre-commit --staged --redact --verbose . || {
  echo "[pre-commit] gitleaks が秘匿情報を検知しました。commit を阻止します。" >&2
  exit 1
}
```

これを `git config --global core.hooksPath "$HOME/dotfiles/git-hooks"` で全コミットに効かせます。各エージェントの hook は別々に登録します。Claude 側は `settings.json` の hooks に、Codex 側は `~/.codex/hooks.json` に、それぞれ動かす条件(`matcher`)を指定して並べます。本体は前の節で引用したもので、`cc-guard-gitleaks` の `--no-verify` 検出と `cc-block-local-info-in-issue-pr` の検出パターンです。どちらも手元では `$HOME` 相対に直すだけで動きます。唯一そのまま貼れないのは Codex 側の `codex-hook-dispatch` で、100KB超の1ファイルに無関係な hook が同居しているからです(機密情報を含むからではありません)。個人情報検出の該当部分だけ上で抜き出しました。

# 似たことをしている人との違い

同じ方向の取り組みは他にもあります。違いを一言でいうと、2つのAIエージェントをまたいで、1本の共有関門 + 各エージェントに複製した hook で守っているところです。

[takna さんの4層防御](https://zenn.dev/takna/articles/secret-leak-prevention-4-layer)は「git はすべての入口を一本化する関門で、コミットは必ずそこを通る」という考え方です。この記事はそこに正面から反例を出しています。PR・Issue の本文のような git を通らない経路、機密情報ではない個人情報、ファイル編集、別のエージェントは git の関門だけでは拾えません。

[sensitive-canary](https://github.com/coo-quack/sensitive-canary)は向きが逆で、AIから外部サービスへ送られる方を止めています。この記事は逆に、開発者自身の個人情報が公開成果物へ出ていく方を止めています。「外から悪意ある指示を紛れ込ませる」攻撃が入ってくる方の話なのに対し、こちらは出ていく方の話です。

[dely_jp さんの3層防御](https://zenn.dev/dely_jp/articles/claude-code-3-layer-defense-git-github)は機密情報や危険な git 操作が対象で、単一エージェントの話です。

攻撃側からは[Flatt さんの "Pwning Claude Code"](https://flatt.tech/research/posts/pwning-claude-code-in-8-different-ways/)が、上で塞いだような逃げ道を攻撃手法として整理しています。あちらが攻撃側の地図なら、この記事はそこで挙げられた逃げ道を実際に塞ぐ hook の方です。読み合わせると噛み合います。

# 最後に

作る前は「git の関門さえ固めれば漏れない」くらいに思っていました。やってみると、入口はそんなに一本化されていません。機密情報はコミットという共通の入口があるので1本で守れますが、個人情報は PR の本文やファイル編集という、エージェントごとに違う入口から出ていきます。だから「1本の共有 git 関門 + 各エージェントに複製した hook」という形に落ち着きました。ただし2つの守りは同じ形ではなく、個人情報の方はかなり揃った一方、機密情報のすり抜け対策は各エージェントの hook に合わせてバラバラです。

ここに手をかけた理由は最初の話に戻ります。一度漏らすと、`git-filter-repo` での後始末は履歴の全書き換えになります。しかも一度 public に push したり clone・fork されたりすると、古いコミットは GitHub のキャッシュや fork 側に残り得ます。[GitHub の公式手順](https://docs.github.com/ja/enterprise-cloud@latest/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)も「露出した認証情報は漏れたものとして扱い、必ず作り直せ」と念を押しています。機密情報の再発行まで要ると考えれば、入口で1回止める事前予防の方がずっと安く済みます。この記事は、その予防側の設計の記録のつもりで書きました。

今分かっている問題も並べておきます。

- Codex 側の `codex-hook-dispatch` から個人情報検出の部分を切り出して、単体で配れる形にできない(今は100KB超の1ファイルに同居しています)。
- Codex 側に `--no-verify` を名指しで止める仕組みがないこと。
- リポジトリ側チェックが任意であること。
- Codex の hook の公式仕様が薄く、実際の動きで裏どりするしかないこと。

同じように、複数のAIエージェントに外向きの作業を任せていて、ローカルのものを漏らしたくない人には、この設計が何かの手がかりになるかもしれません。もっと良いやり方をご存知の方がいたら、記事のコメントで教えてもらえると嬉しいです。
