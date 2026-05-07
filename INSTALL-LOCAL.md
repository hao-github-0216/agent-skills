# INSTALL-LOCAL — bootstrap this fork on a new machine

> Personal install guide for `hao-github-0216/agent-skills` (fork of `addyosmani/agent-skills`). NOT generic upstream documentation. The setup here installs the plugin in **local-dev mode** — your edits to the clone are live in Claude Code immediately, and `git push origin main` propagates them to every machine that runs this recipe.

This file is the canonical bootstrap recipe. The `agent-skills-fork-dev` user-scope skill (at `~/.claude/skills/agent-skills-fork-dev/`) is the day-to-day operating manual that references this file — they are not redundant. This one gets you from "fresh machine" to "plugin works"; that one tells you how to LIVE with it.

## Prerequisites

- `git`
- `gh` (GitHub CLI), authenticated as the fork owner
- Claude Code CLI installed and runnable

Paths use `$HOME` so the same recipe runs on Linux (`/home/...`) and macOS (`/Users/...`). Run every command from a normal user shell — no `sudo`.

## Step 1 — Clone the fork

```bash
gh repo clone hao-github-0216/agent-skills "$HOME/agent-skills" 2>/dev/null || true
cd "$HOME/agent-skills" && git remote add upstream https://github.com/addyosmani/agent-skills.git 2>/dev/null || true
```

Both commands are idempotent — safe to re-run if the clone partially failed before. Verify:

```bash
cd "$HOME/agent-skills" && git remote -v
# Should show:  origin   ...hao-github-0216/agent-skills...
#               upstream ...addyosmani/agent-skills...
```

## Step 2 — Build the wrapper marketplace

Claude Code installs plugins from a "marketplace". We create a tiny local marketplace that points at the clone via symlink, so install / reload pick up live edits.

```bash
mkdir -p "$HOME/.claude/local-marketplaces/my-agent-skills/.claude-plugin" \
         "$HOME/.claude/local-marketplaces/my-agent-skills/plugins"

ln -sfn "$HOME/agent-skills" "$HOME/.claude/local-marketplaces/my-agent-skills/plugins/agent-skills"

printf '%s' '{"name":"my-agent-skills","description":"Local dev marketplace pointing to ~/agent-skills (fork of addyosmani/agent-skills)","owner":{"name":"hao-github-0216"},"plugins":[{"name":"agent-skills","description":"Local development copy of addyosmani/agent-skills, edits sync to fork via git push","source":"./plugins/agent-skills","version":"1.0.0-dev","category":"development","tags":["skills","fork","local-dev"]}]}' \
  > "$HOME/.claude/local-marketplaces/my-agent-skills/.claude-plugin/marketplace.json"

# Sanity-check JSON parses
python3 -m json.tool < "$HOME/.claude/local-marketplaces/my-agent-skills/.claude-plugin/marketplace.json" \
  && echo "✅ marketplace.json valid"
```

If `printf` fails because `cat` is aliased to `bat` or similar, the printf form bypasses the alias entirely.

## Step 3 — Register and install in Claude Code

Inside Claude Code (any project), run:

```
/plugin marketplace add ~/.claude/local-marketplaces/my-agent-skills
/plugin uninstall agent-skills@addy-agent-skills
/plugin install agent-skills@my-agent-skills
/reload-plugins
```

The `uninstall` line only matters if you previously installed `agent-skills` from the upstream `addy-agent-skills` marketplace. If you've never installed it before, that line will be a no-op — safe to run anyway.

## Step 4 — Critical: swap the cache deep-copy for a symlink

By default Claude Code deep-copies plugin sources into `~/.claude/plugins/cache/`, freezing your edits at install time. We replace that cache directory with a symlink to the clone so every edit is live.

```bash
rm -rf "$HOME/.claude/plugins/cache/my-agent-skills/agent-skills/1.0.0"
ln -s "$HOME/agent-skills" "$HOME/.claude/plugins/cache/my-agent-skills/agent-skills/1.0.0"
```

Verify:

```bash
readlink "$HOME/.claude/plugins/cache/my-agent-skills/agent-skills/1.0.0"
# Should print the absolute path of $HOME/agent-skills
```

Back in Claude Code, run `/reload-plugins` once more.

## Step 5 — Verify the install

Inside Claude Code, the system reminder should now list `agent-skills:*` skills. Quick test: invoke one and check its base directory header.

```
/agent-skills:planning-and-task-breakdown
```

The output's "Base directory" line should resolve to a path under `$HOME/agent-skills/skills/`. If it points at a `cache/` path that's NOT a symlink, Step 4 didn't take — re-run it.

## ⚠️ Do NOT run `/plugin update agent-skills@my-agent-skills`

That command will delete the cache directory (including the symlink from Step 4) and replace it with a fresh deep copy from the marketplace source. After that, edits to `$HOME/agent-skills/` are no longer live — they only affect git, not Claude Code.

To pull upstream changes from `addyosmani`:

```bash
cd "$HOME/agent-skills"
git fetch upstream
git merge upstream/main          # or: git rebase upstream/main
git push origin main
# In Claude Code: /reload-plugins
```

If `/plugin update` did get accidentally run, repair with the same symlink swap from Step 4:

```bash
rm -rf "$HOME/.claude/plugins/cache/my-agent-skills/agent-skills/1.0.0" \
  && ln -s "$HOME/agent-skills" "$HOME/.claude/plugins/cache/my-agent-skills/agent-skills/1.0.0"
```

## Daily workflow (after install)

```bash
# Edit anything in the clone
$EDITOR "$HOME/agent-skills/skills/<name>/SKILL.md"

# Apply in current Claude Code session — no git push needed
# In Claude Code: /reload-plugins

# When happy, sync to the fork — every other machine running this recipe will pick up via git pull
cd "$HOME/agent-skills"
git add -A && git commit -m "tweak: ..."
git push origin main
```

## Optional — install the user-scope companion skill

The day-to-day operating manual (philosophy, repair, cross-machine model) lives at `~/.claude/skills/agent-skills-fork-dev/SKILL.md`. It is intentionally NOT in this repo (it's per-machine personal config). On the canonical machine (howardserver) it already exists; on a new machine you can copy it across:

```bash
# From the new machine, pull from howardserver via Tailscale (adjust user/host as needed)
mkdir -p "$HOME/.claude/skills/agent-skills-fork-dev"
scp howard@howardserver.tail1c1d34.ts.net:~/.claude/skills/agent-skills-fork-dev/SKILL.md \
    "$HOME/.claude/skills/agent-skills-fork-dev/SKILL.md"
```

If Tailscale isn't available, just write a minimal version that references this file:

```markdown
---
name: agent-skills-fork-dev
description: Use whenever the user wants to modify, debug, sync, or bootstrap the agent-skills plugin (forked from addyosmani/agent-skills).
---

For install/bootstrap, see `~/agent-skills/INSTALL-LOCAL.md`. For the philosophy and daily workflow, see the canonical version on howardserver at `~/.claude/skills/agent-skills-fork-dev/SKILL.md`.
```

That stub is enough to make Claude Code redirect future questions to the right places.
