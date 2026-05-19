# AGENTS.md

このリポジトリはZenn記事・本の原稿を管理するためのものです。Codexは以下の方針に従って作業してください。

## リポジトリ構成

- `articles/`: Zenn記事のMarkdown原稿。
- `images/<slug>/`: 記事ごとの画像置き場。本文からは `/images/<slug>/...` で参照する。
- `books/`: Zenn本の原稿。
- `.textlintrc`: 日本語校正の設定。
- `.github/workflows/rules/WEB+DB_PRESS.yml`: `prh` 用の用字用語ルール。
- `.github/workflows/textlint.yml`: PR時に`textlint`を`reviewdog`で実行するCI。
- `mise.toml`: `Node.js`/`pnpm`のバージョン管理。
- `package.json`: `Zenn CLI`と`textlint`の依存関係・`npm scripts`。
- `.github/copilot-instructions.md`: Copilot向けの同等ガイド。内容を更新する場合は、必要に応じてこちらとの整合性も保つ。

## 基本方針

- このリポジトリではコードよりもMarkdown原稿の編集が主目的。
- 既存記事の文体・構成に寄せて編集する。
- 変更は依頼範囲に限定し、無関係な表現調整は広げない。
- `:::message` や `:::message alert` は意味がある限り維持する。
- コマンド例はmacOSとfishを前提にする。
- fish前提のため、複数行の入力例では`heredoc`を避け、必要に応じて`printf`や`echo`で代替する。
- JavaScript系のローカル実行は `npx` より `pnpm exec` を優先する。
- `Node.js`/`pnpm`は`mise.toml`の指定を前提にし、必要に応じて`mise install`を案内する。
- 依存関係を変更する場合は`pnpm`を使い、`npm`実行により`package-lock.json`を不用意に更新しない。

## 記事の書き方

- 基本構成は「はじめに」→前提・背景→手順→補足やハマりどころ→まとめ。
- 参考情報や補足は必要に応じて脚注 `[^1]` を使う。
- コマンドやコードはフェンス付きコードブロックを使い、言語名を付ける。
- 画像は `images/<slug>/` に置き、本文では `/images/<slug>/file.png` の形式で参照する。
- 相対パスで画像参照を書かない。

## フロントマター

既存記事に合わせ、最低限次の形式を基準にする。

```md
---
title: "タイトル"
emoji: "🧐"
type: "tech" # tech / idea
topics: ["aws", "terraform"]
published: true
# 予約公開する場合のみ
# published_at: 2025-12-04 07:00
---
```

- 下書きは `published: false`。
- 予約公開する場合は `published: true` と `published_at` を併用する。

## 典型作業

- 新規記事作成: `pnpm exec zenn new:article --title "タイトル" --slug "my-article" --type tech`
- プレビュー: `pnpm exec zenn preview`
- npm scriptでのプレビュー: `pnpm run preview`
- 1ファイルlint: `pnpm exec textlint -f stylish articles/<slug>.md`
- 全体lint: `pnpm exec textlint -f stylish "articles/**/*.md"`
- 自動修正: `pnpm exec textlint --fix "articles/**/*.md"`
- npm scriptでのlint: `pnpm run lint` / `pnpm run lint:fix`

## 編集時の注意

- textlintの指摘は尊重するが、不自然な日本語になるなら機械的に従わない。
- 既存の用語・表記ゆれは周辺記事に合わせる。
- 画像追加が必要なら、記事スラッグと同名のディレクトリを `images/` 配下に作る。
- 既存のfrontmatterや公開設定は、依頼がない限り変更しない。

## 検証

- Markdownを編集したら、可能なら対象ファイルに対して `pnpm exec textlint -f stylish ...` を実行して確認する。
- PRでは`.github/workflows/textlint.yml`により`textlint`が走るため、PR作成前に対象ファイルlintを通しておく。
- 記事全体に影響する変更をした場合のみ、必要に応じて全体lintやプレビューを行う。
