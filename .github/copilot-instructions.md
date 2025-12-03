# Zenn コンテンツ用リポジトリ — AI エージェント向けガイド

このリポジトリは Zenn 記事/本の原稿管理用です。エージェントは以下の流れ・規約に従って作業してください。一般論ではなく、このリポジトリ固有のパターンをまとめています。

## 構成と役割
- `articles/`: 記事の Markdown 原稿を配置。ファイル名はスラッグ（例: `cdktf-for-usual-terraform-users.md`）。
- `images/<slug>/`: 記事ごとの画像ディレクトリ。記事からは `/images/<slug>/...` で参照。
- `books/`: Zenn 本の原稿（未使用の場合あり）。
- `.textlintrc`: 日本語向け校正設定（`prh`, `preset-ja-*`, `spellcheck-tech-word`）。
- `.github/workflows/rules/WEB+DB_PRESS.yml`: `prh` で参照する用字用語ルール集。
- `package.json`: `zenn-cli` と `textlint` 関連パッケージを管理（npm scripts は最小）。

## 典型ワークフロー
- 新規ブランチ: 命名は任意（例: `add/20251204_advent_calendar` など）。
- 記事作成: `npx zenn new:article --title "タイトル" --slug "my-article" --type tech`
- 画像配置: `mkdir -p images/my-article` に画像を保存し、本文から `/images/my-article/xxx.png` で参照。
- ローカルプレビュー: `npx zenn preview` を実行し `http://localhost:8000` を確認。
- 校正（lint）: `npx textlint -f stylish "articles/**/*.md"`／自動修正は `--fix` を付与。
- レビュー後、`main` へ PR。公開は Zenn 側の同期に従う。

## フロントマター規約（実例に基づく）
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
- 予約公開する記事は `published: true` と共に `published_at` を設定（未設定なら即時公開）。
- 下書きは `published: false` を使用。

## 画像と参照パターン
- 記事スラッグと同名のディレクトリを `images/` 直下に作成（例: `images/cdktf-for-usual-terraform-users/`）。
- 本文からの参照例: `![代替テキスト](/images/cdktf-for-usual-terraform-users/cdktf-image.png)`。
- 相対ではなくルート起点（`/images/...`）で参照するのが既存記事の実例。

## ライティングスタイル
- 既存記事の文体・構成・思考過程を踏襲して作成（例: `articles/connect-cloud9-via-remote-ssh.md`, `articles/introduction_of_mise.md`, `articles/cdktf-for-usual-terraform-users.md`）。
- 構成の基本: 「はじめに」で背景と狙い→用語・前提→手順→ハマりどころ/補足→まとめ。
- アドベントカレンダー等は`:::message`ブロックで明示し、必要に応じて`@[card](URL)`を利用。
- 参考情報や補足には脚注（`[^1]`）を活用し、末尾に脚注本文を配置。
- コードやコマンドはフェンス付きコードブロック＋言語指定（`bash`, `ts`, `toml` 等）。
- コマンド例は原則macOS + fish前提。heredocは避け、`printf`/`echo`で代替。
- 用字用語は`.textlintrc`と`WEB+DB_PRESS.yml`に準拠。lint指摘を尊重して修正。
  - ただし、従った結果日本語として不自然な表現になるときは従わないものとする。

## 校正（textlint）
- 設定ファイル: `.textlintrc`。`prh` のルールは `.github/workflows/rules/WEB+DB_PRESS.yml` を参照。
- 実行例:
  - 全体: `npx textlint -f stylish "articles/**/*.md"`
  - 1ファイル: `npx textlint -f stylish articles/<slug>.md`
  - 自動修正: `npx textlint --fix "articles/**/*.md"`
- 注意: ルールファイルのパスを変更する場合は `.textlintrc` の `rulePaths` も更新が必要。

## Zenn CLI の利用
- 依存関係は `package.json` に定義済み（`zenn-cli`）。`npx zenn ...` でローカル実行。
- よく使うコマンド:
  - 新規記事: `npx zenn new:article --title "..." --slug "..." --type tech`
  - プレビュー: `npx zenn preview`

### npm scripts（任意）
- `npm run preview`: `zenn preview`
- `npm run lint`: `textlint -f stylish "articles/**/*.md"`
- `npm run lint:fix`: `textlint --fix "articles/**/*.md"`

## リポジトリの前提・補足
- 既存 README は最小。運用上のコマンドは本ファイルの記載を参照。
- 文章スタイルは日本語の用字用語ルール（WEB+DB PRESS ベース）に準拠する設定。lint 結果を尊重。
- 例示ファイル: `articles/cdktf-for-usual-terraform-users.md`（画像参照やフロントマターの好例）。
-.macOS + fish を前提にコマンド例を記述（heredoc 非推奨）。

---
不明点や追加したい運用（例: 固定の npm scripts、CI での textlint 実行、画像最適化方針など）があればお知らせください。実運用に合わせて本ガイドを拡張します。