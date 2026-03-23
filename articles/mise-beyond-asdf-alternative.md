---
title: "miseはasdf alternativeからどこまで進化したのか"
emoji: "🧐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mise","asdf","direnv","環境構築"]
published: true
published_at: 2026-03-24 08:00
---

:::message
この記事はLLMと壁打ちしながら作成しました。
<!-- textlint-disable-next-line ja-technical-writing/ja-no-mixed-period -->
:::

## 最初に

以前、miseの紹介記事を書きました。[^1]

@[card](https://zenn.dev/yutaro1985/articles/introduction_of_mise)

当時の自分は、miseを「asdf alternativeであり、envとtasksも持つツール」と説明していました。
その整理自体は今も間違っていません。

ただ、2026年時点のドキュメントやREADMEを読むと、いまのmiseはもう少し違う見え方をするようになりました。
単なる寄せ集めではなく、開発環境の準備そのものを定義する道具として育ってきました。

この記事は、その更新版です。
2023年ごろにasdf alternativeとしてmiseを触った人向けに、いまの理解の軸を整理します。

ツールのバージョン管理や、基本的なenv、tasksの使い方は前回記事で触れています。
今回はその先にある話へ進みます。

## mise の思想は「下ごしらえ」をそろえること

`mise-en-place` は、料理でいう「下ごしらえ」を指す言葉です。
READMEでも `The front-end to your dev env` と表現されています。

この名前を前提にすると、miseの機能がかなり自然に見えてきます。
やっていることは、開発に入る前の準備をそろえることです。

- どのツールで動かすか
- どんな状態で動かすか
- どの操作を入口にするか

この3つをバラバラに置かず、1つの設定にまとめる。
それがいまのmiseの中心だととらえると、だいぶ整理しやすくなります。

## いまの mise を理解するためのフレーム

ここから先は、公式の用語定義をそのままなぞるというより、この記事なりの整理で話を進めます。
そのほうが、2023年ごろにmiseをasdf alternativeとして触った人には、今の広がりがつかみやすいと感じるからです。

この記事では、miseを次の対応でとらえます。

| 要素 | 役割 |
| --- | --- |
| `tools` | 実行環境 |
| `env` | 状態 |
| `tasks` | 操作 |

この見方をすると、`mise.toml` は単なるバージョン管理ファイルではありません。
プロジェクトを動かす前提条件を書く「仕様書」のような存在になります。

`tools` だけを見るとasdf alternativeです。
`env` まで含めるとdirenv的な役割も入ります。
`tasks` まで含めると、開発者が触る入口まで定義できます。

ここは、前回記事を書いたころより変化を感じるところでした。

## tools は「実行環境」を定義する層になった

### asdf 互換は残っている

まず大前提として、miseは今もasdf alternativeでなくなったわけではありません。
`.tool-versions` 互換もありますし、asdfプラグイン資産も一応まだ使えます。

ただし、この記事を書いた2026/3/24現在、asdfのpluginはドキュメント上にLegacyと明記されています。
@[card](https://mise.jdx.dev/asdf-legacy-plugins.html#what-are-asdf-legacy-plugins)
新規に整理するならmiseの現在のやり方に従って `mise.toml` を中心にしたほうが分かりやすいです。
今はasdfのalternativeから進化した状態と自分は解釈しています。

### backend が広がり、「どこから入れるか」まで書ける

最近のmiseで印象的なのは、ツールの取得元をbackendとして扱えることです。
言語ランタイムだけでなく、各エコシステムの配布物もそのまま扱えます。

次の例は、その感覚が分かりやすいです。

```toml
[tools]
node = "22"
pnpm = "10"
"npm:prettier" = "3"
"cargo:watchexec-cli" = "2"
```

ここで書いているのは、単なる「バージョン」ではありません。
Node.jsは通常のruntimeとして入れつつ、Prettierはnpmから、watchexecはcargoから入れると宣言しています。
miseはバックエンドを指定できます。
@[card](https://mise.jdx.dev/dev-tools/backends/#backends)
それぞれのツールを、どこからインストールするかまで含めて定義しています。
最近よく見るのはnpmやpipx系のbackendです。
npm backendのデフォルトは `npm` です。
`pnpm` や `bun` を使いたい場合は、`npm.package_manager` などで明示的に設定します。
一方でpipx backendは、公式ドキュメントにもある通り `uv` が入っていればデフォルトで `uvx` を使います。
@[card](https://mise.jdx.dev/dev-tools/backends/npm.html)
@[card](https://mise.jdx.dev/dev-tools/backends/pipx.html)

### shim は非対話環境とのつなぎ役

以前の説明では、ShellにactivateしてPATHを切り替える話が中心でした。
これは今も基本で、対話シェルで使うならまず `mise activate` によるmiseのPATH activationを前提に考えるのが自然です。
`mise activate` について補足すると、各Shellでmiseを有効化するための初期セットアップのようなものです。
インストール手順の中で、Shell起動時にactivateを実行する設定を各Shellの設定ファイルへ記述します。
@[card](https://mise.jdx.dev/installing-mise.html#shells)

一方で、毎回Shell全体へ反映したいわけではない場面もあります。
単発のコマンド実行なら `mise exec`、`mise.toml` に書いたtask名でコマンドを呼びたいなら `mise run` でも十分に扱えます。

そのうえで、いまのドキュメントではshimsの位置付けもだいぶ明確です。
IDEやスクリプト、あるいは通常のPATH activationを前提にしづらい環境で使う選択肢として見ると分かりやすいです。

shimsはmiseが今参照しているものの実体を参照しているシンボリックリンクのようなものです。
たとえば `~/.local/share/mise/shims/node` のようなファイルが作られ、`node` が呼ばれたときにまずそれが実行されます。
shell上でコマンドを実行する時にあまり意識することはないですが、shellじゃない環境でmiseが管理しているツールを指定したい場合などは便利です。

ここで意識したいのは、envの反映範囲の違いです。
通常の `mise activate` では、ディレクトリ移動に応じてshell全体のPATHやenvが更新されます。
一方で `shims` は、shim経由で起動したプロセスの中でmiseの文脈を読む形です。
shell全体の状態としてenvを扱いたいなら、通常のPATH activationを基本にしつつ、場面によって `mise exec` や `mise run` を使い分けるほうが自然です。

自分の理解では、次のように考えるとイメージしやすいです。

- 普段のターミナル作業なら通常のPATH activation
- 単発で1コマンドだけ実行したいなら `mise exec`
- `mise.toml` に書いたtask名でコマンドを呼びたいなら `mise run`
- IDEやスクリプトのように、shellの初期化を前提にしづらいなら `shims`

たとえばローカルで開発していて、`cd` した瞬間に `node` や `pnpm` が切り替わってほしい場面があります。
さらに `AWS_PROFILE` のようなenvまでまとめて反映したいなら、これは通常のPATH activationが一番自然です。

```bash
cd myapp
echo $AWS_PROFILE
pnpm dev
```

一方で、「普段のshell設定には触れたくないが、このコマンドだけはmiseの環境で動かしたい」という場面なら `mise exec` が合っています。

```bash
mise exec -- pnpm test
mise exec -- bash -c 'echo $AWS_PROFILE'
```

`mise run` が分かりやすいのは、長いコマンド列や前提付きの処理をtask名で呼べるようにしたい場面です。
個人的には、開発サーバ起動やdeployのように、チームで同じ名前にそろえたい処理はこの形が分かりやすいです。

```bash
mise run dev
mise run deploy
```

そしてshimsを使う場面として分かりやすいのは、たとえばIDEが `node` や `prettier` を直接呼ぶケースです。
この場合、IDEなどはshellを経由しないでツールを呼び出すため、`activate`で有効化されたPATHを参照できません。
そういうときにshim経由で実行できるようにしておくと、IDEやスクリプト側からは普通の `node` や `prettier` に見えつつ、裏ではmiseが実際の実行ファイルを選んで起動できます。

ただし、ここでも `env` の見え方には差があります。
たとえば通常のPATH activationなら、`echo $AWS_PROFILE` で値をそのまま確認できます。
一方でshim経由の実行は、shimが起動したプロセス側でmiseの文脈を読む形です。
そのため、同じ値がshell全体にそのまま出ているとは限りません。

### lock は「たまたま今入ったもの」を減らす

`tools` に何を書くかだけでなく、実際にどの配布物を取るかもある程度そろえたい、という場面があります。
そのときに関わってくるのが `mise lock` です。

現行のhelpやドキュメントでは、`mise lock` はlockfileに記録されるURLやchecksumを更新するためのしくみとして説明されています。
そのため、「`mise.toml` だけ置いた新しいprojectで最初の `mise.lock` を作るコマンド」と受け取ると少しズレます。
すでに運用しているlockfileを更新するコマンドとして理解しておくほうが自然です。
lockfileがまだない状態では、何が作られるかを表示する挙動になります。
ですので、アプリケーション依存のlockfileと同じものだと考えるより、開発環境で取得する配布物の情報を管理するものと見るほうが近いです。

`mise.toml` が「何を使うか」を表し、`mise.lock` が「どの配布物を取るか」を補う。
この関係で理解しておくと、役割をつかみやすいです。

## env は direnv 代替として見ると分かりやすい

### env は 状態 を書く場所と考えると分かりやすい

まず、公式ドキュメント上の大きい比較軸としては `direnv` です。
実際、Environmentsのページでも `direnv` との比較が前面に出ています。
「Like direnv it manages environment variables for different project directories.」という説明です。
miseの設定ファイルは[様々な形](https://mise.jdx.dev/configuration.html#mise-toml)で作成できますが、各ディレクトリに配置するとカレントディレクトリ内にある設定ファイルが優先的に参照され、重複する内容については参照の優先度に従って上書きされます。
そうすることでdirenvに近いイメージでenvを使うことができます。
このあたりはドキュメントをご参照ください。
@[card](https://mise.jdx.dev/configuration.html#visual-configuration-hierarchy)

そのうえで、ここから先の `状態` という整理は公式の用語定義ではなく、この記事の中で理解しやすくするための定義です。

いまの `env` は静的なkey/valueだけでなく、動的に状態を組み立てるしくみが入っています。

たとえば次のように書けます。

```toml
[env]
APP_ENV = "development"
AWS_REGION = "ap-northeast-1"
AWS_PROFILE = "myapp-dev"
_.file = ".env.shared"
_.path = ["{{config_root}}/node_modules/.bin"]
```

この例でやっていることは4つです。
アプリケーションの環境を指定すること。
利用するAWS Profileを決めること。
共有された `.env` 系ファイルを読むこと。
プロジェクト配下の実行パスを足すことです。

`direnv` はディレクトリごとの環境をどう反映するかが主題です。
miseの `env` も、読む対象が `.env` だけではなく、ディレクトリごとの状態をどう定義するかに重心があります。
その意味で、`.env` は `env._.file` で読める入力形式の1つ、と位置付けるのが自然です。

### `env._` は構成的な読み込み口

`env._` を見ると、だいぶ思想が分かります。
`_` は通常の変数名ではなく、envを組み立てるためのdirectiveです。

代表例は `_.file`、`_.path`、`_.source` です。

- `_.file`: dotenv / JSON / YAMLなどのファイルを読む
- `_.path`: PATHに追加する
- `_.source`: bash scriptをsourceしてexport済みの値を読む

この設計のよいところは、envを「値の一覧」ではなく「作り方の宣言」として書ける点です。
特に `_.source` があると、外部コマンドから得た値もprojectの状態に組み込めます。

### 動的評価は env の重要な進化点

最近のenvは、静的な設定を置くだけではありません。
外部スクリプトやplugin提供の `env._.<name>` directiveで、動的に値を作れます。

たとえばsecret managerから値を取りたい場面では、次のような構成にできます。

```toml
[env]
_.source = { path = "./scripts/load-secrets.sh", redact = true }
```

```bash
#!/usr/bin/env bash
set -euo pipefail

if [ -z "${DATABASE_URL:-}" ]; then
  database_url="$(aws secretsmanager get-secret-value --secret-id app/dev/database-url --query SecretString --output text)"
  [ -n "$database_url" ] || exit 1
  export DATABASE_URL="$database_url"
fi

if [ -z "${DEPLOY_TOKEN:-}" ]; then
  deploy_token="$(aws secretsmanager get-secret-value --secret-id app/dev/deploy-token --query SecretString --output text)"
  [ -n "$deploy_token" ] || exit 1
  export DEPLOY_TOKEN="$deploy_token"
fi
```

ここでは、実在する例としてAWS CLIでSecrets Managerを読む形にしています。
ただし大事なのはコマンド名そのものではなく、外部システムから値を取り込んで `env._.source` でprojectのenvへ渡す流れです。
1Password CLIでも、AWS Secrets Managerを読むwrapperでも、社内ツールでも同じ考え方で組めます。
また、上のように「すでに値が入っているならそのまま使い、空なら外部から読む」形にしておくと、shellや `mise.local.toml` 側で先に入れた値を優先できます。
外部コマンドが空文字を返した時にそのまま通さないよう、最低限の確認を入れておくのも実運用では大事です。

この例で主役なのは、secret store機能そのものというより、外部または暗号化された値を取り込み、それをprojectのenvとして組み立てる流れです。
その見方をすると、`env` の拡張としてsecrets周りを理解しやすくなります。

なお `env._.source` はbash scriptをsourceする前提です。
上の例では説明のためにshebangも付けていますが、実際にはshebangの種類よりも「bashで正しく動くこと」が重要です。miseがこのファイルをサブプロセスとして実行するのではなくbashから`source`する設計になっているため、公式ドキュメントの前提どおりbash互換のscriptとして書いておくのが安全です。

ここまでをまとめると、いまの `env` は単なるkey/valueの置き場ではありません。
値を直接書くだけでなく、外から読み込み、必要に応じて動的に組み立て、その扱い方まで宣言できる層になっています。

この記事で `env = 状態` と整理しているのは、こうした「値そのもの」より「そのプロジェクトがどんな状態で動くか」をまとめて表しやすいからです。

### `required` と `redact` が「チームで読む設定」にしてくれる

envがstateの定義になるなら、足りない値や見せたくない値も宣言できると便利です。
そこで役立つのが `required` と `redact` です。

```toml
[env]
DATABASE_URL = { required = "各自の DB 接続文字列を設定してください" }
DEPLOY_TOKEN = { required = true, redact = true }
```

`required` は、「この値がないと実行できない」を設定に書けます。
READMEやWikiに別で書くより、実際に使う場所へ寄せられます。

`redact` は、taskの出力などで値を見せないための指定です。
つまりenvは、単に値を持つだけでなく、値の扱い方まで宣言できます。

## tasks は「操作」を定義する層になった

### make や package.json の scripts と何が違うのか

tasksだけを見ると、`make` や `package.json` のscriptsでもよいように見えます。
実際、それぞれに長所があります。

それでもmiseのtasksを使う意味は、`tools` と `env` を前提に実行できることです。
入口が1つにまとまるので、説明がかなり減ります。

「Node 22と特定のenvを前提にdev serverを起動してほしい」。
こうした要求を、別々の道具へまたがずに書けます。

### `depends` と並列実行で DAG に近い書き味になる

最近のtasksは、単発コマンドのaliasを超えています。
依存関係を張ると、独立したtaskを並列に流せます。

```toml
[tasks.lint]
description = "Lint the project"
run = "pnpm exec eslint ."

[tasks.test]
description = "Run unit tests"
run = "pnpm test"

[tasks.check]
description = "Run checks"
depends = ["lint", "test"]
```

この例なら、`check` は `lint` と `test` のまとまりです。
しかもmiseは並列実行を前提に持っています。

そのため `check` を「開発前の入口」や「deploy前の関門」として置きやすいです。
`make all` より、開発者向けの動線をそのまま書きやすい印象があります。

### watch も含めると、日常の操作をそのまま載せられる

filesの変化でtaskを再実行したいなら、`mise watch` が使えます。
これは、指定したtaskを一度実行して終わりではなく、関連するファイルの変化を監視しながら繰り返し走らせるためのコマンドです。
たとえば型検査やテストのように、「保存するたびにもう一度流したい」処理と相性がよいです。
ただし `mise watch` は内部で `watchexec` を使うため、利用前に `watchexec` を入れておく必要があります。
このあと載せる実運用例では `cargo:watchexec-cli` を `tools` に含めているのはそのためです。

つまり `mise run typecheck` が1回だけの実行だとすると、`mise watch typecheck` は「監視しながら必要なタイミングでもう一度実行する」イメージです。
もちろん、型検査やlintの一部はIDEのextensionやlanguage serverがその場で見せてくれることもあります。
その場合でも、IDE依存にせずチームで同じ監視コマンドを持ちたい時があります。
また、ターミナル上でまとめて確認したい時にも `mise watch` が使いやすいです。
ただ、常用する `typecheck` に `sources` を付けると、通常の `mise run typecheck` まで「変更がないのでskip」と判定されることがあります。
そのため、watch用のtaskは分けておくほうが扱いやすいです。

```toml
[tasks.typecheck]
description = "Typecheck the app"
run = "pnpm exec tsc --noEmit"

[tasks.typecheck-watch]
description = "Re-run typecheck when files change"
run = "pnpm exec tsc --noEmit"
sources = ["src/**/*.ts", "src/**/*.tsx", "tsconfig.json"]
```

```bash
mise watch typecheck-watch
```

この形にすると、「型検査はどのコマンドで、どのファイルに反応するか」が設定に残ります。
通常の `mise run typecheck` は一発実行のまま残せるので、使い分けもしやすいです。
ローカルのメモやREADMEの一行に依存しにくくなります。

## secrets は env の拡張として見ると収まりがよい

最近のmiseにはsecretsのドキュメントもあります。
たとえば `sops` で暗号化したファイルを読む形や、`age` によるinlineな暗号化も用意されています。

そのうえで、私はこれを独立機能として見るより、`env` の拡張として見たほうが理解しやすいと感じました。

理由は単純で、最終的に必要なのはsecretそのものではなく、実行時に使うenvだからです。
暗号化ファイルを `env._.file` で読む。
外部コマンドで取得して `env._.source` で流し込む。
必要に応じてinlineで暗号化した値を置く。
さらに `redact` で見せ方を制御する。

こうして見ると、公式には `secrets` として独立したまとまりがありつつ、理解のし方としては `env` の延長線上に置くと整理しやすいです。
「機密値を含むstateをどう作るか」という話へ自然に収まります。

## 実運用ではこう書くと整理しやすい

ここまでの話を、1つの `mise.toml` にまとめると次のようになります。
Webアプリケーションをローカルで動かし、確認後にdeployする想定です。

```toml
[tools]
node = "22"
pnpm = "10"
"npm:prettier" = "3"
"cargo:watchexec-cli" = "2"

[env]
APP_ENV = "development"
AWS_REGION = "ap-northeast-1"
_.file = ".env.shared"
_.path = ["{{config_root}}/node_modules/.bin"]

[tasks.setup]
description = "Install dependencies"
run = "pnpm install --frozen-lockfile"

[tasks.dev]
description = "Start the local dev server"
run = "pnpm dev"
env = { _.source = { path = "./scripts/load-secrets.sh", redact = true }, DATABASE_URL = { required = "ローカル開発で使う接続先を設定してください。mise.local.toml か事前の環境変数で渡します" } }

[tasks.lint]
description = "Run eslint and prettier"
run = [
  "pnpm exec eslint .",
  "pnpm exec prettier --check .",
]

[tasks.test]
description = "Run the test suite"
run = "pnpm test"

[tasks.typecheck]
description = "Typecheck the app"
run = "pnpm exec tsc --noEmit"

[tasks.typecheck-watch]
description = "Re-run typecheck when files change"
run = "pnpm exec tsc --noEmit"
sources = ["src/**/*.ts", "src/**/*.tsx", "tsconfig.json"]

[tasks.check]
description = "Run all local checks"
depends = ["lint", "test", "typecheck"]

[tasks.deploy]
description = "Deploy after local checks"
depends = ["check"]
run = "pnpm run deploy"
env = { _.source = { path = "./scripts/load-secrets.sh", redact = true }, DEPLOY_TOKEN = { required = "deploy に使う token を設定してください。mise.local.toml か事前の環境変数で渡します", redact = true } }
```

この例で伝えたいのは、個々の書き方そのものではありません。
どの層に何を書くかが自然に分かれることです。

ここでは `DATABASE_URL` や `DEPLOY_TOKEN` だけでなく、secretを読む `_.source` もtop-levelの `[env]` に置いていません。
`dev` や `deploy` でだけ必要な読み込みまで常に評価すると、`setup` や `lint` のような別のtaskまで止まりやすいからです。
「project全体で常に必要な値」と「一部の操作でだけ必要な値」を分け、後者はtaskごとの `env` に寄せるほうが、実運用では扱いやすいです。

- `tools` に実行環境を書く
- `env` に状態を書く
- `tasks` に操作を書く

この整理をし、これをもとにmiseの設定ファイルが解釈できると、今までは`README`で細かく補足していた説明を減らせます。
新しく入った人にも、「まず`README`と`mise.toml` を見れば前提が分かる」と伝えやすくなります。

さらに配布物の情報までそろえたいなら、`mise.lock` も合わせて運用すると見通しがよくなります。
ここで押さえたいのは、`mise.toml` が「何を使うか」を表し、`mise.lock` が取得する配布物の情報を補う、という役割分担です。
そして `mise lock` は、既存のlockfileに入っているURLやchecksumを更新する時に使うものとして理解しておくと整理しやすいです。
こうしておくと、`mise.toml` が「何を使うか」を表し、`mise.lock` が「何を取るか」を補強します。
この組み合わせは、実務でも扱いやすいです。

## まとめ

miseはasdf alternativeからはっきりと進化しつつあります。

`tools`、`env`、`tasks` をまとめて扱うことで、開発環境の仕様を1つの設定に寄せられます。

私は、いまのmiseは次の形で整理するのが分かりやすいと見ています。

- `tools` は実行環境
- `env` は状態
- `tasks` は操作

この見方に立つと、miseは「便利なversion manager」から一段進みます。
開発に入る前の下ごしらえを定義するツールとして見えてきます。

asdf alternativeとして見ていたころの理解も、いまのmiseを考える入口としては有効です。
ですので、過去の理解を捨てる必要はありません。

むしろ「asdf alternativeとして入り、そこから少しずつ役割が広がってきた」ととらえると、今のmiseはかなり整理しやすいです。

[^1]: 前回記事では、miseの基本的な役割と導入を中心に整理しました。今回の記事はその続編です。
