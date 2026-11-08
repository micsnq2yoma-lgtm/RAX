# RAX 复现计划（arXiv:2610.22496）

**论文**：Scaling Online Exploration for Robot Agents（RAX, Robot Agent eXploration）— Harvard / NVIDIA / UC Berkeley / MIT / Stanford，2026-10-15。项目页 robot-agent-exploration.com
**分工**：Claude 规划与审查，Codex 执行。协作规范见 `../COLLAB_PROTOCOL.md`。
**代码仓库**：GitHub `micsnq2yoma-lgtm/RAX`

## 1. 论文要点（复现所需）
- **核心思路**：冻结权重的通用编码 agent（论文用 Claude Code + Claude Opus 5.5；部分任务用 Codex + GPT-6，xhigh）在有限“探索 episode 预算”内，通过工具直接操控机器人，成功后把决策过程编译成**完整的感知-规划-控制 Python 程序**；评测时程序**脱离 agent 独立运行**在未见场景上。无动作示范、无手工原语、无策略训练。
- **前后端分离**：backend = 模拟器/控制器/传感器/安全检查；frontend = agent 工作区。二者只通过工具接口（MCP）交互；agent 的程序调用的就是 agent 自己用的同一套工具 → 交互代码可原样作为自动程序运行。
- **工作区**：一个指令文件 + skills 文档（robot-agnostic 7 个：manual control、in-context action learning、experience memory、autonomous control、transfer hand-off、multi-env、multi-agent；外加感知/控制层）。agent 从空工作区开始，不带跨会话记忆。
- **指令流程**：读任务 → (1) 用工具手动完成并确认成功 → (2) 写成带前置条件检查/结果验证/`raise Unsure()` 的程序 → (3) 后续每个探索 episode 都跑程序、出问题再修 → (4) 维护 notes/代码/hand-off → 冻结 bundle → rehearsal（无人值守跑一次新场景，消耗 1 个 episode）→ 提交。
- **协议**：单一账本统计所有 episode（含 rehearsal）；探索场景来自训练分布、顺序固定；评测用官方评测集，探索从不使用；评测结果不回传给 agent；错误/超时=失败。agent 容器 16GB/4CPU/无网络；程序运行容器 8GB/4CPU/无网络；可调用 VLM（Gemini Flash），每会话 $10。
- **LIBERO 设置（我们的主目标）**：Franka Panda；动作 = 官方 delta EEF（1 ↔ 5cm / 0.5rad）+ 夹爪，20Hz，每次调用 1–100 行，无运动规划器；观测 agentview + wrist RGB-D 256×256 + 内参/相机位姿 + EEF 位姿 + 手指位置，可像素→3D；步数上限 Spatial 220 / Object 280 / Goal 300 / Long 520（另加 10 步静止）。任务描述 = 官方指令 + 机器人/相机说明 + 步数限制 + 二元成功标准（不含检查细节）+ 3 张官方示范的终帧。预算 ≤20 探索 episode（官方放置分布 fresh reset，非存储初始状态）+ 可选 ≤5 次 LIBERO-Pro 位置扰动适配（seed 从 2000 起）。每 episode 限时 600s + 5s/步。评测：每任务 50 个官方初始状态 + 50 个 LIBERO-Pro 位置扰动状态，二元成功。论文 LIBERO 用 2 个并行环境。

### 目标数字（论文 Table 6）
| | Spatial | Object | Goal | Long | Avg |
|---|---|---|---|---|---|
| LIBERO（RAX） | 99.2 | 100.0 | 98.6 | 96.2 | 98.5 |
| LIBERO-Pro 位置扰动，无适配 | 79.2 | 100.0 | 73.4 | 51.2 | 76.0 |
| LIBERO-Pro 位置扰动，≤5 次适配 | 79.2 | 100.0 | 87.0 | 71.4 | 84.4 |
参照：VPP2 在 LIBERO-Pro 上 42.3；ASPIRE 63.2。

## 2. 复现范围
| 层级 | 内容 | 决定 |
|---|---|---|
| **T1 主目标** | LIBERO 4 个 suite + LIBERO-Pro 位置扰动，完整协议（账本、rehearsal、隔离评测） | 做。先 8 个任务试点（每 suite 2 个），通过后扩到 40 个 |
| **T2 消融** | (a) 探索预算缩放 5/10/20；(b) depth vs RGB-only；(c) 1 vs 2 并行环境；(d) 随探索进行的 token/时延下降 | 在试点任务上做 |
| **T3 不做** | RoboDojo、BEHAVIOR-1K、HumanoidArena、DexJoCo、真机 Franka/RB-Y1 | 需要 GPU 渲染/专有模拟器/真机；论文全部实验 API 花费 >10 万美元。T1 通过后再评估 RoboDojo |

**成功标准（复现判定）**：试点 8 任务在 LIBERO 官方初始状态上平均 ≥90%，LIBERO-Pro 位置扰动无适配 ≥60%（论文 98.5 / 76.0，考虑到我们用的 agent 模型与论文不同，允许差距）；协议无泄漏（评测状态从未用于探索，经审计）。

## 3. 阶段与任务书
| TASK | 内容 | 产出 | 验收 |
|---|---|---|---|
| **00** | 调研：项目页/是否已开源代码（`auto_research_agent/`、`agent_skills/`）、LIBERO-Pro 获取方式、环境搭建 | `status/` 报告 + 环境可跑 | LIBERO 4 suite 能 reset/step/渲染 RGB-D；LIBERO-Pro 扰动状态可加载；若官方代码已开源，计划改为“跑官方代码”，Claude 会重写后续任务书 |
| **01** | Backend：LIBERO 工具服务（MCP server）+ 安全/格式校验 | `rax/backend/libero_server.py` 等 + 单元测试 | 下列工具全部可用并有测试：`reset_episode`、`get_observation`（RGB-D/内参/外参/EEF/手指）、`pixel_to_3d`、`act`（1–100 行 delta 动作）、`check_success`、`budget_status`；步数/时间上限强制执行 |
| **02** | 协议：episode 账本、固定顺序探索 seed、bundle 冻结 + rehearsal runner、隔离评测 runner（每 episode 新进程/容器，无网络） | `rax/protocol/*` | 账本无法被 agent 绕过；rehearsal 计入预算；评测 runner 只读官方 50 init states；评测结果不写入 agent 工作区 |
| **03** | Frontend：工作区模板（指令文件 + 7 个 robot-agnostic skills + LIBERO 感知/控制 skill + 接口契约）、会话启动脚本 | `rax/workspace_template/`、`scripts/run_session.sh` | 用 Codex CLI 和 Claude Code 都能一键起会话；agent 只能通过工具接触模拟器 |
| **04** | 试点：8 个任务各跑一个完整会话（预算 20+5）并评测 | `results/pilot.csv`、每任务 hand-off + 最终程序 | 达到第 2 节成功标准；记录每会话 API 花费、时长、token |
| **05** | 扩展到 40 任务 | `results/libero_full.csv` | 对照 Table 6 |
| **06** | 消融 T2 | `results/ablations/` | 曲线/表格 + 结论 |

参考 skills 和指令内容见论文 §9、Table 12；LIBERO 细节见 §10.7、§11.1、§11.4。

## 4. 预算与风险
- **API 花费是最大风险**：每会话可能数十美元级。TASK-04 前需用户确认每会话上限和总上限（建议：试点总额先封顶，由用户定）。
- **Agent 模型差异**：论文 LIBERO 用 Claude Opus 5.5。我们记录实际使用的 agent/模型/推理强度，结论里说明。
- **VLM 依赖**：论文程序可调 Gemini Flash；LIBERO 上大概率不需要。TASK-03 先不给 VLM，若试点卡在识别再加。
- **网络**：本仓库的云会话默认网络受限；Codex 本地运行时需要能访问 PyPI/GitHub/HuggingFace（LIBERO 资源）。
