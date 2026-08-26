---
title: "miseのbackendは実際に何を使ってインストールしているのか"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mise", "環境構築", "devtools"]
published: false
---

この記事では、miseが各backendで実際に何を使ってインストールするのかを確認します。手元での確認は、2026-08-23にmacOS arm64上で、fishとmise 2026.8.10を使って行いました。

正確な仕様や最新の挙動は、脚注に載せた公式ドキュメントと公式リポジトリの実装を確認してください。本文でも、ドキュメントで確認した内容と手元で試した結果は、できるだけ分けて書きます。

## はじめに

miseの導入、`mise install`や`mise use`、`tools`、`env`、`tasks`の基本は、以前の記事にまとめています。

@[card](https://zenn.dev/yutaro1985/articles/introduction_of_mise)

次の記事では、`tools`を実行環境、`env`を状態、`tasks`を操作として整理しました。

@[card](https://zenn.dev/yutaro1985/articles/mise-beyond-asdf-alternative)

## backendを見る理由

miseの`[tools]`は、runtimeだけでなく普段使うCLIもまとめて管理できます。自分も以前は、runtimeごとにversion managerを使っていました。npm packageとして配布されているCLIは`npm install -g`、Python packageとして配布されているCLIはglobalの`pip install`で入れていました。miseを使い始めたころは、こうしたruntimeやCLIのversionを`[tools]`にまとめて書けるところを便利に感じていました。

ただ、`[tools]`に書いて`mise install`を実行したときに、その裏でどうやってインストールしているのかは、あまり意識していませんでした。あらためてドキュメントを読んでみると、runtimeやCLIによって経路が違い、必要な外部コマンドも変わります。

そこで今回は、backendごとにどこから情報や配布物を取得し、実際には何を使ってインストールしているのかを確認してみます。`mise.lock`に何が残るのかも合わせて見ていきます。

## 用語の整理

最初に、混ざりやすい4つの用語を分けます。

| 用語 | この記事での意味 |
| --- | --- |
| tool | `[tools]`で使いたいruntimeやCLI |
| registry | 裸のtool名をどのbackendへ解決するかの対応表 |
| backend | versionの探索、取得、インストールを担う実装またはエコシステム |
| installerまたは外部コマンド | backendが使う内蔵処理、または`uv`や`pipx`のような実行物 |

まず、`ripgrep`という短縮名がどのbackendへ解決されるのかを確認します。普段使っている設定やcacheに残っている情報を拾わないように、miseのdata、cache、state、config用directoryを空の場所へ分けて実行しました。

```shell
mise registry ripgrep --json --security
```

このときの出力は次のとおりです。

```json
{
  "short": "ripgrep",
  "backends": [
    "aqua:BurntSushi/ripgrep",
    "asdf:https://gitlab.com/wt0f/asdf-ripgrep",
    "cargo:ripgrep"
  ],
  "description": "ripgrep recursively searches directories for a regex pattern while respecting your gitignore",
  "aliases": [
    "rg"
  ],
  "security": [
    {
      "type": "checksum",
      "algorithm": "sha256"
    }
  ]
}
```

`backends`の並びは、registryに記録された優先順です。裸の`ripgrep`は、この同梱registryの優先順から`aqua:BurntSushi/ripgrep`へ解決されます。一方で`"aqua:BurntSushi/ripgrep"`のように完全指定すれば、この対応表を経由しません。[^ripgrep-registry]

短縮名は便利ですが、miseのreleaseやregistry設定が変われば選ばれる経路も変わり得ます。チームで同じbackendからインストールしたいCLIは、先に`mise registry <tool>`を確認してからprefixを明示するのが分かりやすいです。[^registry]

## backendの一覧

公式ドキュメントのBackendsには、custom backendを含む19種類のbackendが載っています。これとは別に、Node.jsやPythonなどをmise内蔵の実装で扱うcore backendがあります。[^backends]

最初に、公式ドキュメントに載っている19種類とcore backendを一覧で確認します。そのあと、自分がこれまで使うことの多かったnpmとpipに関係する経路を中心に、実際に何が動くのかを見ていきます。aquaは自分の過去の利用状況とは別に、registryへtoolを新しく登録するときのTier 1として扱われているため取り上げます。

| backend | 何を使ってインストールするのか | 公式ドキュメント上の扱い |
| --- | --- | --- |
| core | Node.jsやPythonなど、toolごとに用意されたmise内蔵の実装 | 一般backendとは別に記載 |
| aqua | aqua Registryの定義を読み、miseが配布物を取得、展開し、定義されているchecksumや署名を検証する | registry登録のTier 1 |
| asdf | asdf pluginのbash scriptをmiseの互換層から実行する | legacy。新規のregistry登録は受け付けない |
| cargo | 外部の`cargo-binstall`、設定で有効にしたmise内蔵のbinary installer、または`cargo install`でRust crateを入れる | registry登録のTier 3 |
| conda | anaconda.orgから単一のconda packageを直接取得して展開する | registry登録のTier 2 |
| dotnet | `dotnet tool install`で.NET toolを入れる | registry登録のTier 3 |
| forgejo | Forgejoのrelease assetをmiseが取得する | Tierの記載なし |
| gem | `gem install`でRubyGemを入れる | registry登録のTier 3 |
| github | GitHub Releasesのassetをmiseが取得する | registry登録のTier 1 |
| gitlab | GitLab Releasesのassetをmiseが取得する | registry登録のTier 1 |
| go | `go install`でGoモジュールを入れる | registry登録のTier 3 |
| http | 指定したHTTPまたはHTTPS URLからarchiveや実行ファイルを取得する | Tierの記載なし |
| npm | 内蔵のaube、またはnpm、pnpm、bunなどでnpm packageを入れる | registry登録のTier 3 |
| pipx | `uv tool install`または`pipx install`でPython CLIを入れる | registry登録のTier 3 |
| pkgx | pkgx pantryのmetadataをmiseが解決し、bottleをdownloadしてchecksumを確認する | experimental |
| s3 | Amazon S3またはS3互換storageから配布物を取得する | Tierの記載なし |
| spm | artifact bundleをmiseが取得するか、Swift Package Managerでsourceからbuildする | Tierの記載なし |
| ubi | GitHubまたはGitLabのreleaseから実行ファイルを探して入れる | deprecated。新規のregistry登録は受け付けない |
| vfox | mise内蔵のLua runtimeでvfox pluginを実行する | 新規のregistry登録は受け付けない |
| custom | 独自のbackend pluginにversion探索やinstall処理を実装する | Tierの記載なし |

公式ドキュメントでは、Tierについて次のように書かれています。

> Backends fall into the following acceptance tiers for new registry entries:

ここでいうTierはbackendの安定度ではなく、mise registryへtoolを新しく登録するときの受け入れ方針です。Tier 1は優先して受け付けるbackendで、Tier 2、Tier 3の順に登録の条件が厳しくなります。`mise registry ripgrep`で表示されるbackendの優先順とは別の話です。[^registry]

### core、aqua、GitHub

coreは共通のダウンローダではありません。Node.js、Python、Rubyなどは、それぞれmise側の実装が扱います。たとえばNode.jsでは、対象版のarchiveに加えて`SHASUMS256.txt`を取得します。

ただし、この例だけで全core toolの実装を同じだとは断定できません。[^architecture]

aquaはmiseとは別のtool managerで、独自のregistryを持っています。miseの`aqua:` backendはaqua CLIを呼び出さないため、aquaを別途インストールする必要はありません。miseのreleaseに含まれるaqua Registryの情報を読み、mise自身が配布物を取得、展開します。registryの定義と配布元が対応している場合は、checksumや署名、attestationも検証します。配布物自体は各toolのGitHub Releasesなどに置かれています。[^aqua]

registryへtoolを新しく登録する場合は、asdfよりaquaやGitHubが優先されています。aquaではplugin scriptを実行せずに済み、Windowsを含む複数platformを扱えます。

`github:`は解決時にGitHub Releases APIからrelease assetを選び、miseがdownloadと展開を担います。`mise.lock`に現在のplatform用URLがある場合、`mise install --locked`は記録済みのasset URLを使います。このassetの取得ではregistryやRelease APIによる再解決が不要になります。ただし、URLを記録できないbackendまで同じ挙動になるわけではありません。asset名やplatformの組み合わせが特殊なrepositoryでは、`asset_pattern`などを検証用環境で確認する必要があります。[^github]

### npmとpipx

backend名と実際に動くコマンドが一致するとは限りません。ここは古い記事を読むときにも注意が必要な箇所です。

`aube`は、miseと同じ作者が開発しているNode.js向けpackage managerです。mise 2026.8.10の`npm:` backendでは、デフォルトの`auto`がmise内蔵のaubeを使います。Node.js、npm、単体のaube CLIがPATHにない状態でもpackageをインストールできます。この経路はv2026.7.12で追加されました。[^aube][^aube-release]

デフォルト以外を使いたい場合は、`npm.package_manager`でインストールに使うpackage managerを指定できます。たとえばpnpmを使う場合は、次のように書きます。

```toml
[settings.npm]
package_manager = "pnpm"
```

指定できるのは`aube`、`aube_cli`、`npm`、`pnpm`、`bun`です。`aube`はmiseに組み込まれたものを使い、それ以外は対応する外部CLIを実行します。この設定はtoolごとのoptionではなく、設定が有効な範囲の`npm:` backend全体に適用されます。`npm.package_manager`をデフォルトの`auto`にしたまま`npm.shell_out = true`を設定した場合は、version情報の取得とインストールに外部のnpmを使います。

内蔵aubeでは、dependencyのlifecycle scriptがデフォルトで抑止されます。scriptの実行が必要なpackageは、`allow_builds`で個別に許可します。[^npm]

`pipx:`は、BlackやRuffのようなPython製CLIを個別のvirtual environmentへインストールするためのbackendです。アプリケーションからimportするNumPyや`requests`などのライブラリを管理するものではありません。公式ドキュメントでも、CLIの例にはBlack、ライブラリの例にはNumPyと`requests`が使われています。[^pipx]

デフォルトでは、最初にmiseから`uv`を実行できるか確認します。`uv`を実行でき、設定で無効にしていなければ、実際に使われるのは`uv tool install`です。`uv`が見つからない場合や、globalまたはtoolごとの設定で無効にした場合は、外部の`pipx install`を使います。

すべての`pipx:` toolでuvを使わない場合は、設定全体で無効にできます。

```toml
[settings.pipx]
uvx = false
```

特定のtoolだけuvを使わず、pipxでインストールする場合は、そのtoolへ`uvx = false`を指定します。

```toml
[tools]
"pipx:httpie" = { version = "3.2.4", uvx = false }
```

`uvx = false`で明示できるのは、uvを使わずpipxへ切り替えることだけです。[^pipx]

`cargo:` backendにも、`cargo.binstall = false`で`cargo-binstall`ではなく`cargo install`を使う設定があります。自分はCargoを使ったことがないので、ここでは設定があることだけに触れておきます。[^cargo]

Rubyのcore backendでは、`ruby.ruby_install = true`を設定できます。Rubyをsourceからbuildする場合、installerが`ruby-build`から`ruby-install`へ変わります。[^ruby-install]

SPM backendでは、`install_command`でsourceからインストールするときのコマンドを指定できます。[^spm-install-command]

ほかにも、precompiled binaryを使うかsourceからbuildするかを決める設定があります。指定できる内容はそれぞれ異なるため、詳細は公式ドキュメントを参照してください。[^core-compile-settings]

### vfox、asdf、ubi、pkgx

vfoxとasdfは、複雑なinstallerや固有のenv exportが必要なcustom/private pluginで使えます。vfoxはmise内蔵のLua runtimeでplugin hookを実行します。asdfはbashのplugin scriptを互換層から実行します。install時にpluginの処理が実行されるため、使う前にpluginのsourceを確認する必要があります。特にasdfはlegacy扱いで、新規の公式registry登録にはaquaまたはGitHubが勧められています。private pluginを作る場合は、asdfよりvfoxが勧められています。[^vfox][^asdf]

`ubi:`はdeprecatedです。GitHub Releasesの新規設定には`github:`を選びます。`pkgx:`はexperimentalです。使うにはexperimental設定が必要です。どちらも今回の実機確認からは外しました。[^ubi][^pkgx]

## 実際に動くinstallerを確認する

ここからは、記事の冒頭に書いたmacOS arm64環境で実際に確認した結果です。既存のmise設定やcacheがbackendの選択に影響しないように、data、cache、state、config用directoryを新しく作り、普段使っているものとは分けました。

verbose logは長いため、実行したコマンドと、aubeやuvが使われたことを確認できる行を載せます。

### aquaでは配布物のURLとchecksumがlockfileに残る

最初に、ripgrep 14.1.1をaqua backendで指定しました。

```toml
[settings]
lockfile = true

[tools]
"aqua:BurntSushi/ripgrep" = "14.1.1"
```

自分の環境に合わせて、macOS arm64用のlock情報を生成します。

```shell
mise lock --platform macos-arm64
```

生成された`mise.lock`には、次の内容が記録されました。

```toml
# @generated - this file is auto-generated by `mise lock` https://mise.jdx.dev/dev-tools/mise-lock.html

[[tools."aqua:BurntSushi/ripgrep"]]
version = "14.1.1"
backend = "aqua:BurntSushi/ripgrep"

[tools."aqua:BurntSushi/ripgrep"."platforms.macos-arm64"]
checksum = "sha256:24ad76777745fbff131c8fbc466742b011f925bfa4fffa2ded6def23b5b937be"
url = "https://github.com/BurntSushi/ripgrep/releases/download/14.1.1/ripgrep-14.1.1-aarch64-apple-darwin.tar.gz"
url_api = "https://api.github.com/repos/BurntSushi/ripgrep/releases/assets/191355384"
```

続けて、lockfileを使ってインストールし、versionを確認しました。

```shell
mise install --locked --verbose
mise exec -- rg --version | head -n 1
```

```text
ripgrep 14.1.1 (rev 4649aa9700)
```

verbose logでは、lockfileに記録されたURLからarchiveをdownloadし、checksumを確認して展開する処理も確認できました。ただし、URLとchecksumが残るかどうかはbackendによって異なります。

### npm backendは内蔵aubeでインストールできる

`cowsay`は、渡した文字列をASCII artの牛に話させるnpm packageです。npm backendがインストールに必要とするものと、インストール後のCLIが実行時に必要とするものを分けて確認するために使いました。

最初はNode.jsを指定せず、`cowsay`だけを書きます。

```toml
[tools]
"npm:cowsay" = "1.6.0"
```

Node.js、npm、pnpm、bun、単体のaube CLIをPATHから外した状態で実行しました。

```shell
mise install --verbose 2>&1 | rg 'aube: Resolving cowsay@1\.6\.0|Installed 33 packages'
```

verbose logには、内蔵aubeを使ったことを示す行が出力されました。

```text
aube: Resolving cowsay@1.6.0...
Installed 33 packages
```

インストールは成功しましたが、同じPATHのまま実行するとNode.jsがないため失敗します。

```shell
mise exec -- cowsay hello
```

```text
exec: node: not found
```

そこで、同じ`mise.toml`へNode.jsを追加しました。

```toml
[tools]
node = "24.13.0"
"npm:cowsay" = "1.6.0"
```

```shell
mise install
mise exec -- cowsay hello
```

```text
 _______
< hello >
 -------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

今回のcowsayは、Node.jsがなくても内蔵aubeでインストールできました。一方、インストール後のCLIを実行するにはNode.jsが必要でした。npm backendを使っただけでは、実行に必要なNode.jsまで自動的に`[tools]`へ追加されません。

### pipx backendはuvを使うことがある

HTTPieは、APIやHTTP serverの動作確認に使えるPython製のコマンドラインHTTP clientです。ここでは、backend名が`pipx:`でも実際にはuvが使われることを確認するために選びました。

```toml
[tools]
"pipx:httpie" = "3.2.4"
```

pipx CLIを置かず、uvだけを実行できる状態でインストールしました。

```shell
mise install --verbose 2>&1 | rg 'uv tool install httpie==3\.2\.4'
```

verbose logには、miseが実行した次のコマンドが記録されました。

```text
uv tool install httpie==3.2.4
```

インストール後にHTTPieのversionを確認します。

```shell
mise exec -- http --version
```

```text
3.2.4
```

`pipx:`というbackend名から、必ずpipx CLIが実行されるとは限りません。今回のようにuvを使う場合は、installerとして必要なのはuvです。

## プロジェクト設定の例

ここまで確認したNode.js、ripgrep、HTTPieをprojectの`mise.toml`で管理するなら、次のように書けます。`cowsay`はnpm backendの動作確認にだけ使ったため、この設定には含めません。

```toml
[settings]
lockfile = true
minimum_release_age = "7d"

[tools]
node = "24"
uv = "0.12"
"aqua:BurntSushi/ripgrep" = "14"
"pipx:httpie" = "3"

[tasks.tool-versions]
description = "開発用CLIのバージョンを確認する"
run = ["node --version", "rg --version", "http --version"]
```

`node = "24"`は、24系に一致するversionを選ぶ指定です。`24.19.0`のようにpatch versionまで固定する書き方とは異なります。まだlockfileがない場合や、`mise lock --bump`で再解決した場合は、その時点で条件に一致するversionが選ばれます。同じように、`uv = "0.12"`は0.12系、`"aqua:BurntSushi/ripgrep" = "14"`は14系を指定しています。[^lockfile]

この例では、`[tools]`にruntimeとCLI、それをインストールするbackendを書いています。`tasks.tool-versions`は、開発環境がそろっているかをチーム全員が同じコマンドで確認するためのものです。`uv`も`[tools]`に書き、PCへ個別にインストールされたuvへ依存しないようにしています。

この`mise.toml`を空のdata、cache、state、config用directoryで2回試しました。最初に次のコマンドを実行しています。

```shell
mise tasks validate
mise lock --platform macos-arm64
mise install --locked
mise run tool-versions
```

`mise lock`はNode.js 24.19.0、uv 0.12.5、ripgrep 14.1.1、HTTPie 3.2.4を解決しました。生成された`mise.lock`は次のとおりです。

```toml
# @generated - this file is auto-generated by `mise lock` https://mise.jdx.dev/dev-tools/mise-lock.html

[[tools."aqua:BurntSushi/ripgrep"]]
version = "14.1.1"
backend = "aqua:BurntSushi/ripgrep"

[tools."aqua:BurntSushi/ripgrep"."platforms.macos-arm64"]
checksum = "sha256:24ad76777745fbff131c8fbc466742b011f925bfa4fffa2ded6def23b5b937be"
url = "https://github.com/BurntSushi/ripgrep/releases/download/14.1.1/ripgrep-14.1.1-aarch64-apple-darwin.tar.gz"
url_api = "https://api.github.com/repos/BurntSushi/ripgrep/releases/assets/191355384"

[[tools.node]]
version = "24.19.0"
backend = "core:node"

[tools.node."platforms.macos-arm64"]
checksum = "sha256:8294b7aa9b03997481c06babf1e8b270c859358f27da57a11509afe537ac381d"
url = "https://nodejs.org/dist/v24.19.0/node-v24.19.0-darwin-arm64.tar.gz"

[[tools."pipx:httpie"]]
version = "3.2.4"
backend = "pipx:httpie"

[[tools.uv]]
version = "0.12.5"
backend = "aqua:astral-sh/uv"

[tools.uv."platforms.macos-arm64"]
checksum = "sha256:5bb0e5fe008a773c3dbcb97ff79cd89e1241464fe9d2f986d52ad8f1b037bd62"
url = "https://github.com/astral-sh/uv/releases/download/0.12.5/uv-aarch64-apple-darwin.tar.gz"
url_api = "https://api.github.com/repos/astral-sh/uv/releases/assets/514850968"
provenance = "github-attestations"
```

Node.jsとripgrepにはURLとchecksumが残りました。uvには、それらに加えてGitHub Artifact Attestationsを使った検証を示す`provenance`も残っています。`pipx:httpie`に記録されたのはversionとbackendだけです。この差は、公式ドキュメントのbackend別の対応状況とも一致します。[^lockfile]

同じ`mise.toml`と`mise.lock`を別の空のdirectoryへコピーし、もう一度`mise install --locked`と`mise run tool-versions`を実行しました。2回とも同じversionがインストールされ、taskも正常に終了しました。

release日時を取得できるbackendでは、`minimum_release_age = "7d"`により公開後7日未満のversionを候補から外せます。`node = "24"`のように複数のversionが条件へ一致するとき、公開されたばかりのversionをすぐには選ばないための設定です。

公開日時がないversionは、原則として候補から外れません。`24.19.0`のようにversionを明示した場合まで拒否する設定でもありません。また、すべてのbackendで依存packageまで同じ条件が適用されるわけではありません。npmとpipxでは依存packageの取得にもこの値を渡せますが、backendごとに挙動は異なります。[^lockfile][^settings]

## miseで管理するもの、package managerで管理するもの

`mise.toml`はrepositoryごとに置けるため、そのprojectで使うruntimeやCLIのversionもそろえられます。projectの`mise.toml`に書いたCLIも、`node_modules`やPythonのvirtual environmentへ入るわけではなく、miseの管理領域へインストールされます。

アプリケーションから読み込むpackageは、言語ごとのpackage managerで管理します。開発用CLIでも、同じpackage群と組み合わせて使うものはpackage manager側へ置くことがあります。

| 対象 | 主な置き場所 |
| --- | --- |
| Node.js、uv、ripgrep、HTTPieのようなruntimeや、アプリケーションの依存関係から独立して使うCLI | `mise.toml`と`mise.lock` |
| PrettierやTypeScriptなど、projectのnpm packageと組み合わせて使うもの | `package.json`と`pnpm-lock.yaml` |
| Pythonアプリケーションから読み込むライブラリ | `pyproject.toml`と`uv.lock` |
| Goアプリケーションのモジュール | `go.mod`と`go.sum` |

たとえばPrettier本体と`prettier-plugin-*`を組み合わせて使い、CIでも同じ構成でformatするprojectを考えます。この場合は、PrettierとpluginをpnpmのdevDependencyへ入れ、`package.json`と`pnpm-lock.yaml`で一緒に管理するほうが分かりやすいです。`pnpm exec prettier`を使えば、そのprojectへインストールしたversionを実行できます。

project固有のCLIでも、ほかのpackageから独立してversionをそろえたいものなら、projectの`mise.toml`で管理できます。自分はruntimeと、この条件に当てはまるCLIを`[tools]`に書いています。

`mise.lock`に記録される内容はbackendごとに異なります。今回確認したaquaでは配布物のURLとchecksumが残りましたが、npmとpipxではversionとbackendだけでした。公式ドキュメントでも、aquaやGitHubと、npmやpipxではlockfileの対応範囲が分けられています。[^lockfile]

checksumは取得したfileが想定したものと同じかを確認し、provenanceは配布物がどこでbuildされたかを確認するために使います。plugin scriptやpackage lifecycle scriptはinstall時にcodeを実行し、`mise trust`はそのrepositoryの設定を信頼するかを決めます。`mise.lock`があっても、これらをすべて確認できるわけではありません。[^security]

## まとめ

backendのprefixを明示すると、取得元とinstallerの候補を絞れます。また、必要になる外部コマンドや、`mise.lock`に残せる情報を確認しやすくなります。

既存の裸のtool名を置き換える前に、`mise registry <tool>`で候補を確認します。挙動まで確認したい場合は、普段の設定やcacheを使わない小さな環境を用意し、インストールと実行を分けて試します。その結果を見て、必要なものだけprefixを明示します。チーム開発編ではlockfileとCI、タスク実行基盤編ではtasks側を扱う予定です。

[^registry]: [mise Registry](https://mise.jdx.dev/registry.html)と[Registry document v2026.8.10](https://github.com/jdx/mise/blob/v2026.8.10/docs/registry.md)
[^ripgrep-registry]: [mise v2026.8.10のripgrep registry entry](https://github.com/jdx/mise/blob/v2026.8.10/registry/ripgrep.toml)
[^backends]: [mise Backends](https://mise.jdx.dev/dev-tools/backends/)と[Backends document v2026.8.10](https://github.com/jdx/mise/blob/v2026.8.10/docs/dev-tools/backends/index.md)
[^architecture]: [mise Backend Architecture](https://mise.jdx.dev/dev-tools/backend_architecture)
[^aqua]: [mise aqua backend](https://mise.jdx.dev/dev-tools/backends/aqua.html)と[aqua backend document v2026.8.10](https://github.com/jdx/mise/blob/v2026.8.10/docs/dev-tools/backends/aqua.md)
[^github]: [mise github backend](https://mise.jdx.dev/dev-tools/backends/github.html)と[github backend implementation v2026.8.10](https://github.com/jdx/mise/blob/v2026.8.10/src/backend/github.rs)
[^npm]: [mise npm backend](https://mise.jdx.dev/dev-tools/backends/npm.html)と[npm implementation v2026.8.10](https://github.com/jdx/mise/blob/v2026.8.10/src/backend/npm.rs)
[^aube]: [aube](https://github.com/jdx/aube)
[^aube-release]: [mise v2026.7.12 release](https://github.com/jdx/mise/releases/tag/v2026.7.12)
[^pipx]: [mise pipx backend](https://mise.jdx.dev/dev-tools/backends/pipx.html)と[pipx implementation v2026.8.10](https://github.com/jdx/mise/blob/v2026.8.10/src/backend/pipx.rs)
[^cargo]: [mise cargo backend](https://mise.jdx.dev/dev-tools/backends/cargo.html)と[cargo backend document v2026.8.10](https://github.com/jdx/mise/blob/v2026.8.10/docs/dev-tools/backends/cargo.md)
[^ruby-install]: [Ruby settings v2026.8.10](https://github.com/jdx/mise/blob/v2026.8.10/settings.toml#L2373-L2387)
[^spm-install-command]: [mise SPM backend](https://mise.jdx.dev/dev-tools/backends/spm.html)と[SPM backend document v2026.8.10](https://github.com/jdx/mise/blob/v2026.8.10/docs/dev-tools/backends/spm.md#L152-L174)
[^core-compile-settings]: [Python compile setting v2026.8.10](https://github.com/jdx/mise/blob/v2026.8.10/settings.toml#L2120-L2130)
[^vfox]: [mise vfox backend](https://mise.jdx.dev/dev-tools/backends/vfox.html)
[^asdf]: [mise asdf backend](https://mise.jdx.dev/dev-tools/backends/asdf.html)
[^ubi]: [mise ubi backend](https://mise.jdx.dev/dev-tools/backends/ubi.html)
[^pkgx]: [mise pkgx backend](https://mise.jdx.dev/dev-tools/backends/pkgx.html)
[^lockfile]: [mise.lock](https://mise.jdx.dev/dev-tools/mise-lock.html)と[mise.lock document v2026.8.10](https://github.com/jdx/mise/blob/v2026.8.10/docs/dev-tools/mise-lock.md)
[^settings]: [mise Settings](https://mise.jdx.dev/configuration/settings.html#minimum-release-age)
[^security]: [mise Security](https://mise.jdx.dev/security.html)
