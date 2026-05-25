# Refresh 刷新写库 UNIQUE 冲突（500 修复）

---

## 简历写法（可直接用）

**果实生图 refresh 并发写库修复（ScheduleApp）**

- 现象：`GET /fruits/image?refresh=true` 返回 **500**，日志 **SQLite IntegrityError UNIQUE**；即梦侧 **生图已成功**，属「写库失败」而非第三方失败。
- 根因：refresh 采用 **DELETE 旧行 + INSERT 新行**；生图 **~10s** 窗口内重复请求或重试导致 **双 INSERT** 同一 `(user_id, year, month)`。
- 修复：refresh 改为对**同一主键行 UPDATE**（`image_url`、`score_fingerprint`、`image_status`），保证幂等；压测 **500 → 200**，耗时 **~10s**（符合即梦预期）。

---

## 遇到的问题

- 用户点「换一张果实图」→ 500，体验差于慢。  
- 日志：UNIQUE constraint failed: `monthly_fruit_images.user_id, year, month`。  
- 易误判为「缓存坏了」或「即梦挂了」。

---

## 我做了什么

1. **复现与日志对齐**  
   - 连续两次 refresh 或 refresh + 日常 GET 交叉  
   - 确认第一次 DELETE 后、INSERT 前第二次请求又 INSERT  

2. **改 `fruit_service.py` 写路径**  
   - **存在** → `UPDATE ... SET image_url=?, fingerprint=?, status=pending`  
   - **不存在** → `INSERT`  
   - 去掉 refresh 场景的 DELETE  

3. **回归压测**  
   - `fruits/image?refresh=true`：**200**，~10s，无 500  

4. **文档**  
   - 记录「缓存不导致 500」结论，避免团队误判  

---

## 结果

| 场景 | 修复前 | 修复后 |
|------|--------|--------|
| `refresh=true` | **500**（偶发） | **200**，~10s |
| 即梦 | 成功但客户端报错 | 成功且 URL 落库 |
| 日常 cached | 正常 | 不受影响 |

---

## 面试要点（详细）

### Q1：为什么 DELETE+INSERT 在长事务里危险？

**答：**

- 请求 A：DELETE 成功 → 调即梦（10s）  
- 请求 B：DELETE 无行 → INSERT 行1 → 即梦…  
- 请求 A：即梦完成 → INSERT → **UNIQUE 冲突**  
- 本质：**非原子 upsert** + **长耗时中间态**。

**追问：PostgreSQL 呢？**  
→ 同样 UNIQUE；可用 `ON CONFLICT DO UPDATE` 一条 SQL 幂等。

---

### Q2：正确模式是什么？

**答：**

- **Upsert**：`INSERT ... ON CONFLICT UPDATE`  
- 或：**单行 UPDATE**（我们月维度一行一用户）  
- refresh 语义：**覆盖 URL**，不是删记录再建  

---

### Q3：如何向产品解释「refresh 还是要 10s」？

**答：**

- 500 是 **bug**，10s 是 **即梦物理耗时**。  
- 修复后：可靠 200，可先 `pending` + 旧图（配合主题 04）。  

---

### Q4：这和缓存命中率有关系吗？

**答：**

- **无**。读缓存走 SELECT；500 仅 **写路径** refresh。  
- 面试常混淆：「加了缓存为什么还 500」→ 讲清 **读/写分离**。

---

### Q5：幂等性怎么测？

**答：**

- 同一用户 **双击 refresh**、或 refresh 未完成时再 refresh  
- 期望：一条 DB 记录、最终一个 URL、无 500  
- 可加 **请求 id** 去重（进阶）  

---

### Q6：若即梦成功但 UPDATE 失败？

**答：**

- 记录 **orphan URL** 日志，定时清理  
- 重试 UPDATE  
- 客户端轮询 status 发现 `failed` 可重试 refresh  

---

### 易错点

- 只说「数据库锁」不说 **DELETE+INSERT 竞态**。  
- 认为 500 来自缓存命中逻辑错误。  
- 修复用 **重试 INSERT** 而不改模型。
