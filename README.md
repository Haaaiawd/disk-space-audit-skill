# Disk Space Audit Skill

A repeatable, agent-native workflow for auditing disk space on **Windows** and **macOS** (with Linux references). Produces a categorized cleanup report that tells you exactly where your bytes went and what's safe to clean. Cleanup is a separate, explicitly-confirmed phase — the audit itself never touches anything.

Designed for AI coding agents (Devin, Claude Code, Cursor, etc.) that need a structured, safe approach to disk space analysis.

---

# 磁盘空间审计 Skill

一个可重复的、面向 AI agent 的磁盘空间审计工作流，支持 **Windows** 和 **macOS**（附 Linux 参考）。产出是一份分类清理报告，告诉你空间到底去哪了、什么能清、什么不能碰。审计阶段全程只读，清理是独立的、逐项确认的阶段。

适用于 Devin、Claude Code、Cursor 等 AI 编码 agent。

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

## Installation / 安装

### For Devin CLI users

```bash
# Clone to your skills directory
git clone https://github.com/<your-username>/disk-space-audit-skill.git \
  ~/.devin/skills/disk-space-audit
```

### For other AI agents (Claude Code, Cursor, etc.)

Clone anywhere and point your agent to `SKILL.md`:

```bash
git clone https://github.com/<your-username>/disk-space-audit-skill.git
```

Then reference the `SKILL.md` path in your agent's skill configuration.

## Prerequisites / 前置依赖

| Platform | Tool | Install | Purpose |
|----------|------|---------|---------|
| Windows | **WizTree** | `winget install AntibodySoftware.WizTree` | MFT direct read, scans 500GB in ~10s |
| macOS | **disky** | `brew install disky` | APFS-aware, handles sparse files |
| Linux | **ncdu** / **dua** | `apt install ncdu` / `cargo install dua-cli` | Parallel scanning |

The skill will auto-install WizTree via winget if missing on Windows.

## Usage / 用法

Just tell your AI agent something like:

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
disk-space-audit/
├── SKILL.md      # The skill definition (552 lines)
├── README.md     # This file
└── LICENSE       # MIT
```

## License

MIT — see [LICENSE](LICENSE).
