# CI Notes for Exam Prep

Claude Code as a CI worker: install the tooling, understand the CI concept, see it live, then go deep on configuration.

## 1. Get the GitHub CLI (`gh`)

`gh` is GitHub's official CLI. It lets you work with issues, PRs, and Actions runs from a terminal, and Claude Code uses it for the same tasks.

```bash
# macOS
brew install gh

# Windows
winget install --id GitHub.cli

# Debian/Ubuntu: see https://github.com/cli/cli/blob/trunk/docs/install_linux.md

gh auth login        # authenticate (browser or token)
gh auth status       # verify
```

Commands worth knowing:

| Command | Purpose |
|---|---|
| `gh issue list` / `gh issue view <n>` | Browse issues |
| `gh pr create` / `gh pr view` / `gh pr checks` | Work with PRs and their CI status |
| `gh run list` / `gh run view <id> --log-failed` | Inspect workflow runs |
| `gh api <endpoint>` | Call any REST API endpoint |

In GitHub Actions, `gh` is preinstalled on hosted runners and authenticates through the `GITHUB_TOKEN` environment variable.

## 2. The CI Concept with GitHub Projects

CI (continuous integration) means every change is automatically built, checked, and reviewed before it merges.

A GitHub Project (board) can track the work. A typical flow:

1. **Issue created**: a task goes on the Project board as *Todo*.
2. **Trigger**: someone mentions `@claude` in the issue or a comment, or assigns the issue.
3. **Work**: a GitHub Actions workflow starts a runner, checks out the repo, and runs Claude Code.
4. **Output**: Claude creates a branch, commits, pushes, and gives a PR link. The issue moves to *In Progress* or *In Review*.
5. **Checks**: other workflows run on the PR (tests, linters, automated code review).
6. **Human review and merge**: the issue closes and the card moves to *Done*.

Key idea: **GitHub events are the triggers, workflows are the glue, and Claude is one more worker in the pipeline.** Humans keep the approval step.

## 3. Live Demo: Claude as a CI Worker

This repository is the demo.

1. Install the Claude GitHub app and add the workflows. `/install-github-app` in Claude Code sets this up.
2. Open an issue that describes a task. This issue (#3) is an example.
3. Comment `@claude work on this issue and create a pr for it`.
4. Watch the **Actions** tab. The `Claude Code` workflow runs and Claude updates a single tracking comment with a checklist.
5. Claude pushes a `claude/issue-<n>-<timestamp>` branch and links a prefilled PR.
6. When the PR opens, `Claude Code Review` runs and posts review feedback.
7. `Markdown-only check` runs as a normal CI gate. It fails the PR if any non-Markdown file is added.

## 4. In-Depth Configuration of Claude Code as a CI Worker

### Workflows in this repo

| File | Trigger | Role |
|---|---|---|
| `.github/workflows/claude.yml` | `@claude` in issue, comment, or review | Interactive worker that implements and answers |
| `.github/workflows/claude-code-review.yml` | PR opened, synchronized, reopened, or ready for review | Automatic code review |
| `.github/workflows/md-only.yml` | PR or push to `main` | Ordinary CI check, with no AI involved |

### Core action: `anthropics/claude-code-action@v1`

```yaml
- uses: anthropics/claude-code-action@v1
  with:
    claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    # prompt: "..."          # optional; default is to follow the @claude comment
    # claude_args: "--allowed-tools Bash(gh pr *)"
```

### Authentication

- Store `CLAUDE_CODE_OAUTH_TOKEN` (or an API key) in **Settings > Secrets and variables > Actions**. Never commit it.
- `id-token: write` is required by the action.

### Permissions (least privilege)

```yaml
permissions:
  contents: read
  pull-requests: read
  issues: read
  id-token: write
  actions: read   # lets Claude read CI results on PRs
```

Raise `contents`, `pull-requests`, and `issues` to `write` if you want Claude to push commits and post comments directly through the workflow token. Grant only what the job needs.

### Controlling behavior

- **`prompt`**: fixed instructions for automation (for example, a review on every PR). Leave it out for interactive `@claude` use.
- **`claude_args`**: passes CLI flags, such as `--allowed-tools`, `--max-turns`, and `--model`.
- **Tool allowlists**: restrict which Bash commands Claude can run, such as `Bash(gh pr *)` or `Bash(npm test)`.
- **`CLAUDE.md`**: put repo conventions, build and test commands, and style rules in this file at the repo root. Claude reads it on every run.
- **Plugins**: `plugin_marketplaces` and `plugins` load extras, such as the `code-review` plugin used in the review workflow.

### Trigger filters

- Use `if:` conditions so the job runs only on the right events, for example when the body contains `@claude`.
- Use `paths:` on `pull_request` to review only certain files.
- Filter by author, for example to review only first-time contributors.

### Limitations and safety

- The GitHub App cannot modify files under `.github/workflows/`.
- Claude cannot approve PRs or submit formal reviews. A human must approve.
- Treat issue and PR text as untrusted input (prompt injection). Limit tools and permissions accordingly.
- Watch cost and runtime. Use `--max-turns`, timeouts, and narrow triggers.

### Exam-style takeaways

- Claude Code in CI is event-driven. Know the triggers (`issue_comment`, `pull_request`, `issues`, and others).
- Least-privilege permissions plus tool allowlists is the main safety control.
- `CLAUDE.md` is the main way to give Claude project context.
- Keep humans in the loop for merge decisions.
