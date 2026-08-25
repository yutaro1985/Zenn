---
title: "miseのbackendは実際に何を使ってインストールしているのか"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mise", "環境構築", "devtools"]
published: false
---

この記事は、公式ドキュメントと公式リポジトリを読み、実際に使ったときの自分の感覚をもとにまとめています。手元での確認は、2026-08-23にmacOS arm64上のmise 2026.8.10で行いました。

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

たとえば、普段使っているmiseの設定やcacheの影響を受けないように、data、cache、state、configの保存先を分けた検証用環境で次のように確認しました。

```fish
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

公式ドキュメントのBackendsには、18種類のbackendが載っています。これとは別に、Node.jsやPythonなどをmise内蔵の実装で扱うcore backendと、pluginとして追加するcustom backendがあります。[^backends]

| backend | 何を使ってインストールするのか | 公式ドキュメント上の扱い |
| --- | --- | --- |
| core | Node.jsやPythonなど、toolごとに用意されたmise内蔵の実装 | 一般backendとは別に記載 |
| aqua | aqua Registryの定義を読み、miseが配布物を取得、展開し、定義されているchecksumや署名を検証する | registry登録のTier 1 |
| asdf | asdf pluginのbash scriptをmiseの互換層から実行する | legacy。新規のregistry登録は受け付けない |
| cargo | `cargo-binstall`または`cargo install`でRust crateを入れる | registry登録のTier 3 |
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

ここでいうTierは、backendの安定度やsupport levelではなく、mise registryへ新しいtoolを登録するときの受け入れ方針です。Tier 1は優先して受け付けるbackend、Tier 2とTier 3は順に登録の条件が厳しくなります。`mise registry ripgrep`で表示されるbackendの優先順とは別の話です。[^registry]

以降で実際に試すbackendは絞ります。npm backendは、以前使っていた`npm install -g`と同じpackage ecosystemを扱うため取り上げます。

自分はPython CLIをglobalの`pip install`で入れていました。一方、miseには隔離した環境へPython CLIを入れる`pipx:` backendがあるため、その違いを確認します。Rustは使っておらず、cargoも使ったことがないため、cargoは一覧への掲載にとどめました。

### core、aqua、GitHub

coreは共通のダウンローダではありません。Node.js、Python、Rubyなどは、それぞれmise側の実装が扱います。たとえばNode.jsでは、対象版のarchiveに加えて`SHASUMS256.txt`を取得します。

ただし、この例だけで全core toolの実装を同じだとは断定できません。[^architecture]

aquaはmiseとは別のtool managerで、独自のregistryを持っています。miseの`aqua:` backendはaqua CLIを呼び出さないため、aquaを別途インストールする必要はありません。miseに同梱されたaqua Registryを参照し、mise自身が配布物を取得、展開します。registry定義と配布元が対応していれば、checksumや署名、attestationも検証します。配布物自体は各toolのGitHub Releasesなどに置かれています。[^aqua]

新しいregistry entryではasdfよりaquaやGitHubが優先されています。aquaではplugin scriptを実行せずに済み、Windowsを含む複数platformを扱えます。

`github:`は解決時にGitHub Releases APIからrelease assetを選び、miseがdownloadと展開を担います。URLを含むlockfileで`mise install --locked`を実行する場合は、記録済みのasset URLを使えます。この経路ではregistryやRelease APIを解決に使いません。asset名やplatformの組み合わせが特殊なrepositoryでは、`asset_pattern`などを検証用環境で確認する必要があります。[^github]

### npmとpipx

backend名と実コマンドが一致するとは限りません。ここは古い記事を読むときにも注意が必要な箇所です。

mise 2026.8.10の`npm:`は、デフォルトの`auto`でmise内蔵のaubeを使います。Node.js、npm、standaloneのaube CLIがPATHにない状態でもpackageをinstallできました。この内蔵aubeはv2026.7.12で追加された機能です。[^aube-release]

デフォルトの`auto`で`npm.shell_out = true`にすると、version情報の取得とinstallを外部のnpmへ切り替えます。`npm.package_manager`でinstallerを明示した場合、installにはその指定が優先されます。`aube`は内蔵の経路です。`aube_cli`、`npm`、`bun`、`pnpm`には、対応する外部CLIが必要です。内蔵aubeではdependencyのlifecycle scriptがデフォルトで抑止されるため、必要なpackageだけ`allow_builds`で許可します。[^npm]

`pipx:`は、miseから実行できる`uv`があり、globalまたはtoolごとの設定でuvを無効にしていなければ、デフォルトで`uv tool install`を実行します。uvが見つからないか無効にされている場合は、外部の`pipx install`を使います。Python applicationが使うlibraryではなく、HTTPieのようなPython製CLIを個別の環境へインストールするためのbackendです。[^pipx]

### vfox、asdf、ubi、pkgx

vfoxとasdfは、複雑なinstallerや固有のenv exportが必要なcustom/private pluginで使えます。vfoxはmise内蔵のLua runtimeでplugin hookを実行します。asdfはbashのplugin scriptを互換層から実行します。install時にpluginの処理が実行されるため、使う前にpluginのsourceを確認する必要があります。特にasdfはlegacy扱いで、新規の公式registry登録にはaquaまたはGitHubが勧められています。private pluginを作る場合は、asdfよりvfoxが勧められています。[^vfox][^asdf]

`ubi:`はdeprecatedです。GitHub Releasesの新規設定には`github:`を選びます。`pkgx:`はexperimentalです。使うにはexperimental設定が必要です。どちらも今回の実機確認からは外しました。[^ubi][^pkgx]

## 検証用環境で見たinstallerとruntime

ここからは、2026-08-23に普段のmise環境と保存先を分けて確認した結果です。長いverbose logは省き、確認できたコマンドと結果を載せます。

### aquaのartifact lock

`"aqua:BurntSushi/ripgrep" = "14.1.1"`に対して`mise lock --platform macos-arm64`を実行しました。macOS arm64向けのrelease asset URLとSHA-256が`mise.lock`へ記録されました。

```text
ripgrep-14.1.1-aarch64-apple-darwin.tar.gz
SHA-256: 24ad7677...b937be
```

続く`mise install --locked --verbose`は、そのURLをdownloadし、checksumを確認して展開しました。exit 0になり、`rg --version`は`ripgrep 14.1.1`を返しました。aquaの例でURLとchecksumがlockされることは、ほかのbackendにも同じ情報が残るという意味ではありません。

### npmの内蔵aubeとNode.js

`cowsay`は、渡した文字列をASCII artの牛に話させるnpm packageです。ここでは、npm backendによるinstallと、installしたCLIを動かすために必要なruntimeを分けて確認するために使います。

`"npm:cowsay" = "1.6.0"`を、Node.js、npm、pnpm、bun、standaloneのaubeをPATHから外した状態でinstallしました。

```text
aube: Resolving cowsay@1.6.0...
Installed 33 packages
```

installは成功しましたが、同じPATHで`cowsay hello`を実行すると次の結果でした。

```text
exec: node: not found
```

`node = "24.13.0"`を同じ`[tools]`へ足してからinstallすると、`mise exec -- cowsay hello`はexit 0になりました。npm backendのinstall自体にはNode.jsが不要でも、installしたCLIの実行時には必要になる場合があります。今回のcowsayは`npm:cowsay`だけでinstallできました。Node.jsも`[tools]`へ追加しなければ実行できませんでした。

### pipxが使うuv

HTTPieは、APIやHTTP serverの動作確認に使えるPython製のコマンドラインHTTP clientです。

`"pipx:httpie" = "3.2.4"`では、pipx CLIがなくuvだけをPATHに置いた検証用環境で次の外部コマンドが記録されました。

```text
uv tool install httpie==3.2.4
```

`http`、`httpie`、`https`がinstallされ、exit 0でした。ここでは`pipx:`というbackend名でも、実際にはuvが使われています。backendごとに、installer自身の外部依存と、導入後のCLIが求めるruntimeを分けて見る必要があります。

## プロジェクト設定の例

ここまで確認したNode.js、ripgrep、HTTPieをprojectの`mise.toml`で管理するなら、次のように書けます。`cowsay`はnpm backendの確認にしか使っていないため、実際の設定には含めません。

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

この例では、`[tools]`にruntimeとCLI、それをインストールするbackendを書いています。`tasks.tool-versions`は、開発環境がそろっているかをチーム全員が同じコマンドで確認するためのものです。`uv`も`[tools]`に書き、PCにたまたま入っているglobalのuvを使わないようにしています。

このTOMLは、miseのdata、cache、state、configの保存先を空のdirectoryに分けて2回試しました。`mise lock --platform macos-arm64`はNode 24.19.0、uv 0.12.5、ripgrep 14.1.1、HTTPie 3.2.4を解決しました。`mise install --locked`と`mise run tool-versions`は2回ともexit 0でした。

lockfileにはNode.jsとripgrepのURLとchecksumが残りました。uvではURL、checksum、GitHub attestation provenanceが残り、`pipx:httpie`はversionとbackendだけが残りました。

release日時を取得できるbackendでは、`minimum_release_age = "7d"`により公開後7日未満のversionを除外できます。これは、`node = "24"`のようなあいまいなversion指定で使う設定です。公開日時がないversionは原則として除外されず、versionを明示した場合まで拒否する設定でもありません。また、すべてのbackendで依存packageまで同じ条件が適用されるわけではありません。npmとpipxでは依存packageの取得にもこの値を渡せますが、backendごとに挙動は異なります。[^lockfile][^settings]

## miseで管理するもの、package managerで管理するもの

`mise.toml`はrepositoryごとに置けるため、そのprojectで使うruntimeやCLIのversionもそろえられます。projectの`mise.toml`に書いたCLIも、`node_modules`やPythonのvirtual environmentへ入るわけではなく、miseの管理領域へインストールされます。

一方、アプリケーションが使うpackageや、ほかのpackageと一緒に更新したいCLIは、言語ごとのpackage managerで管理します。project固有かどうかだけではなく、何と一緒にversionを固定して更新したいかで分けています。

| 対象 | 主な置き場所 |
| --- | --- |
| Node.js、uv、ripgrep、HTTPieのようなruntimeや、アプリケーションの依存関係から独立して使うCLI | `mise.toml`と`mise.lock` |
| PrettierやTypeScriptなど、ほかのnpm packageと一緒に使うもの | `package.json`と`pnpm-lock.yaml` |
| Python applicationのlibrary | `pyproject.toml`と`uv.lock` |
| Go applicationのモジュール | `go.mod`と`go.sum` |

たとえばPrettierは、pluginやほかのnpm packageと同じタイミングで更新したいなら、pnpmのdevDependencyで管理します。`pnpm exec prettier`を使えば、projectに入っているversionを実行できます。

project固有のCLIでも、ほかのpackageから独立してversionをそろえたいものなら、projectの`mise.toml`で管理できます。自分はruntimeと、この条件に当てはまるCLIを`[tools]`に書いています。

`mise.lock`に記録される内容はbackendごとに異なります。aquaやGitHubではartifact URLとchecksumを記録できますが、npm、pipx、asdfなどはversion中心です。

checksumは取得したfileが想定したものと同じかを確認し、provenanceは配布物がどこでbuildされたかを確認するために使います。plugin scriptやpackage lifecycle scriptはinstall時にcodeを実行し、`mise trust`はそのrepositoryの設定を信頼するかを決めます。`mise.lock`があっても、これらをすべて確認できるわけではありません。[^security]

## まとめ

backendのprefixを明示すると、取得元とinstaller、必要な外部コマンド、`mise.lock`に残せる情報が決まります。

まずは既存の裸のtool名を置き換える前に、`mise registry <tool>`で候補を見ます。次に普段のmise設定から分けた小さな検証用環境で、installと実行を分けて確かめます。その結果を見て、必要なものだけprefixを明示します。チーム開発編ではlockfileとCI、タスク実行基盤編ではtasks側を扱う予定です。

[^registry]: [mise Registry](https://mise.jdx.dev/registry.html)
[^ripgrep-registry]: [mise v2026.8.10のripgrep registry entry](https://github.com/jdx/mise/blob/v2026.8.10/registry/ripgrep.toml)
[^backends]: [mise Backends](https://mise.jdx.dev/dev-tools/backends/)
[^architecture]: [mise Backend Architecture](https://mise.jdx.dev/dev-tools/backend_architecture)
[^aqua]: [mise aqua backend](https://mise.jdx.dev/dev-tools/backends/aqua.html)
[^github]: [mise github backend](https://mise.jdx.dev/dev-tools/backends/github.html)
[^npm]: [mise npm backend](https://mise.jdx.dev/dev-tools/backends/npm.html)と[npm implementation v2026.8.10](https://github.com/jdx/mise/blob/v2026.8.10/src/backend/npm.rs)
[^aube-release]: [mise v2026.7.12 release](https://github.com/jdx/mise/releases/tag/v2026.7.12)
[^pipx]: [mise pipx backend](https://mise.jdx.dev/dev-tools/backends/pipx.html)と[pipx implementation v2026.8.10](https://github.com/jdx/mise/blob/v2026.8.10/src/backend/pipx.rs)
[^vfox]: [mise vfox backend](https://mise.jdx.dev/dev-tools/backends/vfox.html)
[^asdf]: [mise asdf backend](https://mise.jdx.dev/dev-tools/backends/asdf.html)
[^ubi]: [mise ubi backend](https://mise.jdx.dev/dev-tools/backends/ubi.html)
[^pkgx]: [mise pkgx backend](https://mise.jdx.dev/dev-tools/backends/pkgx.html)
[^lockfile]: [mise.lock](https://mise.jdx.dev/dev-tools/mise-lock.html)
[^settings]: [mise Settings](https://mise.jdx.dev/configuration/settings.html#minimum-release-age)
[^security]: [mise Security](https://mise.jdx.dev/dev-tools/security.html)
