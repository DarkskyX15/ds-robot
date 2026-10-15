# pi0.5 远程异步推理 — 仓库检查报告与交接文档

- 日期：2026-10-15
- 检查对象：`/home/darksky/Desktop/code/XLeRobot`（origin: `https://github.com/JoyandAI/XLeRobot.git`，文档上游为 `Vector-Wangel/XLeRobot`）
- 检查目标：梳理"远程异步推理"相关代码；评估部署 pi0.5 远程推理的现成资产与改造点
- **更新（2026-10-15）**：已确定从 `xuweiwu/XLeRobot` fork、以 `pi05_dual_arm` 为工作基线，详见「二.0 工作基线决策」

---

## 一、检查报告

### 1. 结论（TL;DR）

1. **本仓库没有 pi0.5 推理实现**。"远程异步推理"在本仓库分两部分：
   - 文档：指引 LeRobot 官方 `async_inference`（gRPC，端口 8080）以及社区 fork 的 pi0.5 方案；
   - 基础设施：ZMQ host/client 远程 IO（动作下发、观测回传），可直接复用。
2. **最接近"开箱可用"的是 fork `xuweiwu/XLeRobot@pi05_dual_arm`**：
   - 现成推理客户端 `9_pi05_inference_dualarm.py`（openpi WebSocket，端口 8000）；
   - 配套：关节插值器、placo IK、相机低延迟补丁、改造版驱动、双臂 VR 数据采集脚本。
3. **主要缺口**：openpi 服务端训练配置。docs 提到的 `xuweiwu/openpi` 分支 `biso101 training_support` 目前公开仓库已不可见（仅 main）；需按客户端观测键自定义 policy config 并训练 pi0.5。

### 2. 主仓库关键文件索引（远程通信 / 文档）

| 文件 | 内容 |
|---|---|
| `docs/zh/source/software/getting_started/VLA_ACT.md:167-177` | LeRobot 异步推理流程：GPU 机跑 `python -m lerobot.async_inference.policy_server --host=0.0.0.0 --port=8080`，机器人端跑客户端 |
| `docs/zh/source/software/getting_started/LLM_agent.md:196-236` | RoboCrew 工具 `create_vla_single_arm_manipulation`（异步策略客户端），服务端启动命令 |
| `docs/{en,zh}/source/software/getting_started/VLA_pi05.md` | 指向 fork `xuweiwu/XLeRobot`（分支 `pi05 dual arm`，实际为 `pi05_dual_arm`）与 `xuweiwu/openpi`；数据集 `xuweiwu/bimanual-toy-box-cleanup` |
| `software/src/robots/xlerobot/xlerobot_host.py` | 机器人侧：PULL 5555 收动作 / PUSH 5556 发观测（base64 JPEG + JSON），500ms 看门狗停底盘，30Hz 限频 |
| `software/src/robots/xlerobot/xlerobot_client.py` | 控制侧：实现 LeRobot `Robot` 接口，PUSH 发动作 / PULL 收观测 |
| `software/src/robots/xlerobot/config_xlerobot.py:87-130` | Host/Client 配置（端口 5555/5556、watchdog、`remote_ip`） |
| `software/src/robots/xlerobot_sa/`、`xlerobot_2wheels/` | 单臂 / 两轮差速版本，同一 host/client 模式 |
| `simulation/Maniskill/run_xlerobot_sim_host.py` | 仿真 host，协议与真机一致，可无硬件联调 |
| `web_control/server/core/remote_core.py` | `zmq.asyncio` 异步远程核心（心跳 + 数据接收循环）参考实现 |
| `software/install_xlerobot.sh` | 把驱动拷入 `~/lerobot/src/lerobot/robots/` |

### 3. fork 资产索引（`xuweiwu/XLeRobot@pi05_dual_arm`）

临时克隆路径：`/tmp/xlr-pi05`（sparse：`software/` + `docs/`）；对比上游 main 共 7 个提交、16 个文件变更。

| 文件 | 作用 / 关键点 |
|---|---|
| `software/examples/9_pi05_inference_dualarm.py` | **远程推理客户端**。`:69` `OPENPI_SERVER_IP`；`:398` `WebsocketClientPolicy(host, port=8000)`；`:426-438` 观测组装（`image/head`、`image/left_wrist`、`image/right_wrist` 缩放到 224² + 12 维 `state` + `prompt`）；`:442` `client.infer()["actions"]`；`:444-467` `action_horizon=50`、`FPS=50`、P 控制 kp=0.75 执行；`:416-419` 键盘 `^` 急停；episode 间用插值回 `FULL_START_POS`（`:50-63`） |
| `software/src/utils/quadratic_spline_via_ipol.py` | 二次样条关节插值器（限速/限加速度、`scale_to_duration`、`resample`） |
| `software/src/model/kinematics.py` | placo FK/IK（含正则项），采集时做 EE 转换 |
| `software/src/robots/so100_follower/robot_kinematic_processor.py` | `EEReferenceAndDelta` / `InverseKinematicsEEToJoints` 等处理器 |
| `software/src/cameras/opencv/camera_opencv.py`、`realsense/camera_realsense.py` | 延迟补丁：`async_read` 直接取最新帧（对应 HF lerobot PR #2695） |
| `software/src/robots/xlerobot/xlerobot.py`、`config_xlerobot.py` | 改造驱动：`bus_left_base`（左臂+底盘）/ `bus_right_head`（右臂+头 `head_pan`/`head_tilt`）；udev 串口名；相机 `head`(RealSense)/`left_wrist`/`right_wrist`(OpenCV MJPG)；`use_degrees=True` |
| `software/examples/8_vr_teleop_with_dataset_recording_dualarm.py` | 双臂 VR 采集（依赖 `XLeVR` 与上述处理器） |
| `software/src/record.py` | lerobot 0.4 兼容（`init_rerun`） |

### 4. 相关上游一览

| 角色 | 仓库 |
|---|---|
| 机器人主线 | `Vector-Wangel/XLeRobot`（本地 checkout 为 `JoyandAI/XLeRobot`） |
| 机器人侧 pi0.5 实现 | `xuweiwu/XLeRobot` @ `pi05_dual_arm` |
| 策略训练/服务端 | `Physical-Intelligence/openpi`（`serve_policy.py` WebSocket :8000；`packages/openpi-client`）；fork `xuweiwu/openpi`（SO-101 训练支持分支未公开） |
| 异步推理框架（备选） | `huggingface/lerobot`（`lerobot.async_inference.policy_server` :8080；pi05 策略） |
| 异步客户端封装 | `Grigorij-Dudnik/RoboCrew` |
| 训练数据 | HF `xuweiwu/bimanual-toy-box-cleanup` |

### 5. 环境检查

- 本机当前 Python 环境**未安装 lerobot**（`pip show lerobot` 为空）。
- `/tmp/opencode` 属 root、不可写；本次使用 `/tmp/xlr-pi05`（易失，重启清空）。
- GitHub 下载需代理 `http://127.0.0.1:7890`（已验证可用）。
- 检查阶段**未修改本仓库任何文件**；仅创建 `/tmp/xlr-pi05` 临时克隆，以及本交接文档（仓库根目录新增文件）。

### 6. 风险与注意事项

1. **接线差异**：fork 驱动将总线重命名为 `bus_left_base` / `bus_right_head`，与上游布线（bus1=左臂+头，bus2=右臂+底盘）不同。沿用上游接线时需把示例中的 bus 名改回 `bus1`/`bus2`。
2. **观测契约**：相机键（`head`/`left_wrist`/`right_wrist`）、`state` 顺序（`FULL_START_POS` 顺序）、角度单位（`use_degrees=True`）必须与训练数据一致，否则策略失效。
3. **服务端缺口**：需自定义 openpi policy config 与 norm stats；`biso101 training_support` 分支需向 fork 作者确认或自行复现。
4. **端口/网络**：机器人↔GPU 机需可达 TCP 8000（openpi）或 8080（LeRobot async）。
5. **临时副本**：`/tmp/xlr-pi05` 应及时持久化（工作 fork 建立后可删除）。
6. **fork 继任关系**：工作 fork 的父仓库是 `xuweiwu/XLeRobot`（个人 fork），官方更新需自行从 `Vector-Wangel/XLeRobot` 同步；该 pi0.5 分支后续是否继续维护由原作者决定，必要时联系其获取 openpi 训练配置（原 `biso101 training_support` 分支）。

---

## 二、交接文档

### 0. 工作基线决策（2026-10-15）

- **决定**：从 `xuweiwu/XLeRobot` fork 一份仓库，工作分支基于其 `pi05_dual_arm` 分支。
- **理由**：pi0.5 全套改动（7 个提交 / 16 个文件）已在该分支，直接从基线开发，避免自行搬运；同时通过 `upstream` remote 保留跟进官方主线的能力。
- **工作 fork（待创建）**：`https://github.com/<your-account>/XLeRobot`
- **基线提交**：`2970ced`（`pi05_dual_arm` HEAD，基于 `Vector-Wangel/XLeRobot` main `b017b5e`）
- **remote 约定**：
  - `origin` → 你的 fork：`https://github.com/<your-account>/XLeRobot`
  - `upstream` → 官方主线：`https://github.com/Vector-Wangel/XLeRobot`（只 fetch）
  - `xuweiwu` → pi0.5 来源：`https://github.com/xuweiwu/XLeRobot`（只 fetch，跟踪其后续更新）
- **同步策略**：日常在 `feat/pi05-async-inference` 分支开发；需要时 `git fetch upstream && git merge upstream/main`（注意与驱动/相机 diff 的冲突）。GitHub 的 fork 关系挂在 xuweiwu 名下，不影响上述同步与使用。

建仓命令（走代理）：

```bash
# 1) GitHub 网页上 fork https://github.com/xuweiwu/XLeRobot（默认复制全部分支，含 pi05_dual_arm）
# 2) 克隆自己的 fork（blob 过滤 + 稀疏，缩小体积）
git -c http.proxy=http://127.0.0.1:7890 clone --branch pi05_dual_arm \
  --filter=blob:none --sparse https://github.com/<your-account>/XLeRobot.git
cd XLeRobot
git sparse-checkout set software XLeVR docs        # 需要硬件/仿真时: git sparse-checkout add hardware simulation
git switch -c feat/pi05-async-inference
# 3) remotes 与代理（代理仅配置到本仓库）
git remote add upstream https://github.com/Vector-Wangel/XLeRobot.git
git remote add xuweiwu  https://github.com/xuweiwu/XLeRobot.git
git config http.proxy http://127.0.0.1:7890        # 不需要时: git config --unset http.proxy
```

> 若团队后续要求把成果并回 `JoyandAI/XLeRobot` 或官方上游：从本工作 fork 按需提取提交（`git format-patch` / cherry-pick）即可，不影响本基线。

### 1. 交付物

- 本报告（检查结论 + 文件索引 + 风险 + 工作基线决策）。
- **工作 fork（待创建）**：`https://github.com/<your-account>/XLeRobot`，从 `xuweiwu/XLeRobot` fork，工作分支基于 `pi05_dual_arm@2970ced`。
- 参考克隆：`/tmp/xlr-pi05`（上游只读参考，工作 fork 建立后可删除）。
- fork 文件 → lerobot 安装位置映射表（见下）。

### 2. 文件映射表（fork → lerobot 安装目录）

| fork 文件 | 目标位置 |
|---|---|
| `software/src/robots/xlerobot/*.py` | `lerobot/robots/xlerobot/` |
| `software/src/model/kinematics.py` | `lerobot/model/kinematics.py` |
| `software/src/utils/quadratic_spline_via_ipol.py` | `lerobot/utils/quadratic_spline_via_ipol.py` |
| `software/src/robots/so100_follower/robot_kinematic_processor.py` | `lerobot/robots/so100_follower/robot_kinematic_processor.py` |
| `software/src/cameras/opencv/camera_opencv.py` | 覆盖 `lerobot/cameras/opencv/camera_opencv.py` |
| `software/src/cameras/realsense/camera_realsense.py` | 覆盖 `lerobot/cameras/realsense/camera_realsense.py` |
| `software/examples/*.py` | 任意目录运行（依赖 lerobot、XLeVR、openpi_client） |
| `packages/openpi-client`（来自 openpi 仓库） | `pip install -e` 安装 `openpi_client` |

### 3. 后续步骤（建议顺序）

1. **创建/克隆工作 fork**：按「二.0」执行（fork → 克隆 → 建 `feat/pi05-async-inference` → 配 remotes）。
2. **GPU 服务端（openpi）**：`GIT_LFS_SKIP_SMUDGE=1 uv sync` → 定义与客户端观测键对齐的 SO-101 双臂 policy config → 训练 pi0.5 → `uv run scripts/serve_policy.py policy:checkpoint --policy.config=<config> --policy.dir=<ckpt>`（:8000）。
3. **机器人侧**：
   - 安装 `openpi_client`；按映射表拷贝文件进 lerobot；
   - 按实际硬件修改 `config_xlerobot.py`（相机设备、串口、bus 命名、`use_degrees`）；
   - 修改 `9_pi05_inference_dualarm.py:69` 的 `OPENPI_SERVER_IP`。
4. **联调验收**：
   - 服务端 8000 可连通；策略返回 12 维 action chunk；
   - 急停键 `^` 生效；episode 间插值回位正常；50Hz 执行无明显卡顿；
   - 可选：先用 `simulation/Maniskill/run_xlerobot_sim_host.py` 或 `xlerobot_host.py` 独立验证远程 IO。
5. **同步与回馈（可选）**：`git fetch upstream && git merge upstream/main` 保持与官方主线同步；通用性改进可整理成精简提交向 `Vector-Wangel/XLeRobot` 提 PR。

### 4. 回退方案（路线 A：LeRobot 官方 async_inference）

- 将 `software/src/robots/xlerobot*` 拷入 lerobot 0.4.x；
- 在 LeRobot 内训练 pi0.5（`--policy.type=pi05`）；
- GPU 机：`python -m lerobot.async_inference.policy_server --host=0.0.0.0 --port=8080`；
- 机器人端使用 lerobot async robot_client（参见 `docs/.../VLA_ACT.md:167-177`）。

### 5. 速查

- 工作基线：`xuweiwu/XLeRobot@pi05_dual_arm`（HEAD `2970ced` = `Vector-Wangel/XLeRobot` main `b017b5e` + 7 个提交）。
- fork 分支实际名：`pi05_dual_arm`（docs 中写 "pi05 dual arm"）。
- 关键端口：openpi WebSocket 8000；LeRobot async gRPC 8080；ZMQ 5555/5556。
- 关键参数：`action_horizon=50`、`FPS=50`、P 增益 `0.75`、回位插值 `200Hz`。
