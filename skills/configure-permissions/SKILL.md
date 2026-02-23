---
name: configure-permissions
description: Interactive wizard to configure Claude Code permission rules. Sets up granular allow/ask permissions so you can work without --dangerously-skip-permissions while keeping safety nets for destructive commands. Use when user says "configure permissions", "set up permissions", "permission settings", or "stop asking me for permissions".
---

# Configure Permissions Wizard

Interactive wizard that configures Claude Code permission rules. The goal: reach near-skip-permissions convenience while keeping a safety net for destructive commands.

## Overview

This skill asks **2 questions** for most users (scope + preset), then writes the permissions config. Only the "Custom" preset goes deeper with additional questions.

Permissions are written to a Claude Code settings JSON file. The skill deep-merges — it only replaces the `permissions` key and preserves everything else (`model`, `env`, `skipDangerousModePermissionPrompt`, etc.).

## Round 1: Scope + Preset (always asked)

### Question 1 — Scope

Ask the user where permissions should be saved. Use `AskUserQuestion` with these options:

| Option | File | Description |
|--------|------|-------------|
| **All my projects** | `~/.claude/settings.json` | These permissions apply everywhere, across every repo |
| **This project, for everyone** | `.claude/settings.json` | Saved in the repo so teammates get the same permissions when they clone it |
| **This project, just me** | `.claude/settings.local.json` | Only affects you on this machine, invisible to teammates |

### Question 2 — Preset

Ask the user which preset to use. Use `AskUserQuestion` with these options:

| Option | Description |
|--------|-------------|
| **Balanced Dev (Recommended)** | Common dev tools allowed, destructive commands prompt before running |
| **Full Trust** | Everything allowed, destructive commands still prompt before running |
| **Read Only** | Read + web only, everything else prompts |
| **Custom** | Configure each category yourself (more questions follow) |

**If a preset is chosen (not Custom): skip straight to writing the file. No more questions.** This is the fast path.

## Rounds 2-4: Custom Preset Only

Only proceed with these rounds if the user picked "Custom" in Q2.

### Round 2 — Core Permissions (3 questions)

**Q3 - File operations** (multiselect):
- Read
- Edit
- Write
- NotebookEdit

**Q4 - Bash access:**
- Common dev tools (allows specific tool patterns — goes to Round 3 for granularity)
- All Bash (allows `Bash` broadly)
- Ask every time (puts `Bash` in `ask`)

**Q5 - Web & tasks** (multiselect):
- WebSearch
- WebFetch
- Task (subagents)

### Round 3 — Dev Tool Granularity (only if "Common dev tools" chosen in Q4)

**Q6 - Git operations** (multiselect):
- Read-only: `git status`, `git log *`, `git diff *`
- Stage + commit: `git add *`, `git commit *`
- Branch management: `git branch *`, `git checkout *`, `git switch *`, `git merge *`
- Push + pull: `git pull *`, `git fetch *`, `git push *`

**Q7 - Build tools** (multiselect):
- npm/yarn/pnpm: `npm *`, `npx *`, `yarn *`, `pnpm *`
- Python: `python *`, `python3 *`, `pip *`, `pip3 *`, `pytest *`, `uv *`
- System utilities: `ls *`, `cat *`, `head *`, `tail *`, `find *`, `grep *`, `wc *`, `mkdir *`, `cp *`, `mv *`
- Docker: `docker *`, `docker-compose *`

### Round 4 — Safety + Mode (2-3 questions)

**Q8 - Dangerous commands:**
- Ask before running (puts dangerous commands in `ask` — recommended)
- Allow all (no dangerous command protection)

**Q9 - MCP tools** (only if MCP servers are detected in `.mcp.json` or `~/.claude.json`):
- Allow all (adds all detected MCP tool prefixes to `allow`)
- Ask each time (adds them to `ask`)

**Q10 - Default for uncovered tools:**
- Ask (default Claude behavior — prompts for anything not in `allow`)
- This is informational only — Claude Code's default is to ask for anything not explicitly allowed

## Preset Definitions

### Balanced Dev

```json
{
  "permissions": {
    "allow": [
      "Read", "Edit", "Write", "WebFetch", "WebSearch", "Task",
      "Bash(git status)", "Bash(git log *)", "Bash(git diff *)",
      "Bash(git add *)", "Bash(git commit *)", "Bash(git branch *)",
      "Bash(git checkout *)", "Bash(git pull *)", "Bash(git fetch *)",
      "Bash(npm *)", "Bash(npx *)", "Bash(yarn *)", "Bash(pnpm *)",
      "Bash(node *)", "Bash(python *)", "Bash(python3 *)",
      "Bash(pip *)", "Bash(pip3 *)", "Bash(pytest *)", "Bash(uv *)",
      "Bash(ls *)", "Bash(cat *)", "Bash(head *)", "Bash(tail *)",
      "Bash(find *)", "Bash(grep *)", "Bash(wc *)", "Bash(mkdir *)",
      "Bash(cp *)", "Bash(mv *)", "Bash(make *)", "Bash(cargo *)",
      "Bash(bun *)", "Bash(deno *)"
    ],
    "ask": [
      "Bash(rm -rf *)", "Bash(rm -r *)", "Bash(git push --force *)",
      "Bash(git push -f *)", "Bash(sudo *)", "Bash(chmod 777 *)",
      "Bash(chmod -R *)", "Bash(mkfs *)", "Bash(dd *)",
      "Bash(git reset --hard *)", "Bash(git clean -f *)",
      "Bash(> *)", "Bash(curl * | bash*)", "Bash(curl * | sh*)",
      "Bash(wget * | bash*)", "Bash(git push *)"
    ]
  }
}
```

### Full Trust

```json
{
  "permissions": {
    "allow": [
      "Read", "Edit", "Write", "NotebookEdit", "Bash",
      "WebFetch", "WebSearch", "Task"
    ],
    "ask": [
      "Bash(rm -rf *)", "Bash(rm -r *)", "Bash(git push --force *)",
      "Bash(git push -f *)", "Bash(sudo *)", "Bash(mkfs *)", "Bash(dd *)",
      "Bash(git reset --hard *)", "Bash(git clean -f *)"
    ]
  }
}
```

### Read Only

```json
{
  "permissions": {
    "allow": [
      "Read", "WebFetch", "WebSearch", "Task"
    ],
    "ask": [
      "Edit", "Write", "Bash"
    ]
  }
}
```

## MCP Server Detection

Before writing the file, detect MCP servers:

1. Read `.mcp.json` in the current project root (if it exists)
2. Read `~/.claude.json` (if it exists)
3. Extract all server names from the `mcpServers` key in each file
4. For each server name, the MCP tool permission format is: `mcp__<server_name>` (double underscore, with hyphens in server names converted to underscores)

For **Balanced Dev** and **Full Trust** presets: automatically add all detected MCP server tool prefixes to the `allow` list. For example, if servers `exa` and `figma-dev` are found, add `"mcp__exa"` and `"mcp__figma_dev"` to `allow`.

For **Read Only**: add MCP server tool prefixes to `ask`.

For **Custom**: ask the user (Q9) whether to allow or ask.

## Writing the Settings File

### Step 1: Read existing file

Read the target settings file (based on scope choice). If it doesn't exist, start with `{}`.

### Step 2: Build permissions object

Based on the preset (or custom selections), construct the `permissions` object with `allow` and `ask` arrays.

Include detected MCP servers as described above.

### Step 3: Deep merge

Parse the existing JSON. Replace ONLY the `permissions` key with the new permissions object. Keep all other top-level keys intact.

### Step 4: Write file

Write the merged JSON back to the file, formatted with 2-space indentation.

If the directory doesn't exist (e.g., `.claude/` for project-level), create it first.

### Step 5: Confirm

Tell the user:
- Which file was written
- How many rules were added to `allow` vs `ask`
- Remind them that `ask` rules still prompt once (they're not blocked)
- Suggest they restart Claude Code for changes to take effect

## Building Custom Permissions

When the user picks "Custom", build the `allow` and `ask` arrays from their selections across Rounds 2-4:

1. **File operations** (Q3): Selected tools go to `allow`, unselected go to `ask`
2. **Bash access** (Q4):
   - "All Bash": `Bash` goes to `allow`
   - "Common dev tools": specific patterns from Q6+Q7 go to `allow`
   - "Ask every time": `Bash` goes to `ask`
3. **Web & tasks** (Q5): Selected tools go to `allow`, unselected go to `ask`
4. **Git operations** (Q6): Selected patterns go to `allow` as `Bash(pattern)` entries
5. **Build tools** (Q7): Selected patterns go to `allow` as `Bash(pattern)` entries
6. **Dangerous commands** (Q8):
   - "Ask before running": all dangerous patterns go to `ask`
   - "Allow all": dangerous patterns are not added to `ask`
7. **MCP tools** (Q9): detected servers go to `allow` or `ask` based on selection

## Important Rules

- **Never use `deny`** — `deny` blocks Claude entirely, even if the user explicitly requests the action. Always use `ask` instead, which prompts once so the user retains control.
- **Deep-merge only** — never overwrite the entire settings file. Only replace the `permissions` key.
- **`git push` in `ask` for Balanced Dev** — regular push (not just force-push) is in `ask` because pushing is a shared-state action visible to others.
- **Presets skip all further questions** — if the user picks Balanced Dev, Full Trust, or Read Only, write the file immediately after Q2. Do not ask rounds 2-4.
