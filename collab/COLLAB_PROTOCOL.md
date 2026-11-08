# Claude ↔ Codex 协作规范（所有项目通用）

位置：Dropbox `/Workspace/Claude-Codex 协作/`。每个项目一个独立文件夹：`<短名>-<来源ID>/`（如 `RAX-2610.22496/`）。

## 角色
- **Claude（规划/审查）**：读论文，定范围、计划、任务书与验收标准；审查 Codex 的产出，写 review；决定进入下一阶段。
- **Codex（执行）**：按任务书实现、运行实验、记录结果；遇到阻塞写明阻塞点而不是绕过；不擅自改验收标准。

## 项目文件夹结构
| 路径 | 谁写 | 内容 |
|---|---|---|
| `README.md` | Claude | 项目卡片：论文、目标、代码仓库、当前阶段 |
| `PLAN.md` | Claude | 总体复现计划、阶段、验收标准、预算 |
| `tasks/TASK-XX_<名>.md` | Claude | 单个阶段的任务书（Codex 的输入） |
| `status/YYYY-MM-DD_TASK-XX_codex.md` | Codex | 进度报告：做了什么、命令、结果、阻塞、下一步 |
| `reviews/YYYY-MM-DD_TASK-XX_claude.md` | Claude | 审查意见：通过/打回 + 具体修改项 |
| `results/` | Codex | 指标表（CSV/JSON）、图、日志摘要 |

> Dropbox 工具不能原地覆盖文件 → **只追加新文件，不改旧文件**；同名需更新时加后缀 `_v2`。

## 代码
- 代码放 GitHub 仓库（本项目：`micsnq2yoma-lgtm/RAX`），Dropbox 只放文档和结果摘要。
- 每个 TASK 一个分支或一组 commit，commit message 以 `[TASK-XX]` 开头。

## 一次循环
1. Claude 写 `tasks/TASK-XX`。
2. Codex 执行 → 推代码 → 写 `status/..._TASK-XX_codex.md`（必须包含：复现命令、关键数字、与验收标准逐条对照）。
3. Claude 审查 → 写 `reviews/..._TASK-XX_claude.md`：`PASS` 进入下一个 TASK，`REVISE` 列出修改项。

## 状态报告模板（Codex）
```
# TASK-XX 状态 — YYYY-MM-DD
结论: DONE / PARTIAL / BLOCKED
## 完成内容
## 复现命令
## 结果（数字/表格）
## 验收标准对照（逐条 ✅/❌ + 证据）
## 阻塞与问题
## 花费（API $、算力小时）
```
