# 生命树 Fast 接口与 AI 异步 enrichment

---

## 简历写法（可直接用）

**生命树周报 Fast 路径与 LLM 异步化（ScheduleApp）**

- 定位 `GET /tree/weekly` 首次 **9.89s**：`scoring_service` 内 **6 次** DeepSeek 调用（雷达、点评、树叙述等）全部同步阻塞。
- 设计 **Fast 默认路径**：`use_llm_comment=False`、`include_narrative=False`，首屏仅返回规则引擎得分与雷达 JSON；接口 **9.89s → 143ms**（外网），loopback **12ms**。
- 新增 **`weekly_tree_enrichments`** 表与后台任务：LLM 点评/叙述生成后写入，`GET /tree/weekly/enrichment` 供客户端轮询合并展示。
- 与前端约定：先渲染雷达与树态，AI 文案 **渐进加载**，避免白屏 10s。

---

## 遇到的问题

- `tree/weekly` 承担「算分 + 多段 LLM 文案」，首屏必等。  
- 二次 **2.13s** 仍慢：部分逻辑仍触 LLM 或 DB 未全覆盖 fast。  
- 用户需要「树立刻有反应」，文案可晚 2–3s。

---

## 我做了什么

1. **拆分关键路径 vs 增强路径**  
   - **关键路径**：规则打分、雷达维度、树等级（可缓存 `SpiritWeeklyScore`）  
   - **增强路径**：LLM 一句话点评、树叙述、情绪文案  

2. **接口参数与默认值**  
   - `fast=true`（或等价 query）为默认  
   - 响应增加 `ai_enrichment: pending | ready`  

3. **后台 enrichment**  
   - `background_runner` / asyncio 任务：算完后 upsert `weekly_tree_enrichments`  
   - 客户端：`GET /tree/weekly/enrichment?week=...`  

4. **压测与回归**  
   - 对比 fast 前后 `tree/weekly` p50  
   - 确认 `reports/weekly` 不被误伤  

---

## 结果

| 指标 | 优化前 | 优化后 |
|------|--------|--------|
| `GET /tree/weekly` 外网 | 9.89s | **143ms** |
| loopback | ~1.7s | **12ms** |
| LLM 调用（首屏） | 6 次同步 | **0 次**（后台补） |

---

## 面试要点（详细）

### Q1：为什么 6 次 LLM 会到 9.89s？能不能合并一次？

**答：**

- 每次调用含：**网络 RTT + 模型推理 + 业务拼装**；6 次串行近似相加。  
- **合并 prompt** 可减少往返，但：  
  - 输出结构复杂（雷达 vs 叙述 vs 点评）易解析失败；  
  - 单 prompt 过长影响质量与成本。  
- 我们的方案是 **首屏 0 次**，后台慢慢补，比「合并成 1 次仍阻塞首屏」更符合体验。

**追问：后台 6 次还是慢？**  
→ 用户已看到树；后台可串行或限流，不挡交互。

---

### Q2：Fast 接口返回什么？会不会「数据不完整」？

**答：**

- 返回：**分数、雷达、树状态、enrichment 状态 pending**。  
- 不返回：LLM 长文案（或返回占位/上次缓存文案若有）。  
- 前端：**骨架屏 / 上次缓存 / 「AI 正在想」**，enrichment ready 后 merge state。

---

### Q3：异步 enrichment 和「消息队列」比有什么优劣？

**答：**

| 方案 | 适用 |
|------|------|
| **进程内 asyncio + DB** | 单机、任务量小、实现快（本项目） |
| **Celery / Redis 队列** | 多机、可重试、削峰 |
| **SSE 推送到客户端** | 需要实时流式文案 |

我们选 **DB 轮询**：实现简单，与「生图 status 轮询」模式一致，面试可说「后续可换 WebSocket」。

---

### Q4：`SpiritWeeklyScore` 和 enrichment 表分工？

**答：**

- **SpiritWeeklyScore**：结构化分数、可复算、周报核心。  
- **weekly_tree_enrichments**：纯展示型 LLM 产物，可删了重生成，不影响计分。

---

### Q5：如何防止「fast 永远不调 LLM」？

**答：**

- 首次 pending 必触发后台任务（或 lazy 在 enrichment 接口触发）。  
- 监控：pending 超过 N 分钟告警。  
- 产品：用户下拉刷新可 force `include_narrative=true`（慎用，走慢路径）。

---

### Q6：loopback 12ms vs 外网 143ms，差在哪？

**答：**

- 143ms ≈ **12ms 计算 + RTT + TLS + nginx**。  
- 说明 fast 后 CPU/DB 不是瓶颈；优化有效。

---

### 易错点

- 把「异步」说成「多线程」——实际是 **asyncio 协程 + 外部 HTTP**。  
- fast 后仍在某 middleware 里调 LLM。  
- 没讲清 **客户端轮询 enrichment** 的 UX。
