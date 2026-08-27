---
title: "チーム開発の初期セットアップをmiseで簡単にする"
emoji: "🤝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mise", "python", "uv", "pnpm", "githubactions"]
published: false
---

この記事では、チーム開発でrepositoryをcloneした後に必要なtoolのインストールと依存packageの準備を、miseから実行する方法を確認します。例として、Pythonの依存packageをuv、Prettierをpnpmで管理するrepositoryを用意します。同じmise taskを呼び出すGitHub Actionsのworkflowも書きます。

手元での確認は、2026-08-26にmacOS arm64上で、fish 3.7.1とmise 2026.8.14を使って行いました。この検証では、普段のglobal設定やcacheの影響を避けるため、保存先を一時directoryへ変更しています。これは記事の検証時だけに行った設定であり、通常の初期セットアップには必要ありません。Linux用のlock情報も生成しましたが、Linux上でのインストールとGitHub Actionsの実行までは確認していません。

本文は公式ドキュメントと、実際に試したときの自分の理解を元に書いています。正確な仕様や最新の挙動は、リンク先の公式ドキュメントと公式リポジトリの実装を確認してください。

## はじめに

開発プロジェクトへ参加するときは、まずGitHubからrepositoryをcloneし、手元に開発環境を用意します。使用するruntimeやpackage managerが増えると、それぞれのversionを合わせ、依存packageをインストールする必要があります。

この初期セットアップをmiseから実行し、チームの開発者が同じコマンドを使えるようにします。GitHub Actionsから同じmise taskを呼び出す方法も確認します。

前の記事では、miseがtoolをインストールするときに使うbackendを確認しました。記事の最後では、runtimeや独立したCLIを`mise.toml`と`mise.lock`で管理する例も載せています。

@[card](https://zenn.dev/yutaro1985/articles/mise-backends-installers)

Pythonアプリケーションでも、開発に使うtoolがPythonのpackageだけでそろうとは限りません。具体例では、Pythonの依存packageをuvで管理し、MarkdownやJSONのformatにはPrettierを使います。Prettierはnpm packageとして配布されており、一般的にはnpmでインストールします。今回は、自分が普段使っているpnpmで管理します。

Pythonアプリケーションから読み込むpackageは、`pyproject.toml`と`uv.lock`で管理しました。プロジェクトで使うPrettierには、`package.json`と`pnpm-lock.yaml`を使っています。

## 今回使うPythonプロジェクト

例として、`/health`を返すだけの小さなFastAPIアプリケーションを用意しました。Pythonのtestにはpytest、lintにはRuffを使います。PrettierはREADMEと`package.json`のformatを確認します。

repositoryへ置くファイルは次のとおりです。

```text
.
├── .github
│   └── workflows
│       └── check.yml
├── .gitignore
├── README.md
├── app.py
├── mise.lock
├── mise.toml
├── package.json
├── pnpm-lock.yaml
├── pyproject.toml
├── tests
│   └── test_app.py
└── uv.lock
```

各ファイルでは次のものを管理します。

| 管理するもの | 設定ファイル | lockfile |
| --- | --- | --- |
| Python、uv、Node.js、pnpm | `mise.toml` | `mise.lock` |
| FastAPI、Uvicorn、pytest、Ruff | `pyproject.toml` | `uv.lock` |
| Prettier | `package.json` | `pnpm-lock.yaml` |

`[tools]`へ書くpnpmは、npm packageをインストールするためのpackage managerです。Prettierもnpm packageとして配布されています。今回は`npm:prettier`としてmiseへ直接インストールせず、`package.json`へ書きます。

アプリケーションのコードは、`/health`へアクセスすると固定のJSONを返すだけです。

```python
# app.py
from fastapi import FastAPI

app = FastAPI()


@app.get("/health")
def health() -> dict[str, str]:
    return {"status": "ok"}
```

pytestでは`health`の戻り値を確認します。

```python
# tests/test_app.py
from app import health


def test_health() -> None:
    assert health() == {"status": "ok"}
```

Prettierで確認するREADMEは次の2行です。

```md
# mise team example

Pythonアプリケーションの依存packageはuv、Prettierはpnpmで管理します。
```

個人用の`mise.local.toml`と`mise.local.lock`、依存packageのインストール先、testやlintが作るcacheはGitの管理対象から外します。

```gitignore
mise.local.toml
mise.local.lock
.venv/
node_modules/
__pycache__/
.pytest_cache/
.ruff_cache/
```

### Pythonの依存package

`pyproject.toml`は次の内容にしました。

```toml
[project]
name = "mise-team-example"
version = "0.1.0"
description = "miseのチーム開発例で使うPythonアプリケーション"
requires-python = ">=3.13,<3.14"
dependencies = [
  "fastapi>=0.116,<1",
  "uvicorn>=0.35,<1",
]

[dependency-groups]
dev = [
  "pytest>=8,<10",
  "ruff>=0.12,<1",
]

[tool.ruff]
target-version = "py313"
```

アプリケーションが実行時に使うFastAPIとUvicornは`dependencies`へ、開発時に使うpytestとRuffは`dependency-groups.dev`へ書いています。実際にインストールするversionは`uv.lock`へ記録します。

### pnpmで管理するPrettier

`package.json`は次の内容です。

```json
{
  "name": "mise-team-example",
  "private": true,
  "scripts": {
    "format:check": "prettier --check README.md package.json"
  },
  "devDependencies": {
    "prettier": "^3.9.6"
  }
}
```

pnpmのversionは`mise.toml`と`mise.lock`で管理するため、`package.json`には`packageManager`を書いていません。pnpm 10では、`packageManager`に指定したpnpmを自動でダウンロードして実行する設定がデフォルトで有効です。両方へversionを書くと更新箇所が増えるため、今回はmise側だけで管理します。[^pnpm-settings]

Prettierはアプリケーションの実行には使わないため、`devDependencies`へ入れます。format対象のファイルも`package.json`の`scripts`へ書き、miseからは`pnpm --silent run format:check`を呼び出します。Prettierを実行するscriptを`package.json`に残しておけば、設定を変更するときに確認する場所が分かりやすくなります。

## 今回使うmise.toml

`mise.toml`には、toolのversionと、依存packageのインストールや確認に使うtaskを書きました。

```toml
min_version = "2026.8.14"

[settings]
lockfile = true

[tool_config]
locked = true

[tools]
python = "3.13"
uv = "0.12"
pnpm = "10"
node = "24"

[env]
UV_PYTHON = { value = "{{ tools.python.path }}", tools = true }

[tasks.setup]
description = "uvとpnpmで依存packageをインストールする"
run = [
  "uv sync --locked",
  "pnpm install --frozen-lockfile",
]

[tasks.lint]
description = "Pythonと設定ファイルを確認する"
run = [
  "uv run --locked ruff check .",
  "pnpm --silent run format:check",
]

[tasks.test]
description = "Pythonのtestを実行する"
run = "uv run --locked python -m pytest -q"

[tasks.check]
description = "CIと同じ確認を実行する"
depends = ["lint", "test"]
```

`min_version`を指定しているため、この設定で使った機能に対応していない古いmiseでは処理を続けません。

`[settings]`の`lockfile = true`は、`mise.lock`の生成と更新を有効にします。`[tool_config]`の`locked = true`は、このconfig rootで宣言するtoolにlockfileを必須とします。`mise.local.toml`へ個人用toolを書いた場合は、`mise.local.lock`が対応します。lockfileの設定については公式ドキュメントにも説明があります。[^mise-lock]

`UV_PYTHON`には、miseでインストールしたPythonのパスを設定します。uvは同じversionのPythonを自分で管理している場合や、systemにインストール済みの場合も検出します。この設定を入れておけば、`uv sync`は`mise.lock`に記録したPythonを使って`.venv`を作成します。miseのPython向けドキュメントにも同じ設定例があります。[^mise-python]

`setup`では、uvとpnpmのlockfileを変更せずに依存packageをインストールします。uvの`--locked`は`uv.lock`が古い場合に失敗し、pnpmの`--frozen-lockfile`は`pnpm-lock.yaml`が`package.json`と合っていない場合に失敗します。[^uv-sync][^pnpm-install]

`check`は`lint`と`test`を実行します。ローカルとCIで`mise run check`を使うため、確認内容をGitHub ActionsのYAMLへ重ねて書かずに済みます。taskの定義方法や`depends`の詳しい動作は、miseのTask Configurationを参照してください。[^task-configuration]

## 3つのlockfileを作る

`mise.toml`の`python = "3.13"`は、3.13系の中から条件に合うversionを選ぶ指定です。`node = "24"`や、ほかのtoolも同じようにmajorまたはminor versionまでしか指定していません。このままではlockfileを作る時期によって、選ばれるpatch versionが変わります。

そこで、開発環境で使うtoolを`mise.lock`へ記録します。今回は自分のmacOS arm64環境とGitHub ActionsのLinux x64環境を指定しました。

```shell
mise lock --platform macos-arm64,linux-x64
```

この例の`mise.lock`に含めたplatformは、macOS arm64とLinux x64だけです。ほかのplatformを使う開発者がいる場合は、必要なplatformを`--platform`へ追加してlockfileを更新します。[^mise-lock]

Pythonの依存packageとPrettierにも、それぞれのpackage managerでlockfileを作ります。

```shell
mise install
mise exec -- uv lock
mise exec -- pnpm install --lockfile-only
```

生成された`mise.lock`は63行でした。全体を掲載する代わりに、次のコマンドでtool、確定したversion、backend、platformの見出しを取り出しました。

```shell
grep -E '^(lockfile_version|\[\[tools\.|version =|backend =|\[tools\..*platforms\.)' mise.lock
```

```text
lockfile_version = 1
[[tools.node]]
version = "24.19.0"
backend = "core:node"
[tools.node."platforms.linux-x64"]
[tools.node."platforms.macos-arm64"]
[[tools.pnpm]]
version = "10.34.5"
backend = "aqua:pnpm/pnpm"
[tools.pnpm."platforms.linux-x64"]
[tools.pnpm."platforms.macos-arm64"]
[[tools.python]]
version = "3.13.15"
backend = "core:python"
[tools.python."platforms.linux-x64"]
[tools.python."platforms.macos-arm64"]
[[tools.uv]]
version = "0.12.5"
backend = "aqua:astral-sh/uv"
[tools.uv."platforms.linux-x64"]
[tools.uv."platforms.macos-arm64"]
```

前の記事で確認したbackendが、今回の`mise.lock`にも記録されています。mise 2026.8.14では、PythonとNode.jsはcore backend、uvとpnpmはaqua backendへ解決されました。将来も同じbackendになると決めつけず、lockfileの差分で確認します。

:::message alert
検証の途中では、`[tools]`へNode.js、pnpmの順に書いていました。この設定では、`mise which pnpm`がNode.jsと一緒に入ったCorepackのshimを指すことがありました。pnpm、Node.jsの順にすると、`aqua:pnpm/pnpm`から入れたpnpm 10.34.5を使いました。

これはmise 2026.8.14で確認したPATHの結果であり、設定順の仕様として保証されているとは考えていません。miseでpnpmを管理する場合は、`mise which pnpm`も確認します。Corepackを使う方針なら、miseの`[tools]`からpnpmを外し、どちらでversionを管理するのかを決めます。[^mise-node]
:::

実際の各platformの項目には、配布物のURLとSHA-256 checksumも記録されました。前の記事で載せたripgrepの例と同じく、backendによってlockfileへ記録できる内容は異なります。

`npm`、`pipx`、`cargo`、`go`のように配布物のURLを記録できないbackendは、strict lockの検査対象外です。`locked = true`を指定しても、すべてのbackendを同じ方法で固定できるわけではありません。[^mise-lock]

3つのlockfileをcommitすると、管理する対象は次のようになります。

| lockfile | 今回記録されたもの |
| --- | --- |
| `mise.lock` | Python 3.13.15、uv 0.12.5、pnpm 10.34.5、Node.js 24.19.0とplatformごとの配布物 |
| `uv.lock` | FastAPI、Uvicorn、pytest、Ruffと、それらが利用するPython package |
| `pnpm-lock.yaml` | Prettierと、それが利用するnpm packageの解決結果 |

`mise.lock`だけではアプリケーションの依存packageまで固定されません。反対に、`uv.lock`と`pnpm-lock.yaml`だけでは、実行に使うPython、uv、Node.js、pnpmのversionまではそろいません。

## 新しい環境での準備と確認

ここではmise CLIをインストール済みで、`mise`コマンドを実行できることを前提にします。インストールしていない場合は、公式のGetting Startedを参照してください。[^mise-getting-started]

ここまでに作成したファイルをrepositoryへcommitしたあと、新しい開発環境を準備する流れは次のとおりです。

1. repositoryをcloneし、そのdirectoryへ移動する
2. `mise.toml`の内容を確認してtrustする
3. `mise install`でtoolをインストールする
4. `mise tasks validate`でtaskの定義を確認する
5. `mise run setup`と`mise run check`を実行する

まず、cloneした`mise.toml`の内容を確認します。miseのtaskには任意のコマンドを書けます。内容を読んでからtrustします。[^trust]

```shell
sed -n '1,220p' mise.toml
mise trust mise.toml
```

trustした後、lockfileに記録されたtoolをインストールします。

```shell
mise install
```

ここでは`mise install`を使っています。`[tool_config]`の`locked = true`により、このrepositoryの`mise.toml`に書いたtoolにはlockfileが必要です。

コマンドへ`--locked`を付けると、ユーザー設定を含めたtool全体にlocked modeが適用されます。今回はrepositoryの設定だけを対象にしたいため、`mise install --locked`は使いませんでした。[^mise-lock]

インストール後にversionを確認します。

```shell
mise exec -- python --version
mise exec -- uv --version
mise exec -- node --version
mise exec -- pnpm --version
```

```text
Python 3.13.15
uv 0.12.5 (210d1f678 2026-08-14 aarch64-apple-darwin)
v24.19.0
10.34.5
```

次に、taskの参照先が存在するか、循環した依存関係がないかなどを確認します。`mise tasks validate`はtaskを実行せず、定義だけを検証するコマンドです。[^task-validate]

```shell
mise tasks validate
```

```text
✓ All 4 task(s) validated successfully
```

依存packageをインストールし、ローカルで確認します。

```shell
mise run setup
mise run check > /tmp/mise-run-check.log 2>&1
```

どちらのコマンドも成功しました。`mise run check`のlogから、Ruff、pytest、Prettierの結果を`grep`で取り出します。

```shell
grep -E 'All checks passed!|[0-9]+ passed in|All matched files use Prettier code style!' /tmp/mise-run-check.log
```

```text
[lint] All checks passed!
[test] 1 passed in 0.14s
[lint] All matched files use Prettier code style!
```

Ruff、pytest、Prettierを個別に覚えなくても、`mise run check`でまとめて確認できました。失敗した場合は、保存したlog全体を確認します。

## GitHub Actionsのworkflowへ同じtaskを書く

GitHub Actionsでは、`jdx/mise-action`でmiseとtoolを準備した後、ローカルと同じtaskを呼び出すworkflowを書きます。

```yaml
name: check

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  check:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    env:
      MISE_TASK_OUTPUT: prefix
    steps:
      - uses: actions/checkout@d23441a48e516b6c34aea4fa41551a30e30af803 # v6
      - uses: jdx/mise-action@c2a87611a18de5b3828c5652fe268e992400cb5c # v4.3.0
      - run: mise tasks validate
      - run: mise run setup
      - run: mise run check
```

`jdx/mise-action`の`version`は指定していないため、CIでは実行時点の最新releaseが使われます。`min_version`は、このrepositoryが対応するmiseの最低versionを示すもので、mise自体を固定する設定ではありません。miseは外部のregistryやbackendと連携するため、公式ドキュメントでは特定versionへの固定より最低versionの指定が推奨されています。[^mise-action][^mise-min-version]

この記事を書いた時点の`jdx/mise-action` v4.3.0は、`mise.lock`を作業directoryまたは親directoryから探します。検出した場合、内部で`mise install --locked`を実行します。

まっさらなGitHub Actions runnerではユーザーのglobal設定を考えなくてよいため、この動作をそのまま使います。詳しい条件はmise-actionのREADMEを確認してください。[^mise-action]

`MISE_TASK_OUTPUT=prefix`を指定すると、先ほどの出力にある`[lint]`や`[test]`のようにtask名が付きます。並行して動くtaskの出力も区別しやすくなります。

このworkflowはactionlintで構文を確認しました。ただし、先に書いたとおり、実際のGitHub Actions runnerではまだ実行していません。

### GitHub Actionsのworkflowを生成するコマンド

miseにはGitHub Actionsのworkflowを生成するコマンドもあります。

```shell
mise generate github-action --task=check
```

mise 2026.8.14で生成すると、`MISE_EXPERIMENTAL: true`と`jdx/mise-action@v3`を含むworkflowが出力されました。この記事の例ではexperimentalな機能を使っておらず、mise-actionの最新major versionはv4だったため、そのまま採用していません。

生成されたYAMLはひな型として確認し、actionのversion、権限、timeout、実際に必要なtaskを見直す必要があります。現在の生成処理は公式リポジトリでも確認できます。[^generate-github-action]

## 個人ごとの値を追加する場合

今回のFastAPIアプリケーションには、個人ごとに変わる設定やsecretはありません。`mise.toml`の`[env]`にはuvが使うPythonのパスだけを書き、個人ごとに変わる値は入れていません。

あとから外部APIを使う機能を追加し、開発用serverの起動時にtokenが必要になった場合は、taskへ`required`と`redact`を指定できます。次の構文は`mise tasks validate`で確認しましたが、今回のアプリケーションでは実行していません。

```toml
[tasks.dev]
run = "uv run --locked uvicorn app:app --reload"
env = { EXTERNAL_API_TOKEN = { required = "mise.local.tomlまたは実行環境でtokenを設定してください", redact = true } }
```

先に示したrepositoryの`.gitignore`には、`mise.local.toml`と`mise.local.lock`を記載しています。これらをGitの管理対象から外したうえで、各開発者が自分の`mise.local.toml`へ値を書きます。

```toml
[env]
EXTERNAL_API_TOKEN = "各自のtoken"
```

`required`は値を保存する機能ではありません。環境変数や、あとから読み込まれる設定に値がなければ、指定したメッセージを表示して処理を中止します。CIではGitHub ActionsのSecretsなどから環境変数を渡します。miseの環境変数設定については公式ドキュメントを参照してください。[^environments]

`redact = true`を指定すると、miseが整形するtaskの出力では値が伏せられます。ただし、secretを暗号化して保存する機能ではなく、raw形式のtaskが直接出力する内容までは隠しません。secret managerやCIのsecret管理を置き換えるものとは考えないほうがよいです。

secretの扱いは確認することが多いため、別の記事でもう少し詳しく試す予定です。

## 依存packageのインストールは明示的に実行する

miseには、directoryへ入ったときやtoolをインストールした後にコマンドを実行するhookがあります。`mise run setup`をhookから自動実行することも考えられますが、今回は明示的なtaskとして残しました。

`enter`や`cd`のhookは、shellでmiseをactivateしていないと実行されません。ここでいうactivateは、`mise activate`が出力するshell用の設定を読み込み、directoryの移動に合わせてmiseが環境変数やhookを更新できる状態にすることです。[^activate]

今回の手順ではhookを使わないため、次のactivate設定は必要ありません。hookを使う場合は、使用しているshellに合わせて設定します。

macOSのデフォルトであるzshでは、次の行を`~/.zshrc`へ追加します。

```zsh
eval "$(mise activate zsh)"
```

今回の実機確認で使ったfishでは、`~/.config/fish/config.fish`へ次の行を追加します。

```fish
mise activate fish | source
```

`postinstall`はactivateしなくても動きますが、toolの追加がない`mise install`でも実行されます。[^hooks]

repositoryへ入っただけで依存packageのインストールが始まるより、READMEに次の2コマンドを書いておくほうが、自分には分かりやすいです。

```shell
mise install
mise run setup
```

hookを使う場合も、案内の表示や軽い確認から始め、時間のかかる処理を入れる前に実行条件を確認したほうがよさそうです。

## versionを更新するとき

toolの新しいversionを確認する場合は、最初にdry-runできます。

```shell
mise lock --bump --dry-run --json
```

手元で確認した時点では、更新候補がなかったため空の配列が返りました。

```text
[]
```

更新候補がある場合の出力は、公式ドキュメントに次のサンプルが掲載されています。これは今回のrepositoryで実行した結果ではありません。Node.js 22.14.0を記録したlockfileに対し、22.15.0が候補になった場合の例です。[^mise-lock]

```json
[
  {
    "name": "node",
    "backend": "core:node",
    "lockfile": "~/src/myproj/mise.lock",
    "old_versions": ["22.14.0"],
    "new_versions": ["22.15.0"]
  }
]
```

`old_versions`が現在のlockfileに記録されたversion、`new_versions`が更新候補です。`--dry-run`を付けているため、このコマンドでは`mise.lock`を書き換えません。

更新候補がある場合は内容を確認し、`mise.lock`を更新します。

```shell
mise lock --bump
```

Pythonの依存packageは、`uv add`や`uv lock --upgrade`で更新します。Prettierは`pnpm add -D prettier@latest`など、pnpmのコマンドを使います。

Pull Requestでは、変更した設定と対応するlockfileを一緒に確認します。

- `mise.toml`と`mise.lock`
- `pyproject.toml`と`uv.lock`
- `package.json`と`pnpm-lock.yaml`

## 記事の検証時だけ設定した環境変数

:::message
この節の環境変数は、記事の内容を検証するためにだけ設定したものです。通常の初期セットアップでは設定しません。
<!-- textlint-disable-next-line ja-technical-writing/ja-no-mixed-period -->
:::

新しくcloneしたrepositoryでも、普段使っているmiseのglobal設定やcacheは参照されます。手元のglobal設定や既存のcacheに頼らず初期セットアップできることを確認するため、検証時はmise、uv、pnpmなどの保存先を一時directoryへ変更しました。

各コマンドの前に`/usr/bin/env`を付け、環境変数を渡しています。実際のパスには`mktemp -d`で作成した一時directoryを使いました。次の例では、読みやすいように`/tmp/mise-team-verification`へ置き換えています。

```shell
/usr/bin/env \
  MISE_CONFIG_DIR=/tmp/mise-team-verification/mise-config \
  MISE_DATA_DIR=/tmp/mise-team-verification/mise-data \
  MISE_CACHE_DIR=/tmp/mise-team-verification/mise-cache \
  MISE_STATE_DIR=/tmp/mise-team-verification/mise-state \
  MISE_NODE_DEFAULT_PACKAGES_FILE=/dev/null \
  MISE_PYTHON_DEFAULT_PACKAGES_FILE=/dev/null \
  XDG_CONFIG_HOME=/tmp/mise-team-verification/xdg-config \
  XDG_CACHE_HOME=/tmp/mise-team-verification/xdg-cache \
  XDG_DATA_HOME=/tmp/mise-team-verification/xdg-data \
  XDG_STATE_HOME=/tmp/mise-team-verification/xdg-state \
  UV_CACHE_DIR=/tmp/mise-team-verification/uv-cache \
  NPM_CONFIG_USERCONFIG=/dev/null \
  NPM_CONFIG_STORE_DIR=/tmp/mise-team-verification/pnpm-store \
  GIT_CONFIG_GLOBAL=/dev/null \
  GIT_CONFIG_NOSYSTEM=1 \
  mise install
```

同じ環境変数を`mise trust`、`mise tasks validate`、`mise run setup`、`mise run check`にも渡しました。普段使っているmiseのglobal設定、default packageの設定、cache、pnpmのstore、Gitのglobal設定を検証結果から除くためです。repositoryは、これらの一時directoryとは別の場所へcloneしました。

## まとめ

今回は、Pythonの依存packageをuv、Prettierをpnpmで管理するプロジェクトを例に、repositoryへ何を置き、clone後とCIでどのコマンドを実行するかを確認しました。

repositoryをcloneした後は、`mise install`、`mise run setup`、`mise run check`の順に実行します。GitHub Actionsのworkflowからも同じtaskを呼び出せば、Ruff、pytest、PrettierのコマンドをYAMLへ重ねて書かずに済みます。今回はactionlintによる構文確認だけです。GitHub Actions runnerでの実行は確認していません。

`mise.lock`、`uv.lock`、`pnpm-lock.yaml`は、同じものを固定しているわけではありません。toolのversionと配布物、Pythonの依存package、Prettierとその依存packageをそれぞれ記録しています。この違いを前の記事で確認したbackendと合わせて見ると、どのファイルを更新すればよいのか判断しやすくなりました。

次は、taskどうしの依存関係、並列実行、ファイル監視など、miseをタスク実行基盤として使う場合をもう少し詳しく確認する予定です。

[^mise-lock]: [Lockfiles | mise-en-place](https://mise.jdx.dev/dev-tools/mise-lock.html)
[^uv-sync]: [Locking and syncing | uv](https://docs.astral.sh/uv/concepts/projects/sync/)
[^pnpm-install]: [pnpm install | pnpm](https://pnpm.io/cli/install)
[^pnpm-settings]: [Settings: managePackageManagerVersions | pnpm 10.x](https://pnpm.io/10.x/settings#managepackagemanagerversions)
[^task-configuration]: [Task Configuration | mise-en-place](https://mise.jdx.dev/tasks/task-configuration.html)
[^mise-node]: [Node | mise-en-place](https://mise.jdx.dev/lang/node.html)
[^mise-python]: [Python: mise & uv | mise-en-place](https://mise.jdx.dev/lang/python.html#mise-uv)
[^mise-getting-started]: [Getting Started | mise-en-place](https://mise.jdx.dev/getting-started.html)
[^trust]: [mise trust | mise-en-place](https://mise.jdx.dev/cli/trust.html)
[^activate]: [mise activate | mise-en-place](https://mise.jdx.dev/cli/activate.html)
[^task-validate]: [mise tasks validate | mise-en-place](https://mise.jdx.dev/cli/tasks/validate.html)
[^mise-action]: [jdx/mise-action](https://github.com/jdx/mise-action/tree/c2a87611a18de5b3828c5652fe268e992400cb5c)
[^mise-min-version]: [Minimum mise version | mise v2026.8.14](https://github.com/jdx/mise/blob/v2026.8.14/docs/configuration.md#L314-L336)
[^generate-github-action]: [generate/github_action.rs | jdx/mise v2026.8.14](https://github.com/jdx/mise/blob/v2026.8.14/src/cli/generate/github_action.rs)
[^environments]: [Environments | mise-en-place](https://mise.jdx.dev/environments/)
[^hooks]: [Hooks | mise-en-place](https://mise.jdx.dev/hooks.html)
