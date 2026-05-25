# 生命树 Fast 接口与 AI 异步补全

## 遇到的问题

`GET /tree/weekly`（植物状态 / 雷达）首次 **~9.89s**，二次仍 **~2.13s**（树叙述仍走 LLM）。

压测基线：`tree/weekly` 外网 **1558–1763ms**（已有得分时）；用户无得分首次更高。

---

## 根因

主路径同步执行：

1. `ScoringService.calculate_all_spirits`：`use_llm_comment=true` → 5 次精灵 LLM 点评  
2. 可选 `quality_note` 批量 LLM 校准  
3. `TreeService._generate_tree_narrative`：1 次树叙述 LLM  

植物页**必须先等**该接口返回才能请求生图。

---

## 解决方法

### 1. Fast 主接口

`build_tree_data(..., fast=True)`：

- `calculate_all_spirits(..., use_llm_comment=False, calibrate_quality_notes=False)` → 规则 fallback 点评  
- `include_narrative=False` → 先用 `weekly_summary_line`  
- 响应增加 `ai_enrichment: pending | ready`

### 2. 后台补全

- 表 `weekly_tree_enrichments`  
- `schedule_ai_enrichment()`：后台跑 LLM 点评 + 树叙述  
- `GET /tree/weekly/enrichment` 供轮询  

---

## 数据

| 场景 | 优化前 | 优化后（19:45） |
|------|--------|-----------------|
| `tree/weekly` 外网 | 9.89s（首次） | **143ms** |
| `tree/weekly` loopback | ~1726ms | **12ms** |
| `tree/weekly/enrichment` | — | **286ms**（查状态） |

植物页 API 串行「雷达+生图占位」：约 **0.23s**（见 round2 报告 §4）。

---

## 面试要点

**Q：为什么不让 LLM 全异步？**  

A：雷达/得分依赖 DB 聚合，要快；**文案类**可异步。原则：**关键路径只返回可渲染的最小集**，重计算/贵调用后台化 + 轮询或 WebSocket。

**Q：fast 后数据会不会不准？**  

A：得分来自规则计算（非 LLM）；变的是**展示文案**；后台补全后 enrichment 接口给完整叙述。需产品接受「叙述晚几秒出现」。
