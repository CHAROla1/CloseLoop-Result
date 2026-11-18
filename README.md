# CloseLoop-Result

闭环评测结果仓库，保存两类内容：可直接浏览的批量报告（`batches/`），以及按批归档的每帧中间结果与真值自车轨迹（`perframe/`）。场景原始数据、`rollout.asl`、模型、日志及未归档的中间产物不放在这里。

## 目录

```text
batches/
  <batch-id>/
    index.html
    catalog.json
    <report-id>/
      index.html
      manifest.json
      data/*.json
      media/rollout_*/*.mp4
perframe/
  <batch-id>/
    README.md          # 字段说明与数据来源
    mapping.csv        # task → 场景与来源映射
    tasks/task-*/
      ego_trajectory.csv          # 自车执行轨迹
      planned_trajectories.jsonl  # 模型规划轨迹
      vlm_reasoning.jsonl         # 每帧 VLM 推理
      metrics.parquet             # 每帧指标
      gt_ego_trajectory.json      # 场景真值自车轨迹
```

每个 `<batch-id>` 对应一次导出的结果快照。新增或重做批次时使用新的目录名，避免覆盖已发布结果。

## 批量生成并导入

在主项目中完成 cluster 闭环评测后，按[主项目 README v2 的第 8 步](https://github.com/CHAROla1/Alpasim_WorldModel_ClosedLoop#闭环评测全流程-v2)准备 `REPORT_INPUTS`，然后运行：

```bash
export REPO=/mnt/data/WM_Group/tang-workspace/alpasim_render_bk/Alpasim_WorldModel_ClosedLoop
export RESULT_REPO=/mnt/data/WM_Group/tang-workspace/CloseLoop-Result
export CLUSTER_ID=wyp-v2-01
export REPORT_INPUTS="$REPO/reports/inputs-$CLUSTER_ID"
export REPORT_DIR="$REPO/reports/$CLUSTER_ID"
export RESULT_BATCH="$CLUSTER_ID"

cd "$REPO"
"$RUNTIME_PYTHON" scripts/postprocess/alpasim/closed_loop_report.py build-all \
  --runs-dir "$REPORT_INPUTS" --output "$REPORT_DIR" --media-mode copy

/usr/local/bin/python scripts/postprocess/alpasim/export_report_bundle.py \
  --source "$REPORT_DIR" --destination "$RESULT_REPO/batches/$RESULT_BATCH"
```

`RUNTIME_PYTHON` 和 `REPORT_INPUTS` 的准备命令见主项目 README。`build-all` 只生成本地报告；后一条导出命令才将必要文件复制进本仓库。导出器仅接收总览中成功报告引用的资源，复制旧报告的软链接目标，并在单文件超过 100 MiB 时中止。导出失败不会留下半成品批次目录。

## 检查和提交

```bash
cd "$RESULT_REPO"
git status --short
git add "batches/$RESULT_BATCH"
git diff --cached --stat
git commit -m "results: add $RESULT_BATCH"
git tag "batch/$RESULT_BATCH"                       # 批次快照 tag，供按需拉取定位

# 本机代理对大推送会 HTTP 408 超时，推送需绕过代理直连
env -u http_proxy -u https_proxy -u HTTP_PROXY -u HTTPS_PROXY git push origin main --tags
```

提交前检查暂存列表和视频体积（`du -sh "batches/$RESULT_BATCH"`）。不要使用 `git add .` 把本地其他文件一起提交。

若同时归档每帧数据：改 `tools/extract_perframe.py` 顶部的 `REG_V5 / REG_V6 / DST` 常量后运行，再 `git add "perframe/$RESULT_BATCH"`，tag 名用 `data/$RESULT_BATCH-perframe`。

## 本地浏览

```bash
cd "$RESULT_REPO/batches/$RESULT_BATCH"
python -m http.server 18765 --bind 0.0.0.0
```

浏览器打开 `http://<主机地址>:18765/`。网页通过 HTTP 加载 JSON 和视频，直接双击 `index.html` 可能无法正常读取资源。

## 按需拉取某个批次

仓库随批次累积增大，**查看单个批次不要全量 clone**。用 partial clone + sparse-checkout，只下载所选目录：

```bash
git clone --filter=blob:none --no-checkout https://github.com/CHAROla1/CloseLoop-Result.git
cd CloseLoop-Result
git sparse-checkout set batches/2026-11-13-v6   # 换成想看的批次目录，可空格分隔多个
git checkout main
```

如上例只下载 v6 批次（~224 MB），而非整个仓库。之后可 `git sparse-checkout add perframe/2026-11-13-v5` 追加目录，`git pull` 增量更新。

各批次快照已打 tag（`batch/<batch-id>`、`data/<batch-id>-perframe`），可用 `git ls-remote --tags origin` 查看；要检出到某快照时 `git checkout <tag>`。

注意：默认 `git clone`（不带参数）仍是全量下载。
