# EtheRus

一个纯前端的网页音乐游戏（4 键下落式），无需构建，双击 `index.html` 即可游玩。

## 功能

- 4 键（D / F / J / K）下落式音符，Perfect / Good / Miss 三档判定
- 4 首内置合成乐曲（约 90 秒，含鼓组 / 贝斯 / 钢琴 / 主音），旋律确定性生成
- 支持导入本地音频文件，自动分析节拍生成谱面
- 简单 / 普通 / 困难三档难度
- 三阶段演出特效、暂停（Esc）、结算分段恭喜语
- 昵称系统 + 排行榜（本机 / 在线可切换）
- 管理员模式：开始界面底部「管理员入口」，密码见源码 `ADMIN_PASSWORD`，解锁测试模式（自动判定，成绩不入榜）

## 部署到 GitHub Pages

项目是纯静态站点，`index.html` 在仓库根目录即可。

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

- `index.html` 顶部脚本里的 `LEADERBOARD_API` 留空 → 使用本机 localStorage（现状）
- 填入服务器地址（如 `https://api.example.com`，不带末尾斜杠）→ 自动切换为在线排行榜；网络失败时自动回退本机

服务器只需实现一个接口：

```
POST {LEADERBOARD_API}/scores?song=<歌曲id>&diff=<难度key>
Content-Type: application/json

{ "name": "玩家昵称", "score": 12345, "acc": 98.5, "date": 1718000000000 }
```

期望响应：

```json
{
  "top": [{ "name": "...", "score": 12345, "acc": 98.5, "date": 0 }],
  "meIndex": 0
}
```

- `top`：该「歌曲 + 难度」下按分数降序的前 10 名
- `meIndex`：本次提交玩家在 `top` 中的下标（用于高亮），不在前 10 则传 `-1`
- 同名玩家建议只保留最好成绩

可以用任何语言实现（Node/Express、Cloudflare Workers、Vercel Functions 等均可）。注意要开 CORS（`Access-Control-Allow-Origin: *`）。

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
