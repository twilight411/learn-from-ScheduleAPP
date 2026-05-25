# learn-from-ScheduleAPP

从 **ScheduleApp**（精灵日程 / Flutter Web + FastAPI）线上优化实战中提炼的学习笔记，面向**面试复盘**与**性能排查方法论**。

## 仓库结构

```
learn-from-ScheduleAPP/
├── README.md                 # 本文件
├── benchmark-reports/        # 原始压测报告与 JSON（可引用数据）
└── topics/                   # 按主题命名的笔记（问题 → 方法 → 数据）
```

## 主题索引

| 文件 | 一句话 |
|------|--------|
| [01-植物页首屏慢-三层根因与压测](topics/01-植物页首屏慢-三层根因与压测.md) | 首屏慢 = Web 包 + 同步 LLM + 同步即梦；如何用压测量化 |
| [02-果实生图DB缓存](topics/02-果实生图DB缓存.md) | `fruits/image` 从每次 ~10s 到缓存 **&lt;300ms** |
| [03-生命树Fast接口与AI异步](topics/03-生命树Fast接口与AI异步.md) | `tree/weekly` 从 **9.89s** 到 **~143ms** |
| [04-生图异步化与轮询](topics/04-生图异步化与轮询.md) | `wait=false` + `/image/status`，首屏不等即梦 |
| [05-Flutter-Web静态资源部署](topics/05-Flutter-Web静态资源部署.md) | `AssetManifest.bin.json` 404 与完整 `build/web` 部署 |
| [06-Refresh刷新写库UNIQUE冲突](topics/06-Refresh刷新写库UNIQUE冲突.md) | 缓存不是 500 原因；DELETE+INSERT 竞态 |
| [07-生图缓存机制与面试要点](topics/07-生图缓存机制与面试要点.md) | 指纹缓存、占位 URL、cached 字段；常见面试题 |

## 环境快照

- **API**：`http://47.118.28.102:8000`
- **压测日**：2026-05-25
- **详细数据**：见 [benchmark-reports/](benchmark-reports/)

## 来源项目

笔记内容来自 ScheduleApp 仓库 `.deploy/docs/` 压测与部署实践，不代表原仓库官方文档。
