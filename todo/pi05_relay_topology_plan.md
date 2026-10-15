# pi0.5 远程推理：Relay 拓扑方案 · 仓库现状 · 代价备忘

- 日期：2026-10-15
- 关联文档：`pi05_async_inference_handover.md`（仓库根目录，检查报告与交接文档）
- 状态：方案备忘（未实施），供后续排期与实施参考

---

## 一、想法：Relay（中继）拓扑

### 1.1 架构

```
[机器人板载电脑/Pi]            [Laptop 中继]                          [GPU 机/云端]
XLerobot 驱动 + ZMQ Host  ←ZMQ→  ZMQ Client（收观测/发动作）           openpi Host
  5555(动作) / 5556(观测)        + 策略执行逻辑（P 控制/插值/急停）      = serve_policy + pi0.5
  30Hz 环 / 看门狗 500ms         + openpi Client（发观测/收动作块） ←WS→  (:8000)
```

要点：

- **机器人端只跑 ZMQ Host**（`xlerobot_host.py`），不装 lerobot 之外的重依赖，不直连 GPU。
- **Laptop 是中继+执行器**：从 ZMQ 拿观测 → 预处理（224² 图像、12 维 state、prompt）→ 通过 openpi Client 发给云端 → 拿到动作块后在本地做 P 控制（kp=0.75）与插值回位 → 经 ZMQ 发动作流给机器人。
- **云端跑 openpi Host（pi0.5）**：`uv run scripts/serve_policy.py policy:checkpoint --policy.config=pi05_bimanual_so101_lora --policy.dir=<ckpt>/19999`。
- 角色澄清：**openpi Client 自身没有机器人控制能力**（纯推理 RPC）；控制逻辑在 Laptop 的执行代码里。ZMQ 路径的"完整控制" = Host（控硬件）+ Client（发指令）成对出现。

### 1.2 动机/收益

- 机器人带板载电脑、可自由移动，不需要与执行机 USB 直连。
- 所有"大脑"（遥操作、web_control、pi0.5、LLM agent）共用同一条机器人接入通道（ZMQ），支持手动接管与策略热切换。
- 云端 GPU 可换/可扩，机器人侧保持轻量。

---

## 二、可行性评估与改造点

**结论：可行**（即"统一"落地的具体形态），但需以下改造/权衡：

| # | 项目 | 说明 | 处理 |
|---|---|---|---|
| 1 | 代码改造 | `9_pi05_inference_dualarm.py` 现直接读 `robot.bus_*`（`sync_read`）、`get_camera_observation()`；走 ZMQ 需改用 `XLerobotClient` 的 `get_observation()` / `send_action()` | 中继脚本可基于 9 号示例改造，或新写 |
| 2 | 频率错配 | ZMQ host 环 `max_loop_freq_hz=30`；命令 CONFLATE=1 只保留最新 → 50Hz 执行实际被压到 ≤30Hz | 接受降频（P 控制容忍）或把 host 调 50Hz（Pi CPU 需实测） |
| 3 | 观测新鲜度 | 经 ZMQ 拿"最新一帧"（≤33ms 粒度 + 网络 + JPEG 解码），比直连多几十 ms | 对 1 秒/chunk 的节奏影响小；实测确认 |
| 4 | 失败模式 | 云端断连时 Laptop 卡在 `infer()`；ZMQ 看门狗 500ms 停底盘（手臂保持最后目标位） | 保留键盘急停 + 建议机器人侧物理断电 |
| 5 | 云端链路 | openpi WS 默认明文、无 TLS；公网直连有隐私/安全风险 | 内网优先；公网需 VPN 或 wss 反代；可选 api_key |
| 6 | 控制权 | CONFLATE/PUSH 语义下单控制方 | 遥操作与推理切换需停止一方；同刻只一个在发动作 |

**推荐场景**：机器人必须自由移动（板载 Pi、不拖线）、或需要远程操控。

**不推荐场景**：Laptop 能 USB 直连机器人时——"本地硬件 + 云端 openpi"更简单、延迟最低、图像无损。

---

## 三、代价与优化：ZMQ 路径的压缩

### 3.1 带宽账（3 相机、640×480、30fps）

| 方案 | 单帧 | 三路上行带宽 | 结论 |
|---|---|---|---|
| 原始 RGB | ~920KB | **~650 Mbps** | WiFi 完全不可行 |
| 现状：JPEG q90 + base64 + JSON | ~30–60KB | ~22–43 Mbps | 能跑，但吃满 WiFi、抖动敏感 |
| 优化：320×240 q85、15fps | ~10–20KB | **~4–7 Mbps** | 舒适 |
| 进一步：224² q85、10fps | ~5–8KB | ~1.5–2.4 Mbps | 绰绰有余（策略输入本就是 224²） |

结论：**ZMQ 路径必须压缩**（现状已做 JPEG q90，方向正确），且值得再优化。云链路（Laptop→GPU）无需额外压缩：openpi 协议已把图缩到 224² 原始 uint8，约 0.45MB/s。

### 3.2 优化清单（按性价比）

1. **降分辨率**（最大收益）：host 编码前缩到 320×240 或直接 224²。
2. **降视频帧率**：状态保持 30Hz，视频 10–15fps（host 加计数器即可）；policy 每 ~1s 才查一次。
3. **质量**：q90 → q80~85。
4. **去 base64+JSON**（可选）：换 msgpack/二进制帧，减 33% 体积与 CPU；不改协议也可接受。
5. 保留现状的优点：CONFLATE=1 不积压；压缩带来 5–15ms 编码延迟，可接受。

### 3.3 其他代价

- 多一跳：LAN 下 +30–60ms 往返；公网部署延迟叠加，需评估。
- 两个故障域（ZMQ 链路 / 云端链路），需日志与重连策略。
- 安全：明文 WS + 明文 ZMQ，仅限可信网络。

---

## 四、仓库与资产现状（截至 2026-10-15）

### 4.1 本仓库 ds-robot（fork 自 `xuweiwu/XLeRobot`）

- 远端：`origin`=DarkskyX15/ds-robot（fetch=HTTPS+代理 `socks5h://127.0.0.1:7890`；push=SSH）；`upstream`=Vector-Wangel/XLeRobot（已 fetch）；`xuweiwu`=xuweiwu/XLeRobot（已 fetch）。
- 分支：
  - `main` = `3707155`（2025-12-29，落后官方最新 80 个提交）；
  - `pi05_dual_arm` = `2970ced`（pi0.5 全套改动：7 提交/16 文件/+3993 行；基点为 `c05fb6f`）；
  - **`feat/pi05-async-inference`（工作分支）** = `fc43289`（已含 `.gitignore` 增加 `.ref/`），已推送并跟踪 origin。
- 本机 git 身份（仓库局部）：`DarkskyX <darkskyx@foxmail.com>`。
- pi0.5 关键文件：`software/examples/9_pi05_inference_dualarm.py`、`software/src/robots/xlerobot/{xlerobot,config_xlerobot,xlerobot_host,xlerobot_client}.py`、`software/src/utils/quadratic_spline_via_ipol.py`、`software/src/model/kinematics.py`、相机补丁等。

### 4.2 服务端资产（`.ref/openpi` @ `xuweiwu/openpi`）

- 分支 `biso101_training_support`，HEAD `73ef478`（基于上游 main `bd70b8f`，落后 origin/main `981483d` 4 个提交，功能无影响）。
- 新增：`biso101_policy.py`、`LeRobotBiSO101DataConfig`（repack：`observation.images.{head,left_wrist,right_wrist}`→`image/*`、`observation.state`→`state`、`task`→`prompt`；关节增量+夹爪绝对：`make_bool_mask(5,-1,5,-1)`）、4 个 `pi05_bimanual_so101_*` 训练配置（LoRA 为主）、lerobot v3 数据集支持（lerobot rev `cb920235…`）。
- **预训练权重**：HF `xuweiwu/pi05_bimanual_so101_lora`（2026-02-20）：
  - 末步 checkpoint `19999/`（params + assets/**norm_stats.json** + train_state）；
  - 全库 13.39GB；起服务只需 `19999/params` + `19999/assets`（约 7GB）——**无需训练，可直接起服务**。
- 起服务命令：
  ```bash
  uv run scripts/serve_policy.py policy:checkpoint \
    --policy.config=pi05_bimanual_so101_lora --policy.dir=<ckpt>/19999
  ```

### 4.3 通信协议（已核对对齐）

- **openpi**：WebSocket（`ws://host:8000`，无 TLS，可选 Api-Key），msgpack+numpy 二进制，双方 `compression=None`；上行 3×224² uint8 + state[12] + prompt（≈441KB/次），下行 `actions` float32 [50,12] + `server_timing`；`/healthz` 健康检查。
- **ZMQ**：PUSH/PULL 5555（动作 JSON）/ 5556（观测 JSON：17 维状态 + 全相机 base64 JPEG q90），CONFLATE=1，看门狗 500ms 停底盘，环 30Hz。
- **客户端↔服务端契约**：图像键、12 维 state（左6+右6，度）、action 切片 [:, :12]、action_horizon=50、FPS 50 —— 全部对齐 ✅。

### 4.4 参考仓库（`.ref/`，完整克隆）

| 仓库 | 来源 | HEAD | 相对状态 |
|---|---|---|---|
| `XLeRobot-vanilla` | Vector-Wangel（官方） | `b017b5e`（2026-10-06） | 最新官方主线；比本库 main 多 80 提交，含 lerobot 插件化打包（#150/#151） |
| `XLeRobot-custom` | JoyandAI（卖家） | `8e8eabf`（2026-03-10） | 与官方共同祖先 `9540be0`（2026-02-04）；自家 26 提交：lerobot 0.4 适配、VR 参数/文档、键盘遥操作修复等；缺官方后 45 提交 |

### 4.5 本机环境现状

- 无 `uv`、无 lerobot 安装、`~/lerobot` 不存在；Python 3.14.4；GPU=MX250（**不足以做推理服务端**）。
- SSH push 已配置可用；GitHub 访问需代理 `socks5h://127.0.0.1:7890`。
- `.ref/` 已加入 .gitignore；`pi05_async_inference_handover.md`、`openspec/`、`.opencode/`、`todo/` 目前未跟踪。

---

## 五、实施 TODO（建议顺序）

- [ ] **服务端（GPU 机）**：装 uv → `GIT_LFS_SKIP_SMUDGE=1 uv sync` → 下载权重（`--include "19999/params/*" "19999/assets/*"`）→ 起 `serve_policy`（:8000）→ 健康检查 + 假观测验证返回 [50,12]。
- [ ] **机器人侧**：安装 lerobot（0.4.x）+ 按映射表拷入 fork 文件；`pip install -e .ref/openpi/packages/openpi-client`（numpy>=2）；改 `config_xlerobot.py`（串口 udev/相机设备）；`xlerobot/__init__.py` 取消注释 host/client；启动 `xlerobot_host`。
- [ ] **Relay 中继脚本**：基于 `9_pi05_inference_dualarm.py` 改造为 `XLerobotClient` 模式（get_observation/send_action；30Hz 适配；保留 ^ 急停与回位插值）。
- [ ] **ZMQ 优化**：host 端降分辨率（320×240 或 224²）+ 视频 10–15fps + q85；实测对策略的影响。
- [ ] **云链路**：优先同网段；公网评估 VPN/wss；设置 api_key；断连/看门狗/急停演练。
- [ ] **联调验收**：12 维动作 chunk、50Hz（或 30Hz）执行、急停生效、回位插值、看门狗、端到端延迟测量。
- [ ] **（可选）** 双模式：直连（USB + 云端 openpi）与 Relay（ZMQ 中继）通过配置开关切换。

### 待定问题

1. Relay 与直连的优先级：机器人是否必须自由移动（决定是否值得做 Relay）。
2. 最终控制频率 30Hz 还是调 host 至 50Hz。
3. 视频降帧/降分辨率对策略成功率的影响需实测。
4. 是否把中继做成通用"机器人网关"（服务于遥操作/网页/agent 全部接入）。
