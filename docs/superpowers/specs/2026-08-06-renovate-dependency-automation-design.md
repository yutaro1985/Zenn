# Renovateによる依存更新自動化 — Design

**日付:** 2026-08-06
**対象Issue:** [#45](https://github.com/yutaro1985/Zenn/issues/45)

## Goal

pnpmを正とする依存管理へ整理し、Renovateで通常の依存更新と脆弱性修正PRを自動作成できる状態にする。依存更新PRでは、lockfileの整合性とZenn CLIの動作を検証し、PRに新しいHigh以上の脆弱性が持ち込まれないことを確認する。

脆弱性PRのマージは自動化せず、人間が差分とCI結果を確認して判断する。

## Context and evidence

- リポジトリはGitHub上のpublic repository `yutaro1985/Zenn`で、既定ブランチは`main`。
- `package.json`と`pnpm-lock.yaml`が現在のpnpm運用を構成している。
- 古い`package-lock.json`も追跡されており、同一依存について両方のlockfileにDependabot alertが生成されている。
- GitHub Dependabot security updatesは有効で、既存のDependabot PR #34/#36はopenのまま残す。
- pnpm 9.15.9で`pnpm install --frozen-lockfile`は成功した。
- `pnpm exec zenn --version`は成功した。
- `pnpm run lint`は既存記事の572件のエラーで失敗するため、今回の依存更新検証ゲートには採用しない。
- `pnpm audit --audit-level=high`は既存mainで21件（high 16件、moderate 5件）を検出するため、導入直後の必須ゲートには採用しない。
- `main`には現在ブランチ保護が設定されていない。Dependency Reviewの必須化はリポジトリ外部設定として扱う。

## Design decisions

### 1. Renovate configuration

ルートの`renovate.json`をRenovateのリポジトリ設定とする。

- `config:recommended`を基礎にする。
- 通常の更新は`Asia/Tokyo`の週次スケジュールにする。
- Renovateの`npm` managerだけを有効にする。Renovateでは`package.json`と`pnpm-lock.yaml`をこのmanagerで扱う。
- 脆弱性修正は通常スケジュールを待たず、即時PRを作成する。
- 通常更新・脆弱性修正とも自動マージは無効にする。
- 脆弱性修正PRには既存の`security`ラベルを付与する。
- GitHub Actions自体の更新はRenovateの対象にせず、当面は既存のDependabot security updatesを維持してカバレッジを失わない。

設定の意図を次のように固定する。

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended", "schedule:weekly"],
  "enabledManagers": ["npm"],
  "timezone": "Asia/Tokyo",
  "automerge": false,
  "vulnerabilityAlerts": {
    "enabled": true,
    "schedule": [],
    "prCreation": "immediate",
    "automerge": false,
    "labels": ["security"]
  }
}
```

### 2. Canonical lockfile

pnpmを唯一のパッケージマネージャとして扱い、`package-lock.json`を削除する。これにより、npmとpnpmの解決結果が分岐する状態と、GitHub alertの重複を解消する。

既存のDependabot PR #34/#36は自動クローズしない。`package-lock.json`削除後にGitHub上で不要になったことを確認し、別の外部操作として必要なら手動で扱う。

### 3. Dependency verification workflow

`.github/workflows/dependency-verification.yml`を追加する。対象外PRでrequired checkがPendingになることを避けるため、workflow-levelのpath filterは使わず、すべてのPull Requestと`main`へのpushで実行する。

同一PRまたは同一refに対する古い実行を停止するため、workflow-levelで次のconcurrencyを設定する。

```yaml
concurrency:
  group: dependency-verification-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true
```

権限は次の最小設定にする。

```yaml
permissions:
  contents: read
```

パッケージ検証jobは次の手順を実行する。

1. `actions/checkout`を検証済みのfull commit SHA `d23441a48e516b6c34aea4fa41551a30e30af803`（`v6`）でcheckoutし、`persist-credentials: false`を指定する。
2. `pnpm/action-setup`を検証済みのfull commit SHA `0977fd99725f1db4007ccb2928dbb4e90d06cc86`（`v6`）で実行し、pnpm 9.15.9をセットアップする。
3. `actions/setup-node`を検証済みのfull commit SHA `820762786026740c76f36085b0efc47a31fe5020`（`v7`）で実行し、Node.js 24.13.0をセットアップする。`pnpm-lock.yaml`をキーにpnpm cacheを有効化する。
4. `pnpm install --frozen-lockfile`を実行する。
5. `pnpm exec zenn --version`を実行する。

Dependency Review jobはPull Requestでのみ実行し、先に同じcheckoutを行ったうえで、`actions/dependency-review-action`を検証済みのfull commit SHA `a1d282b36b6f3519aa1f3fc636f609c47dddb294`（`v5`）で実行し、次を指定する。

```yaml
fail-on-severity: high
fail-on-scopes: runtime, development, unknown
license-check: false
```

これは既存の脆弱性を一括解消するジョブではなく、PR差分による新規のHigh以上脆弱性混入を防ぐゲートである。ライセンス方針はこの導入の対象外とするため、Dependency Reviewのライセンス検査は無効化する。`pnpm audit`は現行baselineが解消されるまで必須jobにしない。

既存の`.github/workflows/textlint.yml`も`package-lock.json`削除後にnpm経路へ戻らないよう、Node.js 24.13.0・pnpm 9.15.9・`pnpm install --frozen-lockfile`を使用する。PRコードをcheckout・実行するtextlint jobは`contents: read`だけにし、履歴を使ってPR base SHAから実行中のSHAまでを差分としてreviewdogでcheckstyleレポートをフィルタした`rdjson` artifactへ変換して保存する。別のreview-posting jobはPRコードをcheckout・実行せず、artifactだけを`-filter-mode=nofilter`で処理してreviewdogでPRレビューを投稿するため、そのjobだけに`pull-requests: write`を付与する。追加するartifactとreviewdogのaction参照も検証済みのfull commit SHAに固定し、リリースタグをコメントで残す。

### 4. Documentation

`README.md`に次を短く追記する。

- pnpmがcanonical package managerであること。
- Renovate Appのインストールがリポジトリ設定とは別に必要であること。
- Renovateの脆弱性PRは自動作成されるが、マージは手動であること。
- Dependency Verificationがlockfile整合性とZenn CLI smoke testを行うこと。
- `pnpm audit`の既存alertは今回の導入PRの成功条件ではないこと。

## External operations

リポジトリファイルの変更だけではRenovateは実行されない。設定PRをmainへマージした後、次の外部操作を行う。

1. Mend Renovate Appを`yutaro1985/Zenn`だけにインストールする。
2. AppがDependabot alertsのread権限を持つことを確認する。
3. Renovateの初回onboardingまたはDependency Dashboardが想定どおり作成されることを確認する。
4. `dependency-review` jobをブランチ保護またはrulesetでrequired checkにする。
5. RenovateがGitHub Actionsを更新対象に含めない間は、Dependabot security updatesを無効化しない。

これらはGitHub repository settingsまたはApp installationへの外部書き込みであり、設定ファイルのPRとは分けて報告する。

ActionsはこのPRで検証済みのfull commit SHAに固定し、Dependabotが更新を検知できるよう各SHAの後ろにリリースタグコメントを残す。将来のaction更新では、タグの変更先を確認してSHAを更新する。

## Error handling and residual risk

- 既存textlintエラーは依存更新PRの失敗原因にしない。textlint workflowはpnpmのfrozen install後に差分フィルタ済みのrdjsonレポートを生成し、記事変更を含むPRにはartifactを介してreviewdogのレビュー警告を残す。
- `pnpm audit`の既存21件はこのIssueでは解消しない。Renovate/Dependabotが作成するPRを別途評価する。
- `package-lock.json`削除により、既存Dependabot PR #34/#36が不要またはconflictになる可能性があるが、自動クローズしない。
- `main`のブランチ保護が未設定のため、Dependency Review jobを追加しただけではマージを強制できない。
- Renovate Appが未インストールの場合、設定ファイルをマージしてもPRは作成されない。

## Verification strategy

ローカルでは次を実行する。

```bash
mise x pnpm@9.15.9 -- pnpm install --frozen-lockfile
mise x pnpm@9.15.9 -- pnpm exec zenn --version
```

設定検証では次を確認する。

- `renovate-config-validator --strict`が成功する。
- workflow YAMLが構文上有効である。
- `git diff --check`が成功する。
- `package-lock.json`が削除され、意図しないlockfile変更がない。
- Solの独立レビュー指摘が受け入れ条件と実装計画に反映されている。

## Non-goals

- 既存572件のtextlintエラーの解消。
- 既存21件のaudit alertの一括修正。
- RenovateまたはDependabot PRの自動マージ。
- GitHubブランチ保護・rulesetのリポジトリファイルからの変更。
- 既存Dependabot PRの自動クローズ。
