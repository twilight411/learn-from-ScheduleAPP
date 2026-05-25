# Flutter Web 静态资源完整部署

## 遇到的问题

浏览器控制台大量 **404**：

- `/assets/AssetManifest.bin.json`
- `/assets/FontManifest.json`
- `/manifest.json`

表现：引擎反复报错，图标/资源加载异常，页面像「初始化很慢或坏了」。

服务器 `web_build/assets/` **只有子目录，没有 Manifest 文件**（残缺部署包约 **12MB**；完整包约 **20MB**）。

---

## 根因

1. 曾用不完整 `flutter build` 产物打 tar 上传（或构建未完成）  
2. `deploy_web_build.py` 只检查 `main.dart.js`，未校验 Manifest  
3. 用户通过 **:8000 uvicorn 直出** 静态文件，缺文件即 404  

**与 API 慢无关**，但叠加后「打开 App」体感很差。

---

## 解决方法

1. `flutter clean && flutter build web --release`  
2. 确保 `build/web` 含 `assets/AssetManifest.bin.json`、`FontManifest.json`、`manifest.json`  
3. 整包 tar 部署到 `/app/spirit-scheduler/web_build`  
4. `index.html` 增加 `window.flutterConfiguration = { renderer: "auto" }`  
5. `deploy_web_build.py` 增加 **REQUIRED_FILES** 校验  

修复后 curl：`/assets/AssetManifest.bin.json` → **HTTP 200**。

---

## 数据

| 指标 | 修复前 | 修复后 |
|------|--------|--------|
| 部署包大小 | ~12.4 MB | **~20.2 MB** |
| `AssetManifest.bin.json` | MISSING | **12362 bytes** |
| API 植物接口 | 仍可能慢 | 独立优化（见其他主题） |

---

## 面试要点

**Q：Flutter Web 部署要检查什么？**  

A：不止 `main.dart.js`；必须包含 `assets/AssetManifest*`、`FontManifest.json`、`canvaskit/` 或 skwasm、`flutter_bootstrap.js`；用清单校验；强刷缓存；区分静态服务器与 API 反代路径。

**Q：CanvasKit 太大怎么办？**  

A：`renderer: auto` / skwasm；nginx 对 `.wasm` 开 gzip 与正确 `Content-Type`；考虑 CDN；代码分割不能省略 assets 目录。
