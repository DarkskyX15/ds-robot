# pi0.5 直连部署流程（Runbook）

- 日期：2026-10-16
- 拓扑：**直连（client-host）** —— 机器人机用 USB 直连硬件、本地跑 lerobot + `9_pi05_inference_dualarm.py`（openpi client），经 WebSocket 连 GPU 服务端；GPU 机跑 `serve_policy.py`（openpi host + pi0.5）
- 关联文档：
  - `dev-docs/pi05_async_inference_handover.md`（检查报告/交接）
  - `todo/pi05_relay_topology_plan.md`（备选：ZMQ relay 拓扑，含压缩/带宽分析）
  - `openspec/changes/align-xlerobot-bus-wiring`（✅ 驱动总线接线适配已实施，2026-10-16；真机验证见任务 3/4）

## 0. 当前状态速览（2026-10-16）

| 项 | 状态 |
|---|---|
| 服务端 | ✅ 已部署：`10.10.64.57:20721`，`/healthz` OK，单次推理 ~150ms、客户端往返 ~200–300ms（实测 3 次） |
| 服务端权重 | `xuweiwu/pi05_bimanual_so101_lora`（step 19999，含 norm_stats；起服务无需训练） |
| 机器人侧环境 | ❌ 待部署（lerobot + 补丁 + openpi-client + 校准） |
| 驱动接线适配 | ✅ 已实施（2026-10-16）：`bus1`=左臂+头、`bus2`=右臂+底盘，端口字段 `port1`/`port2`；真机验证待做 |

## 1. Host 侧（GPU 机）

### 1.1 前置条件

- NVIDIA GPU ≥ 8GB 显存；Ubuntu 22.04 推荐；`nvidia-smi` 正常
- 网络可达 GitHub、HuggingFace（国内需代理，见 §3）
- 磁盘 ~20GB 余量

### 1.2 步骤

```bash
# ① 获取代码（xuweiwu fork 的 biso101 分支；submodule 不需要）
git clone -b biso101_training_support https://github.com/xuweiwu/openpi.git && cd openpi

# ② 安装 uv（若没有）
curl -LsSf https://astral.sh/uv/install.sh | sh

# ③ 创建环境
GIT_LFS_SKIP_SMUDGE=1 uv sync
GIT_LFS_SKIP_SMUDGE=1 uv pip install -e .
uv run python -c "import jax; print(jax.devices())"     # 期望 [CudaDevice(id=0)]；CPU 回退会慢 ~100 倍

# ④ 下载权重（只下服务所需 ≈7GB；全库 13.4GB 含 train_state）
uvx --from 'huggingface_hub[cli]' hf download xuweiwu/pi05_bimanual_so101_lora \
  --include "19999/params/*" "19999/assets/*" \
  --local-dir ~/checkpoints/bimanual_so101
# 国内可加：HF_ENDPOINT=https://hf-mirror.com

# ⑤ 起服务（--port 必须写在 policy:checkpoint 之前！写后面会报 Unrecognized options）
uv run scripts/serve_policy.py --port=20721 policy:checkpoint \
  --policy.config=pi05_bimanual_so101_lora \
  --policy.dir=~/checkpoints/bimanual_so101/19999

# ⑥ 自检（服务机本机）
curl http://127.0.0.1:20721/healthz        # → OK
# 再跑一次假观测（脚本见 §2.5，HOST 换 127.0.0.1），期望 actions (50,12)、infer 百毫秒级
```

- 防火墙放行 TCP 20721（机器人机所在网段）；跨公网需 VPN/wss（协议默认明文）。
- 常驻：tmux/systemd；客户端每 5s 自动重连，服务端重启无需重启客户端。
- 服务重启后**首次请求**含 JIT 编译可能耗时较长（分钟级）；建议先预热一发。

## 2. 机器人侧

### 2.1 前置检查（先记录设备映射）

```bash
lsusb
ls -l /dev/serial/by-id/ 2>/dev/null
ls -l /dev/ttyACM* /dev/ttyUSB* 2>/dev/null
ls -l /dev/xlerobot* 2>/dev/null          # 本机已有：/dev/xlerobot_port1(左臂), /dev/xlerobot_port2(右臂)
ls -l /dev/v4l/by-id/ 2>/dev/null
sudo apt install -y v4l-utils && v4l2-ctl --list-devices   # 注意：/dev/mediaN 不能采集，要 /dev/videoN 或 by-id
groups | grep dialout || sudo usermod -aG dialout $USER   # 串口权限（重登生效）

# RealSense 序列号（若头部仍是 RealSense）
python -c "import pyrealsense2 as rs; ctx=rs.Context(); [print(d.get_info(rs.camera_info.name),'| SN:',d.get_info(rs.camera_info.serial_number)) for d in ctx.query_devices()]"
```

需要记录的 4 样：两条总线设备路径、左右腕相机路径、头部相机路径/SN。

### 2.2 五步安装

```bash
# ① Clone 仓库（需要代理时按 §3 加 -c http.proxy=...）
git clone -b feat/pi05-async-inference https://github.com/DarkskyX15/ds-robot.git
cd ds-robot

# ② venv + 依赖（lerobot 固定 0.4.3；extras：串口 + RealSense）
python3 -m venv ~/pi05-env && source ~/pi05-env/bin/activate
pip install "lerobot[feetech,intelrealsense]==0.4.3" scipy

# ②-1 下载 openpi 仓库（浅克隆、跳过 LFS 大文件）
GIT_LFS_SKIP_SMUDGE=1 git clone --depth 1 --branch biso101_training_support \
  https://github.com/xuweiwu/openpi.git ~/openpi
# ②-2 从本地目录安装 client
pip install -e ~/openpi/packages/openpi-client

# ③ 拷 4 个 fork 补丁文件进 lerobot —— 在【仓库根目录 + venv 已激活】下执行：
L=$(python -c "import lerobot,os;print(os.path.dirname(lerobot.__file__))"); echo "$L"   # 确认指向 venv 的 site-packages
cp -r software/src/robots/xlerobot "$L/robots/"
cp    software/src/utils/quadratic_spline_via_ipol.py "$L/utils/"
cp    software/src/cameras/opencv/camera_opencv.py "$L/cameras/opencv/"
cp    software/src/cameras/realsense/camera_realsense.py "$L/cameras/realsense/"

# ④ 配置修改（见 §2.3 / §2.4）

# ⑤ 校准 + 运行（见 §2.6）
```

### 2.3 配置修改点

`software/src/robots/xlerobot/config_xlerobot.py`：

1. **相机 3 处**：`left_wrist` / `right_wrist` 的 `index_or_path` 换成实际设备（优先 `/dev/v4l/by-id/...`）；`head` 若为普通 USB 相机则整块替换为 `OpenCVCameraConfig`（若仍是 RealSense 则保留 `RealSenseCameraConfig`，只改 SN）。
2. **串口 2 处**：默认值已指向 `/dev/xlerobot_port1`、`/dev/xlerobot_port2`（适配后无需修改；如设备节点不同再改）。

`software/examples/9_pi05_inference_dualarm.py`：

3. `OPENPI_SERVER_IP = "10.10.64.57"`
4. 第 398 行 `port=8000` → `port=20721`（硬编码，无配置项）

### 2.4 总线接线（已适配出厂接线，2026-10-16）

- 出厂接线（卖家/官方）：**port1（bus1）= 左臂+头；port2（bus2）= 右臂+底盘**
- 驱动已按此适配：总线变量 `bus1`/`bus2`，头部电机命名 `head_motor_1/2`，配置端口字段 `port1`/`port2`
- 首次真机运行前，仍需执行一次路由安全检查（头/轮不互串，见 §4 验收清单）

### 2.5 快速验证

```bash
# import 冒烟
python -c "from lerobot.robots.xlerobot import XLerobot, XLerobotConfig; from lerobot.utils.quadratic_spline_via_ipol import QuadraticSplineInterpolator; from openpi_client import websocket_client_policy; print('imports OK')"

# 连通服务端 + 假观测（期望 (50,12)）
python - <<'EOF'
import numpy as np
from openpi_client import websocket_client_policy
c = websocket_client_policy.WebsocketClientPolicy(host="10.10.64.57", port=20721)
obs = {k: np.zeros((224, 224, 3), np.uint8) for k in ("image/head", "image/left_wrist", "image/right_wrist")}
obs["state"] = np.zeros(12, np.float32); obs["prompt"] = "test"
r = c.infer(obs)
print("actions:", r["actions"].shape, "| infer_ms:", round(r["server_timing"]["infer_ms"], 1))
EOF
```

### 2.6 校准 + 运行

- 若机器人已有卖家格式校准文件（`~/.cache/huggingface/lerobot/calibration/`），接线适配完成后可直接复用；否则先做一次校准。
- 运行（值守 `^` 急停）：

```bash
python software/examples/9_pi05_inference_dualarm.py
# 预期：双臂插值到 FULL_START_POS → 发观测 → 每 ~1s 一个 (50,12) chunk、50Hz 执行
```

## 3. 代理（192.168.31.224:7890）

```bash
# 一次性
git -c http.proxy=http://192.168.31.224:7890 clone <repo>
pip install --proxy http://192.168.31.224:7890 <pkg>

# 环境变量法（git/pip/curl 通吃）
export http_proxy=http://192.168.31.224:7890 https_proxy=http://192.168.31.224:7890
export no_proxy=localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16,172.16.0.0/12   # 必须含内网，否则连 10.10.64.57 会被导进代理
# 用完 unset
```

## 4. 验收清单

- [ ] 服务端 `/healthz` OK；假观测返回 `(50, 12)` 且 infer_ms 在百毫秒级
- [ ] 机器人侧：imports OK；udev/设备路径记录完整；相机抓帧正常
- [ ] 真机路由安全检查（头/轮不互串）—— 驱动已适配（`align-xlerobot-bus-wiring`）
- [ ] 校准完成（17 个电机）
- [ ] 9 号示例跑通：回位插值、观测/动作循环、`^` 急停、50Hz 无卡顿

## 5. 依赖/待办

1. 真机硬件验证（部署 + 路由安全检查 + 校准 + 集成验收，对应 change 任务 3/4）
2. 机器人侧环境安装（§2.2）+ 设备映射确认（§2.1）
3. （可选）把 `packages/openpi-client` 内嵌进仓库，去掉 GitHub 拉取依赖
