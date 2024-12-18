---
title: "言語・ツールのバージョン管理、環境変数の管理、タスクランナーとしてまで使えるmiseというツールについて"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mise","環境構築"]
published: true
---

:::message
この記事は
[みすてむず いず みすきーしすてむず その2 Advent Calendar 2024](https://adventar.org/calendars/10269)[^1]の12日目です。
:::

表題のとおり、miseというツールがあります。
miseについての紹介記事もだいぶ増えてきましたが、まだ積極的に使っている人をあんまり見ないので、その有用性をお伝えするために、何番煎じとなってしまってもよいので書くことにしました。
※この記事は2024/12/12 00:00:00時点の情報、つまり`v2024.12.5`バージョンを元にしています。（この直後に`v2024.12.6`がリリースされました…）

## miseとは

公式
[mise](https://mise.jdx.dev/)

miseはざっくりいうとRust製の[asdf](https://asdf-vm.com/) Alternativeにあたるツールです。
asdfはいろんな言語や関連ツールのバージョン管理を統合的に行えるツールです。
実際、asdfのプラグインとして対応していたツールもmiseでインストールできます。
asdfのほうを使ったことがないので間違っていたら申し訳ないのですが、asdfは言語のバージョン管理に特化しているのに対して、miseは言語だけでなく、ツールや環境変数の管理、タスクランナーとしても使えるというのが特徴です。

[miseのGitHubリポジトリのREADME](https://github.com/jdx/mise)には以下のように書かれています。

> Like asdf (or nvm or pyenv but for any language) it manages dev tools like node, python, cmake, terraform, and hundreds more.
> Like direnv it manages environment variables for different project directories.
> Like make it manages tasks used to build and test projects.

asdfは主にShellで書かれていますが、前述の通りmiseはRustで書かれています。
それによりWin、Mac、Linuxとあまり環境のことを考えずに使えるのが良いところですね。

miseはかつてrtxという名前で、たいへん検索性が低かったです。
miseという名前もそんなに検索性が高いかと言われると難しいですが。

> Pronounced "MEEZ ahn plahs"

と公式にある通り、miseは「ミーズ」と読みます。
フランス語ですね。英語だと思っちゃうとなかなか読みづらいです。
mise-en-placeという言葉はフランス語で料理の下準備とか下ごしらえのことを表します。

以後、まずはmiseの簡単な使い方の説明から始めます。
また、miseはドキュメントでもDev Tools、Environments、Tasksと3つに分かれているのでそれぞれについて書いていきます。

## miseの導入方法

公式の[Getting Started](https://mise.jdx.dev/getting-started.html)を参照しましょう。
丸写しになってしまうと良くないので細かくは上記ページを見てほしいのですが、とりあえずGetting Startedではcurlでインストールしています

```bash
curl https://mise.run | sh
```

他にも[Installing Mise](https://mise.jdx.dev/installing-mise.html)に、Homebrew、dnf、apt、cargoなど(他にもたくさんあります)でのインストール方法が書かれています。
個人的にはbrewでインストールしています。
Shellもいろんなものにツール自体で対応しており、bash,zshだけでなくfish,xonsh,elvish,nushellにも対応しています（Completionは残念ながらbash,zsh,fishのみが公式に対応しているもののようです）。
今後増える可能性もありますし、増やしてほしいならIssueやPRを送るとよいでしょう。

IDEと組み合わせて使う場合のこともドキュメントに書いてあるので詳しくはそちらを御覧ください。
[IDE Integration](https://mise.jdx.dev/ide-integration.html)

英語にはなってしまいますが、ドキュメントの情報量がそこそこ充実しているのもmiseの良いところです。

## 設定ファイル

miseは過去の類似ツール(`.*env`といった名前のツール)で使われていた`.*-version`のようなファイルや、asdfの`.tool-versions`との互換性もあり、それらを使って言語やツールのバージョンを管理することもできます。
しかし、mise自体には`mise.toml`というファイルを扱ってバージョンを扱う機能があるので、新しくmiseを使う場合はそちらを使うことをお勧めします。

※asdfとの違いについては公式に[Comparison to asdf](https://mise.jdx.dev/dev-tools/comparison-to-asdf.html)というページがあるのでそちらを参照してみてください。

`mise.toml`とその設定内容については[こちらのページ](https://mise.jdx.dev/configuration.html)に詳しく書いてあります。

`mise.toml`と書いていますが、実際には下記のようにさまざまなファイルを認識できます。

> mise.toml is the config file for mise. They can be at any of the following file paths (in order of precedence, top overrides configuration of lower paths):
mise.local.toml - used for local config, this should not be committed to source control
mise.toml
mise/config.toml
.config/mise.toml - use this in order to group config files into a common directory
.config/mise/config.toml
.config/mise/conf.d/*.toml - all files in this directory will be loaded in alphabetical order

※書いてないですが、`.mise.toml`も認識されるようです。

`mise cfg`コマンドで現在miseが認識しているファイルを確認できます。
グローバルな設定は通常`.config/mise/config.toml`に書くことが多いです。
プロジェクトに個別に`mise.toml`を配置もできますし、`mise.local.toml`を使って個人の環境に合わせた設定もできます。
また、ディレクトリごとに`mise.toml`を配置することもできます。

各言語やツールのバージョンの設定はもちろん、環境変数やタスクランナーの設定もこの`mise.toml`にまとめて記述します。
だからこそ、miseの設定はちゃんと`mise.toml`の形式で書くことを推奨します。

Dev Tools、Environments、Tasksの設定については後述します。

設定可能なすべての項目は[Settings](https://mise.jdx.dev/configuration/settings.html)を参照してください。
自分も正直まだ全然把握しきれていません…。

新しく`mise.toml`やそれに類するファイルを作った時は`mise trust`コマンドでmiseに認識させる必要があるのでその点はご注意ください。
認識させてない状態で動かそうとするとメッセージが出るはずです。

## Dev Tools

※ここに書いてあることは[Dev Tools](https://mise.jdx.dev/dev-tools.html)にも書いてあるので、一次情報としてはそちらを参照してください。

Dev Toolsとしての機能では、言語のツールやバージョンの管理を行います。
個人的にはこれがmiseのメインの機能だと思っています。
asdfを使っていた人にはイメージしやすいでしょう。

バージョンは`mise.toml`に記述します。

参考までに、自分のMacの`.config/mise/config.toml`を載せておきます。
※ややバージョンが古いのは気にしないでください…。

```toml
[tools]
node = "20.10.0"
terraform = "1.9.7"
terraform-docs = "0.17.0"
python = "3.11.6"
```

`mise.toml`にあらかじめこのように書いておいて`mise install`をすることで、`mise.toml`に書かれたバージョンの各ツールがインストールされます。

また、`mise install node@20.0.0`のように特定バージョンを指定してインストールもできます。
`mise install node@20`だと最新の20系がインストールされます。
`mise install node`のようにバージョン指定をしなければ`mise.toml`に書かれたバージョンがインストールされます。
こうしてツールをインストールしたうえで`mise use`コマンドで実際に使用するバージョンを指定します。
`mise use node@20.0.0`のように指定すると、現在参照している`mise.toml`にここで指定したバージョンが記載されます。
`mise exec`または`mise x`で、miseでインストールした言語やツールのバージョンを指定したうえでコマンド実行することもできます。

重ねてになりますが、詳しくは[Dev Tools](https://mise.jdx.dev/dev-tools.html)をご参照ください。

## Environments

※より詳しいことは[Environments](https://mise.jdx.dev/environments.html)をご参照ください。

miseでは環境変数を管理することもできます。
これの何がうれしいかというと、前述の通り`mise.toml`はいろんなディレクトリに配置できるので、そのディレクトリ事に環境変数を設定できます。
これは従来[direnv](https://github.com/direnv/direnv)を使って行っていたことですが、その役割をmiseが担うことができます。
※ちなみに、miseとdirenvは併用できます。[こちらのドキュメント](https://mise.jdx.dev/direnv.html)を参照。個人的には混乱の元なのでどちらか（可能であればmise側）に寄せることをお勧めします。

自分がよく使うのは、AWS_PROFILEの切替ですね。

```toml
[env]
AWS_PROFILE = "foo"
AWS_SDK_LOAD_CONFIG = 1
```

少し話が逸れるのですが、AWS_PROFILEを環境変数に設定しておくと、`~/.aws/config`に定義してあるprofileを使用できます。
これにより、ディレクトリごとに使うprofileを切り替えてTerraformの実行先を切り替えるというのを自分はよくやります。
これを今までdirenvでやっていたのですが、今はmiseを使うようにしています。

もちろん、Dev Toolsで記載していた言語やツールのバージョン設定も一緒に記述できるので、バージョン管理と環境変数をまとめられます。

その他の使い方は[Environments](https://mise.jdx.dev/environments.html)を参照してください。

## Tasks

自分は最近まで知らなかったのですが、miseはタスクランナーとしても使えます。
あんまりよく知らなかったので自分も使い方を知るつもりで記述します。

たとえば、`mise.toml`に以下のように記述します。

```toml
[tasks.build]
description = "Build the CLI"
run = "cargo build"
```

こうすると`mise run build`で`cargo build`が実行されます。
miseの他のコマンドと被らなければ`mise build`のようにrunを省略しても実行できるようになったそうですが、混乱の元になりそうなので毎回`mise run`にしたほうが良さそうです。
自分はローカルで以下のように試してみました。

```toml
[tasks.sl]
description = "Run SL"
run = "sl"
```

これにより`mise run sl`または`mise sl`でSLが走ることを確認しました。

なお、実行可能なTaskは`mise tasks`で確認できます。
他にもタスクをグループ化する、ファイルを監視して変化した時にタスクを実行するなど高度な使い方もあるそうですので、詳細は[Running Tasks](https://mise.jdx.dev/tasks/running-tasks.html)を参照してください。
タスクは別ファイルに切り出すこともできるそうで、そのへんの使い方は[File Tasks](https://mise.jdx.dev/tasks/file-tasks.html)に詳しく書いてあります。

個人的にもあまりまだ使えてないので今後活用したいなと思っています。

※参考
かなり詳細に書いているので「どう書いたらいいんだっけ」みたいなのはだいたい解決すると思います
[Tasks](https://mise.jdx.dev/tasks/)
[Running Tasks](https://mise.jdx.dev/tasks/running-tasks.html)
[TOML-based Tasks](https://mise.jdx.dev/tasks/toml-tasks.html)
[File Tasks](https://mise.jdx.dev/tasks/file-tasks.html)

## 注意点

miseはさまざまな言語やツールを統合的に管理できますが、そのぶん各言語内で管理を完結させようとするツールとの相性が悪めです。
個人的には特に相性が悪いと感じたのは[Volta](https://volta.sh/)でしょうか。
Voltaは言語のバージョンも`packege.json`で管理しようとするため、ツールの責任範囲がバッティングしてしまいます。
JSのバージョン管理だけVoltaでやって、他のツールの管理だけmiseでやる、というのはちょっと面倒です。
自分自身がJSのエコシステムに慣れていないのもあるのですが、このへんをうまく解決できませんでした。
うまく使い分けているよという方はぜひコメントで教えてください。
他にも、言語やツールのバージョン管理の責任範囲がバッティングするケースがあるはずですので、何をどのように管理しているかには注意を配りましょう（最近だとuvでPythonを管理するケースなど？）。

## まとめ

今回はmiseというすばらしいツールの紹介をしました。
言語やツールのバージョン管理は今までにもさまなツールがあり、言語やツールごとに乱立していましたが、miseはasdfと同様にそれらを統合的に管理できるツールです。
それに加えて、環境変数の管理やタスクランナーとして使えるのはこのあたりを管理する人にとっては非常に助かるのではないでしょうか。
みなさんがこの記事を見ることでmiseを知って、言語やツールのバージョン管理に対する悩みが解消されてくれることを願っています。

[^1]: [みすてむず いず みすきーしすてむず](https://misskey.systems/) とは、オープンソースのプラットフォームMisskeyのインスタンスのひとつで、主にITに関わる人が参加しています。最近はXよりもそちらに入り浸っています。
