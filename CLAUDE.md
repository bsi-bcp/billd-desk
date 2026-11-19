# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> ⚠️ 本文件是该 fork（`bsi-bcp/billd-desk`）**后加的**，上游 `galaxy-s10/billd-desk` 没有此文件。本仓只是第三方开源客户端的个人 fork，勿把本项目（BCP）内部信息、凭据或业务代码提交/PR 回上游。

## 这是什么

BilldDesk 开源版客户端：跨平台远程桌面控制（类 ToDesk / 向日葵），**Vue3 + Vite + Electron**，MIT。

- `README.md` 是 **BilldDeskPro** 文案；开源版功能口径看 `README_OpenSource.md`。Pro 源码闭源，不在本仓。
- 项目**从未发布 1.0 稳定版**，main 即 dev，任意 commit 都可能含破坏性变更（作者明示）。
- 本仓只有客户端；服务端在同级另一个仓 `billd-desk-server`（Koa2 + TS + Socket.io + MySQL/Redis），两者必须配套联调。
- 文档与代码可能不同步：`doc/start-client.md` 里的 `build:web` / `build` 等脚本名已过时，**以 `package.json` 的 scripts 为准**。

## 架构

- `src/` = Vue3 renderer；`electron-main/` = Electron 主进程 + preload，由 `vite-plugin-electron/simple` 构建为 `electron-dist/index.cjs`（package.json 的 `main`）与 `preload.mjs`。
- 两种产物：
  - Web 构建：`VITE_APP_RELEASE_PROJECT_ISWEB=true` 只改变 Vite `base`（静态资源前缀指向 CDN）；electron 插件未被裁剪，同一次 build 仍会产出 `electron-dist/`，部署时按需只取 `dist/`。
  - 桌面构建：再经 electron-builder 打包到 `electron-release/<version>/`（win nsis / mac dmg / linux AppImage）。
- 路由为 **hash 模式**；主进程 `createWindow` 以 `#/<route>?windowId=` 打开多个窗口（`WINDOW_ID_ENUM`：remote / webrtc / about / version），`src/event.ts` 的 `IPC_EVENT` 是主进程与窗口间的全局协议。`src/utils` 的 `ipcRenderer` 取自 `window.electronAPI`，浏览器里为空 → IPC 能力自动降级。
- 全局状态（pinia）：`networkStore.wsMap(roomId)` / `rtcMap(socketId)` 管信令与 WebRTC 实例；`cacheStore`（persist 到 localStorage）存本机设备码/密码与已连接设备；`appStore` 存窗口信息与多路远程会话配置。

**远程控制一次完整链路**（跨多个文件，改动前按序理解）：

1. 设备身份：首次运行 `POST /desk_user/create` 领取 8 位 uuid + 密码，存 `cacheStore`（`src/api/deskUser.ts`、`src/store/cache`）。
2. 信令：socket.io，**房间号 = 设备码字符串**（字段名沿用直播域的 `live_room_id` / `roomId`）。`src/hooks/use-websocket.ts` 发 `billdDeskJoin`；主控端发 `billdDeskStartRemote`（双方 uuid + 密码），服务端校验后回 `billdDeskStartRemoteResult`（含被控端 socketId）。
3. 媒体：纯 P2P WebRTC。**被控端**（`src/views/remote/index.vue`，持 `desktopCapturer`/`getUserMedia` 屏幕流）发 offer，控制端（`src/views/webrtc/index.vue`）回 answer；`nativeWebRtcOffer / nativeWebRtcAnswer / nativeWebRtcCandidate` 由服务端在房间内转发（`src/hooks/webrtc/remoteDesk.ts`、`src/utils/network/webRTC.ts`）。
4. 控制：控制端把鼠标/键盘操作封成 `billdDeskBehavior` WS 消息；被控端经 `use-ipcRendererSend.handleRtcBilldDeskBehavior`（按屏幕尺寸 × 缩放比换算坐标）→ `IPC_EVENT` → `electron-main/index.ts` 用 `@nut-tree-fork/nut-js` 注入系统。

- `src/hooks/webrtc/` 下 `live.ts`、`srs.ts`、`tencentcloudCss.ts`、`meeting*.ts` 是 billd-live 直播域遗留代码；远程桌面只用 `remoteDesk.ts` 与 `nativeWebRtc*` 消息，改公共 hook 前先确认调用方。
- 配置：`src/spec-config.ts` 的 `prodDomain` 是单点 → `src/constant.ts` 派生 API/WS/TURN 地址（dev 固定 `ws://localhost:4300`、`/api` 走 Vite 代理）；运行时可在设置页用 `LS_KEY.axiosBaseUrl / wssUrl / coturnUrl` 覆盖（`src/utils/localStorage/app.ts`）。

## 命令

```bash
pnpm i               # .npmrc：@billd* 走私有镜像 registry.hsslive.cn，其余 npmmirror（不可达时装不上）
npm run dev          # rimraf electron-dist dist && vite（renderer + Electron 主进程一起起）
npm run lint         # eslint；dev 时 vite-plugin-checker 还会跑 vue-tsc
npm run lint:fix
npm run build:prod   # Web 部署产物 → dist/（build:beta 同脚本；electron-dist 也会一起生成）
npm run build:win    # 桌面包；build:mac / build:linux / build:ele / build:mw / build:deb
npm run gen-icons
```

- `build:win / build:mac / build:linux / build:ele / build:mw / build:deb` 均以 `standard-version` 开头：**会改 package.json 版本并发 tag**，要求当前 git 工作区干净，否则打包失败。
- `deploy:beta` / `deploy:prod` / `copy` 依赖 `deploy/` 目录，而该目录被 `.gitignore` 排除 → 干净 checkout 下不可用。

## 联调与测试

- 联调：先起服务端（需要 MySQL/Redis，开发端口 4300），再在本仓 `npm run dev`。Vite 已把 `/api`、`/betaapi`、`/prodapi` 分别代理到 4300/4300/4200 并去掉前缀。最省事的验证：同一浏览器开两个窗口，一个领设备码当被控端，另一个输入设备码发起连接。
- 本仓**没有单元测试框架，也没有 `test` 脚本**；`test/` 只有手工脚本。可用的门禁：`npm run lint`、dev 时 `vite-plugin-checker`（vue-tsc + eslint）、手工双端联调。

## 提交与发布

- 约定式提交（commitizen：`npm run cz`）。
- `standard-version` 依据提交生成 CHANGELOG 与 tag（`v0.x.y`，`release.md` 是更新日志）。
