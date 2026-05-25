# 简历 6 条 Bullet 逐条拆解 · 面试深度版

> 配合 [00-简历总览](00-简历-ScheduleApp性能优化总览.md) 使用。  
> 数据出处：[round2 压测](../benchmark-reports/api-load-benchmark-2026-05-25-round2.md)

---

## 先搞懂：简历里的 `~10s` 是什么意思？

| 写法 | 含义 |
|------|------|
| **`~10s`** | 「约 10 秒」的缩写，来自压测 **p50 约 9.89s～10.03s**，不是精确 10.000s |
| **为什么用约** | 外部 API（即梦生图、LLM）每次耗时会有 **±1～2s 波动**，面试时说「十秒级」比背死 9.89 更稳 |
| **什么时候会出现 ~10s** | ① **冷路径**：DB 无缓存，必须调即梦；② **`refresh=true`**：用户强制换图；③ **优化前**同步等待即梦返回 |
| **优化后日常还会 ~10s 吗** | **不会**。日常打开走缓存 **&lt;300ms**；只有主动 refresh 或指纹变了才仍 ~10s（且可后台 pending，不挡首屏） |

**面试一句话**：  
「~10s 指即梦生图或同步 LLM 链路的**外部依赖耗时**，不是我们服务器算 10 秒；优化目标是**日常路径不走这条慢链路**。」

---

## Bullet 1：性能压测与定位

### 简历原文

> 编写 Python 压测脚本模拟 Flutter 真实调用链（登录 → 雷达 → 生图 → 周报/果实），结合外网与本机 loopback 对比，将 tree/weekly 从 9.89s 归因到同步 6 次 LLM 调用，排除「服务器资源不足」（load 0.01，内存 available 1.3GB）。

### 拆开讲（3 分钟版）

1. **背景**：用户说登录快、植物页慢，不能凭感觉优化。  
2. **做法**：写脚本按 `PlantProvider` 顺序打 API，记录每步耗时；同时 SSH 在服务器上对 `127.0.0.1:8000` 打 loopback。  
3. **发现**：`tree/weekly` 外网 **9.89s**，loopback 优化前也慢 → 不是单纯网速；读代码发现 `scoring_service` 里 **6 次 DeepSeek** 串行。  
4. **排除机器瓶颈**：`load 0.01`、`free` 显示 available **~1.3GB**，uvicorn 内存 **~110MB**。  
5. **结论**：优先砍 **同步 LLM** 和 **同步生图**，不是加 CPU。

### 关键名词

| 名词 | 解释 |
|------|------|
| **Flutter 真实调用链** | 不是乱打接口，而是对齐客户端：先 `tree/weekly`，再 `tree/weekly/image`，并行 `reports/weekly` + `fruits/image` |
| **loopback** | 在服务器本机请求 `127.0.0.1:8000`，去掉公网 RTT，看「纯后端」多快 |
| **load 0.01** | Linux 平均负载，1 分钟窗口；&lt;1 表示 CPU 很闲 |
| **9.89s** | round1 压测 `GET /tree/weekly` **首次**耗时（含网络），优化前基线 |

### 面试官可能问什么 · 怎么答

**Q：你怎么确定是 6 次 LLM，不是数据库慢？**  
→ 读 `tree_service` / `scoring_service` 调用栈；loopback 下任务列表 **11ms**；慢接口与 LLM/即梦 URL 调用时间吻合；加日志数 HTTP  outbound 次数。

**Q：压测脚本怎么保证和真用户一致？**  
→ 同一测试账号 token；顺序、query 参数（是否 refresh）与 Flutter `ApiPlantRepository` 一致；报告里分冷/热路径。

**Q：外网 143ms 和 loopback 12ms 差这么多？**  
→ 143 ≈ 12 + RTT + TLS + nginx；说明 fast 后计算本身极轻。

**Q：5 并发登录看什么？**  
→ 看登录、token 接口在并发下是否退化；植物慢的主因仍是单用户路径上的 LLM/生图，不是登录。

**Q：如果面试官质疑「9.89 只测了一次」？**  
→ 报告有多轮热路径、JSON 原始数据；可答「以 p50/多轮中位数为准，并复测 round2」。

### 易错点

- 说成「服务器性能差」——你有 load 数据，要强调 **已排除**。  
- 说不出 **6 次 LLM** 在哪——至少说 scoring/weekly 聚合逻辑。  
- 把 9.89s 说成生图——那是 **tree/weekly** 接口，主要是 LLM。

---

## Bullet 2：生命树/果实生图缓存

### 简历原文

> 设计 weekly_tree_images / monthly_fruit_images 表与得分指纹失效策略，果实接口由每次 ~10s 降至缓存命中 &lt;300ms（cached=true），植物页热路径由 12–15s 降至 1.5–2s。

### 拆开讲

1. **问题**：生命树有 `weekly_tree_images`，二次打开 **~11ms**；果实没有表，每次 **16.58s**（LLM 选果 + 即梦）。  
2. **设计**：按月/周存 `image_url` + **score_fingerprint**（当月/周任务得分 hash）。  
3. **读路径**：`get_or_generate_fruit_image()` — 指纹匹配 → 直接返回，`cached=true`。  
4. **失效**：任务完成度/得分变了 → 指纹变 → 下次 miss，重新生图。  
5. **数据**：果实 **16.58s → 133ms**；整页 API 从 **12–15s → 1.5–2s**（果实缓存后、尚未 fast+异步前的一轮）。

**注意数字时间线（面试别混）：**

| 阶段 | 植物页 API 合计 |
|------|-----------------|
| 优化前 | 12–15s |
| 仅果实+树图 DB 缓存 | **1.5–2s** |
| + fast + 异步（round2） | **0.35–0.45s** |

简历写「1.5–2s」是 **中间里程碑**，若面试官问「你简历不是说 0.4s」→ 答「后续又做了 fast 和 wait=false，最终 round2 到 0.4s 级」。

### 关键名词

| 名词 | 解释 |
|------|------|
| **得分指纹** | 对影响「画什么图」的输入（任务 ID、分数等）算 hash，变则重绘 |
| **cached=true** | 响应 JSON 字段，表示**没调即梦**，读的是 DB 里已有 URL |
| **&lt;300ms** | 外网压测命中缓存时 **122～133ms**，写简历取整说「亚秒级/三百毫秒内」 |

### 面试官可能问什么 · 怎么答

**Q：为什么不用 Redis？**  
→ 月/周级数据，读少写少，要持久化、多实例共享；DB 单行 PK 查询够用。

**Q：指纹太敏感怎么办？**  
→ 与产品对齐哪些字段进 hash；太敏感会频繁生图费钱，太粗图与数据不一致。

**Q：怎么证明是 DB 缓存不是内存？**  
→ 重启 uvicorn 后仍 133ms；响应有 `cached=true`。

**Q：12–15s 怎么算出来的？**  
→ 客户端串行+并行：`weekly`(~10s) + max(生图~10s, 果实~16s) 量级叠加，压测脚本模拟后实测。

### 易错点

- 只说「加了缓存表」不说**指纹**。  
- 把 1.5–2s 和 0.4s 说成同一阶段——要会讲**迭代顺序**。

---

## Bullet 3：Fast 主路径 + AI 后台化

### 简历原文

> GET /tree/weekly 默认 fast=true，首屏仅返回规则得分与雷达；LLM 点评/树叙述迁入 weekly_tree_enrichments 后台任务 + /enrichment 轮询，接口 9.89s → 143ms（外网压测）。

### 拆开讲

1. **问题**：`tree/weekly` 一次请求里同步跑 **6 次 DeepSeek**（点评、叙述等），首屏必等。  
2. **Fast**：默认 `use_llm_comment=False`、`include_narrative=False`，只算**规则引擎分+雷达 JSON**（可命中 `SpiritWeeklyScore`）。  
3. **后台**：`background_runner` 起 asyncio 任务写 `weekly_tree_enrichments`。  
4. **客户端**：先渲染树和雷达；定时或进入页后调 `GET /tree/weekly/enrichment`，`pending→ready` 后补文案。  
5. **结果**：外网 **9.89s → 143ms**；loopback **12ms**。

### 关键名词

| 名词 | 解释 |
|------|------|
| **关键路径** | 用户认为「页面好了」所必须的数据 |
| **增强路径** | AI 文案，可晚 2～3 秒 |
| **enrichment** | 增强内容，与计分表分离 |

### 面试官可能问什么 · 怎么答

**Q：6 次 LLM 能否合成 1 次？**  
→ 可减往返，但输出结构复杂、解析失败成本高；我们选**首屏 0 次、后台再跑**。

**Q：后台任务挂了怎么办？**  
→ enrichment 一直 pending；前端显示默认文案；监控+重试；用户下拉可触发重算。

**Q：为什么用轮询不用 SSE？**  
→ 文案一次性就绪，2～3s 轮询足够；与 SSE 流式场景不同（可对比 Sky-Chat）。

**Q：fast 会不会导致「没 AI」？**  
→ 只是**延迟展示**；最终仍从 enrichment 表读；产品接受渐进加载。

### 易错点

- 把 143ms 说成「LLM 优化到 143ms」——是 **不调 LLM** 才 143ms。  
- 说不出 `weekly_tree_enrichments` 存什么——**纯展示 LLM 产物**。

---

## Bullet 4：生图异步化

### 简历原文

> 新增 wait=false 与 /image/status，先返占位图并 asyncio 后台调即梦，客户端 2s 轮询；日常 tree/weekly/image、fruits/image ~122ms，不再阻塞首屏 ~10s。

### 拆开讲

1. **问题**：即使最终会缓存，**第一次**或无缓存时 HTTP 仍同步等即梦 **~10s**，worker 被占。  
2. **wait=false**：接口立刻返回 `image_status=pending` + **旧图/占位图 URL**。  
3. **后台**：asyncio 调即梦，写完 DB 设 `ready`。  
4. **status 接口**：轻量查询，Flutter **每 2s** 轮询直到 ready。  
5. **日常**：DB 已有图时 **122/133ms** 且 `cached=true`，不再等满 10s。

**~10s 在这里的角色**：  
- 优化前：**HTTP 连接阻塞 ~10s**  
- 优化后：10s 发生在**后台**，用户先看占位/旧图

### 面试官可能问什么 · 怎么答

**Q：轮询 2s 会不会太慢？**  
→ 生图总长约 10s，2s 粒度足够；比 100ms 轮询省请求。

**Q：占位图策略？**  
→ 上月图 / 默认 SVG / 固定宽高防抖动（可联想 Sky-Chat layout shift）。

**Q：并发 100 人同时生图？**  
→ 单机 asyncio 不够要队列+限流；即梦配额要熔断；当前量级进程内任务够用。

**Q：和 Bullet 2 缓存区别？**  
→ 缓存：**不调即梦**；异步：**必须调即梦但不挡请求**。

### 易错点

- 说「异步后生成只要 122ms」——122ms 是**有缓存或 pending 立即返回**，不是即梦变快。

---

## Bullet 5：线上故障修复

### 简历原文

> 修复 fruits/image?refresh=true 因 DELETE+INSERT 与长耗时生图并发导致的 SQLite UNIQUE 500（生图已成功但写库失败），改为 UPDATE 就地刷新；修复 Flutter Web 残缺部署导致的 AssetManifest 404（部署包 12MB→20MB）。

### 拆开讲（两件事）

#### 5a UNIQUE 500

1. **现象**：refresh 偶发 **500**，日志 `UNIQUE constraint failed`。  
2. **根因**：refresh **DELETE 再 INSERT**；即梦 **~10s** 期间用户双击或重试 → 两次 INSERT 同一 `(user_id, year, month)`。  
3. **关键**：即梦**已成功**，失败在**写库**——所以不是「第三方挂了」。  
4. **修复**：改 **UPDATE** 同一行（url、fingerprint、status）。  
5. **验证**：refresh 稳定 **200**，仍 ~10s（合理）。

#### 5b AssetManifest 404

1. **现象**：浏览器 404 `AssetManifest.bin.json`，白屏。  
2. **根因**：只上传了部分 `build/web`（如只换了 main.dart.js）。  
3. **修复**：`flutter clean && build web` 全量 tar；脚本 **REQUIRED_FILES** 校验。  
4. **12MB→20MB**：补全 assets/Manifest，不是「变臃肿」，是**之前缺文件**。

### 面试官可能问什么 · 怎么答

**Q：为什么 DELETE+INSERT 会错？**  
→ 画时间线：A DELETE→调即梦；B INSERT；A 再 INSERT → UNIQUE。标准解：**UPSERT / UPDATE**。

**Q：缓存会导致 500 吗？**  
→ **不会**；500 是写路径竞态，读缓存只 SELECT。

**Q：Manifest 作用？**  
→ Flutter 映射资源路径；缺则无法加载字体/图片。

### 易错点

- 把 500 说成即梦失败——强调 **生图成功、写库失败**。  
- 说 20MB 是性能变差——要解释是**完整包**。

---

## Bullet 6：部署与静态资源

### 简历原文

> 规范 build/web 必检清单（Manifest / wasm / bootstrap）；Nginx 配置 .wasm gzip 与 application/wasm；flutterConfiguration.renderer=auto 降低 CanvasKit 首包依赖。

### 拆开讲

1. **必检清单**：部署后自动检查 `AssetManifest.bin.json`、`flutter_bootstrap.js`、`main.dart.js`、wasm 目录等，缺则失败。  
2. **Nginx wasm**：`gzip_types` 含 `application/wasm`；Correct `Content-Type`，否则浏览器编译失败；压缩减传输。  
3. **renderer=auto**：不强制 **CanvasKit**（~7.1MB wasm）；让 Flutter 按环境选 HTML/Canvas 或 Skwasm，弱网首屏更好。  
4. **与 API 优化关系**：前端首包和 API 是**两层瓶颈**；API 0.4s 后用户仍可能卡在下载 wasm——都要做。

### 面试官可能问什么 · 怎么答

**Q：CanvasKit vs HTML？**  
→ CanvasKit 绘制一致、包大；HTML 包小、部分效果差异；日程类可 auto。

**Q：你怎么排查 404？**  
→ DevTools Network → 对比本地 build/web 与服务器 ls → 发现缺 Manifest。

**Q：wasm gzip 能小多少？**  
→ wasm 压缩率好，具体看 nginx 配置；要点是**类型要对**。

### 易错点

- 把本条说成「后端性能优化」——这是 **前端交付/运维**。  
- 说不出和 Bullet 5b 的关系——可合并讲「部署完整性」。

---

## 全局串联：面试官让你「用一条线讲完项目」

**推荐 90 秒话术：**

> 植物页慢，我先用 Python 按客户端真实顺序压测，定位到 tree/weekly 要 9.89 秒主要是 6 次同步 LLM，果实和树图各要十秒级即梦，而且服务器 load 很低不是机器问题。  
> 第一步给果实和树加 DB 指纹缓存，命中后一百多毫秒。  
> 第二步把 weekly 改成 fast，LLM 文案放后台 enrichment 轮询，weekly 降到 143 毫秒。  
> 第三步生图 wait=false 加 status 轮询，首屏不再阻塞十秒。  
> 同时修了 refresh 的 SQLite UNIQUE 500 和 Flutter Web 缺 Manifest 的 404。  
> 最终植物页 API 从十二到十五秒降到零点四秒级；十秒级只留在用户主动 refresh 或必须重新生图时，而且可以不挡界面。

---

## 数字速查表（背这张即可）

| 说法 | 精确值 | 含义 |
|------|--------|------|
| ~10s | 9.89～16.58s | 同步 LLM 链或即梦生图 |
| 9.89s → 143ms | tree/weekly 外网 | Fast 去掉同步 LLM |
| 16.58s → 133ms | fruits/image | DB 缓存命中 |
| 10.03s → 122ms | tree/weekly/image | 缓存+异步 |
| 12–15s → 0.4s | 植物页 API 合计 | round2 最终 |
| 12–15s → 1.5–2s | 中间态 | 仅 DB 缓存后 |
| load 0.01 | 服务器负载 | 排除 CPU 瓶颈 |
| 12MB→20MB | 部署包 | 补全缺失静态资源 |

---

## 和 Sky-Chat 简历的对比（若面试官问「两个项目性能点有何不同」）

| 维度 | Sky-Chat | ScheduleApp 本专项 |
|------|----------|-------------------|
| 瓶颈类型 | 高频 setState、SSE 解析 | 外部 LLM/即梦、DB 缓存 |
| 核心手段 | rAF 批更新、FSM、虚拟列表 | Fast API、指纹缓存、异步+轮询 |
| 指标 | 120次/s→40次/s、TBT 90ms | 接口 9.89s→143ms、整页 0.4s |
| ~10s | 不涉及 | **第三方 API 物理耗时** |
