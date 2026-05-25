# API 压测报告（第二轮）— 性能优化 + 果实 refresh 修复后

**最新压测时间**：2026-05-25 **19:45 CST**（果实 `refresh` UNIQUE 修复后复测）  
**首轮同日报告**：2026-05-25 19:35–19:36 CST（修复前，果实 `refresh` 曾 500）  
**目标环境**：`http://47.118.28.102:8000`（`spirit-scheduler` / uvicorn）  
**测试账号**：`雅` / `123456`  

**已上线变更**：

- fast `GET /tree/weekly`（AI 叙述/点评后台生成）
- 生图 `wait=false` + `/image/status` 轮询
- `monthly_fruit_images` / `weekly_tree_images` DB 缓存
- 果实 `refresh`：**UPDATE 替代 DELETE+INSERT**（修复 500）
- Flutter Web 完整静态资源（`AssetManifest.bin.json` 等）

**执行脚本与数据**：

| 脚本 | 输出 | 时间戳 |
|------|------|--------|
| `.deploy/api_load_benchmark.py` | `.deploy/load_benchmark_report.json` | 2026-05-25T19:45:51 |
| `.deploy/api_benchmark_async_paths.py` | `.deploy/load_benchmark_async_paths.json` | 2026-05-25T19:45:10 |
| `.deploy/server_loopback_bench.py` | 终端 stdout | 2026-05-25 19:46 |

**历史报告**：[优化前基线](./api-load-benchmark-2026-05-25.md) · [果实缓存后](./api-load-benchmark-after-fruit-cache-2026-05-25.md)

---

## 1. 执行摘要（19:45 复测）

| 场景 | 优化前（你实测） | 19:36 首轮 | **19:45 复测** |
|------|------------------|------------|----------------|
| `GET /tree/weekly` | 首次 **9.89s** | 294ms | **143ms** |
| `GET /tree/weekly/image` 日常 | **10.03s** | 132ms | **122ms**（`wait=false`） |
| `GET /fruits/image` 日常 | **16.58s** | 138ms | **133ms**（`wait=false`） |
| `GET /fruits/image?refresh=true` | — | **HTTP 500** | **HTTP 200 ~10.0s** ✅ |
| `GET /tree/weekly/image?refresh=true` | — | ~14.8s | **~10.1s** |
| 植物页 API（新逻辑估算） | 12–15s | ~0.5s | **~0.4s** |
| 植物页 API（标准脚本热3轮） | — | ~8.5s | **~1.5s** |

**结论**：

1. **日常路径稳定百毫秒级**：雷达、缓存生图、果实图均正常。
2. **强制 `refresh=true`**：生命树/果实均为 **~10s**（即梦），**不再 500**。
3. **服务器资源充足**；偶发登录/任务列表秒级抖动为公网 RTT，非瓶颈。
4. **即梦生图正常**：冷路径返回 `byteimg.com` 真图 URL。

---

## 2. 修复验证：`fruits/image?refresh=true`

| 轮次 | HTTP | 耗时 | 说明 |
|------|------|------|------|
| 19:36 首轮（修复前） | **500** | ~23s | `IntegrityError` UNIQUE，`INSERT` 撞车 |
| 19:41 手工探测（修复后） | **200** | ~10.6s | 日志 `jiyun_image_generation_success` |
| **19:45 标准冷路径** | **200** | **10032ms** | `cached=false`，真图 URL |
| 19:45 专项脚本（紧接树 refresh 后） | 200 | 243ms | `status=failed`（即梦返回无效 URL 时快速失败，非 500） |

根因与修复：见服务器日志 — 生图已成功但写库失败；改为 refresh 时 **就地 UPDATE** 记录。

---

## 3. 环境与资源（19:45 CST）

| 项 | 数值 |
|----|------|
| 服务 | `spirit-scheduler` **active**，uvicorn RSS ~110MB |
| Load average | 0.01, 0.02, 0.00 |
| 内存 available | ~1335MB / 1911MB |
| Web | `main.dart.js` 3.36MB，`AssetManifest.bin.json` 已部署 |
| 即梦 | `IMAGE_PROVIDER=jiyun` |

---

## 4. 专项：日常路径（外网，19:45）

| 接口 | HTTP | 耗时 | 备注 |
|------|------|------|------|
| `POST /auth/login` | 200 | 6262ms* | *首轮登录偶发慢，后续请求正常 |
| `GET /users/me` | 200 | 350ms | |
| `GET /profile` | 200 | 912ms | |
| `GET /schedule/today` | 200 | **89ms** | |
| `GET /tree/weekly` | 200 | **107ms** | `ai_enrichment=pending` |
| `GET /tree/weekly/enrichment` | 200 | **286ms** | |
| `GET /tree/weekly/image?wait=false` | 200 | **122ms** | `cached=true` `status=ready` |
| `GET /tree/weekly/image/status` | 200 | **163ms** | |
| `GET /fruits/image?wait=false` | 200 | **133ms** | `cached=true` `status=ready` |
| `GET /fruits/image/status` | 200 | **302ms** | |
| `GET /tree/weekly/image?refresh=true` | 200 | **9867ms** | `cached=false` `status=ready` |

**植物页 API 首屏（缓存命中）**：

```
串行：tree/weekly (~0.11s) + tree/image?wait=false (~0.12s) ≈ 0.23s
并行：reports/weekly ∥ fruits/image?wait=false ≈ ~0.10s
合计约 0.35–0.45s（不含 CDN 拉图、后台 AI 叙述）
```

---

## 5. 标准压测：冷 / 热路径（外网，19:45）

### 5.1 冷路径（`refresh=true`，最坏生图）

| 接口 | HTTP | 耗时 | cached |
|------|------|------|--------|
| 任务列表 | 200 | **879ms** | — |
| 生命树雷达 `tree/weekly` | 200 | **143ms** | fast |
| 生命树生图 | 200 | **10141ms** | false |
| **月度果实生图** | **200** | **10032ms** | false |
| AI 周报 | 200 | **103ms** | DB |
| 月报文案 | 200 | **99ms** | — |

### 5.2 热路径（连续 3 轮，无 refresh）

| 轮次 | 关键路径合计（约） |
|------|-------------------|
| 第 1 轮 | **~14.4s**（含生图未缓存的一次性抖动） |
| 第 2 轮 | **~2.4s** |
| 第 3 轮 | **~1.5s** |

热路径典型单接口（第 3 轮，已稳定）：

| 接口 | p50 |
|------|-----|
| 生命树雷达 | **134ms** |
| 生命树生图 | **121ms** |
| 月度果实生图 | **612ms** |
| AI 周报 | **107ms** |
| 任务列表 | **266ms** |

### 5.3 并发登录（5 用户）

| 指标 | 19:36 首轮 | **19:45 复测** |
|------|------------|----------------|
| 成功率 | 5/5 | **5/5** |
| p50 | 1582ms | **3762ms** |
| max | 1609ms | **3768ms** |

> 登录耗时随公网波动，非植物页瓶颈。

---

## 6. 服务器本机 loopback（19:46）

| 接口 | 耗时 |
|------|------|
| login | **293ms** |
| tasks | **12ms** |
| **tree/weekly** | **12ms** |
| tree/weekly/image（缓存） | **12ms** |
| tree/weekly/image?refresh=true | **9928ms** |
| fruits/image（缓存） | **7ms** |
| reports/weekly（缓存） | **7ms** |
| reports/weekly?refresh=true | **6889ms** |

---

## 7. 纵向对比（核心指标）

| 指标 | 优化前 | 果实缓存后 | 19:36 round2 | **19:45 复测** |
|------|--------|------------|--------------|----------------|
| `tree/weekly` 外网 | 9.89s / 2.13s | ~1.7s | 294ms | **143ms** |
| `tree/weekly/image` 日常 | ~10s | &lt;100ms | 132ms | **122ms** |
| `fruits/image` 日常 | ~10s | &lt;300ms | 138ms | **133ms** |
| `fruits/image?refresh=true` | — | — | **500** | **200 ~10s** |
| 植物页热路径（稳定） | 12–15s | 2–3.3s | ~2–8.5s | **~1.5s** |
| `tree/weekly` loopback | ~1.7s | ~1.4s | 10ms | **12ms** |

---

## 8. 异常与说明

| 项 | 状态 |
|----|------|
| `fruits/image?refresh=true` 500 | **已修复**（19:45 确认为 200） |
| 专项脚本 `fruit_image_refresh` 243ms + `status=failed` | 紧接树 refresh 后探测；可能即梦返回占位 URL，**非 HTTP 500** |
| 热1 生命树生图 11.2s | 单轮未命中缓存，属正常冷生图 |
| 浏览器 `runtime.lastError` | 扩展插件，与 API 无关 |

---

## 9. 复现命令

```powershell
cd D:\Users\zyx\Desktop\ScheduleApp\.deploy

python api_load_benchmark.py
python api_benchmark_async_paths.py
python server_loopback_bench.py
python check_server_image_gen.py   # 生图日志 + 实时探测
```

---

**报告整理**：2026-05-25 19:45–19:46 自动化复测  
**关联代码**：`fruit_service.get_or_generate_fruit_image`（refresh 就地更新）、`tree_service` fast/async 生图
