# EtheRus

一个纯前端的网页音乐游戏（4 键下落式），无需构建，双击 `index.html` 即可游玩。

**当前版本：v1.0.0**（版本号在 `index.html` 的 `GAME_VERSION`，主页右下角展示）

## v1.0.0 更新内容

- Phigros 风滚动选曲 + 选曲高潮试听（带频谱可视化）
- 在线排行榜：token + HMAC 签名防篡改、IP 限流防刷、每曲独立榜单、头像与等级徽章
- 经验值/等级系统（服务端结算，等级分段换色）
- 音符币经济：每日签到 + 结算奖励 + 商店（主题/头像框/音符配色）
- 中文/English 双语界面、判定偏移校准滑块、音频输出设备选择
- 开发者面板（密码哈希 + IP 白名单）、数据导出、调试信息
- 结算演出：评级平滑弹出动画 + 结算音乐、界面全面融入网页背景
- 联系方式角落（QQ / bilibili）与一个隐藏小彩蛋

## 谱面规则（官方关卡统一标准，已审核定稿）

- **基础音符**：频谱起始检测（onset），阈值自适应，密度由难度 `minGap`/`noteCap` 控制
- **轨道分配**：频段偏好 + 近期拥挤度平衡，避免连打同一键
- **双押**：只出现在重音（强度高于中位数）上，按难度节流（普通 1/8、困难 1/6、噩梦 1/4），长按期间不出双押
- **长按**：只出现在小节最后一拍或留白处（与下一音符间隔 ≥1.5 拍），时长为整数拍（1~2 拍），按难度节流；任意时刻全场最多一条长按；同轨道长按结束后留 150ms 恢复时间
- **简单难度**：只有单键点按，无长按无双押
- 冲突校验标准：长按重叠 0、同轨侵占 0、长按内双押 0、非整拍长按 0

## 功能

- 4 键（D / F / J / K）下落式音符，Perfect / Good / Bad / Miss 判定，长按与双押
- Phigros 风纵向滚动选曲（大封面卡片、惯性滚动、吸附居中），Arcaea 风详情面板
- 选曲自动试听高潮片段（onset 密度最高的 15 秒，循环淡入淡出）
- 官方歌曲（songs/ 目录）+ 导入本地音频自动分析生成谱面
- 简单 / 普通 / 困难 / 噩梦四档难度（噩梦需困难 A 解锁）
- 三阶段演出特效、暂停（Esc）、结算分段恭喜语
- 账号系统（昵称 + 密码，本机存储）+ 在线排行榜（分歌曲独立榜单、显示头像、token + 签名防篡改、IP 限流防刷）
- 音符币经济：每日签到（7 天循环递增）+ 结算奖励，商店兑换主题 / 头像框 / 音符配色（登录限定）
- 延迟校准关（冰与火之舞式跟拍测偏移，BGM 即主菜单开门曲）+ 手动偏移滑块 + 音频输出设备选择
- 中文 / English 界面切换（设置页）
- 开发者面板：主菜单底部「开发者入口」，密码验证（SHA-256 哈希比对）+ 服务端 IP 白名单双重校验后进入，含测试模式（自动判定）、谱面编辑器、数据管理、调试信息，并解锁全部难度与商店内容

## 部署到 GitHub Pages

项目是纯静态站点，`index.html` 在仓库根目录即可。完整开发上下文见同目录 `AGENTS.md`。

1. 在 GitHub 新建一个公开仓库，例如 `EtheRus`
2. 上传本文件夹内容，或在本地执行：

   ```bash
   git init
   git add .
   git commit -m "EtheRus 初版"
   git branch -M main
   git remote add origin https://github.com/<你的用户名>/EtheRus.git
   git push -u origin main
   ```

3. 仓库页面 → **Settings → Pages** → Source 选 `main` 分支、目录选 `/ (root)` → 保存
4. 等一两分钟，访问 `https://<你的用户名>.github.io/EtheRus/`

## 在线排行榜接口

游戏内的排行榜客户端已经写好，逻辑是：

- `index.html` 顶部脚本里的 `LEADERBOARD_API` 留空 → 使用本机 localStorage
- 填入服务器地址（如 `https://api.example.com`，不带末尾斜杠）→ 自动切换为在线排行榜；网络失败时自动回退本机

本仓库自带实现：`leaderboard.ps1`（PowerShell HttpListener，localhost:8081，`scores.json` / `users.json` / `avatars.json` 存储）。接口如下：

```
POST /register   { name, pwdHash } → { ok, token }   # 每 IP 每天最多注册 3 个号
POST /login      { name, pwdHash } → { ok, token }   # 换发 token
GET  /scores?song=<id>&diff=<key>  → { top: [...], meIndex }
POST /scores?song=<id>&diff=<key>  # 提交成绩
     Headers: Content-Type: application/json, X-Sig: <HMAC-SHA256(请求体原文, LB_SECRET)>
     Body: { name, score, acc, date, fc, token }   # fc=是否全连，供服务端结算经验值
     → { top, meIndex, xpGain, xp, level }
POST /avatar     { token, dataUrl }  # 头像 ≤12KB，必须是 data:image/ 开头
POST /devauth    { pwdHash }         # 开发者模式校验：SHA-256 密码哈希 + IP 白名单（$DEV_IPS）
```

- **经验值/等级**：每次有效提交由服务端结算 XP（round(acc) + 全连 20），累计进 users.json；升级所需经验递增（升 L+1 需 100×L）；榜单条目带 `level` 字段，客户端在玩家名旁显示等级徽章（按等级段换色：1-9 灰 / 10-24 绿 / 25-49 蓝 / 50-99 紫 / 100+ 金）

- `top`：该「歌曲 + 难度」按分数降序前 10 名（含 `avatar` 字段），同名只保留最好成绩
- 提交校验：token 与名字绑定、X-Sig 签名逐字节校验、时间戳 ±5 分钟防重放、score 0~1,000,000 / acc 0~100 范围校验
- 限流（按 IP，Cloudflare 隧道走 `CF-Connecting-IP` 头）：全局 60 次/分，注册 5 次/时，提交 10 次/分，超限 429
- `LB_SECRET` 内嵌在客户端，只能提高刷分门槛，无法根除改内存——纯前端游戏的固有限制
- 音符币 / 签到 / 商店数据只存在本机（装饰性货币），不经过排行榜服务器

CORS 已开（`Access-Control-Allow-Origin: *`）。用其他语言重写时请保持同一套协议。

## 移植为电脑软件版的准备

代码已按可打包标准整理：单文件、无外部依赖、无构建步骤，任何壳都能直接包。

推荐两条路线：

### Electron（生态成熟，体积 ~150MB）

```bash
npm init -y
npm install --save-dev electron
```

新建 `main.js`：

```js
const { app, BrowserWindow } = require("electron");
app.whenReady().then(() => {
  const win = new BrowserWindow({ width: 500, height: 780, autoHideMenuBar: true });
  win.loadFile("index.html");
});
```

`package.json` 加 `"main": "main.js"` 和 `"scripts": { "start": "electron ." }`，打包用 `electron-builder`。

### Tauri（体积小 ~10MB，需要 Rust 环境）

```bash
cargo install tauri-cli
cargo tauri init   # 前端目录指向本文件夹
cargo tauri build
```

注意事项：

- 游戏音频依赖浏览器自动播放策略，已用「开始游戏」按钮满足用户手势要求，壳内同样适用
- localStorage 在 Electron/Tauri 中均可正常使用，本机排行榜无需改动
- 若配置了 `LEADERBOARD_API`，桌面版自动共享同一个在线排行榜

## 局域网试玩（可选）

`server.ps1` 是一个极简局域网服务器（PowerShell 编写，无需安装任何环境）：

```powershell
powershell -ExecutionPolicy Bypass -File server.ps1
```

启动后同一 Wi-Fi 下的设备访问 `http://<本机IP>:8080/` 即可游玩。仅供试玩，不作为正式部署方案。
