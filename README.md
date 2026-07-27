I am an evil, malicious imp!!! I commit from a sockpuppet account.

# Four-eyes-principle-breach

Test repo for showcasing the solution to a security problem of breaching the four-eyes principle in this time of AI and bot PRs.

## The problem

With classic branch protection, a PR opened by a bot (Dependabot, an AI agent, a CI machine user) plus a single human approval satisfies "require 1 approval" — but only **one** human ever looked at the change. The four-eyes principle is silently breached.

## The solution

The workflow [`.github/workflows/bot-pr-two-approvals.yml`](.github/workflows/bot-pr-two-approvals.yml) publishes a commit status `bot-pr/two-human-approvals`:

- **PR authored by a human with write access** → status is `success` immediately; native branch protection rules apply as usual.
- **PR from an untrusted author** — a bot, a machine-user account, or **any human account without write/admin access** → status stays `pending` until **2 humans with write access** approve the current head commit, where neither approver authored or committed any commit in the PR. Approvals from accounts without write/admin permission are ignored.
- **Untrusted-author PR with an unverified commit** → status is `failure`: an unsigned commit can spoof its author email and defeat the contributor detection, so such PRs are rejected outright.

Treating write-less humans the same as bots closes the **sockpuppet attack**: a collaborator creates a throwaway account, pushes a commit from it via a fork PR, then approves the PR from their main account. The author isn't a bot, so a bot-only check would wave the PR through with that single self-serving approval. Here the throwaway author is untrusted, so the PR needs 2 approvals from write-access non-committers — the attacker's own approval counts as at most 1 of them, forcing a genuine independent review.

The status is marked as **required** in branch protection, so bot PRs cannot merge without two independent human reviews.

## Required branch protection settings (on `main`)

The workflow is only half of the mechanism — without these settings the demo does not protect anything:

1. **Require status checks to pass before merging** and add `bot-pr/two-human-approvals` as a required check.
2. **Require a pull request before merging** with at least 1 approval (covers human PRs).
3. **Dismiss stale pull request approvals when new commits are pushed.**
4. **Require review from Code Owners** — together with [`.github/CODEOWNERS`](.github/CODEOWNERS) this guards the workflow file itself: the `pull_request_review` trigger runs the workflow version from the PR merge commit, so an unreviewed workflow edit could otherwise fake the required status.
5. **Do not allow bypassing the above settings** (include administrators).
6. Recommended: **Require signed commits** — the workflow already fails bot PRs containing unverified commits, but the branch protection setting extends the same guarantee to human PRs as defense in depth.

## Test checklist

| # | Scenario | Expected status |
|---|----------|-----------------|
| 1 | Bot opens a PR | `pending` |
| 2 | First human approves | still `pending` |
| 3 | Second human approves | `success` |
| 4 | New commit pushed to the PR | back to `pending` (stale approvals don't count) |
| 5 | A human who pushed a commit to the PR approves | their approval is **not** counted |
| 6 | Human **with write access** opens a PR | `success` immediately, standard rules apply |
| 7 | A user **without write access** approves | their approval is **not** counted |
| 8 | Untrusted-author PR contains an unverified (unsigned) commit | `failure` |
| 9 | Human **without write access** opens a PR (fork / sockpuppet) | `pending`, same rules as a bot PR |
| 10 | Sockpuppet PR approved from its owner's main account | still `pending` — that approval is only 1 of the required 2 |

The intentionally outdated `lodash` pin in [`package.json`](package.json) plus [`.github/dependabot.yml`](.github/dependabot.yml) generate real bot PRs to test with — authored by `dependabot[bot]` with signed commits, they exercise the full bot path of the workflow. The workflow itself is content-agnostic (it only looks at the PR author, committers and reviews), so no demo source code is needed.

## Known limitations

- On PRs **from forks**, `pull_request_review` runs get a read-only token and cannot update the status. This fails closed — the status stays `pending` and the PR cannot merge — but it also means approvals on fork PRs only take effect on the next `pull_request_target` event: a push, reopen, ready-for-review, or adding a label (the `labeled` trigger exists exactly for this manual nudge).
- Machine-user accounts (regular accounts used as bots) are not auto-detected — add their logins to `MACHINE_USER_BOTS` in the workflow.
- Any workflow in this repo running with `statuses: write` could set the same status context; for maximum robustness publish the status from a GitHub App with its own token instead of `GITHUB_TOKEN`. That would also solve the fork-PR limitation above, making the setup suitable for fully open-source repos.
