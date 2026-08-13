# CLAUDE.md — Arch Dev Issues 仓库操作指南

本文件供 Claude Code 在操作此仓库时参考。每次提交/新增 Issue 均需遵循以下规范。

---

## 仓库目的

记录 Arch Linux 开发与运维中遇到的各类问题，每份文档包含：问题现象 → 排查过程 → 根因分析 → 解决方案。

仓库地址: `https://github.com/victcity/arch-dev-issues`

---

## 目录结构

```
arch-dev-issues/
├── CLAUDE.md                    # 本文件 (Claude Code 操作指南)
├── README.md                    # 中英双语总览 + 索引表格
├── LICENSE                      # CC0 1.0
├── issues/
│   └── YYYY-MM-DD-short-slug/
│       ├── README.zh.md         # 简体中文
│       ├── README.en.md         # English (与中文内容一致)
│       └── assets/              # 截图、日志等附件
└── .claude/                     # Claude Code 配置 (自动生成)
```

### 命名规则

- 目录名: `YYYY-MM-DD-` + 短横线连接的英文关键词，如 `2026-08-13-hyprland-global-shortcuts`
- `short-slug` 应简洁描述问题，不超过 5 个单词
- 日期取问题首次记录的日期

---

## 新增 Issue 工作流

### 步骤 1: 创建目录

```bash
mkdir -p issues/YYYY-MM-DD-short-slug/assets
touch issues/YYYY-MM-DD-short-slug/assets/.gitkeep
```

### 步骤 2: 撰写中文文档 (README.zh.md)

必须包含以下章节，按顺序：

1. **标题** — `# 问题简述`
2. **概述** — 2-3 句话说明问题和结论
3. **环境** — 设备型号 / OS 版本 / 软件版本 / 关键配置
4. **背景**（可选）— 相关架构、依赖关系等前置知识
5. **复现步骤** — 精确到命令行的步骤，确保他人可重现
6. **排查过程** — 按时间线记录尝试了什么、发现了什么
7. **根因分析** — 对根本原因的判断（已确认 / 推测）
8. **解决方案** — 明确的修复步骤或 workaround
9. **相关链接** — 上游仓库 / Issue / 文档

### 步骤 3: 撰写英文文档 (README.en.md)

- 内容与中文版一致，章节结构相同
- 确保技术术语翻译准确
- 命令、代码块、日志输出无需翻译
- 表格保持一致格式

### 步骤 4: 更新 README.md 索引表

在 `## 目录 / Index` 下的表格中新增一行：

```markdown
| YYYY-MM-DD | [中文标题](issues/slug/README.zh.md) | 领域标签 | [English](issues/slug/README.en.md) |
```

领域标签示例: `Hyprland / Wayland / keyboard`、`kitty / terminal`、`opencode / TUI`、`systemd / service`

### 步骤 5: 撰写提交信息

格式: `docs: <英文简述> (zh/en)`
