---
name: project-setup
description: Bootstrap a new project on Phil's machine (macOS or Windows) by creating a local project folder, initializing git, creating a private GitHub repo via the GitHub CLI, and linking the project to Vercel for deployments. Use this skill whenever the user asks to "start a new project", "set up a new project", "scaffold a project", "create a new repo and deploy", or mentions wanting a fresh project folder + GitHub + Vercel together. Trigger even if the user only says something like "new project called X" — assume they want the full local + GitHub + Vercel chain unless they say otherwise.
---

# Project Setup

Bootstraps a new project: a local project folder, a private GitHub repo, and a linked Vercel project for deployments. Works on both of Phil's machines (macOS and Windows). The GitHub repo is the cross-machine bridge — set up on one machine, `git clone` on the other. Don't put working folders in cloud-synced storage (Drive/iCloud); it corrupts `.git` and thrashes `node_modules`.

## Platform — detect first, then use the matching column

Run `uname` (or check the session environment). "Darwin" → macOS. Otherwise assume Windows / PowerShell.

| | macOS | Windows |
|---|---|---|
| Shell | zsh / bash | PowerShell |
| Projects root | `~/Documents/Claude/projects` | `C:\Claude\projects` |
| Project path | `~/Documents/Claude/projects/<name>` | `C:\Claude\projects\<name>` |

Most commands (git, gh, vercel, npm, npx, node) are **identical on both** — run them as-is. Only folder ops, file writing, and toolchain install hints differ; those are given per-platform below. If the active shell doesn't match the OS (e.g. bash on Windows / WSL), either switch shells or wrap PowerShell commands with `powershell -Command "..."`.

## When to use

Trigger on phrases like:
- "set up a new project called X"
- "start a new project, name it X"
- "scaffold X with GitHub and Vercel"
- "new project X" (assume full chain unless told otherwise)

## Architecture

This skill uses two subagents to split planning from execution, since the two phases benefit from different model strengths:

| Phase | Subagent | Model | Why |
|---|---|---|---|
| 1–3: Gather inputs, verify toolchain, pre-flight conflict checks | `project-planner` | Opus | Careful judgment on naming, conflict resolution, decisions that affect the whole run |
| 4: Execute the bootstrap commands | `project-executor` | Sonnet | Fast, deterministic command execution and output verification |

The subagents must be installed at the user agents directory — `~/.claude/agents/` on macOS, `%USERPROFILE%\.claude\agents\` on Windows. Phil keeps the canonical copies in the phlzzz/claude-config GitHub repo (cloned to C:\Claude\claude-config on Windows / ~/code/claude-config on macOS), with ~/.claude/agents/ symlinked into the repo so both machines and remote web sessions share one source of truth (web sessions get them via the session-start hook). If either subagent is missing, fall back to running everything in the main session and tell the user the subagents aren't installed.

## Workflow

### Step 1 — Delegate planning to `project-planner`

Invoke the planner explicitly:

> Use the project-planner subagent to gather inputs and run pre-flight checks for a new project setup.

Pass along whatever the user already said (e.g. "they said the project is called my-cool-thing"). The planner will:

1. Confirm/normalize the project name
2. Ask scaffold type via `ask_user_input_v0`
3. Ask for an optional description
4. Ask about repo visibility only if the user signaled they want public
5. Verify the toolchain (git, gh, gh auth, vercel, vercel auth, node)
6. Run pre-flight conflict checks for folder, GitHub repo name, and Vercel project name
7. Return a structured plan back to the main session

The planner returns a JSON-shaped summary like:

```
projectName: my-cool-thing
scaffold: nextjs
description: "A weekend hack to test something"
visibility: private
vercelMode: create-new   # or "link-existing" if user chose that
status: ready-to-execute
```

If the planner returns `status: cancelled`, stop and tell the user.

### Step 2 — Delegate execution to `project-executor`

Pass the planner's output into the executor:

> Use the project-executor subagent to bootstrap the project with these inputs: [paste the plan].

The executor will run Phase 4 (folder creation, scaffold, git init, GitHub repo, Vercel link, deploy) and return the resulting URLs.

### Step 3 — Report back

Summarize for the user in one short paragraph: project name, local path, GitHub URL, Vercel production URL. No fluff.

---

## Fallback (if subagents aren't installed)

If the `project-planner` or `project-executor` subagent is missing from the user agents directory, run the full workflow in the main session using the inline reference below, and tell the user once at the end:

> "Heads up — the project-planner and project-executor subagents aren't installed, so I ran everything in the main session. To get the Opus-for-planning / Sonnet-for-execution split, clone phlzzz/claude-config and symlink ~/.claude/agents into it (macOS: ln -s ~/code/claude-config/agents ~/.claude/agents; Windows PowerShell: New-Item -ItemType SymbolicLink -Path $HOME\.claude\agents -Target C:\Claude\claude-config\agents)."

### Inline fallback workflow

**Phase 1 — Gather inputs**

Project name (free text, normalize to lowercase-hyphen-separated, confirm if changed).

Scaffold via `ask_user_input_v0`:
```
question: "What scaffold should the project use?"
options:
  - "Empty (README + .gitignore)"
  - "Next.js (Vercel default, recommended for web apps)"
  - "Vite + React + TypeScript"
  - "Other (I'll describe it)"
```

Optional description (free text, can skip).

Visibility: default private. Only ask if user mentions "public", "open source", "share".

**Phase 2 — Verify toolchain**

Same on both platforms:
```
git --version; gh --version; gh auth status; vercel --version; vercel whoami; node --version
```

If anything fails, point to the fix and stop:

| Missing | macOS fix | Windows fix |
|---|---|---|
| `git` | `brew install git` (or `xcode-select --install`) | install from git-scm.com |
| `gh` | `brew install gh` | `winget install GitHub.cli` |
| `gh auth status` fails | `gh auth login` | `gh auth login` |
| `vercel` | `npm i -g vercel` | `npm i -g vercel` |
| `vercel whoami` fails | `vercel login` | `vercel login` |
| `node` | `brew install node` | install from nodejs.org |

**Phase 3 — Pre-flight checks**

macOS:
```bash
test -d ~/Documents/Claude/projects/<name> && echo EXISTS
gh repo view "<name>" 2>&1
vercel project ls 2>&1 | grep -i "<name>"
```

Windows:
```powershell
Test-Path "C:\Claude\projects\<name>"
gh repo view "<name>" 2>&1
vercel project ls 2>&1 | Select-String -SimpleMatch "<name>"
```

For each conflict, ask via `ask_user_input_v0` (different name / delete or link / cancel — see subagent files for exact options).

**Phase 4 — Execute**

1. Create the folder and `cd` into it (`mkdir -p` on macOS / `New-Item` on Windows — see executor).
2. Scaffold (Empty / `npx create-next-app@latest . ...` / `npm create vite@latest . -- --template react-ts` — the npx/npm commands are identical on both).
3. `git init -b main` (skip if scaffold did it), `git add .`, `git commit -m "Initial commit"` — identical on both.
4. `gh repo create <name> --private --source=. --remote=origin --push` — identical on both.
5. Vercel — must run all four, in order (identical on both):
   - `vercel projects add <name>` (skip if `vercelMode: link-existing`)
   - `vercel link --yes --project <name>`
   - `vercel git connect <repo URL> --yes` ← **without this, pushes from elsewhere create an orphan auto-imported project**
   - `vercel --prod --yes`
6. Capture URLs and report.

For the full step-by-step including edge cases and the per-platform file-writing blocks, see `agents/project-executor.md`.

---

## Don't

- Don't open VS Code at the end (Phil declined this).
- Don't add a license, CI workflow, or other extras unless asked.
- Don't make the repo public by default.
- Don't put the working folder in Google Drive / iCloud / any cloud-synced path — GitHub is the cross-machine sync.
- Don't proceed past a failed step without acknowledging it.
- Don't skip the pre-flight checks. They take 5 seconds and save messy cleanup.
