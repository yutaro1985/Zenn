# Mise Backend Article Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Issue #44の連載第1回として、mise v2026.8.10のbackendがツールをどこから取得し、実際に何を使ってインストールするのかを説明する日本語記事の下書きを作る。

**Architecture:** 既存2記事の基本説明を再掲せず、`tool`、`registry`、`backend`、実際のinstallerを最初に分ける。その後、代表backendの比較と隔離fixtureの実測結果を使い、従来の`tools`/`tasks`と現在のbackend・lockfileを組み合わせた運用例、責任分界、安定度を1本の記事へまとめる。

**Tech Stack:** Zenn Markdown、mise 2026.8.10、TOML、fish、pnpm、textlint、Zenn CLI

**Spec:** GitHub Issue #44の合意済み執筆計画 https://github.com/yutaro1985/Zenn/issues/44#issuecomment-5273116051

## Global Constraints

- 今回作る記事はbackend編の1本だけとし、次の記事には着手しない。
- 新規ファイルは`articles/mise-backends-installers.md`とし、frontmatterは`published: false`にする。
- 記事タイトルは「miseのbackendは実際に何を使ってインストールしているのか」とする。
- 2026-08-23にmacOS arm64上のmise 2026.8.10で確認したことを明記する。
- 公式ドキュメント、jdx/miseのv2026.8.10タグのソース、公式release notes、実機CLI出力を一次情報として使う。
- live docsとv2026.8.10の実装がずれる可能性がある場合は、v2026.8.10のタグ固定ソースと実機結果を優先する。
- 既存の`articles/introduction_of_mise.md`と`articles/mise-beyond-asdf-alternative.md`は変更しない。
- npm backendの既定installerはmise内蔵aubeであり、Node/npm/aube CLIなしでもインストールできることを書く。ただし、導入したCLIがNodeを実行時に必要とする場合はNodeを別途`[tools]`へ置く必要がある。
- pipx backendはuvがPATHにあれば`uv tool install`、なければ`pipx install`を使う。Pythonアプリケーションの依存管理とは分ける。
- cargo backendは外部cargoが必須であり、cargo-binstallの有無と条件で経路が変わる。常にsource buildまたは常にbinary downloadとは書かない。
- aqua backendはaqua CLIを呼ばず、miseに組み込まれた実装と同梱registry snapshotを使う。github backendもmise自身がrelease assetを処理する。
- ubiはdeprecated、asdf pluginはlegacy、pkgxはexperimentalとして扱う。vfoxは複雑なinstallerやenv exportが必要なprivate/custom pluginの候補とし、plugin自体のreviewとtrustが必要であること、公式registryへの新規登録ではaqua/githubが優先されることを書く。
- `mise.lock`のURL/checksum/provenance能力を全backendへ一般化しない。npm、pipx、cargo、asdfはv2026.8.10ではversionのみである。
- `minimum_release_age`、`trust`、Safe mode、install scriptは安全性・再現性編で詳しく扱うため、本記事ではbackend選択に必要な最低限に留める。
- コマンド例はmacOSとfishを前提にし、複数行のshell入力でheredocを使わない。
- 日本語は既存記事の一人称と温度感へ寄せ、命題型H2、過剰な二項対比、主体のない抽象文、同じ長さの段落、AI偏愛語を避ける。
- 記事内の外部リンクは公式miseドキュメント、jdx/miseのタグ固定ソース、既存のZenn記事を中心にする。
- 対象記事へ`mise exec -- pnpm exec textlint -f stylish articles/mise-backends-installers.md`を実行し、exit 0を得る。
- frontmatterに`published: false`が1行だけあることを確認し、Zenn CLIでMarkdownを解析して記事一覧へ認識されることを確認する。
- ローカルコミットまで行い、push、PR作成、Issueの状態変更はしない。

---

### Task 1: backend編の下書きを作成する

**Files:**
- Create: `articles/mise-backends-installers.md`
- Reference: `articles/introduction_of_mise.md`
- Reference: `articles/mise-beyond-asdf-alternative.md`
- Test: `articles/mise-backends-installers.md`

**Interfaces:**
- Consumes: Issue #44の合意済み計画、`/private/tmp/mise-backend-impl-research.md`、`/private/tmp/mise-backend-status-research.md`、`/private/tmp/mise-backend-editorial-research.md`、`/private/tmp/mise-backend-fixture-report.md`
- Produces: Zennで下書きとして認識され、textlintを通る`articles/mise-backends-installers.md`

- [ ] **Step 1: 一次情報と既存記事との差分を確認する**

  4つの調査メモと既存2記事を読む。特に既存記事の「npm backendのデフォルトはnpm」「pipx backendはuvxを使う」という2026-03時点の記述を、新記事で現行挙動として引き継がない。既存記事自体は変更しない。

- [ ] **Step 2: frontmatterと導入を書く**

  次のfrontmatterを使う。

  ```md
  ---
  title: "miseのbackendは実際に何を使ってインストールしているのか"
  emoji: "📦"
  type: "tech" # tech: 技術記事 / idea: アイデア
  topics: ["mise", "環境構築", "devtools"]
  published: false
  ---
  ```

  冒頭で既存2記事をZenn cardとして示し、基本的な`mise install`/`mise use`、`tools`/`env`/`tasks`の説明は既存記事へ委ねる。検証条件として2026-08-23、macOS arm64、mise 2026.8.10を明記し、仕様と実機確認を分けて書く。

  続けて「backendを見る理由」という短い節を置く。従来はruntimeごとのversion managerと`npm install -g`、`pipx install`、`cargo install`のような個別コマンドを組み合わせていたこと、以前のmiseではcore toolやasdf pluginを同じ`[tools]`へ寄せるところが中心だったこと、現在はregistryと複数backend、`mise.lock`により取得経路まで選べるようになったことを順番に書く。この節は歴史の網羅ではなく、同じ`[tools]`の一行でも実際の処理が違うという本題への接続に留める。

- [ ] **Step 3: 最初に4つの用語を分ける**

  `tool`、`registry`、`backend`、`installerまたは外部コマンド`を小さな表で区別する。`mise registry ripgrep --json --security`の実機結果を短く載せ、裸の`ripgrep`が同梱registryの優先順位で`aqua:BurntSushi/ripgrep`へ解決されること、完全指定ならその対応表を迂回できることを書く。

- [ ] **Step 4: 代表backendを実装方式で比較する**

  次の4群に分け、取得元、実際のinstaller、外部依存、lockfileの情報量を表にする。

  - core / aqua / github: mise側がprovider固有のdownload・展開・検証を多く担当する。
  - npm / pipx / cargo: package ecosystemを使うが、backend名と実コマンドは一致するとは限らない。
  - vfox / asdf: pluginがversion解決やinstall手順を提供する。vfoxはmise内蔵Lua、asdfはBash plugin scriptを実行し、どちらもplugin sourceをreviewしてtrustする境界が残る。
  - ubi / pkgx: ubiはdeprecated、pkgxはexperimentalとして新規の主例から外す。

  coreはproviderごとに実装が違うため、Nodeの`SHASUMS256.txt`確認を例にしつつ全core toolへ一般化しない。aquaはaqua CLIを呼ばず、githubはGitHub Releases APIとassetを使うことを書く。

- [ ] **Step 5: fixtureで確認した3つの具体例を書く**

  実測結果を中心に、次の順で書く。

  1. `aqua:BurntSushi/ripgrep@14.1.1`では`mise lock`がmacOS arm64向けURLとSHA-256を記録し、`mise install --locked`がchecksum確認後に展開した。
  2. `npm:cowsay@1.6.0`はNode/npm/aube CLIをPATHから外しても内蔵aubeでインストールできたが、実行は`node: not found`で失敗した。`node = "24.13.0"`を従来の`tools`設定へ追加すると動いた。
  3. `pipx:httpie@3.2.4`はpipx CLIなし・uvありのPATHで`uv tool install httpie==3.2.4`を実行した。対照として`cargo:eza@0.18.18`はcargoなしでは外部`cargo install`を起動できずに失敗した。

  長いverbose logは貼らず、判断に必要な数行だけコードブロックへ抜き出す。成功例だけでなく期待どおり失敗した例も残し、backendのinstaller依存と導入後runtime依存を分ける。

- [ ] **Step 6: 従来機能と組み合わせたプロジェクト例を書く**

  次の設定を土台にし、runtime、配布済みCLI、Python CLIを1つの入口へまとめる例を示す。`cowsay`は検証専用なので運用例には残さない。

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

  `tools`で導入経路を選び、`mise.lock`でbackendが対応する範囲を固定し、`tasks`をチーム共通の確認入口にする、という従来機能と追加機能の組み合わせを説明する。`uv`を同じ`[tools]`へ明示することで、`pipx:httpie`が未宣言のglobal uvへ偶然依存しない構成にする。`minimum_release_age`は曖昧なversion解決に効くが、明示pinや全backendの推移依存を守る機能ではないと添える。

  この設定は`/private/tmp/mise-backend-combined-20260823`で別々の空のmise data/cache/state/config領域へ2回適用済みである。`mise lock --platform macos-arm64`はNode 24.19.0、uv 0.12.5、ripgrep 14.1.1、HTTPie 3.2.4を解決し、`mise install --locked`と`mise run tool-versions`は2回ともexit 0だった。実測値とlockfileの差は`/private/tmp/mise-backend-fixture-report.md`を参照し、記事のTOMLをこの検証済み設定から変えない。

- [ ] **Step 7: miseへ寄せる責任と残す責任を書く**

  runtimeと独立CLIはmiseの候補にし、アプリケーション依存は`package.json`/`pnpm-lock.yaml`、`pyproject.toml`/`uv.lock`、`Cargo.toml`/`Cargo.lock`へ残す。たとえばプロジェクト固有のPrettierは`npm:prettier`へ移すことを既定の推奨にせず、pnpmのdevDependencyに残す判断を明記する。

  `mise.lock`が持つ情報を、aqua/github等のartifact lockと、npm/pipx/cargo/asdfのversion lockに分ける。checksum、provenance、plugin script、package lifecycle script、`trust`は防ぐ対象が違うため「lockfileがあれば安全」とまとめない。

- [ ] **Step 8: 安定度とまとめを書く**

  本記事でstableと書く場合は「公式docsにexperimental表記がなく、mise 2026.8.10のCLIで利用できた」という調査上の区分に限定し、将来互換を保証する公式ラベルのように扱わない。最近変わったnpmの内蔵aube、deprecatedのubi、legacyのasdf、experimentalのpkgxを同列に推奨しない。vfoxは複雑なinstallerやenv exportが必要なcustom/private pluginの候補に留め、公式registryへ新規登録する経路としてはaqua/githubが優先されると説明する。

  まとめでは、backendのprefixは取得元の名前だけでなく、実際のinstaller、外部依存、lockfileが保持できる情報を選ぶ、と短く戻す。チーム開発編や安全性編の予告は1〜2文に留める。

- [ ] **Step 9: 日本語を編集する**

  全文を読み返し、次を修正する。

  - 「AではなくB」の連発を減らし、必要な差は具体的に書く。
  - H2を名詞句または作業名にし、見出し直後に同じ主張を言い換えない。
  - 「本質」「思想」「解像度」「設計」「手触り」「重要」といった抽象語を根拠なく使わない。
  - 段落の長さと説明密度を均一にしすぎない。
  - 一次情報に基づく仕様と、自分の判断を同じ断定文へ混ぜない。

  stop-ai-slop-jpの5軸（文体、リズム、主体、語彙、構造）を各10点で自己評価し、合計35点未満なら再編集する。評価と修正点はimplementer reportへ記録し、記事本文には載せない。

- [ ] **Step 10: 対象記事を検証する**

  Run:

  ```bash
  mise exec -- pnpm exec textlint -f stylish articles/mise-backends-installers.md
  ```

  Expected: exit 0、error 0件。

  Run:

  ```bash
  rg -n '^published: false$' articles/mise-backends-installers.md
  ```

  Expected: frontmatter内の1行だけがmatchする。

  Run:

  ```bash
  mise exec -- pnpm exec zenn list:articles
  ```

  Expected: exit 0で、`mise-backends-installers`が記事一覧に現れる。このコマンドはMarkdownの解析と一覧認識を確認するもので、draft状態は直前のfrontmatter確認で判定する。コマンドが現在のZenn CLIに存在しない場合は、`mise exec -- pnpm exec zenn --help`で同等の読み取り専用検証コマンドを確認し、使用したコマンドと結果をreportへ記録する。

- [ ] **Step 11: 差分を自己レビューしてコミットする**

  Run:

  ```bash
  git diff --check
  git diff -- articles/mise-backends-installers.md
  ```

  Expected: whitespace errorなし。既存記事や公開設定の変更なし。

  Commit:

  ```bash
  git add articles/mise-backends-installers.md
  git commit -m "docs(mise): draft backend installer article"
  ```
