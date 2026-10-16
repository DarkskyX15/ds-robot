# pi0.5 数据微调指南（openpi 链路）

- 日期：2026-10-16
- 链路：**机器人侧采集 → HF/本地数据集 → openpi 加配置 → (norm stats) → 训练 → serve 新权重**
- 关联文档：`pi05_deployment_runbook.md`（部署）、`pi05_async_inference_handover.md`（检查报告）
- 载体：GPU 机上的 `xuweiwu/openpi` @ `biso101_training_support`（`.ref/openpi` 为同版本参考副本）

## 0. 链路总览

```
[机器人机]                        [GPU 机 ~/openpi]
8_vr_teleop_with_dataset_recording_dualarm.py
        │  采集（3 相机 + 12 维 + 任务文本）
        ▼
   HF 数据集（或 ~/.cache/huggingface/lerobot/<repo_id>/）
        │                                        ┌─ scripts/compute_norm_stats.py（方案 b 需要）
        ▼                                        ▼
   src/openpi/training/config.py 新增 TrainConfig ──→ scripts/train.py ──→ checkpoints/<config>/<exp>/<step>/
                                                                                    │
                                                             scripts/serve_policy.py policy:checkpoint
                                                                                    ▼
                                                                       机器人侧（IP/端口常量，提示词与训练一致）
```

## 1. 起点现状（现有权重是怎么来的）

| 项 | 值 |
|---|---|
| 数据 | `xuweiwu/bimanual-toy-box-cleanup`：43 episodes / 206,883 帧 / 单任务 / **fps=60** / v3.0 格式 |
| 相机键 | `observation.images.{head,left_wrist,right_wrist}` |
| state / action | 均 12 维（左臂 6 + 右臂 6，度；夹爪 0–100） |
| 训练配置 | `pi05_bimanual_so101_lora`（LoRA，`action_dim=32` → 切片 12，`action_horizon=50`，`use_delta_transform=True`） |
| 权重 | `xuweiwu/pi05_bimanual_so101_lora`（HF，step 19999，含 norm_stats） |
| 服务 | `10.10.64.57:20721`（`serve_policy`） |

## 2. 采集数据（机器人侧）

- 脚本：`software/examples/8_vr_teleop_with_dataset_recording_dualarm.py`（与现有数据同一条管线；VR 遥操作 + `LeRobotDataset` 录制）
- **契约不能动**（改了旧 norm stats / 变换全失效）：
  - 相机键名与位置（`head` / `left_wrist` / `right_wrist`）
  - 12 维 state/action 顺序（左 6 + 右 6）、单位（度）、夹爪范围
  - 采集脚本默认帧率流程（这套数据为 60fps）
- 数据量参考：原任务 **43 episodes** 即有演示效果；单任务建议 **≥50** 起步，按效果补采
- 任务文本：采集时的 task 字符串会进训练 prompt（`prompt_from_task=True`）——**推理时要发同一句**
- 存放两种方式：
  1. **上传 HF 数据集**（推荐，私有亦可）：`hf upload <账号>/<数据集名> <本地dataset目录> --repo-type dataset`
  2. **离线**：把数据集放到 lerobot 缓存目录 `~/.cache/huggingface/lerobot/<repo_id>/`（openpi 的 loader 直接读取该目录，不触发下载）

## 3. GPU 机：新增 TrainConfig

在 `src/openpi/training/config.py` 中复制现有 `pi05_bimanual_so101_lora` 块，改三处（**方案 a：复用现有 norm stats + warm-start，同机器人同量纲时推荐**）：

```python
TrainConfig(
    name="pi05_redbox_lora",                      # ① 新名字
    model=pi0_config.Pi0Config(pi05=True, action_dim=32, action_horizon=50,
                               paligemma_variant="gemma_2b_lora"),
    data=LeRobotBiSO101DataConfig(
        repo_id="<账号>/<新数据集>",                # ② 新数据集
        base_config=DataConfig(prompt_from_task=True),
        use_delta_transform=True,                 # 别动
        assets=AssetsConfig(assets_dir="<旧ckpt>/19999/assets",   # 方案 a：复用 norm stats
                            asset_id="xuweiwu/bimanual-toy-box-cleanup"),
    ),
    batch_size=32,                                # 显存不够降到 8~16
    freeze_filter=pi0_config.Pi0Config(pi05=True, action_dim=32, action_horizon=50,
                                       paligemma_variant="gemma_2b_lora").get_freeze_filter(),
    lr_schedule=_optimizer.CosineDecaySchedule(warmup_steps=1_000, peak_lr=2.5e-5,
                                               decay_steps=30_000, decay_lr=2.5e-6),
    optimizer=_optimizer.AdamW(clip_gradient_norm=1.0),
    ema_decay=None,
    weight_loader=weight_loaders.CheckpointWeightLoader("<旧ckpt>/19999/params"),  # 方案 a：warm-start
    num_train_steps=20_000, num_workers=12, wandb_enabled=True,
)
```

- **方案 b**（换了机器人 / 量纲变化）：不用 `assets=` 复用，`weight_loader` 保持 `CheckpointWeightLoader("gs://openpi-assets/checkpoints/pi05_base/params")`，并在下一步重算 norm stats。

## 4. 计算 norm stats（仅方案 b / 量纲变化时）

```bash
cd ~/openpi
uv run scripts/compute_norm_stats.py --config-name pi05_redbox_lora
# 输出：./assets/pi05_redbox_lora/<repo_id>/norm_stats.json（训练时自动打进 checkpoint）
```

## 5. 训练

```bash
cd ~/openpi
XLA_PYTHON_CLIENT_MEM_FRACTION=0.9 uv run scripts/train.py pi05_redbox_lora \
  --exp-name=redbox --overwrite
# 多卡：--fsdp-devices=N（按卡数分摊显存）
```

- 显存要求（官方表）：推理 >8GB；**LoRA 微调 >22.5GB**；全量微调 >70GB
- ⚠️ 与 `serve_policy` 同机时会抢显存——先停服务再训（或换卡）
- wandb 默认开启；无网时设离线/关闭
- 产物：`checkpoints/pi05_redbox_lora/redbox/<step>/`（params + assets/norm_stats）

## 6. 部署新权重

```bash
uv run scripts/serve_policy.py --port=20721 policy:checkpoint \
  --policy.config=pi05_redbox_lora \
  --policy.dir=checkpoints/pi05_redbox_lora/redbox/19999
```

- `--port` 必须写在 `policy:checkpoint` **之前**
- 机器人侧无需改代码（`OPENPI_SERVER_IP`/`OPENPI_SERVER_PORT` 已是常量）
- **推理提示词必须与训练任务文本一致**

## 7. 验收

- 服务端 `/healthz` + 假观测返回 `(50, 12)`
- 机器人侧跑 `9_pi05_inference_dualarm.py`：新任务成功率/流畅度对比旧权重
- 记录：数据集规模、训练步数、最终 ckpt、推理延迟，便于复现

## 8. 踩坑清单

1. **提示词一致性**——训练用数据集 task 文本，推理发同一句（刚才提示词改回的原因）。
2. **别改数据契约**：相机键、维度顺序、单位、`use_delta_transform=True`；采集用 `8_` 脚本即天然一致。
3. **fps 保持采集脚本默认**（这套数据 60fps），执行端 50Hz 是既定行为，不要手动改任一侧。
4. **数据规模**：43 episodes 是演示级基准；场景变难就多采。
5. **显存**：LoRA ≥22.5GB；不足则降 `batch_size` 或 `--fsdp-devices` 多卡。
6. **norm stats 归属**：方案 a 复用旧 stats 时，新数据必须与旧数据同机器人、同量纲；否则用方案 b 重算。
7. **离线数据集**：openpi 只认 HF id 或 `~/.cache/huggingface/lerobot/<repo_id>/` 目录结构，直接用任意本地路径不被支持。

## 附：文件路径速查（GPU 机，`~/openpi`）

| 事项 | 路径 |
|---|---|
| 训练配置 | `src/openpi/training/config.py` |
| 数据集加载/变换 | `src/openpi/training/data_loader.py`、`src/openpi/transforms.py`、`src/openpi/policies/biso101_policy.py` |
| norm stats 脚本 | `scripts/compute_norm_stats.py`（输出 `./assets/<config>/<repo_id>/`） |
| 训练入口 | `scripts/train.py`（产物 `./checkpoints/<config>/<exp>/`） |
| 服务入口 | `scripts/serve_policy.py` |
