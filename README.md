# claude-workflows

Shared GitHub Actions workflow: mention `@claude` on an issue and Claude implements it, runs the tests, takes Playwright screenshots, and opens a PR with them.

## One-time setup (central repo)

1. Create a repo named `claude-workflows` and push this folder to it.
2. If the repo is **private**: Settings → Actions → General → *Access* → allow access from repositories owned by your account. Without this, other repos can't call the workflow.

## Per project repo

1. Install the Claude GitHub App on the repo (https://github.com/apps/claude), or run `/install-github-app` from Claude Code inside the repo.
2. Add ONE auth secret:
   - Pro/Max subscription: run `claude setup-token`, then `gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo OWNER/REPO`
   - API key: `gh secret set ANTHROPIC_API_KEY --repo OWNER/REPO`
3. Copy `templates/claude.yml` and `templates/claude-review.yml` to `.github/workflows/`, replace `YOUR_USER`, and set the inputs (install/test/start commands, app URL).
4. Copy `templates/ISSUE_TEMPLATE/` to `.github/ISSUE_TEMPLATE/` so every issue has an *Acceptance criteria* section.
5. For releases: copy `templates/release.yml` to `.github/workflows/`, and `templates/release-please-config.json` + `templates/.release-please-manifest.json` to the repo root (set `release-type` and the current version). In the repo settings, enable *Allow GitHub Actions to create and approve pull requests* and use **squash merging** with the PR title as the commit message.
6. Commit, open an issue from the template, and write `@claude` in it.

## Review workflow

`claude-review.yml` runs on every non-draft PR, and manually from the Actions tab (enter a PR number). It is independent of the implementer:

- New session with no implementer context; it only sees the issue, the diff and the repo.
- Read-only token: it can comment but cannot push, and its comments can't re-trigger `@claude`.
- Checks each acceptance criterion as MET / NOT MET / CANNOT VERIFY with evidence, takes its own screenshots (in the run's artifacts), and posts one comment with a PASS/FAIL verdict.
- Runs your test command separately as an objective check. The job fails if the tests fail or the verdict is FAIL, so you can make it a required check in branch protection.
- Optional `model` input lets the reviewer use a different model than the implementer.

The verdict is a comment plus a pass/fail check rather than a GitHub "approve", because a PR can't be approved by its own author's identity.

To set the secret on several repos at once, loop over their names with `gh secret set ... --repo`.

## Inputs

| Input | Default | Purpose |
|---|---|---|
| `node_version` | `20` | Node version |
| `install_command` | `npm ci` | Installs dependencies |
| `start_command` | `npm run dev` | Starts the app for screenshots |
| `app_url` | `http://localhost:3000` | Where the app answers |
| `extra_instructions` | empty | Project-specific notes for Claude (avoid single quotes) |
| `max_turns` | `60` | Caps how long Claude works per run |

## Notes

- Screenshots are committed under `pr-screenshots/issue-N/` so they render in the PR; they are also uploaded as a workflow artifact for 14 days.
- Runs on GitHub-hosted runners; private repos use your free Actions minutes.
- Only people with write access should be able to trigger it; keep the project repos private or restrict who can comment.

## Release workflow

`release.yml` keeps one open release PR on each repo, updated after every merge to `main`, with the changelog and the next version. Merging the release PR creates the tag and the GitHub Release, and attaches the build if `build_command` and `artifact_path` are set.

- Version bumps come from squash-merged PR titles: `feat:` → minor, `fix:` → patch, `feat!:` → major (while below 1.0.0, breaking changes bump minor). `chore:`/`docs:`/`test:` don't trigger a release.
- Claude is instructed to title its PRs this way.
- The reviewer skips release PRs automatically.
