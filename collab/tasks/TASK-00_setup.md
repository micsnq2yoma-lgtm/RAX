# TASK-00：调研 + 环境搭建（Codex 执行）

**背景**：见 `../PLAN.md`。复现 RAX 在 LIBERO / LIBERO-Pro 上的结果。本任务只做调研和环境，不写 agent 逻辑。

## 要做
1. **查官方代码**：访问 https://robot-agent-exploration.com 和 https://arxiv.org/abs/2610.22496 ，查找 GitHub 仓库（论文提到目录 `auto_research_agent/`、`agent_skills/`）。如果找到：记录 URL、license、commit、包含哪些 benchmark 集成，以及 LIBERO 部分能否直接运行。**找到后先停下报告**，不要继续第 3 步之后的定制工作。
2. **查 LIBERO-Pro**（Zhou et al., 2025）：仓库地址、位置扰动（objects exchange initial regions）的 50 个评测状态如何生成/加载。
3. **搭环境**（Python 3.10+，CPU 可跑，MuJoCo/robosuite 无头渲染用 `MUJOCO_GL=egl` 或 `osmesa`）：
   - 安装 LIBERO，下载资源；在 `libero_spatial/object/goal/10(long)` 每个 suite 跑一次：reset → 10 步零动作 → 随机动作 20 步，保存 agentview + wrist 的 RGB 和 **depth**（需开启 camera depth），256×256。
   - 取出相机内参和相机位姿（world/base frame），写一个 `pixel_to_3d(u, v, depth, K, T)` 并用一个已知物体位置（从 sim state 读取）验证误差 < 1 cm。
   - 加载 LIBERO-Pro 位置扰动的一个初始状态并渲染。
   - 确认官方动作尺度：delta 1.0 ↔ 5 cm / 0.5 rad；各 suite 步数上限 220/280/300/520。
4. 把以上写成 `scripts/check_env.py`（一条命令跑完所有检查），提交到 `micsnq2yoma-lgtm/RAX`，commit 以 `[TASK-00]` 开头。

## 产出
- 代码：`scripts/check_env.py`、`requirements.txt` 或 `environment.yml`、`docs/setup.md`
- Dropbox：`status/YYYY-MM-DD_TASK-00_codex.md`（按协作规范模板），附 4 张 suite 截图（RGB+depth）放 `results/task00/`

## 验收标准
- [ ] 报告中明确：官方代码是否开源（含证据链接）
- [ ] LIBERO-Pro 获取方式与扰动状态加载方式已确认
- [ ] `python scripts/check_env.py` 在干净环境一次通过，打印 4 个 suite 的检查结果
- [ ] depth 为米制，`pixel_to_3d` 验证误差 < 1 cm（给出数字）
- [ ] 记录机器配置、安装耗时、遇到的坑

## 不要做
- 不要写 agent/MCP/协议代码（TASK-01/02 的范围）
- 不要使用 LIBERO 的人工示范动作数据做任何训练（只允许取示范终帧图像作为任务图像，留到 TASK-03）
