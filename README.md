# Vue2-TodoList 云服务器部署指南

本指南记录了如何将 `Vue2-TodoList` 项目部署至 Linux 云服务器（以宝塔面板 Nginx + Node.js 环境为例）。

## 1. 环境要求
- **操作系统**: Linux (如 Ubuntu, CentOS, OpenCloudOS)
- **Web 服务器**: Nginx
- **运行时**: Node.js **v20.x** 或更高版本 (Vite 6+ 的要求)
- **管理工具**: 宝塔面板 (可选，但建议用于简化配置)

---

## 2. 后端服务部署 (Node.js)

项目后端由 `server.js` 驱动，运行在 `4096` 端口，通过 `tasks.json` 持久化存储数据。

### 步骤：
1. **上传代码**: 将项目源码上传至服务器目录（如 `/www/wwwroot/vue.austoin.asia`）。
2. **安装依赖**:
   在项目根目录打开终端，执行：
   ```bash
   npm install
   ```
3. **放行端口**: 
   - 在宝塔“安全”菜单放行 `4096` 端口。
   - 在云厂商控制台（腾讯云/阿里云）的安全组中放行 `4096` 端口。
4. **启动服务**:
   在宝塔 **“Node项目”** 中添加项目：
   - **项目目录**: 选择项目根目录。
   - **启动文件**: `server.js`。
   - **端口**: `4096`。
   - 确认状态为“运行中”。

---

## 3. 前端配置与打包 (Vue + Vite)

为了解决 **HTTPS 网页禁止请求 HTTP 接口 (Mixed Content)** 的安全限制，必须使用相对路径并配合 Nginx 反向代理。

### 步骤：
1. **修改 API 地址**:
   编辑 `src/App.vue`（或相关配置文件），将硬编码的 IP 地址改为相对路径：
   ```javascript
   // 错误做法：const API_URL = 'http://43.x.x.x:4096/api' (会导致混合内容报错)
   // 正确做法：
   const API_URL = '/api'
   ```
2. **修复文件权限**:
   确保打包工具对目录有操作权限：
   ```bash
   chown -R www:www /www/wwwroot/vue.austoin.asia
   chmod -R 755 /www/wwwroot/vue.austoin.asia
   ```
3. **执行编译**:
   在终端执行打包命令，生成 `dist` 文件夹：
   ```bash
   npm run build
   ```

---

## 4. Nginx 站点配置

通过 Nginx 统一入口，处理静态资源并转发 API 请求。

### 步骤：
1. **设置运行目录**:
   在“网站设置”中，将 **网站目录** 指向项目的根目录，将 **运行目录** 设置为 `/dist`。
2. **配置反向代理 (核心)**:
   由于前端使用 `/api` 相对路径，需要在 Nginx 配置文件中手动添加转发规则。点击“设置”->“配置文件”，在 `server` 块内添加：
   ```nginx
   # 拦截所有 /api 开头的请求并转发给后端 Node 服务
   location ^~ /api/ {
       proxy_pass http://127.0.0.1:4096;
       proxy_set_header Host $host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header REMOTE-HOST $remote_addr;
       
       # 允许跨域
       add_header 'Access-Control-Allow-Origin' '*';
   }
   ```
3. **防止刷新 404 (可选)**:
   在“伪静态”中添加：
   ```nginx
   location / {
       try_files $uri $uri/ /index.html;
   }
   ```

---

## 5. 常见问题排查 (FAQ)

### Q1: 为什么提示 `Mixed Content` 错误？
**原因**: 你的站点启用了 HTTPS (SSL)，但前端 JS 脚本尝试请求 `http://IP:4096`。
**解决**: 必须按第 3 步修改 `API_URL = '/api'`，并通过 Nginx 反向代理进行转发。

### Q2: `npm run build` 报错找不到 `index.html`？
**原因**: 可能是 Linux 下文件名大小写敏感或权限不足。
**解决**: 确认 `index.html` 在根目录，并执行 `chmod -R 755` 赋予权限。

### Q3: 为什么修改了代码但网页没变化？
**原因**: 
1. 没执行 `npm run build`。
2. 浏览器缓存了旧的 `dist/assets` 下的 JS 文件。
**解决**: 重新打包后，在浏览器中使用 `Ctrl + F5` 强制刷新，或使用无痕模式访问。

### Q4: 为什么 Node 项目启动失败？
**原因**: 可能是从 Windows 上传了不兼容的 `node_modules`。
**解决**: 删除服务器上的 `node_modules` 和 `package-lock.json`，在服务器端重新执行 `npm install`。

---

## 6. 维护命令参考
- **查看后端日志**: 在宝塔 Node 项目列表中点击“日志”。
- **重启后端**: `npm run server` 或在宝塔界面点击“重启”。
- **更新代码**: 
  1. 上传新源码。
  2. 终端执行 `npm run build`。
  3. 重启 Node 项目。

---

**部署人**: [Austoin]  
**日期**: 2026-01-17