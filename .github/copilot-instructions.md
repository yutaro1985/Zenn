# Zenn コンテンツ用リポジトリ — AI エージェント向けガイド

このリポジトリはZenn記事/本の原稿管理用です。エージェントは以下の流れ・規約に従って作業してください。一般論ではなく、このリポジトリ固有のパターンをまとめています。

## 構成と役割
- `articles/`: 記事のMarkdown原稿を配置。ファイル名はスラッグ（例: `cdktf-for-usual-terraform-users.md`）。
- `images/<slug>/`: 記事ごとの画像ディレクトリ。記事からは `/images/<slug>/...` で参照。
- `books/`: Zenn本の原稿（未使用の場合あり）。
- `.textlintrc`: 日本語向け校正設定（`prh`, `preset-ja-*`, `spellcheck-tech-word`）。
- `.github/workflows/rules/WEB+DB_PRESS.yml`: `prh` で参照する用字用語ルール集。
- `.github/workflows/textlint.yml`: PR時に`textlint`を`reviewdog`で実行するCI。
- `docs/article-writing-style.md`: 記事の表現、構成、レビューで指摘された言い換えをまとめたガイド。
- `mise.toml`: `Node.js`/`pnpm`のバージョン管理。
- `package.json`: `zenn-cli` と `textlint` 関連パッケージを管理（npm scriptsは最小）。
- `AGENTS.md`: Codex向けの同等ガイド。内容を更新する場合は、必要に応じてこちらとの整合性も保つ。

## 典型ワークフロー
- 新規ブランチ: 命名は任意（例: `add/20251204_advent_calendar` など）。
- `Node.js`/`pnpm`は`mise.toml`の指定を前提とし、必要に応じて`mise install`を実行。
- 記事作成: `pnpm exec zenn new:article --title "タイトル" --slug "my-article" --type tech`
- 画像配置: `mkdir -p images/my-article` に画像を保存し、本文から `/images/my-article/xxx.png` で参照。
- ローカルプレビュー: `pnpm exec zenn preview` または `pnpm run preview` を実行し `http://localhost:8000` を確認。
- 校正（lint）: `pnpm exec textlint -f stylish "articles/**/*.md"` または `pnpm run lint`／自動修正は `--fix` を付与。
- レビュー後、`main` へPR。公開はZenn側の同期に従う。

## フロントマター規約（実例ベース）
- 基本例（`articles/cdktf-for-usual-terraform-users.md` ほか）
  ```md
  ---
  title: "タイトル"
  emoji: "🧐"
  type: "tech" # tech / idea
  topics: ["aws","apigateway"]
  published: true
  # 予約公開する場合のみ
  # published_at: 2025-12-04 07:00
  ---
  ```
- 予約公開する記事は `published: true` とともに `published_at` を設定（未設定なら即時公開）。
- 下書きは `published: false` を使用。

## 画像と参照パターン
- 記事スラッグと同名のディレクトリを `images/` 直下に作成（例: `images/cdktf-for-usual-terraform-users/`）。
- 本文からの参照例: `![代替テキスト](/images/cdktf-for-usual-terraform-users/cdktf-image.png)`。
- 相対ではなくルート起点（`/images/...`）で参照するのが既存記事の実例。

## ライティングスタイル
- 表現と構成の詳細は`docs/article-writing-style.md`を参照。
- 既存記事の文体・構成・思考過程を踏襲して作成。
- 例: `articles/connect-cloud9-via-remote-ssh.md`
- 例: `articles/introduction_of_mise.md`
- 例: `articles/cdktf-for-usual-terraform-users.md`
- 構成の基本:「はじめに」で背景と狙い→用語・前提→手順→ハマりどころ/補足→まとめ。
- アドベントカレンダー等は`:::message`ブロックで明示し、必要に応じて`@[card](URL)`を利用。
- 参考情報や補足には脚注（`[^1]`）を活用し、末尾に脚注本文を配置。
- コードやコマンドはフェンス付きコードブロック＋言語指定（`shell`, `ts`, `toml` 等）。
- コマンド例は原則macOS前提。bash、zsh、fishで同じように実行できるコマンドは`shell`、shell固有の構文は実際のshell名を指定する。必要な場合だけshell別の例を併記する。
- 実機確認に使ったshellは本文へ記載する。fishで実行する複数行の入力例ではheredocを避け、`printf`/`echo`で代替する。
- JavaScript系のローカル実行は`npx`より`pnpm exec`を優先し、依存関係を変更するときは`npm`実行で不要な`package-lock.json`を生成しない。
- 用字用語は`.textlintrc`と`WEB+DB_PRESS.yml`に準拠。lint指摘を尊重して修正。
  - ただし、従った結果日本語として不自然な表現になるときは従わないものとする。

## 校正（textlint）
- 設定ファイル: `.textlintrc`。`prh` のルールは `.github/workflows/rules/WEB+DB_PRESS.yml` を参照。
- 実行例:
  - 全体: `pnpm exec textlint -f stylish "articles/**/*.md"` または `pnpm run lint`
  - 1ファイル: `pnpm exec textlint -f stylish articles/<slug>.md`
  - 自動修正: `pnpm exec textlint --fix "articles/**/*.md"` または `pnpm run lint:fix`
- 記事PRでは`.github/workflows/textlint.yml`により`articles/`配下へ`textlint`が走るため、PR作成前に対象記事のlintを通しておく。
- 記事以外のMarkdownを編集した場合も、可能なら対象ファイルへ個別に`textlint`を実行する。

## Zenn CLI の利用
- 依存関係は `package.json` に定義済み（`zenn-cli`）。`pnpm exec zenn ...` でローカル実行。
- よく使うコマンド:
  - 新規記事: `pnpm exec zenn new:article --title "..." --slug "..." --type tech`
  - プレビュー: `pnpm exec zenn preview` または `pnpm run preview`

### npm scripts（任意）
- `pnpm run preview`: `zenn preview`
- `pnpm run lint`: `textlint -f stylish "articles/**/*.md"`
- `pnpm run lint:fix`: `textlint --fix "articles/**/*.md"`

## リポジトリの前提・補足
- 既存READMEは最小。運用上のコマンドは本ファイルの記載を参照。
- 文章スタイルは日本語の用字用語ルールに準拠する設定。lint結果を尊重。
- 例示ファイル: `articles/cdktf-for-usual-terraform-users.md`（画像参照やフロントマターの好例）。
- shellに依存しないコマンド例とshell固有の構文を、コードフェンスの言語指定で区別する。

---
不明点や追加したい運用があればお知らせください。
例: 固定のnpm scripts、CIでのtextlint実行、画像の最適化方針など。
実運用に合わせて本ガイドを拡張します。
