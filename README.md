# Disk Space Audit

[![skills.sh](https://skills.sh/b/Haaaiawd/disk-space-audit-skill)](https://skills.sh/Haaaiawd/disk-space-audit-skill)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

**[中文文档](./README.zh-CN.md)**

A repeatable, agent-native workflow for auditing disk space on **Windows** and **macOS** (with Linux references). Produces a categorized cleanup report that tells you exactly where your bytes went and what's safe to clean. Cleanup is a separate, explicitly-confirmed phase — the audit itself never touches anything.

Designed for AI coding agents (Devin, Claude Code, Cursor, Codex, etc.) that need a structured, safe approach to disk space analysis. Compatible with [`npx skills`](https://github.com/vercel-labs/skills) — the open agent skills ecosystem by Vercel Labs.

## Quick install

### Via `npx skills` (recommended)

Supports **70+ agents** — Claude Code, Cursor, Codex, OpenCode, Devin, and more.

```bash
# Install globally (available across all projects)
npx skills add Haaaiawd/disk-space-audit-skill -g

# Install to specific agents
npx skills add Haaaiawd/disk-space-audit-skill -a claude-code -a cursor

# Install to current project
npx skills add Haaaiawd/disk-space-audit-skill

# Non-interactive (CI/CD)
npx skills add Haaaiawd/disk-space-audit-skill -g -a claude-code -y
```

### Via git clone

```bash
# For Devin CLI
git clone https://github.com/Haaaiawd/disk-space-audit-skill.git \
  ~/.devin/skills/disk-space-audit

# For Claude Code
git clone https://github.com/Haaaiawd/disk-space-audit-skill.git \
  ~/.claude/skills/disk-space-audit

# Generic — clone anywhere, point your agent to SKILL.md
git clone https://github.com/Haaaiawd/disk-space-audit-skill.git
```

## Features

- **Read-only audit** — scan and report never deletes, moves, or modifies files
- **Multi-platform** — Windows (WizTree MFT), macOS (disky APFS-aware), Linux (ncdu/dua)
- **Incremental caching** — WizTree CSV cached for 24h, reused on subsequent scans
- **Duplicate file detection** — three-level filter (size → 4KB hash → full hash), >50MB only
- **Categorized report** — 🟢🟡🟠🔴 four-tier ranking by size × safety
- **Per-item cleanup confirmation** — no batch approvals, no ambiguous consent
- **Re-download cost awareness** — states bandwidth/time cost for every cache deletion
- **Anti-pattern table** — explicitly lists what NOT to delete (uv cache, Docker volumes, etc.)

## Workflow

| Phase | Description |
|-------|-------------|
| 1. Recon | Disk overview, platform detection |
| 2. Deep scan | WizTree (Win) / disky (macOS) / ncdu (Linux) |
| 2.5. Duplicate detection | Optional, on request only |
| 3. Investigate | Web search + content inspection for unknowns |
| 4. Categorize | Group by type, rank by size × safety |
| 5. Report | Markdown report with consequence column |
| 6. Cleanup | Only when explicitly asked, per-item confirmation |

## Prerequisites

| Platform | Tool | Install | Purpose |
|----------|------|---------|---------|
| Windows | **WizTree** | `winget install AntibodySoftware.WizTree` | MFT direct read, scans 500GB in ~10s |
| macOS | **disky** | `brew install disky` | APFS-aware, handles sparse files |
| Linux | **ncdu** / **dua** | `apt install ncdu` / `cargo install dua-cli` | Parallel scanning |

The skill will auto-install WizTree via winget if missing on Windows.

## Usage

After installing, just tell your AI agent something like:

```
"scan my C drive"
"analyze disk space"
"find what's eating my storage"
"clean up system drive"
"find duplicate files"
```

The agent will run the audit (read-only) and produce a categorized report. Cleanup only starts when you explicitly ask, and every item requires individual confirmation.

### Try without installing

```bash
# Generate a prompt and pipe to claude
npx skills use Haaaiawd/disk-space-audit-skill | claude

# Start a specific agent with the skill
npx skills use Haaaiawd/disk-space-audit-skill --agent claude-code
```

## Key design decisions

1. **Audit ≠ Cleanup.** The report is the deliverable. Cleanup is a separate, explicitly-confirmed phase. This prevents accidental data loss from "helpful" agents.

2. **Speed via MFT.** PowerShell `Get-ChildItem -Recurse` hangs on AppData junctions and million-file caches. WizTree reads the NTFS Master File Table directly — 500GB in 10 seconds (admin mode).

3. **Value over vanity.** Clearing 50GB of useful cache to show a big number is harmful. Every recommendation weighs re-acquisition cost against space saved.

4. **Per-item confirmation, always.** "Clean some stuff" is not authorization. Each deletion gets its own yes/no. New information (app started running, symlink found) resets prior approval.

## File structure

```
disk-space-audit-skill/
├── SKILL.md          # The skill definition (552 lines)
├── README.md         # English docs (this file)
├── README.zh-CN.md   # 中文文档
├── LICENSE           # MIT
└── .gitignore
```

## Compatibility

This skill follows the [open agent skills format](https://github.com/vercel-labs/skills) — a `SKILL.md` with YAML frontmatter (`name` + `description`) at the repo root. It works with any agent supported by `npx skills`, including:

| Agent | `--agent` flag |
|-------|---------------|
| Claude Code | `claude-code` |
| Cursor | `cursor` |
| Codex | `codex` |
| OpenCode | `opencode` |
| Devin | `devin` |
| Amp / Replit / Universal | `amp` / `replit` / `universal` |
| ...and 65 more | [Full list](https://github.com/vercel-labs/skills#supported-agents) |

## License

MIT — see [LICENSE](LICENSE).
