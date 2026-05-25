# Refresh 刷新与 UNIQUE 约束 500

## 遇到的问题

压测 `GET /fruits/image?month=2026-05&refresh=true`：

- **HTTP 500**
- 耗时仍 **~23s**（即梦已跑完）

日志：

```text
sqlalchemy.exc.IntegrityError: UNIQUE constraint failed:
  monthly_fruit_images.user_id, monthly_fruit_images.month
[SQL: INSERT INTO monthly_fruit_images ...]
```

**误解**：「是不是缓存导致 500？」—— **不是**。

---

## 根因

`refresh=true` 时旧逻辑：

1. `DELETE` 该行 → `row = None`  
2. 同步 `generate_fruit_image()` **~10–23s**  
3. 结束时 `INSERT` 新行  

在步骤 2 期间：

- 同 session/并发请求可能已有行；或 delete 与 insert 竞态  
- 生图已成功，**写库 INSERT 撞 UNIQUE** → 500  

生命树 refresh 用 SQL `DELETE` 后 INSERT，压测多为 **200**；果实路径更容易在长跑生图时撞车。

---

## 解决方法

**不再 DELETE**，`refresh` 时：

- 跳过缓存命中判断（`not refresh`）  
- 生图完成后对**已有行 UPDATE**（`score_fingerprint`、`image_url`、`image_status`）  
- 无行才 INSERT  

---

## 数据

| 轮次 | HTTP | 耗时 |
|------|------|------|
| 19:36 修复前 | **500** | ~23s |
| 19:41 修复后探测 | **200** | ~10.6s |
| **19:45 标准冷路径** | **200** | **10032ms** |

日常 **无 refresh**：`cached=true`，**109–133ms**，**不会 500**。

---

## 面试要点

**Q：缓存和 refresh 矛盾吗？**  

A：不矛盾。缓存服务**读路径**；refresh 是**失效策略**。应用层应 **upsert/update**，避免「先删后插」在长耗时操作中间留空窗。

**Q：如何设计 refresh API？**  

A：返回 `job_id` 或 `status=pending`；后台生成；完成后 UPDATE；或软失效（版本号/指纹变化）而非物理 DELETE。

**关联**：[07-生图缓存机制与面试要点](07-生图缓存机制与面试要点.md)
