# ANSWER.md

## 1. Why did you choose your scanning tool?

I went with Gitleaks CLI (installed manually) instead of `gitleaks/gitleaks-action@v2` for two reasons:

- The official action requires a license key for org-owned repos. Installing the CLI directly avoids that.
- The CLI gives me a JSON report that I can parse to build a custom PR comment with file names, rule types, and fix instructions.

Gitleaks has 25k+ GitHub stars and ships with 170+ built-in detection rules. However, the secrets in this repo use non-standard formats (e.g. `AKIA_FLOWACCOUNT_PROD_2024` instead of a real AWS key pattern), so I added a custom `.gitleaks.toml` config with regex rules tailored to catch them.

## 2. What does each step in your workflow do?

**Checkout code** -- Clones the repo with full history using `actions/checkout@v4` with `fetch-depth: 0`.

**Install Gitleaks** -- Downloads the v8.30.0 binary from GitHub Releases. The version is set as a workflow-level `env` variable so it's easy to update.

**Run Gitleaks scan** -- Runs `gitleaks dir .` with `--config .gitleaks.toml` to scan all files using our custom rules. `--exit-code 0` prevents the step from failing immediately, so the comment step can still run. After the scan, a shell check reads the JSON report and sets `leaks_found=true` or `false` as a step output.

**Post PR comment** -- Only runs when `leaks_found` is `true`. Uses `actions/github-script@v7` to parse the JSON report, group findings by file, and post a markdown comment with a table of findings and step-by-step fix instructions.

**Fail if secrets detected** -- Runs `exit 1` as the last step. Placed last on purpose so the comment gets posted before the workflow fails.

## 3. How does the workflow block the merge?

`exit 1` makes GitHub mark the check as failed. The PR page shows "Some checks were not successful" and the merge button shows a warning.

I also configured a branch protection rule on `main` requiring this check to pass, which fully blocks the merge button when secrets are found.

## 4. How does it post the PR comment?

Uses `actions/github-script@v7` which provides an Octokit API client. The script:
1. Reads and parses `gitleaks-report.json`
2. Groups findings by file and builds a markdown table
3. Calls `github.rest.issues.createComment()` to post it on the PR

This needs `pull-requests: write` permission, declared at the workflow level.

Secret values are truncated to 20 characters in the comment to avoid exposing them in full.

## 5. What would you improve with more time?

- Scan only files changed in the PR instead of the whole repo.
- Upload SARIF report to GitHub Code Scanning for inline annotations.
- Cache the Gitleaks binary to skip downloading on every run.
- Update the existing bot comment instead of posting a new one on each push.
- Add Slack notification for the security team.
- Set up a pre-commit hook to catch secrets before they get pushed.
