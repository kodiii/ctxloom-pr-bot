# ctxloom PR review

A **GitHub Action** that posts a risk-scored summary comment and inline review notes on every pull request. Runs entirely inside your CI — **no hosted service, no LLM calls, no external accounts**, no per-PR cost.

Uses ctxloom's local dependency graph + git overlay to find:

- Which files in the diff have the highest blast radius
- Who has historically owned the high-risk modules (reviewer suggestions)
- Co-change patterns from git history that imports alone wouldn't surface

> **Want LLM-narrated reviews on top?** See **[AI-REVIEWS.md](./AI-REVIEWS.md)** for the optional Claude integration — four specialist agents (security, architecture, testing, performance) running in parallel via Anthropic's official `claude-code-action`, with the ctxloom MCP server providing structural evidence so the LLM never hallucinates the graph. Risk-gated so you only spend tokens on PRs that actually need attention.

---

## Quick start

### Option 1 — `ctxloom install-pr-bot` (one command)

If you have ctxloom installed (`npm install -g ctxloom-pro` — unpinned is fine for local dev; for CI use the exact-version pin shown in the workflow example), the fastest path is:

```bash
cd /path/to/your/repo
ctxloom install-pr-bot
git add .github/workflows/ctxloom-review.yml
git commit -m "ci: enable ctxloom pr-bot"
git push
```

The command writes the workflow file shown below into your repo, with the right defaults (default branch detected from git, action pinned to `@v1`). Pass `--force` to overwrite an existing file or `--ref v1.2.1` to pin to a specific release.

The interactive `ctxloom setup` wizard also offers to install the workflow at the end if you'd rather click through.

### Option 2 — manual

Or just paste this file yourself:

```yaml
# .github/workflows/ctxloom-review.yml
name: ctxloom review

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write
  checks: write

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # needed for git overlay (co-change history)

      - uses: kodiii/ctxloom/apps/pr-bot@v1
```

That's it. The first run builds the dependency graph for your repo (~10–60s for typical projects), subsequent runs reuse the workflow cache.

---

## Cost

| Resource | Who pays | When |
|---|---|---|
| GitHub Actions minutes | The repo owner | Per PR (each run takes ~30–90s on `ubuntu-latest`) |
| Docker image hosting | GitHub (GHCR) | Free, cached per runner |
| LLM tokens | Nobody | This action does not call any LLM |
| Cloud server hosting | Nobody | No hosted service exists |

Free for public repos. For private repos, runs against the standard 2,000 free Actions minutes/month included with GitHub Free / Pro.

---

## Configure (`.ctxloom.yml`)

Optional. Place at your repo root to override defaults:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/kodiii/ctxloom/main/apps/pr-bot/schema/ctxloom.schema.json

risk_threshold: 0.7            # 0-1, comments fire above this
inline_comments: true          # post per-file inline notes
suggested_reviewers: true      # nominate reviewers from git history
check_run: false               # set true to block merge on high risk
excluded_paths: []             # globs the bot ignores
max_inline_per_pr: 10          # cap on inline comments (avoids spam)
```

The published [`schema/ctxloom.schema.json`](schema/ctxloom.schema.json) gives editor autocomplete in VS Code's YAML extension and JetBrains IDEs.

---

## Privacy

- **Your code never leaves your runner.** Analysis happens in the same job that checked out the code.
- The Action calls the GitHub API exclusively — to post comments, read the diff, and (optionally) post a check run. No other network calls except:
  - Optional Sentry crash reports, **only if** you set `SENTRY_DSN` in your workflow's `env:` (off by default).
  - The Docker image is pulled from GHCR on the first run per runner.
- No telemetry of any kind is collected.

---

## What the bot posts

A typical summary comment looks like:

```
🧵 ctxloom review · HIGH risk

3 files changed, max risk score 0.81.

| File | Risk | Impact |
|---|---|---|
| src/auth/session.ts | critical | 14 dependents |
| src/db/migrations/047_add_role.sql | high | 8 dependents |
| README.md | low | 0 dependents |

Suggested reviewers: @alice (touched src/auth/* in 4 of last 10 PRs)
```

Inline comments fire only on files with risk ≥ `risk_threshold`. They reference specific hunks rather than whole files.

---

## Troubleshooting

**The bot posts a summary but no inline comments.**
The Action only posts inline comments on lines that exist in the PR's unified diff. Files in the PR with no content change in the new side (binary files, pure renames, deletions) are skipped — the summary table still lists them, but no inline anchors are possible. This is by design; inline at a line that's not in the diff makes GitHub reject the whole review with `422`.

**The bot says risk scores are based on file count only.**
Means the dependency graph couldn't be built from your checkout. Most common cause: `actions/checkout@v4` without `fetch-depth: 0`. The git overlay needs full history to compute co-change and ownership. Make sure your workflow has:

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0
```

**The action exits with `fatal: detected dubious ownership`.**
You're on an old image tag. Re-pin to `@v1` (or any tag ≥ `v1.2.4`) — the runtime container marks `/github/workspace` as a safe directory globally so this can't happen with the published image.

---

## Self-host the bot as a long-running service (advanced)

Not supported in v1. The Action is the only deployment surface — it's strictly better for distribution and cost, and there's no hosted version to maintain.

If you have a use case that genuinely requires a long-running webhook server (e.g. you want the bot to respond to issue comments outside of PR events), please open an issue describing the workflow.

---

## Development

```bash
# From the monorepo root
npm test -w @ctxloom/pr-bot          # 52 tests, all unit
npm run build -w @ctxloom/pr-bot     # tsup bundle to dist/index.js
```

The Docker image is built and tested in CI ([`pr-bot-ci.yml`](../../.github/workflows/pr-bot-ci.yml)). Releases bake the version of `ctxloom-pro` from the root `package.json` into the bundle as `__CTXLOOM_VERSION__`, so Sentry events from the Action are correlated with the corresponding release tag.

### How the image gets to users

[`pr-bot-publish-image.yml`](../../.github/workflows/pr-bot-publish-image.yml) builds the Dockerfile **with the monorepo root as the Docker build context** (so `COPY packages/core/ ...` resolves) and pushes the resulting image to `ghcr.io/kodiii/ctxloom-pr-bot` on every `v*` tag. The image is tagged with the version (`1.2.2`), the floating major (`v1`), and `latest`.

`action.yml` references `image: 'docker://ghcr.io/kodiii/ctxloom-pr-bot:v1'`, **not** `image: 'Dockerfile'`. That's the standard pattern for Docker actions in a monorepo: GitHub Actions' built-in Dockerfile resolution uses the action's directory as the build context, which can't see into other workspace packages. Pre-building in our own workflow sidesteps that limitation and is also faster for consumers — the runner pulls a finished image (~3s) instead of rebuilding from scratch (~30s).
