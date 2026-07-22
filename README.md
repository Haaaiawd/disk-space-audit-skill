# Disk Space Audit Skill

[![skills.sh](https://skills.sh/b/Haaaiawd/disk-space-audit-skill)](https://skills.sh/Haaaiawd/disk-space-audit-skill)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A repeatable, agent-native workflow for auditing disk space on **Windows** and **macOS** (with Linux references). Produces a categorized cleanup report that tells you exactly where your bytes went and what's safe to clean. Cleanup is a separate, explicitly-confirmed phase — the audit itself never touches anything.

Designed for AI coding agents (Devin, Claude Code, Cursor, Codex, etc.) that need a structured, safe approach to disk space analysis. Compatible with [`npx skills`](https://github.com/vercel-labs/skills) — the open agent skills ecosystem by Vercel Labs.

---

# 磁盘空间审计 Skill

一个可重复的、面向 AI agent 的磁盘空间审计工作流，支持 **Windows** 和 **macOS**（附 Linux 参考）。产出是一份分类清理报告，告诉你空间到底去哪了、什么能清、什么不能碰。审计阶段全程只读，清理是独立的、逐项确认的阶段。

适用于 Devin、Claude Code、Cursor、Codex 等 AI 编码 agent。兼容 [`npx skills`](https://github.com/vercel-labs/skills) —— Vercel Labs 出品的开源 agent skills 生态工具。

## Quick install / 快速安装

### Via `npx skills` (recommended / 推荐)

Supports **70+ agents** — Claude Code, Cursor, Codex, OpenCode, Devin, and more.

支持 **70+ 种 agent** —— Claude Code、Cursor、Codex、OpenCode、Devin 等。

```bash
# Install globally (available across all projects / 全局安装)
npx skills add Haaaiawd/disk-space-audit-skill -g

# Install to specific agents / 安装到指定 agent
npx skills add Haaaiawd/disk-space-audit-skill -a claude-code -a cursor

# Install to current project / 安装到当前项目
npx skills add Haaaiawd/disk-space-audit-skill

# Non-interactive (CI/CD) / 非交互式
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

## Features / 特性

- **Read-only audit** — scan and report never deletes, moves, or modifies files
  - **只读审计** — 扫描和报告阶段绝不删除、移动、修改任何文件
- **Multi-platform** — Windows (WizTree MFT), macOS (disky APFS-aware), Linux (ncdu/dua)
  - **多平台** — Windows 用 WizTree 读 MFT，macOS 用 disky 处理 APFS，Linux 用 ncdu/dua
- **Incremental caching** — WizTree CSV cached for 24h, reused on subsequent scans
  - **增量缓存** — WizTree CSV 缓存 24 小时，后续扫描直接复用
- **Duplicate file detection** — three-level filter (size → 4KB hash → full hash), >50MB only
  - **重复文件检测** — 三级过滤（大小 → 4KB 哈希 → 全文件哈希），仅扫 >50MB 文件
- **Categorized report** — 🟢🟡🟠🔴 four-tier ranking by size × safety
  - **分类报告** — 🟢🟡🟠🔴 四级排序，按大小 × 安全度
- **Per-item cleanup confirmation** — no batch approvals, no ambiguous consent
  - **逐项清理确认** — 不接受批量批准，不接受模糊授权
- **Re-download cost awareness** — states bandwidth/time cost for every cache deletion
  - **重下载成本意识** — 每个缓存删除建议都标注带宽/时间成本
- **Anti-pattern table** — explicitly lists what NOT to delete (uv cache, Docker volumes, etc.)
  - **反模式表** — 明确列出不该删的东西（uv cache、Docker volumes 等）

## Workflow / 工作流

| Phase | Description | 描述 |
|-------|-------------|------|
| 1. Recon | Disk overview, platform detection | 磁盘概览，平台检测 |
| 2. Deep scan | WizTree (Win) / disky (macOS) / ncdu (Linux) | 深度扫描 |
| 2.5. Duplicate detection | Optional, on request only | 重复文件检测（可选，按需触发） |
| 3. Investigate | Web search + content inspection for unknowns | 调查不明目录 |
| 4. Categorize | Group by type, rank by size × safety | 分类排序 |
| 5. Report | Markdown report with consequence column | 生成报告（含删除后果列） |
| 6. Cleanup | Only when explicitly asked, per-item confirmation | 清理（仅显式要求时执行） |

## Prerequisites / 前置依赖

| Platform | Tool | Install | Purpose |
|----------|------|---------|---------|
| Windows | **WizTree** | `winget install AntibodySoftware.WizTree` | MFT direct read, scans 500GB in ~10s |
| macOS | **disky** | `brew install disky` | APFS-aware, handles sparse files |
| Linux | **ncdu** / **dua** | `apt install ncdu` / `cargo install dua-cli` | Parallel scanning |

The skill will auto-install WizTree via winget if missing on Windows.

Windows 上如果没装 WizTree，skill 会自动通过 winget 安装。

## Usage / 用法

After installing, just tell your AI agent something like:

安装后，直接跟你的 AI agent 说：

```
# English
"scan my C drive"
"analyze disk space"
"find what's eating my storage"
"clean up system drive"
"find duplicate files"

# 中文
"扫描 C 盘"
"分析磁盘空间"
"清理 C 盘"
"找重复文件"
```

The agent will run the audit (read-only) and produce a categorized report. Cleanup only starts when you explicitly ask, and every item requires individual confirmation.

Agent 会执行只读审计并生成分类报告。清理操作仅在显式要求时启动，且每项需单独确认。

### Try without installing / 免安装试用

```bash
# Generate a prompt and pipe to claude / 生成 prompt 并传给 claude
npx skills use Haaaiawd/disk-space-audit-skill | claude

# Start a specific agent with the skill / 用指定 agent 启动
npx skills use Haaaiawd/disk-space-audit-skill --agent claude-code
```

## Key design decisions / 关键设计决策

1. **Audit ≠ Cleanup.** The report is the deliverable. Cleanup is a separate, explicitly-confirmed phase. This prevents accidental data loss from "helpful" agents.
   - **审计 ≠ 清理。** 报告是交付物。清理是独立阶段，需显式确认。防止"热心"agent 误删数据。

2. **Speed via MFT.** PowerShell `Get-ChildItem -Recurse` hangs on AppData junctions and million-file caches. WizTree reads the NTFS Master File Table directly — 500GB in 10 seconds (admin mode).
   - **用 MFT 提速。** PowerShell 递归扫描会在 AppData junction 和百万文件缓存上卡死。WizTree 直接读 MFT，500GB 只要 10 秒（admin 模式）。

3. **Value over vanity.** Clearing 50GB of useful cache to show a big number is harmful. Every recommendation weighs re-acquisition cost against space saved.
   - **价值优先。** 为凑数字清掉 50GB 有用缓存是有害的。每条建议都权衡重获取成本和空间收益。

4. **Per-item confirmation, always.** "Clean some stuff" is not authorization. Each deletion gets its own yes/no. New information (app started running, symlink found) resets prior approval.
   - **逐项确认，没有例外。** "清一些"不算授权。每个删除项单独确认。发现新情况（app 在跑、symlink）则重新确认。

## File structure / 文件结构

```
disk-space-audit-skill/
├── SKILL.md      # The skill definition (552 lines)
├── README.md     # This file
├── LICENSE       # MIT
└── .gitignore
```

## Compatibility / 兼容性

This skill follows the [open agent skills format](https://github.com/vercel-labs/skills) — a `SKILL.md` with YAML frontmatter (`name` + `description`) at the repo root. It works with any agent supported by `npx skills`, including:

本 skill 遵循 [开源 agent skills 格式](https://github.com/vercel-labs/skills) —— repo 根目录放一个带 YAML frontmatter（`name` + `description`）的 `SKILL.md`。兼容 `npx skills` 支持的所有 agent：

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
