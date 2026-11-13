# CloseLoop-Result

闭环评测结果仓库，仅保存可直接浏览的批量报告：HTML、报告 JSON 和网页引用的 MP4 视频。场景原始数据、`rollout.asl`、模型、日志及未引用的中间产物不放在这里。

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
git push
```

提交前检查暂存列表和视频体积。不要使用 `git add .` 把本地其他文件一起提交。

## 本地浏览

```bash
cd "$RESULT_REPO/batches/$RESULT_BATCH"
python -m http.server 18765 --bind 0.0.0.0
```

浏览器打开 `http://<主机地址>:18765/`。网页通过 HTTP 加载 JSON 和视频，直接双击 `index.html` 可能无法正常读取资源。
