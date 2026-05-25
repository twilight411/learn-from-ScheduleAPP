# 简历写法 · ScheduleApp 性能优化（总览）

> 技术栈示例：**Flutter Web** + **FastAPI** + **SQLite/PostgreSQL** + **即梦生图 API** + **DeepSeek LLM** + **Nginx**  
> 可并入「精灵日程 / ScheduleApp」项目下的子模块描述，或单独写「线上性能优化专项」。

---

## 项目一句话（Overview）

全栈日程管理 Web 应用；负责**植物页首屏性能优化**：通过压测定位三层瓶颈（Web 首包、同步 LLM、同步生图），落地 DB 指纹缓存、Fast API、异步生图与部署校验，将植物页 API 等待从 **12–15s 降至 0.4s 级**（缓存命中）。

---

## 简历 bullet（推荐 4–6 条，按你实际参与度选）

- **性能压测与定位**：编写 Python 压测脚本模拟 Flutter 真实调用链（登录 → 雷达 → 生图 → 周报/果实），结合外网与本机 loopback 对比，将 `tree/weekly` 从 **9.89s** 归因到同步 **6 次 LLM** 调用，排除「服务器资源不足」（load **0.01**，内存 available **1.3GB**）。
- **生命树/果实生图缓存**：设计 `weekly_tree_images` / `monthly_fruit_images` 表与**得分指纹**失效策略，果实接口由每次 **~10s** 降至缓存命中 **&lt;300ms**（`cached=true`），植物页热路径由 **12–15s 降至 1.5–2s**。
- **Fast 主路径 + AI 后台化**：`GET /tree/weekly` 默认 `fast=true`，首屏仅返回规则得分与雷达；LLM 点评/树叙述迁入 `weekly_tree_enrichments` 后台任务 + `/enrichment` 轮询，接口 **9.89s → 143ms**（外网压测）。
- **生图异步化**：新增 `wait=false` 与 `/image/status`，先返占位图并 `asyncio` 后台调即梦，客户端 2s 轮询；日常 `tree/weekly/image`、`fruits/image` **~122ms**，不再阻塞首屏 **~10s**。
- **线上故障修复**：修复 `fruits/image?refresh=true` 因 DELETE+INSERT 与长耗时生图并发导致的 **SQLite UNIQUE 500**（生图已成功但写库失败），改为 **UPDATE 就地刷新**；修复 Flutter Web 残缺部署导致的 **AssetManifest 404**（部署包 **12MB→20MB**）。
- **部署与静态资源**：规范 `build/web` 必检清单（Manifest / wasm / bootstrap）；Nginx 配置 `.wasm` gzip 与 `application/wasm`；`flutterConfiguration.renderer=auto` 降低 CanvasKit 首包依赖。

---

## 关键词（简历 / 面试）

`性能压测` `p50/p95` `loopback 对照` `指纹缓存` `读写分离（读快写慢）` `异步任务` `轮询` `SSE 无关本专题` `LLM 调用链拆分` `外部 API 超时` `SQLite UNIQUE` `Flutter Web 部署`

---

## 与各主题笔记对应

| 主题文件 | 简历可拆出的点 |
|----------|----------------|
| [01 三层根因与压测](01-植物页首屏慢-三层根因与压测.md) | 方法论、压测脚本、定位过程 |
| [02 果实 DB 缓存](02-果实生图DB缓存.md) | 指纹缓存、cached 字段 |
| [03 Fast + AI 异步](03-生命树Fast接口与AI异步.md) | 关键路径瘦身、后台 enrichment |
| [04 生图异步](04-生图异步化与轮询.md) | wait=false、状态机 pending/ready |
| [05 Flutter Web 部署](05-Flutter-Web静态资源部署.md) | 前端部署、404 排查 |
| [06 Refresh 500](06-Refresh刷新写库UNIQUE冲突.md) | 并发写库、upsert 思维 |
| [07 缓存与面试](07-生图缓存机制与面试要点.md) | 综合问答 |

数据出处：[benchmark-reports/api-load-benchmark-2026-05-25-round2.md](../benchmark-reports/api-load-benchmark-2026-05-25-round2.md)

---

## 面试前必读：6 条 Bullet 逐条拆解

→ **[00b-简历 bullet 逐条拆解（面试深度版）](00b-简历bullet逐条拆解-面试深度版.md)**  
含：每条「拆开怎么讲」、名词解释、**~10s 是什么意思**、高频追问与参考答案、90 秒串联话术、数字速查表。
