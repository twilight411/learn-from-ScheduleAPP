# 压测报告归档

从 ScheduleApp 项目 `.deploy/docs/` 与 `.deploy/*.json` 复制，便于本仓库笔记引用。

| 文件 | 说明 |
|------|------|
| [api-load-benchmark-2026-05-25.md](api-load-benchmark-2026-05-25.md) | 优化前基线 |
| [api-load-benchmark-after-fruit-cache-2026-05-25.md](api-load-benchmark-after-fruit-cache-2026-05-25.md) | 果实 DB 缓存上线后 |
| [api-load-benchmark-2026-05-25-round2.md](api-load-benchmark-2026-05-25-round2.md) | fast + async + refresh 修复后复测（**推荐作最终数据**） |

复现脚本（在 ScheduleApp 仓库）：

```powershell
cd ScheduleApp/.deploy
python api_load_benchmark.py
python api_benchmark_async_paths.py
python server_loopback_bench.py
```
