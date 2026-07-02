# claude-config — machine setup

This repo holds Phil's **global Claude Code configuration**: agents, skills, and user
`settings.json`. It is *not* a working project — it's the thing that makes every project
on a machine share the same agents/skills/settings.

It works by symlinking three paths under `~/.claude/` at this clone. Anything under
`~/.claude/` is user-global, so once these links exist, **every** project on the machine
(including ones created by the `project-setup` skill) automatically has all agents and
skills — there is nothing to connect per project.

| Path in `~/.claude/` | Links to |
|---|---|
| `agents` | `<clone>/agents` |
| `skills` | `<clone>/skills` |
| `settings.json` | `<clone>/settings/settings.json` |

---

## New machine — one-time bootstrap

Clone somewhere **local**. Never put this in iCloud / Google Drive / OneDrive — cloud
sync corrupts `.git` and thrashes symlinks.

### Windows (PowerShell)

Symlink creation needs Developer Mode on (Settings → For developers), or run the
shell as Administrator.

```powershell
# 1. Clone (local disk, not a synced folder)
git clone https://github.com/phlzZz/claude-config.git C:\Claude\claude-config

# 2. Ensure ~/.claude exists
New-Item -ItemType Directory -Force "$HOME\.claude" | Out-Null

# 3. Point the three paths at the clone (removes any existing link/folder first)
$repo = "C:\Claude\claude-config"
foreach ($p in "agents","skills") {
  $link = "$HOME\.claude\$p"
  if (Test-Path $link) { (Get-Item $link).Delete() }
  New-Item -ItemType SymbolicLink -Path $link -Target "$repo\$p" | Out-Null
}
$s = "$HOME\.claude\settings.json"
if (Test-Path $s) { (Get-Item $s).Delete() }
New-Item -ItemType SymbolicLink -Path $s -Target "$repo\settings\settings.json" | Out-Null
```

### macOS

```bash
# 1. Clone (local disk, not iCloud/Drive)
mkdir -p ~/Claude && git clone https://github.com/phlzZz/claude-config.git ~/Claude/claude-config

# 2. Ensure ~/.claude exists
mkdir -p ~/.claude

# 3. Point the three paths at the clone
repo=~/Claude/claude-config
ln -sfn "$repo/agents" ~/.claude/agents
ln -sfn "$repo/skills" ~/.claude/skills
ln -sf  "$repo/settings/settings.json" ~/.claude/settings.json
```

> **Before running step 3**, check whether `~/.claude/agents` or `~/.claude/skills`
> already exist as **real folders** (not symlinks) with content in them — e.g. from an
> older Google Drive setup. If so, move that content into this repo first so nothing is
> lost. `ln -sfn` / the PowerShell delete will otherwise replace them.

---

## Keeping machines in sync

- **Pull the latest config on a machine:** `git pull` inside the clone. The symlinks mean
  changes take effect immediately — no re-linking.
- **Make a change:** edit files in the clone, commit, push. Then `git pull` on the other
  machines.
- `settings/settings.json` is **shared across all machines** (default model,
  `autoUploadSessions`, shared permissions). Anything that should be machine-specific
  belongs in `~/.claude/settings.local.json` instead, which is not part of this repo.
