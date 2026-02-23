# /configure-permissions

An interactive wizard skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that configures granular permission rules — so you can stop using `--dangerously-skip-permissions` without getting prompted on every action.

## What it does

- **2-question fast path** — pick a scope and a preset, done in under 30 seconds
- **4 presets** — Balanced Dev, Full Trust, Read Only, and Custom
- **Auto-detects MCP servers** — adds permissions for your configured MCP tools
- **Deep-merges settings** — only touches the `permissions` key, preserves your other config
- **Never uses `deny`** — destructive commands go to `ask` (prompts once) so you always retain control

## Quick Install

### Option A: Plugin Marketplace (easiest)

```
/install-skill ashraymalhotra/claude-permissions-wizard
```

### Option B: Manual Copy

```bash
# Create the skills directory if it doesn't exist
mkdir -p ~/.claude/skills/configure-permissions

# Download the skill file
curl -sL https://raw.githubusercontent.com/ashraymalhotra/claude-permissions-wizard/main/skills/configure-permissions/SKILL.md \
  -o ~/.claude/skills/configure-permissions/SKILL.md
```

### Option C: Git Clone

```bash
git clone https://github.com/ashraymalhotra/claude-permissions-wizard.git
cp -r claude-permissions-wizard/skills/configure-permissions ~/.claude/skills/
```

## Presets at a Glance

| | Balanced Dev | Full Trust | Read Only | Custom |
|---|:---:|:---:|:---:|:---:|
| Read / Edit / Write | Allow | Allow | Read only | You choose |
| Git (status, log, diff) | Allow | Allow | — | You choose |
| Git (add, commit, branch) | Allow | Allow | — | You choose |
| Git push | **Ask** | Allow | — | You choose |
| npm / yarn / pnpm / bun | Allow | Allow | — | You choose |
| Python / pip / pytest | Allow | Allow | — | You choose |
| System utils (ls, find, grep…) | Allow | Allow | — | You choose |
| Web search & fetch | Allow | Allow | Allow | You choose |
| MCP tools | Auto-allow | Auto-allow | Ask | You choose |
| Destructive commands (rm -rf, sudo, force-push…) | **Ask** | **Ask** | Ask | You choose |

**Ask** = Claude prompts you once before running. Not blocked, just confirmed.

## Usage

Start a Claude Code session and run:

```
/configure-permissions
```

The wizard asks two questions:

1. **Scope** — Where should permissions be saved?
   - All projects (`~/.claude/settings.json`)
   - This project, for everyone (`.claude/settings.json`)
   - This project, just me (`.claude/settings.local.json`)

2. **Preset** — Which permission level?
   - **Balanced Dev** (recommended) — common dev tools allowed, destructive commands prompt
   - **Full Trust** — everything allowed, destructive commands still prompt
   - **Read Only** — read + web only, everything else prompts
   - **Custom** — configure each category yourself (8 more questions)

That's it. The wizard writes your settings file and you're done.

## How it Works

The skill generates a Claude Code settings JSON with two arrays:

- **`allow`** — tools and commands Claude can use without asking
- **`ask`** — tools and commands that prompt you once before running

It reads your existing settings file, replaces only the `permissions` key (preserving `model`, `env`, and other config), and writes it back with 2-space indentation.

For MCP servers, it scans `.mcp.json` and `~/.claude.json`, extracts server names, and adds the corresponding `mcp__<server_name>` permission entries.

## Contributing

Contributions are welcome! Feel free to:

- Add new presets for specific workflows (e.g., DevOps, Data Science)
- Improve MCP server detection
- Add support for more build tools and runtimes

Please open an issue first for larger changes.

## License

[MIT](LICENSE)
