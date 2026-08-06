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

## 依存関係の更新

このリポジトリでは `pnpm` と `pnpm-lock.yaml` を依存管理の正とします。

通常の依存更新と脆弱性修正PRは Renovate が作成します。Renovateを動作させるには、リポジトリの設定ファイルとは別に Mend Renovate App のインストールが必要です。脆弱性修正PRを含め、自動マージは行わず、差分とCI結果を確認して手動でマージします。

`Dependency Verification` workflowは、すべてのPull Requestと `main` へのpushで依存インストールとZenn CLIのsmoke testを実行します。Pull RequestではDependency Reviewも実行し、新たに追加されるHigh以上の脆弱性を検出します。

既存の `pnpm audit` alertとtextlintエラーは、この導入PRの成功条件には含めません。

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
