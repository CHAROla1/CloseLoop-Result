# 每帧中间结果与 GT 自车轨迹 — 2026-11-13-v5

闭环评测批次 `2026-11-13-v5`（64 个 task）的每帧中间数据与对应场景真值，从主项目 registry 提取。

## 目录

```text
mapping.csv                    # task_id → sample / source_name / scene_id / 数据来源
tasks/task-XXXXX/
  ego_trajectory.csv           # 自车执行轨迹（位姿/速度/控制量，含表头）
  planned_trajectories.jsonl   # 模型规划轨迹（自 steps.jsonl 提取的子集字段）
  vlm_reasoning.jsonl          # 每帧 VLM 推理文本
  metrics.parquet              # 每帧评测指标
  gt_ego_trajectory.json       # 场景真值自车轨迹（101 帧 @10Hz）
```

## 字段说明

### ego_trajectory.csv（~89 行 = 表头 + 8.4 s @10 Hz）

`timestamp_us, x, y, z, qx, qy, qz, qw`（自车位姿）, `vx, vy, wz`（速度/横摆角速度）,
`u_steering_angle, u_longitudinal_actuation`（控制量）, `ref_traj_*`（参考轨迹点）,
`front_steering_angle, acceleration`, `x_ref_0, y_ref_0, yaw_ref_0`（参考位姿）

### planned_trajectories.jsonl（85 行 = 85 个规划步，每行一步）

| 键 | 含义 |
|---|---|
| `step` / `time_now_us` / `time_query_us` / `t0_us` | 步序号与时间戳（µs） |
| `scene_id` | 场景 clip id |
| `anchor_pose` | 当前自车锚定位姿 `{xyz[3], quat_xyzw[4]}`（世界系） |
| `trajectory_rig_xyz` / `trajectory_rig_heading` | 模型输出的**选定规划轨迹**（rig 系，32 点） |
| `trajectory_local` | 选定轨迹的本体表示，33 × `[t_us, x, y, z, ...]` |
| `selected_candidate` | 6 候选中被选中的序号 |

规划轨迹为 rig 坐标系，配合 `anchor_pose` 可变换到世界系；与 `ego_trajectory.csv` 按 `timestamp_us`（100 ms 间隔）对齐。

### vlm_reasoning.jsonl

每行 `{now_time_us, query_time_us, reasoning_text}`。

### metrics.parquet

每帧指标（`collision_any`、`offroad`、`wrong_lane`、`dist_to_gt_trajectory` 等）。
时间列为 INT96，读取需较新版 pandas/pyarrow 或 duckdb。

### gt_ego_trajectory.json

场景真值：`rig_trajectories`（GT 自车位姿序列，101 帧 @10 Hz）、`world_to_nre`、`camera_calibrations`。
该轨迹即 `min_ade@*(gt)` / `dist_to_gt_*` 指标的对比基准。

## 数据来源

- 56 个任务取自 registry `2026-11-13-v5` 的原始 run；
- **8 个任务（task-00056~00063，sample_00427~00434）在 v5 中提前终止**（offroad 等原因，缺 VLM 记录），按重跑结果改取 registry `2026-11-13-v6` 中同场景的成功 run，`mapping.csv` 的 `source_registry` / `source_task` 列记录了实际来源；
- `gt_ego_trajectory.json` 与 run 无关，始终取 `data/scenes/<sample>/trajectories/ego.json`。

## 不包含

模型输入 dump（`model_inputs_*.pt`）、`rollout.asl` 二进制日志、视频、渲染模型、周车真值轨迹。完整原始数据在主项目 `cluster_config/cluster_registry/` 下。
