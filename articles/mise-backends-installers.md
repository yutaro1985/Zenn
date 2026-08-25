---
title: "miseのbackendは実際に何を使ってインストールしているのか"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mise", "環境構築", "devtools"]
published: false
---

この記事では、2026-08-23にmacOS arm64上のmise 2026.8.10で確認した挙動を扱います。公式ドキュメントの仕様と、隔離したfixtureでの実機確認は分けて記述します。

ここでいう「安定」は、公式ドキュメントにexperimental表記がなく、この版のCLIで利用できたという調査上の区分です。将来の互換性を保証するラベルではありません。

## はじめに

miseの導入、`mise install`や`mise use`、`tools`、`env`、`tasks`の基本は、以前の記事にまとめています。

@[card](https://zenn.dev/yutaro1985/articles/introduction_of_mise)

次の記事では、`tools`を実行環境、`env`を状態、`tasks`を操作として整理しました。

@[card](https://zenn.dev/yutaro1985/articles/mise-beyond-asdf-alternative)

## backendを見る理由

miseの`[tools]`は、runtimeだけでなく普段使うCLIもまとめて管理できます。自分も以前はruntimeごとのversion managerに加えて、`npm install -g`、`pipx install`、`cargo install`を使い分けていました。miseを使い始めたころは、それらを`[tools]`へ寄せられるところを便利に感じていました。

ただ、`[tools]`に書いて`mise install`を実行したときに、その裏でどうやってインストールしているのかは、あまり意識していませんでした。あらためてドキュメントを読んでみると、runtimeやCLIによって経路が違い、必要な外部コマンドも変わります。

そこで今回は、backendごとにどこから情報や配布物を取得し、実際には何を使ってインストールしているのかを確認してみます。`mise.lock`に何が残るのかも合わせて見ていきます。

## 用語の整理

最初に、混ざりやすい4つの用語を分けます。

| 用語 | この記事での意味 |
| --- | --- |
| tool | `[tools]`で使いたいruntimeやCLI |
| registry | 裸のtool名をどのbackendへ解決するかの対応表 |
| backend | versionの探索、取得、インストールを担う実装またはエコシステム |
| installerまたは外部コマンド | backendが使う内蔵処理、または`uv`や`cargo`のような実行物 |

たとえば、fixtureで次のように確認しました。

```fish
mise registry ripgrep --json --security
```

関連する出力の抜粋です。並びは優先順を表します。

```text
aqua:BurntSushi/ripgrep
asdf:https://gitlab.com/wt0f/asdf-ripgrep
cargo:ripgrep
```

出力には、優先順で`aqua:BurntSushi/ripgrep`、`asdf:https://gitlab.com/wt0f/asdf-ripgrep`、`cargo:ripgrep`が含まれていました。裸の`ripgrep`は、この同梱registryの優先順から`aqua:BurntSushi/ripgrep`へ解決されます。一方で`"aqua:BurntSushi/ripgrep"`のように完全指定すれば、この対応表を経由しません。

短縮名は便利ですが、miseのreleaseやregistry設定が変われば選ばれる経路も変わり得ます。チームで配布元まで意図を残したいCLIは、先に`mise registry <tool>`を確認してからprefixを明示するのが分かりやすいです。[^registry]

## 代表的なbackend

すべてのbackendを列挙するより、installerの持ち主で4群に分けると見通しがよくなります。

| backend群 | 取得元とinstaller | 外部依存 | `mise.lock`で保持できる主な情報 |
| --- | --- | --- | --- |
| core / aqua / GitHub | mise側がproviderごとのdownload、展開、検証を多く担当する | coreはtoolごと。aquaは同梱registryとartifact URLを使う。GitHubは解決時にReleases APIを使う。`--locked`では記録済みasset URLを使える | coreはtoolごとに異なる。aquaとGitHubはURL、checksum、sizeなど |
| npm / pipx / cargo | package ecosystemを使う | npmは設定次第。pipxは`uv`または`pipx`、cargoは`cargo`が必要 | version中心 |
| vfox / asdf | pluginがversion解決やinstall手順を提供する | pluginの実装次第 | vfoxは一部のURLやprovenance、asdfはversion中心 |
| ubi / pkgx | 旧経路または実験的な経路 | backendごとに異なる | 新規の主例には使わない |

### core、aqua、GitHub

coreは共通のダウンローダではありません。Node.js、Python、Rubyなどは、それぞれmise側の実装が扱います。たとえばNode.jsでは、対象版のarchiveに加えて`SHASUMS256.txt`を取得します。

ただし、この検証方法を全core toolに当てはめると、正確性を欠きます。[^architecture]

`aqua:`はaqua CLIを呼ぶ薄いwrapperでもありません。miseに同梱されたaqua registryのsnapshotを参照し、mise自身がURLを取得して展開し、checksumを確認します。registry定義の配布物はupstreamにあるため、公開物の可用性までmiseが肩代わりするわけではありません。[^aqua]

`github:`は解決時にGitHub Releases APIからrelease assetを選び、miseがdownloadと展開を担います。URLを含むlockfileで`mise install --locked`を実行する場合は、記録済みのasset URLを使えます。この経路ではregistryやRelease APIを解決に使いません。asset名やplatformの組み合わせが特殊なrepositoryでは、自動選択を読まずに`asset_pattern`などをfixtureで確認する必要があります。[^github]

### npm、pipx、cargo

backend名と実コマンドが一致するとは限りません。ここは古い記事を読むときにも注意が必要な箇所です。

mise 2026.8.10の`npm:`は、デフォルトの`auto`でmise内蔵のaubeを使います。Node.js、npm、standaloneのaube CLIがPATHにない状態でもpackageをinstallできました。この内蔵aubeはv2026.7.12で追加された機能です。[^aube-release]

`npm.shell_out = true`では外部のnpmを使います。package managerに明示した`aube`は内蔵の経路です。`aube_cli`、`npm`、`bun`、`pnpm`には、対応する外部CLIが必要です。内蔵aubeではdependencyのlifecycle scriptがデフォルトで抑止されるため、必要なpackageだけ`allow_builds`で許可します。[^npm]

`pipx:`はデフォルト設定で、PATHに`uv`があり、globalまたはtoolごとの設定でuvの使用を無効にしていなければ、`uv tool install`を実行します。`uvx`という名前から別の実行ファイルを想像しがちですが、確認した実コマンドは`uv tool install`でした。それ以外では、外部の`pipx install`へfallbackします。Pythonのアプリケーション依存ではなく、隔離したPython CLIを置く経路として使います。[^pipx]

`cargo:`は外部の`cargo`へ委ねます。`cargo-binstall`が利用可能な設定ではprebuilt binaryを試す経路があります。

fixtureではcargoなしのため、`cargo install eza@0.18.18 --locked ...`を起動しようとして失敗しました。Rust製CLIを入れれば常にbinary downloadで済む、という理解では困ります。[^cargo]

### vfox、asdf、ubi、pkgx

vfoxとasdfは、複雑なinstallerや固有のenv exportが必要なcustom/private pluginの候補です。vfoxはmise内蔵のLua runtimeでplugin hookを実行します。asdfはbash plugin scriptを互換層から実行します。どちらもplugin sourceをreviewし、trustする境界が残ります。特にasdfはlegacy扱いで、新規の公式registry登録にはaquaまたはGitHubが勧められています。[^vfox][^asdf]

`ubi:`はdeprecatedです。GitHub Releasesの新規設定には`github:`を選びます。`pkgx:`はexperimentalです。使うにはexperimental設定が必要です。両者を「安定」の主例から外します。[^ubi][^pkgx]

## fixtureで見たinstallerとruntime

ここからは、2026-08-23に状態を隔離したfixtureで確認した結果です。長いverbose logは省き、判断に必要な行だけ残します。

### aquaのartifact lock

`"aqua:BurntSushi/ripgrep" = "14.1.1"`に対して`mise lock --platform macos-arm64`を実行しました。macOS arm64向けのrelease asset URLとSHA-256が`mise.lock`へ記録されました。

```text
ripgrep-14.1.1-aarch64-apple-darwin.tar.gz
SHA-256: 24ad7677...b937be
```

続く`mise install --locked --verbose`は、そのURLをdownloadし、checksumを確認して展開しました。exit 0になり、`rg --version`は`ripgrep 14.1.1`を返しました。aquaの例でURLとchecksumがlockされることは、npmやcargoにも同じ情報が残るという意味ではありません。

### npmの内蔵aubeとNode.js

`"npm:cowsay" = "1.6.0"`を、Node.js、npm、pnpm、bun、standaloneのaubeをPATHから外した状態でinstallしました。

```text
aube: Resolving cowsay@1.6.0...
Installed 33 packages
```

installは成功しましたが、同じPATHで`cowsay hello`を実行すると次の結果でした。

```text
exec: node: not found
```

`node = "24.13.0"`を同じ`[tools]`へ足してからinstallすると、`mise exec -- cowsay hello`はexit 0になりました。npm backendのinstallerはNode.jsなしで動けても、installしたCLIの実行時にNode.jsが必要になる場合があります。この違いを設定レビューで落とすと、初回installだけ通る構成になります。

### pipxとcargoの外部依存

`"pipx:httpie" = "3.2.4"`では、pipx CLIがなくuvだけをPATHに置いたfixtureで次の外部コマンドが記録されました。

```text
uv tool install httpie==3.2.4
```

`http`、`httpie`、`https`がinstallされ、exit 0でした。対照として、`"cargo:eza" = "0.18.18"`をcargoとcargo-binstallなしで試すと、次を起動したあとexit 1で失敗しました。

```text
cargo install eza@0.18.18 --locked --root <isolated install path>
```

成功例だけを設定の根拠にしないため、この失敗も残しておきます。backendごとに、installer自身の外部依存と、導入後のCLIが求めるruntimeを分けて見る必要があります。

## プロジェクト設定の例

runtime、配布済みCLI、Python CLIを1つの入口へ集めるなら、次のように書けます。`cowsay`はinstaller確認専用ですので、運用設定には含めません。

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

この例では、`[tools]`で導入経路を選び、`mise.lock`でbackendが対応できる範囲を固定し、`tasks`をチーム共通の確認入口にしています。`uv`も`[tools]`に明示することで、`pipx:httpie`が未宣言のglobal uvへ偶然依存しません。

このTOMLは、別々の空のmise data、cache、state、config領域で2回適用しました。`mise lock --platform macos-arm64`はNode 24.19.0、uv 0.12.5、ripgrep 14.1.1、HTTPie 3.2.4を解決しました。`mise install --locked`と`mise run tool-versions`は2回ともexit 0でした。

lockfileにはNode.jsとripgrepのURLとchecksumが残りました。uvではURL、checksum、GitHub attestation provenanceが残り、`pipx:httpie`はversionとbackendだけが残りました。

`minimum_release_age = "7d"`は、`node = "24"`のようなあいまいなversion解決で新しすぎるreleaseを避ける設定です。明示pinを守る機能でも、すべてのbackendの推移依存を固定する機能でもありません。npmとpipxでは依存の取得へ渡せる場合がありますが、同じ条件を他のbackendへ広げて考えないほうがよいです。[^lockfile][^settings]

## miseへ寄せる範囲

自分はruntimeと、repositoryをまたいで使う開発CLIを`[tools]`の候補にしています。一方でアプリケーション依存は、それぞれのpackage managerへ残します。

| 対象 | 主な置き場所 |
| --- | --- |
| Node.js、uv、ripgrep、HTTPieのようなruntimeやCLI | `mise.toml`と`mise.lock` |
| project固有のPrettierやTypeScript package | `package.json`と`pnpm-lock.yaml` |
| Python applicationのlibrary | `pyproject.toml`と`uv.lock` |
| Rust applicationのcrate | `Cargo.toml`と`Cargo.lock` |

たとえばproject固有のPrettierをデフォルトで`npm:prettier`へ移すつもりはありません。pnpmのdevDependencyに残せば、projectの依存関係と一緒に更新できます。

miseの`[tools]`に置く対象を、アプリケーションから独立して使うCLIだけにすると、責任が混ざりにくくなります。

`mise.lock`も1つの安全機能としてまとめません。aquaやGitHubではartifact URLとchecksumを記録できます。npm、pipx、cargo、asdfのようにversion中心のbackendとは情報量が違います。

checksum、provenance、plugin script、package lifecycle script、`mise trust`は、それぞれ防ぐ対象が異なります。lockfileがあるから安全、という結論にはしません。[^security]

## まとめ

backendのprefixは、取得元の名前だけではありません。どのinstallerが動くか、何を外部に要求するか、`mise.lock`が何を保持できるかを選ぶ指定です。

まずは既存の裸のtool名を置き換える前に、`mise registry <tool>`で候補を見ます。次に小さなfixtureでinstallと実行を分けて確かめ、必要なものだけprefixを明示します。チーム開発編ではlockfileとCI、タスク実行基盤編ではtasks側を扱う予定です。

[^registry]: [mise Registry](https://mise.jdx.dev/registry.html)
[^architecture]: [mise Backend Architecture](https://mise.jdx.dev/dev-tools/backend_architecture)
[^aqua]: [mise aqua backend](https://mise.jdx.dev/dev-tools/backends/aqua.html)
[^github]: [mise github backend](https://mise.jdx.dev/dev-tools/backends/github.html)
[^npm]: [mise npm backend](https://mise.jdx.dev/dev-tools/backends/npm.html)と[npm implementation v2026.8.10](https://github.com/jdx/mise/blob/v2026.8.10/src/backend/npm.rs)
[^aube-release]: [mise v2026.7.12 release](https://github.com/jdx/mise/releases/tag/v2026.7.12)
[^pipx]: [mise pipx backend](https://mise.jdx.dev/dev-tools/backends/pipx.html)と[pipx implementation v2026.8.10](https://github.com/jdx/mise/blob/v2026.8.10/src/backend/pipx.rs)
[^cargo]: [mise cargo backend](https://mise.jdx.dev/dev-tools/backends/cargo.html)と[cargo implementation v2026.8.10](https://github.com/jdx/mise/blob/v2026.8.10/src/backend/cargo.rs)
[^vfox]: [mise vfox backend](https://mise.jdx.dev/dev-tools/backends/vfox.html)
[^asdf]: [mise asdf backend](https://mise.jdx.dev/dev-tools/backends/asdf.html)
[^ubi]: [mise ubi backend](https://mise.jdx.dev/dev-tools/backends/ubi.html)
[^pkgx]: [mise pkgx backend](https://mise.jdx.dev/dev-tools/backends/pkgx.html)
[^lockfile]: [mise.lock](https://mise.jdx.dev/dev-tools/mise-lock.html)
[^settings]: [mise Settings](https://mise.jdx.dev/configuration/settings.html#minimum-release-age)
[^security]: [mise Security](https://mise.jdx.dev/dev-tools/security.html)
