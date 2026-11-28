# John Blog (AstroPaper v6 / Astro 7) — AI Agent Guidelines

## 1. Development Environment & Single Source of Truth (Debian 13 VM)

- **唯一代码真源**：本项目唯一开发与 Git 真源位于 `debian-dev`（Debian 13 `ext4`）`/opt/stacks/john`（已停用 Windows 侧同步，严禁在 Windows 本地修改）。
- **连接与预览**：
  - 人类通过 IDE `Remote-SSH` 连接 `debian-dev:/opt/stacks/john` 开发；
  - 容器 `john-web` 直接挂载 `/opt/stacks/john` 提供实时热更新预览（Vite / Astro Dev Server）。
- **宿主机可用工具链**：
  - Node.js `v26.13.1` / npm `11.20.0`、`git`（已配 GitHub SSH）、`rg`、`fd`、`jq`、`ast-grep (sg)`、`agy` CLI（`/usr/local/bin/agy`）。
- **提交前质量门禁**：
  ```bash
  npm run format:check && npm run lint && npm run build
  ```

---

## 2. Default Review Gate & Codex / agy Workflow

1. **默认 `REVIEW` 模式**：未收到人类明确固定口令 `确认执行方案X第N步` 前，禁止创建、修改、删除任何文件或执行 Git 提交/推送/容器重启。
2. **变更卡与可复制审批口令**：任何写操作前必须先输出方案编号、目标、文件白名单、非目标、命令白名单、风险、验证方式与停止点，并以独立 `text` 代码块给出具体的 `确认执行方案X第N步`。
3. **单写入者边界（如启用 Codex + agy 协同）**：Codex 负责只读评审与独立验收，统一通过 `agy -p --mode accept-edits --model gemini-4-hydrogen-medium --effort medium --print-timeout 10m < prompt.txt` 调度 `agy` 执行写入，严禁手写 JS/Node 流式包装脚本。
