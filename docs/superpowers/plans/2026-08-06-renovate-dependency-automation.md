# Implementation Plan: Renovate Dependency Automation

**Goal:** Introduce weekly Renovate dependency updates, immediate manually merged vulnerability PRs, and required dependency-change verification while standardizing the repository on pnpm.

**Architecture:** Store Renovate policy in the repository, use pnpm as the sole lockfile authority, and run dependency installation plus Zenn CLI smoke validation on every pull request and main push. Run Dependency Review only at the job level for pull requests so the workflow itself is never omitted by path filtering. Keep App installation and required-check configuration as explicit post-merge GitHub operations.

**Tech Stack:** Renovate, GitHub Actions, Node.js 24.13.0, pnpm 9.15.9, Zenn CLI, Dependency Review Action v5, reviewdog v0.21.0, GitHub CLI.

**Proposed plan path:** docs/superpowers/plans/2026-08-06-renovate-dependency-automation.md

## Global constraints

- Execute from the repository root in the isolated implementation worktree.
- Use branch codex/renovate-dependency-automation.
- Do not modify files outside the current worktree or unrelated user changes.
- Keep pnpm-lock.yaml unchanged; delete only the stale package-lock.json.
- Do not enable automerge.
- Do not add workflow-level path filters.
- Do not make pnpm audit or pnpm run lint required gates.
- Do not close Dependabot PRs #34 or #36.
- Do not disable Dependabot security updates.
- Do not install Apps or change repository settings from the implementation branch.

## File map

- Create renovate.json: repository Renovate policy.
- Delete package-lock.json: remove the stale npm lockfile.
- Create .github/workflows/dependency-verification.yml: install, smoke, and dependency-diff verification.
- Modify README.md: document pnpm ownership, Renovate behavior, and CI boundaries.
- Preserve package.json, pnpm-lock.yaml, and mise.toml.
- Modify .github/workflows/textlint.yml so it uses pnpm after package-lock.json removal and separates lint execution from review posting permissions.

## Task 1: Confirm the isolated execution boundary

Files: read-only repository inspection.

- [ ] Verify the worktree and branch:

    test "$(git branch --show-current)" = "codex/renovate-dependency-automation"
    git status --short --branch
    git worktree list

- [ ] Confirm the approved inputs exist:

    test -f docs/superpowers/specs/2026-08-06-renovate-dependency-automation-design.md
    test -f package-lock.json
    test -f pnpm-lock.yaml

- [ ] Fetch and inspect origin/main. If the branch is behind, stop and reconcile it before implementation, then rerun every validation:

    git fetch origin main
    git log --oneline HEAD..origin/main

No implementation commit is created for this task.

## Task 2: Add the Renovate policy

Files:

- Create: renovate.json

- [ ] Create renovate.json with:

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

- [ ] Validate JSON syntax:

    mise x node@24.13.0 -- node -e 'JSON.parse(require("fs").readFileSync("renovate.json", "utf8")); console.log("renovate.json: valid JSON")'

- [ ] Confirm the validator command exposed by the Renovate package, then run strict validation:

    mise x pnpm@9.15.9 -- pnpm --package=renovate@latest dlx renovate-config-validator --help
    mise x pnpm@9.15.9 -- pnpm --package=renovate@latest dlx renovate-config-validator --strict --no-global renovate.json

If this pnpm dlx form is unsupported by the installed pnpm, inspect pnpm help dlx and use the Renovate package's renovate-config-validator binary; do not substitute an unrelated validator.

- [ ] Commit only the policy:

    git add renovate.json
    git diff --cached --check
    git commit -m "chore(deps): configure Renovate"

## Task 3: Make pnpm the sole lockfile authority

Files:

- Delete: package-lock.json
- Preserve: pnpm-lock.yaml

- [ ] Confirm no pending pnpm-lock.yaml change:

    git diff --exit-code HEAD -- pnpm-lock.yaml

- [ ] Delete the stale npm lockfile:

    git rm package-lock.json

- [ ] Confirm the staged change contains only that deletion:

    git diff --cached --name-status
    git diff --cached --check

Expected name-status: D package-lock.json.

- [ ] Commit the migration separately:

    git commit -m "chore(deps): remove stale npm lockfile"

Rollback: restore package-lock.json from the pre-change commit only if a real npm consumer is identified. Do not regenerate it opportunistically.

## Task 4: Add dependency verification

Files:

- Create: .github/workflows/dependency-verification.yml

- [ ] Create the workflow with this behavior:

    name: Dependency Verification

    on:
      pull_request:
    push:
      branches:
        - main

    concurrency:
      group: dependency-verification-${{ github.event.pull_request.number || github.ref }}
      cancel-in-progress: true

    permissions:
      contents: read

    jobs:
      install-and-smoke:
        name: dependency-install-and-smoke
        runs-on: ubuntu-latest
        steps:
          - name: Check out repository
            uses: actions/checkout@d23441a48e516b6c34aea4fa41551a30e30af803 # v6
            with:
              persist-credentials: false

          - name: Set up pnpm
            uses: pnpm/action-setup@0977fd99725f1db4007ccb2928dbb4e90d06cc86 # v6
            with:
              version: 9.15.9

          - name: Set up Node.js
            uses: actions/setup-node@820762786026740c76f36085b0efc47a31fe5020 # v7
            with:
              node-version: 24.13.0
              cache: pnpm
              cache-dependency-path: pnpm-lock.yaml

          - name: Install dependencies
            run: pnpm install --frozen-lockfile

          - name: Smoke-test Zenn CLI
            run: pnpm exec zenn --version

      dependency-review:
        name: dependency-review
        if: github.event_name == 'pull_request'
        runs-on: ubuntu-latest
        steps:
          - name: Check out repository
            uses: actions/checkout@d23441a48e516b6c34aea4fa41551a30e30af803 # v6
            with:
              persist-credentials: false

          - name: Review dependency changes
            uses: actions/dependency-review-action@a1d282b36b6f3519aa1f3fc636f609c47dddb294 # v5
            with:
              fail-on-severity: high
              fail-on-scopes: runtime, development, unknown
              license-check: false

- [ ] Parse the YAML:

    ruby -e 'require "yaml"; YAML.parse_file(ARGV.fetch(0)); puts "workflow: valid YAML"' .github/workflows/dependency-verification.yml

- [ ] Run GitHub Actions semantic linting when actionlint is available:

    mise registry actionlint
    mise x actionlint@latest -- actionlint .github/workflows/dependency-verification.yml

If the registry lookup does not provide actionlint, record that validation as unavailable and rely on YAML parsing plus the implementation pull request's real Actions run.

- [ ] Verify the trigger boundary statically:

    rg -n '^  pull_request:|^  push:|^    branches:|^      - main$' .github/workflows/dependency-verification.yml
    rg -n "if: github.event_name == 'pull_request'" .github/workflows/dependency-verification.yml
    ! rg -n 'paths:|paths-ignore:' .github/workflows/dependency-verification.yml

The implementation pull request validates the pull request path: both jobs must run. The first merged main push validates the push path: the workflow and dependency-install-and-smoke run, while dependency-review is skipped by its job-level condition. Do not move the condition to the workflow trigger.

- [x] Update the existing .github/workflows/textlint.yml so package-lock.json removal cannot route it through npm and PR write access is isolated:

  - Check out with actions/checkout SHA d23441a48e516b6c34aea4fa41551a30e30af803 (v6), set persist-credentials to false, and preserve submodules.
  - Set up pnpm 9.15.9 with pnpm/action-setup SHA 0977fd99725f1db4007ccb2928dbb4e90d06cc86 (v6).
  - Set up Node.js 24.13.0 with actions/setup-node SHA 820762786026740c76f36085b0efc47a31fe5020 (v7), then run pnpm install --frozen-lockfile.
  - Run textlint with checkstyle output and upload it with actions/upload-artifact SHA ea165f8d65b6e75b540449e92b4886f43607fa02 (v4.6.2).
  - Use a separate review-posting job with no checkout that downloads the artifact using actions/download-artifact SHA d3f86a106a0bac45b974a628896c90dbdf5c8093 (v4.3.0), sets up reviewdog with reviewdog/action-setup SHA d8edfce3dd5e1ec6978745e801f9c50b5ef80252 (v1.4.0), and grants that job the only pull-requests: write permission.

- [x] Commit the workflow and security boundary changes:

    git add .github/workflows/dependency-verification.yml .github/workflows/textlint.yml
    git diff --cached --check
    git commit -m "ci: verify dependency updates"

## Task 5: Document dependency-update operations

Files:

- Modify: README.md

- [ ] Add a section after the setup instructions covering:

    ## 依存関係の更新

    このリポジトリでは pnpm と pnpm-lock.yaml を依存管理の正とします。

    通常の依存更新と脆弱性修正PRは Renovate が作成します。Renovateを動作させるには、リポジトリの設定ファイルとは別に Mend Renovate App のインストールが必要です。脆弱性修正PRを含め、自動マージは行わず、差分とCI結果を確認して手動でマージします。

    Dependency Verification workflowは、すべてのPull Requestと main へのpushで依存インストールとZenn CLIのsmoke testを実行します。Pull RequestではDependency Reviewも実行し、新たに追加されるHigh以上の脆弱性を検出します。

    既存の pnpm audit alertとtextlintエラーは、この導入PRの成功条件には含めません。

- [ ] Verify terminology and scope:

    rg -n 'pnpm-lock.yaml|Mend Renovate App|自動マージ|Dependency Verification|pnpm audit' README.md
    git diff --check

- [ ] Commit documentation separately:

    git add README.md
    git diff --cached --check
    git commit -m "docs: document dependency updates"

## Task 6: Run integrated validation

- [ ] Install from the unchanged pnpm lockfile:

    mise x pnpm@9.15.9 -- pnpm install --frozen-lockfile

Expected: exit 0 and no lockfile rewrite.

- [ ] Run the Zenn CLI smoke test:

    mise x pnpm@9.15.9 -- pnpm exec zenn --version

Expected from the current lockfile: 0.1.162.

- [ ] Confirm the canonical lockfile stayed unchanged:

    git diff --exit-code HEAD -- pnpm-lock.yaml

- [ ] Repeat configuration and workflow validation:

    mise x node@24.13.0 -- node -e 'JSON.parse(require("fs").readFileSync("renovate.json", "utf8"))'
    mise x pnpm@9.15.9 -- pnpm --package=renovate@latest dlx renovate-config-validator --strict --no-global renovate.json
    ruby -e 'require "yaml"; YAML.parse_file(ARGV.fetch(0))' .github/workflows/dependency-verification.yml
    git diff --check

- [ ] Confirm the final file inventory. The only implementation surfaces should be renovate.json, deleted package-lock.json, both dependency workflows, and README.md, plus the previously approved design and plan documentation:

    git diff --name-status origin/main...HEAD
    git status --short

- [ ] Record known non-gates without treating them as regressions:

  - pnpm run lint: skipped as an acceptance gate; baseline is 572 existing errors.
  - pnpm audit --audit-level=high: informational only; baseline is 21 existing vulnerabilities.
  - package.json test script: not run because it intentionally exits with "no test specified".

Do not claim the existing vulnerabilities or lint errors are resolved.

## Task 7: Publish and converge review

- [ ] Re-fetch Issue #45 immediately before publication:

    gh issue view 45 --repo yutaro1985/Zenn --json number,title,state,body,labels,comments

Confirm it remains open and the implementation still matches its acceptance criteria.

- [ ] Verify commit boundaries and branch state:

    git log --oneline origin/main..HEAD
    git status --short --branch

- [ ] Push the branch:

    git push -u origin codex/renovate-dependency-automation

- [ ] Create a Ready-for-Review pull request, not a draft:

    gh pr create \
      --repo yutaro1985/Zenn \
      --base main \
      --head codex/renovate-dependency-automation \
      --title "Renovateで依存更新とセキュリティ修正を自動化する" \
      --body $'Closes #45\n\n- Renovateの週次更新と即時脆弱性PRを設定\n- pnpmを唯一のlockfile管理へ統一\n- 依存インストール、Zenn CLI、Dependency Reviewの検証を追加\n- 自動マージは無効'

- [ ] Exercise the pull-request-only behavior through the real pull request:

    pr_number="$(gh pr view --repo yutaro1985/Zenn --json number --jq .number)"
    gh pr checks "$pr_number" --repo yutaro1985/Zenn --watch

Confirm dependency-install-and-smoke and dependency-review both complete on the pull request.

- [ ] Request an independent Sol/high review with a bounded brief covering origin/main...HEAD, Issue #45, security, migration, workflow triggers, and external-operation boundaries. Accept the result as Sol only when runtime metadata confirms gpt-5.6-sol/high.

- [ ] Fetch unresolved review threads:

    gh api graphql \
      -f owner='yutaro1985' \
      -f name='Zenn' \
      -F number="$pr_number" \
      -f query='query($owner:String!,$name:String!,$number:Int!){repository(owner:$owner,name:$name){pullRequest(number:$number){reviewDecision reviewThreads(first:100){nodes{id isResolved comments(first:50){nodes{id body path line url author{login}}}}}}}}'

- [ ] For each actionable thread: verify the claim, implement the narrow fix, rerun affected and integrated checks, commit, push, reply with evidence, then resolve the thread. Do not resolve unsupported or unfixed comments.

- [ ] Repeat until all are true:

  - Sol returns no must_fix.
  - GitHub has zero unresolved actionable review threads.
  - Pull request checks are green on the current head SHA.
  - Local HEAD, upstream SHA, and pull request headRefOid match.
  - Issue #45 remains open until merge is confirmed.

## Task 8: Perform post-merge external operations

These operations are outside the implementation branch and require explicit confirmation before each external write.

- [ ] Confirm the pull request actually merged:

    gh pr view "$pr_number" --repo yutaro1985/Zenn --json state,mergedAt,mergeCommit,closingIssuesReferences

- [ ] Install Mend Renovate App for yutaro1985/Zenn only.

- [ ] Capture evidence from the App installation settings that it can read Dependabot alerts. Where supported, inspect the installation through:

    gh api repos/yutaro1985/Zenn/installation --jq '{id,app_slug,target_type,permissions}'

Treat an unsupported or unauthorized API response as unavailable evidence, not as confirmation.

- [ ] Observe the first Renovate run and confirm an onboarding pull request or Dependency Dashboard appears:

    gh pr list --repo yutaro1985/Zenn --state all --json number,title,author,headRefName,state,url --jq '.[] | select(.author.login | ascii_downcase | contains("renovate"))'
    gh issue list --repo yutaro1985/Zenn --state all --json number,title,author,state,url --jq '.[] | select(.title | ascii_downcase | contains("dependency dashboard"))'

- [ ] Using the exact check name observed on the implementation pull request, configure dependency-review or the agreed Dependency Verification checks as required through branch protection or a repository ruleset. Re-fetch the settings afterward and retain the returned rule/protection evidence.

- [ ] Confirm the merged main push ran Dependency Verification, with the install/smoke job completed and the pull-request-only Dependency Review job skipped.

- [ ] Verify Dependabot PRs #34 and #36 were not automatically closed:

    gh pr view 34 --repo yutaro1985/Zenn --json number,state,title,url
    gh pr view 36 --repo yutaro1985/Zenn --json number,state,title,url

- [ ] Keep Dependabot security updates enabled while Renovate remains limited to the npm manager.

## Residual risks and containment

- Dependabot overlap: duplicate dependency PRs may temporarily occur. Do not auto-close #34/#36; evaluate them after the canonical lockfile change is visible on main.
- GitHub Actions coverage: Renovate does not update Actions under this configuration. Retain Dependabot security updates.
- Existing vulnerability baseline: the 21 existing audit findings remain. Track remediation separately and do not weaken Dependency Review.
- Action integrity: all action references changed by this implementation are pinned to verified full commit SHAs with release-tag comments. Future action updates require verifying the new tag target before changing the SHA.
- Lifecycle scripts: frozen installation executes dependency lifecycle scripts on an ephemeral GitHub-hosted runner. Keep permissions at contents: read; if a dependency behaves unexpectedly, block its pull request and investigate before merging.
- Lockfile deletion: if an actual npm-only consumer is discovered, contain by reverting the lockfile-removal commit and revisiting the package-manager decision.
- Renovate malfunction or PR flood: disable or suspend the App installation and revert the Renovate configuration commit. Automerge remains disabled, limiting impact.
- Workflow failure after merge: revert the workflow commit or remove its required status only with explicit approval and recorded evidence; do not bypass the gate silently.

## Sol execution-plan metadata

- status: implementation-complete-pr-under-review
- requested_route: gpt-5.6-sol/high
- observed_route: gpt-5.6-sol/high
- route_confirmation: confirmed by delegated task runtime metadata
- Implementation status: renovate.json, both dependency workflows, README, lockfile cleanup, permission separation, and validation evidence are recorded in this PR.
- Pending verification: CodeRabbit review convergence, merged-main behavior, Mend Renovate App permissions and first run, and required-check enforcement remain to be confirmed.
- Follow-up: finish PR review convergence, merge after human approval, then perform the documented external operations and record live evidence.
