## 1. Why did you choose your scanning tool?

I chose Gitleaks (CLI, installed manually) as the secret scanning tool.

The main reason is that the official GitHub Action (`gitleaks/gitleaks-action@v2`) requires a license key for organization-owned repos. By installing the CLI directly, I avoid that dependency entirely.

Another reason is output control. The CLI lets me generate a JSON report (`--report-format json`) which I can parse in the next step to build a custom PR comment. With the action, the output is abstracted away and harder to customize.

Gitleaks also has wide community adoption (25k+ stars on GitHub) and ships with 170+ built-in rules that cover AWS keys, Stripe tokens, database passwords, and many other common secret patterns -- which is exactly what this repo needs.

## 2. What does each step in your workflow do?

The workflow has 5 steps that run in order:

**Checkout code** -- Uses `actions/checkout@v4` with `fetch-depth: 0` to clone the full repo history. The runner starts as an empty machine, so this step is required before we can scan anything.

**Install Gitleaks** -- Downloads the Gitleaks v8.30.0 binary from GitHub Releases and places it in `/usr/local/bin`. The version is defined as a workflow-level `env` variable so it's easy to update in one place.

**Run Gitleaks scan** -- Runs `gitleaks detect --no-git --source . --report-format json --report-path gitleaks-report.json --exit-code 0`. The `--no-git` flag scans files as they are in the working directory. `--exit-code 0` is important -- it tells Gitleaks not to fail the step even if secrets are found. Instead, I check the JSON report myself and set a `leaks_found` output variable to `true` or `false`. This way the next step (posting a comment) still runs before the workflow fails.

**Post PR comment with findings** -- Only runs when `leaks_found` is `true`. Uses `actions/github-script@v7` to read the JSON report, group findings by file, and build a markdown comment with a table showing the file, rule name, line number, and a redacted match. It also includes step-by-step fix instructions. The comment is posted using the GitHub REST API (`issues.createComment`).

**Fail if secrets detected** -- The last step. Runs `exit 1` when secrets are found, which makes the workflow fail. This step is intentionally placed last so the comment gets posted before the check turns red.

## 3. How does the workflow block the merge?

When the last step runs `exit 1`, GitHub marks the check as failed. On the PR page, this shows up as "Some checks were not successful" and the merge button indicates that merging is blocked.

For a stricter setup, you would also configure branch protection rules on `main` and require this check to pass before merging. That way the merge button is fully disabled, not just showing a warning.

## 4. How does it post the PR comment?

The comment step uses `actions/github-script@v7`, which provides a JavaScript environment with the Octokit API client already available.

The script does three things:
1. Reads `gitleaks-report.json` and parses it.
2. Groups findings by file path and builds a markdown table for each file.
3. Calls `github.rest.issues.createComment()` to post the comment on the PR.

This requires `pull-requests: write` permission, which is declared at the workflow level. Without it, the API call would return a 403 error.

The comment includes fix instructions telling the developer to remove hardcoded secrets, use environment variables instead, rotate any exposed credentials, and clean the git history.

## 5. What would you improve or add if you had more time?

- Scan only the files changed in the PR diff instead of the entire repo, to speed things up on larger codebases.
- Upload findings as a SARIF report to GitHub Code Scanning, so secrets show up as inline annotations on the diff.
- Cache the Gitleaks binary with `actions/cache` to skip downloading it on every run.
- Update the existing bot comment instead of posting a new one each time the PR is pushed to.
- Add a Slack or email notification to alert the security team when secrets are found.
- Set up Gitleaks as a pre-commit hook so developers catch secrets locally before pushing.
