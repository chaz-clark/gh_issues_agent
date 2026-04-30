# GitHub Issues Agent Guide

## Agent Instructions
1. Read this file for mission, principles, quickstart, and pitfalls.
2. Parse `gh_issues_agent.json` for structured data: label taxonomy, API patterns, file format spec, and validation.
3. Read `knowledge/agile_sprint.md` to understand the current sprint, what is in progress, and what is next. Always check sprint status before proposing work.
4. Read `knowledge/gh_issues_agent_mission.md` for the full milestone rationale and issue triage philosophy.
5. Do not parse this Markdown for structured data.

---

## Mission

Manage GitHub issues for any repo as a local, file-based workflow. Pull issues down as readable markdown, triage by label, work through fixes and features in priority order, and close issues with a linked commit reference when done. Auto-detects the target repo from the git remote of the directory the tools are run in.

**What it does**: Syncs open GitHub issues + comments into `.github_issues/open/` as markdown files, helps triage and prioritize work by label, and closes issues via the GitHub API when resolved.

**Why it exists**: Keeps issue tracking local and integrated with the development workflow — no browser context-switching, full comment history available while coding, and a clear audit trail of what was fixed and when.

**Who uses it**: The repo maintainer working through reported bugs and feature requests.

---

## Agent Quickstart

1. **Check the sprint**: Read `knowledge/agile_sprint.md` — find the active sprint, identify the next `[ ]` issue in order
2. **Sync issues**: Run `gh_sync.py` to pull all open issues + comments into `.github_issues/open/`
3. **Pick work**: Open the `.md` file for the next issue in the sprint plan, read the full description + comments
4. **Fix it**: Make the code change — do not commit yet
5. **Commit**: After verifying the fix locally — commit with a descriptive message referencing the issue (`Fixes #42`)
6. **Close**: Run `gh_close.py --issue 42 --comment "Fixed in commit abc123."` — posts comment, closes on GitHub, moves file to `closed/`
7. **Update the sprint**: Mark the issue `[x]` in `agile_sprint.md` with the commit hash
8. **Re-sync**: Run `gh_sync.py` to confirm all sprint issues are in `closed/`

---

## File Organization: JSON vs MD

### This Markdown File Contains:
- Mission, principles, and workflow narrative
- Common pitfalls and why they happen
- External system lessons (GitHub API quirks)

### The JSON File Contains:
- `label_taxonomy` — priority order and handling rules per label
- `api_endpoints` — correct request patterns for issues and comments
- `issue_file_format` — the markdown frontmatter spec written by `gh_sync.py`
- `validation` checklists

---

## Key Principles

### 1. Propose Before Execute on Bulk Actions
When closing multiple issues at once or making label changes in bulk, show the full list and wait for confirmation. Single-issue closes can execute immediately.

**Why**: Closing the wrong issue or batch-closing a set that wasn't fully reviewed is hard to undo cleanly — GitHub doesn't have an "undo close."

### 2. Always Include a Commit Reference When Closing
Every `gh_close.py` call should include `--comment` with the commit hash or PR number.

**Why**: Without a link, there's no way to trace what change resolved the issue. GitHub cross-references work automatically when the comment includes `fixes #N` or a direct commit URL.

### 3. Work Bugs Before Features
The `bug` label is priority 1 — resolve all open bugs before picking up enhancements.

**Why**: A toolkit with broken behavior erodes trust faster than a toolkit with fewer features. Users who hit a bug and file it deserve a fast response.

### 4. Batch Enhancements by Topic
Group `enhancement` issues by the file or module they affect before starting work — keep related changes in one pass instead of context-switching across modules.

**Why**: Context switching between tools mid-session costs more than the overhead of grouping. Related enhancements often share a code path and can be addressed in one pass.

### 5. Never Edit Files in `.github_issues/` by Hand
The `open/` and `closed/` files are generated output — treat them as read-only. Edits will be overwritten on the next `gh_sync.py` run.

**Why**: The sync script overwrites files on each run. Local edits create a false record that disappears without warning.

---

## How to Use This Agent

### Prerequisites
- `.env` with `GH_TOKEN` set (or `GH_TOKEN=$(gh auth token)` exported in the shell). `repo` scope for private repos, `public_repo` for public.
- `uv sync` completed (requests and python-dotenv are already in `pyproject.toml`)
- `.github_issues/` folder exists (created on first sync)

### Existing Tooling

| Tool | Purpose | When to use |
|---|---|---|
| `tools/gh_sync.py` | Pull all open issues + comments → `.github_issues/open/` | At the start of each work session and after any push |
| `tools/gh_close.py` | Post comment + close issue + move local file to `closed/` | After a fix is committed and pushed |

### Workflow

```bash
# Start of session
uv run tools/gh_sync.py

# Browse open issues
ls .github_issues/open/

# After fixing issue #42 and pushing commit abc123
uv run tools/gh_close.py --issue 42 --comment "Fixed in commit abc123. <short description of the fix>"

# Re-sync to confirm and pick next issue
uv run tools/gh_sync.py
```

### Token Scopes Required
- Public repo: `public_repo`
- Private repo: `repo`

Generate at: Canvas → Account → Settings → Approved Integrations → New Access Token *(wait, wrong app)* — GitHub → Settings → Developer Settings → Personal Access Tokens → Tokens (classic) → New token.

---

## Common Pitfalls and Solutions

### 1. gh_sync.py Pulls Pull Requests Too

**Problem**: The sync output includes PR entries mixed with issues.

**Why it happens**: GitHub's `/issues` endpoint returns both issues and pull requests. The script filters them by checking for the `pull_request` key, but if a PR was opened without a linked issue it still appears.

**Solution**: `gh_sync.py` already filters PRs via `"pull_request" not in issue`. If you see PRs in `open/`, check the filter logic — the field may have changed in the API response.

### 2. Issue File Disappears from open/ Without Being Closed

**Problem**: A file that was in `open/` is gone after re-running `gh_sync.py`, but it's not in `closed/` either.

**Why it happens**: The issue was closed directly on GitHub (in the browser or by another tool) between syncs. `gh_sync.py` only writes currently-open issues; it moves files for issues no longer in the open set to `closed/`.

**Solution**: Check `closed/` first. If it's there, the issue was resolved. If it's missing entirely, it may have been deleted on GitHub — check the repo's closed issues list.

### 3. Rate Limiting on Large Repos

**Problem**: `gh_sync.py` fails mid-run with a 403 or 429 error on repos with many issues.

**Why it happens**: GitHub's REST API rate limit is 5,000 requests/hour for authenticated users. Each issue + its comments is multiple requests; repos with 50+ issues and heavy comment threads can hit this.

**Solution**: The script uses pagination (100 per page) to minimize requests. If you hit rate limits, wait an hour or use a fine-grained token with higher limits. Check remaining rate limit: `GET /rate_limit`.

### 4. Closing the Wrong Issue Number

**Problem**: `gh_close.py --issue 42` closes a different issue than intended.

**Why it happens**: Issue numbers are reused across repos — if `GITHUB_REPO` is not set correctly and the script detects the wrong remote, it closes an issue on the wrong repo.

**Solution**: Always verify `GITHUB_REPO` resolves correctly before a close. Add `echo $GITHUB_REPO` or check the script output — it prints the repo name before acting.

---

## External System Lessons

### GitHub API — Issues Endpoint Returns Pull Requests

**Behavior**: `GET /repos/{owner}/{repo}/issues` returns both issues and pull requests. PRs have a `pull_request` key in the response object; plain issues do not.

**Why it matters**: If not filtered, PRs will appear as issue files in `open/`, creating noise.

**How to handle it**: Filter with `"pull_request" not in item` before processing.

### GitHub API — Rate Limit Headers

**Behavior**: Every API response includes `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers. At 0 remaining, requests return 403 until the reset timestamp.

**Why it matters**: A sync run on a large issue backlog can exhaust the hourly budget.

**How to handle it**: The scripts don't currently inspect rate limit headers — if you hit limits, add a check on `resp.headers.get("X-RateLimit-Remaining")` and sleep until reset if needed.

### GitHub API — Closed Issues Are Not Returned by Default

**Behavior**: `GET /issues?state=open` only returns open issues. Closed issues require `?state=closed` or `?state=all`.

**Why it matters**: `gh_sync.py` only syncs open issues by design. If an issue was closed externally it will be missing from the next sync's open set and moved to `closed/` locally.

**How to handle it**: This is the correct behavior. The `closed/` folder serves as the local archive.

---

## Validation

Pre-run:
- [ ] `GH_TOKEN` set in `.env`
- [ ] Token has `repo` or `public_repo` scope on the target repo
- [ ] `uv sync` completed (requests, python-dotenv installed)

Post-sync:
- [ ] `.github_issues/open/` contains `.md` files matching open issues on GitHub
- [ ] Each file has frontmatter (issue number, title, labels, url)
- [ ] Comments section present on issues that have comments

Post-close:
- [ ] Issue shows as closed on GitHub
- [ ] File moved from `open/` to `closed/`
- [ ] Comment with commit reference visible on the GitHub issue

---

## Resources

### Agent Files
- `gh_issues_agent.json` — label taxonomy, API patterns, file format spec
- `tools/gh_sync.py` — sync script
- `tools/gh_close.py` — close script
- `knowledge/github_issues_reference.md` — GitHub API reference notes

