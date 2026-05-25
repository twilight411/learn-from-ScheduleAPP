# Flutter Web 静态资源部署与首包优化

---

## 简历写法（可直接用）

**Flutter Web 线上部署与首包优化（ScheduleApp）**

- 线上出现 **`AssetManifest.bin.json` 404**、白屏：排查为 **残缺部署**（仅上传部分 `build/web`，缺 Manifest / 部分 assets）。
- 编写 **`deploy_web_build.py`**：打包前 `flutter clean && build web`，部署后校验 **REQUIRED_FILES**（`AssetManifest.bin.json`、`flutter_bootstrap.js`、`main.dart.js`、wasm 等），包体 **12MB → 20MB** 全量。
- **Nginx**：为 `.wasm` 配置 **gzip** 与 `Content-Type: application/wasm`，降低 CanvasKit 传输体积。
- **`web/index.html`**：设置 `flutterConfiguration.renderer = "auto"`，避免强制 CanvasKit（**~7.1MB wasm**），优先 HTML/Canvas 或 Skwasm 以改善弱网首屏。

---

## 遇到的问题

- 浏览器控制台：`Failed to load resource: AssetManifest.bin.json 404`。  
- 用户感觉「登录后植物页慢」——部分来自 **前端首包 10MB+** 下载与解析，与 API 慢叠加。  
- 仅替换 `main.dart.js` 导致版本与 Manifest **不一致**。

---

## 我做了什么

1. **根因定位**  
   - 对比本地 `build/web` 与服务器目录文件列表  
   - 确认缺 `AssetManifest.bin.json` 及 assets 子目录  

2. **部署流程硬化**  
   - `flutter clean` → `flutter build web` → tar 全量上传  
   - 脚本列出 REQUIRED_FILES，缺一则失败退出  

3. **Nginx 调优**（`patch_nginx_wasm.py`）  
   - `gzip_types` 含 `application/wasm`  
   - 避免 wasm 被当 `application/octet-stream` 且无压缩  

4. **渲染器策略**  
   - 去掉强制 `canvaskit`  
   - `renderer: auto` 让 Flutter 按环境选择  

---

## 结果

| 项 | 说明 |
|----|------|
| 404 | 全量部署后消失 |
| 部署包 | **12MB → 20MB**（完整 assets） |
| CanvasKit wasm | 仍 **~7.1MB**（若走 canvaskit）；auto 可减轻 |
| API 慢 | 与前端首包 **独立**，需分别优化（见主题 01） |

---

## 面试要点（详细）

### Q1：AssetManifest 是干什么的？

**答：**

- Flutter Web 用其映射 **逻辑 asset 名 → 实际 URL/hash 路径**。  
- 缺文件则 **无法加载字体、图片、本地化**，常在 bootstrap 阶段失败 → 白屏。  
- 与 **Service Worker** 缓存策略相关（若启用）。

---

### Q2：为什么「只更新 main.dart.js」会挂？

**答：**

- `main.dart.js` 引用的 asset hash 在 Manifest 里；**旧 Manifest + 新 js** → 路径对不上。  
- 正确做法：**整包原子替换**（或 CDN 带版本目录 `build/20260525/`）。

---

### Q3：CanvasKit vs HTML renderer 权衡？

**答：**

| | CanvasKit | HTML/Canvas |
|--|-----------|-------------|
| 一致性 | 高，接近移动端 | 部分效果差异 |
| 首包 | **+7MB wasm** | 较小 |
| 弱网 | 差 | 较好 |

产品偏 **日程工具** 可接受 auto；设计-heavy 才强推 CanvasKit。

---

### Q4：nginx 对 wasm 要注意什么？

**答：**

- `Content-Type: application/wasm`（否则编译失败）  
- **gzip/brotli**（wasm 可压缩性好）  
- **HTTP/2** 多路复用减 RTT  
- 缓存：`Cache-Control` 带 hash 文件名可长期缓存  

---

### Q5：如何区分「前端慢」和「API 慢」？

**答：**

- DevTools **Network**：看 `main.dart.js`、wasm 下载时间 vs `/tree/weekly` 等待。  
- Performance：**TTFB** 高是 API；**下载 7MB** 是静态资源。  
- 我们压测证明 API 可 **143ms**；首包仍是另一战场。

---

### Q6：部署校验清单你会列哪些？

**答：**

- `index.html`、`flutter_bootstrap.js`、`main.dart.js`  
- `AssetManifest.bin.json`（及 `.json` 变体）  
- `assets/`、`canvaskit/` 或 `skwasm/` 目录  
- `flutter_service_worker.js`（若启用）  
- 部署后 **curl -I** 抽查 Content-Type 与 200  

---

### 易错点

- 把 API 优化当成 Web 也会快——**要分开测**。  
- 忘记 **clean build** 导致旧 hash 残留。  
- wasm 未 gzip 却怪 Flutter 太大。
