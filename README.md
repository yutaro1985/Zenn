# Zenn

## 前提

- バージョン管理は `mise` を使用
- パッケージマネージャは `pnpm` を使用

## セットアップ

1. `mise` で Node.js / pnpm を用意

   - 例: `mise.toml` に Node.js / pnpm を指定
   - 具体的なバージョンはチーム方針に合わせて調整

   ```bash
   mise install
   ```

2. 依存インストール

   - `pnpm install`

## よく使うコマンド

### プレビュー

- `pnpm run preview`

### 校正 (textlint)

- `pnpm run lint`
- `pnpm run lint:fix`

### 記事作成

- `pnpm exec zenn new:article --title "..." --slug "..." --type tech`

## 一時実行したい場合

`pnpm dlx` で `npx` 相当の動作が可能です。

- `pnpm dlx zenn preview`
