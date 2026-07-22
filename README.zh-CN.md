# 磁盘空间审计 Skill

[![skills.sh](https://skills.sh/b/Haaaiawd/disk-space-audit-skill)](https://skills.sh/Haaaiawd/disk-space-audit-skill)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

**[English](./README.md)**

一个可重复的、面向 AI agent 的磁盘空间审计工作流，支持 **Windows** 和 **macOS**（附 Linux 参考）。产出是一份分类清理报告，告诉你空间到底去哪了、什么能清、什么不能碰。审计阶段全程只读，清理是独立的、逐项确认的阶段。

适用于 Devin、Claude Code、Cursor、Codex 等 AI 编码 agent。兼容 [`npx skills`](https://github.com/vercel-labs/skills) —— Vercel Labs 出品的开源 agent skills 生态工具。

## 快速安装

### 通过 `npx skills`（推荐）

支持 **70+ 种 agent** —— Claude Code、Cursor、Codex、OpenCode、Devin 等。

```bash
# 全局安装（所有项目可用）
npx skills add Haaaiawd/disk-space-audit-skill -g

# 安装到指定 agent
npx skills add Haaaiawd/disk-space-audit-skill -a claude-code -a cursor

# 安装到当前项目
npx skills add Haaaiawd/disk-space-audit-skill

# 非交互式（CI/CD）
npx skills add Haaaiawd/disk-space-audit-skill -g -a claude-code -y
```

### 通过 git clone

```bash
# Devin CLI
git clone https://github.com/Haaaiawd/disk-space-audit-skill.git \
  ~/.devin/skills/disk-space-audit

# Claude Code
git clone https://github.com/Haaaiawd/disk-space-audit-skill.git \
  ~/.claude/skills/disk-space-audit

# 通用 —— clone 到任意位置，让你的 agent 指向 SKILL.md
git clone https://github.com/Haaaiawd/disk-space-audit-skill.git
```

## 特性

- **只读审计** — 扫描和报告阶段绝不删除、移动、修改任何文件
- **多平台** — Windows 用 WizTree 读 MFT，macOS 用 disky 处理 APFS，Linux 用 ncdu/dua
- **增量缓存** — WizTree CSV 缓存 24 小时，后续扫描直接复用
- **重复文件检测** — 三级过滤（大小 → 4KB 哈希 → 全文件哈希），仅扫 >50MB 文件
- **分类报告** — 🟢🟡🟠🔴 四级排序，按大小 × 安全度
- **逐项清理确认** — 不接受批量批准，不接受模糊授权
- **重下载成本意识** — 每个缓存删除建议都标注带宽/时间成本
- **反模式表** — 明确列出不该删的东西（uv cache、Docker volumes 等）

## 工作流

| 阶段 | 描述 |
|------|------|
| 1. Recon | 磁盘概览，平台检测 |
| 2. Deep scan | WizTree (Win) / disky (macOS) / ncdu (Linux) |
| 2.5. 重复文件检测 | 可选，按需触发 |
| 3. Investigate | 调查不明目录（web search + 内容检查） |
| 4. Categorize | 分类排序（按大小 × 安全度） |
| 5. Report | 生成报告（含删除后果列） |
| 6. Cleanup | 仅显式要求时执行，逐项确认 |

## 前置依赖

| 平台 | 工具 | 安装命令 | 用途 |
|------|------|----------|------|
| Windows | **WizTree** | `winget install AntibodySoftware.WizTree` | MFT 直读，500GB 约 10 秒 |
| macOS | **disky** | `brew install disky` | APFS 感知，处理稀疏文件 |
| Linux | **ncdu** / **dua** | `apt install ncdu` / `cargo install dua-cli` | 并行扫描 |

Windows 上如果没装 WizTree，skill 会自动通过 winget 安装。

## 用法

安装后，直接跟你的 AI agent 说：

```
"扫描 C 盘"
"分析磁盘空间"
"清理 C 盘"
"找重复文件"
```

Agent 会执行只读审计并生成分类报告。清理操作仅在显式要求时启动，且每项需单独确认。

### 免安装试用

```bash
# 生成 prompt 并传给 claude
npx skills use Haaaiawd/disk-space-audit-skill | claude

# 用指定 agent 启动
npx skills use Haaaiawd/disk-space-audit-skill --agent claude-code
```

## 关键设计决策

1. **审计 ≠ 清理。** 报告是交付物。清理是独立阶段，需显式确认。防止"热心"agent 误删数据。

2. **用 MFT 提速。** PowerShell 递归扫描会在 AppData junction 和百万文件缓存上卡死。WizTree 直接读 MFT，500GB 只要 10 秒（admin 模式）。

3. **价值优先。** 为凑数字清掉 50GB 有用缓存是有害的。每条建议都权衡重获取成本和空间收益。

4. **逐项确认，没有例外。** "清一些"不算授权。每个删除项单独确认。发现新情况（app 在跑、symlink）则重新确认。

## 文件结构

```
disk-space-audit-skill/
├── SKILL.md          # Skill 定义（552 行）
├── README.md         # 英文文档
├── README.zh-CN.md   # 中文文档（本文件）
├── LICENSE           # MIT
└── .gitignore
```

## 兼容性

本 skill 遵循 [开源 agent skills 格式](https://github.com/vercel-labs/skills) —— repo 根目录放一个带 YAML frontmatter（`name` + `description`）的 `SKILL.md`。兼容 `npx skills` 支持的所有 agent：

| Agent | `--agent` 参数 |
|-------|---------------|
| Claude Code | `claude-code` |
| Cursor | `cursor` |
| Codex | `codex` |
| OpenCode | `opencode` |
| Devin | `devin` |
| Amp / Replit / Universal | `amp` / `replit` / `universal` |
| ...还有 65 种 | [完整列表](https://github.com/vercel-labs/skills#supported-agents) |

## 开源协议

MIT — 见 [LICENSE](LICENSE)。
