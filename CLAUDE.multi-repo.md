# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> 归档说明：本文件按**容器根**（即本仓 `billd-desk/` 的父目录）视角编写，描述 5 个子仓（billd-desk / billd-desk-server / billd-desk-admin / billd-desk-flutter / billd-desk-electron）的现状与协作关系；放在本仓根仅为随 fork 留存，文中路径均以容器根为基准。

## 目录性质（先读）

本目录是**第三方项目**（上游 `galaxy-s10`）的**多仓容器**，自身不是 git 仓；提交、打 tag、回滚必须 `cd` 进具体子仓分别进行。与 BCP 业务无关，勿把 BCP 相关内容或凭据提交进任何子仓。

| 子目录 | 状态 | 说明 |
|---|---|---|
| `billd-desk/` | 有代码 | 核心客户端：Vue3 + Vite + Electron 主进程/渲染进程，MIT，唯一可运行的前端仓（`main`，HEAD `v0.1.46-136`） |
| `billd-desk-server/` | 有代码 | 后端：Koa2 + TypeScript + Sequelize(MySQL) + Redis + Socket.io（`main`，最后提交 2025-07-15） |
| `billd-desk-admin/` | **仅文档** | 磁盘上只有 README 与 md 文档（约 23 个文件），**无源码** |
| `billd-desk-flutter/` | **仅文档** | 同上，**无 Flutter 源码** |
| `billd-desk-electron/` | **空壳** | 只剩 README，声明 Electron 端已并入 `billd-desk/` |

关键口径：

- 主仓 `billd-desk/README.md` 是 **BilldDeskPro** 文案；开源版功能口径看 `README_OpenSource.md`。Pro 源码闭源，不在此仓库。
- 项目**从未发布 1.0 稳定版**，main 即 dev，任意 commit 都可能有破坏性变更（作者明示）。
- 线上：客户端 `https://desk.hsslive.cn`，API `https://desk-api.hsslive.cn`；接口文档在 Apifox（链接见 README）。
- 文档与代码可能不同步：`billd-desk/doc/start-client.md` 里的 `build:web` / `build` 等脚本名已过时，**以 `package.json` 的 scripts 为准**。

## 架构总览

### 客户端 `billd-desk/`（一份代码，两种产物）

- **Vue3 renderer**（`src/`）+ **Electron main**（`electron-main/`，`vite-plugin-electron/simple` 构建）。
- 两种构建：
  - Web 版：`VITE_APP_RELEASE_PROJECT_ISWEB=true` 只改变 Vite `base`（静态资源前缀指向 CDN）；`electron()` 插件未被裁剪，同一次 build 仍会产出 `electron-dist/`，部署时按需只取 `dist/`。浏览器里 `window.electronAPI?.ipcRenderer` 为空，IPC 能力自动降级。
  - 桌面版：同一次构建额外产出 `electron-dist/index.cjs`（package.json 的 `main`）与 `preload.mjs`，再由 electron-builder 打包到 `electron-release/<version>/`。
- 路由为 **hash 模式**，Electron 主进程 `createWindow` 用 `#/<route>?windowId=` 打开多个窗口（`WINDOW_ID_ENUM`：remote / webrtc / about / version），`src/event.ts` 的 `IPC_EVENT` 是主进程与各窗口通信的全局约定。
- 全局状态（pinia）：
  - `networkStore.wsMap: Map<roomId, WebSocketClass>`、`rtcMap: Map<socketId, WebRTCClass>` —— WebRTC/信令实例总线；
  - `cacheStore`（persist 到 localStorage）存本机设备码/密码、已连接设备列表等；
  - `appStore` 存窗口信息、多路远程会话配置。

**远程控制一次完整链路**（跨多个文件，改动前务必按序理解）：

1. 被控端首次运行调 `POST /desk_user/create` 领取 8 位 uuid + 密码，持久化在 `cacheStore`（`src/api/deskUser.ts`、`src/store/cache`）。
2. 双方以**设备码字符串**作为 socket.io 房间号（字段名沿用直播域的 `live_room_id` / `roomId`，实际值就是 deskUserUuid）：`src/hooks/use-websocket.ts` 发 `billdDeskJoin` → 服务端 `handleWsBilldDeskJoin` 把 `deskUserUuid → socketId` 写入 Redis（10s TTL）。
3. 主控端发 `billdDeskStartRemote`（带双方 uuid + 密码）→ 服务端校验两边密码、被控端在线，回 `billdDeskStartRemoteResult`（含被控 socketId）。
4. 纯 P2P WebRTC：由**被控端**（`src/views/remote/index.vue`，持有 `desktopCapturer`/`getUserMedia` 屏幕流）发 offer，控制端（`src/views/webrtc/index.vue`）回 answer；`nativeWebRtcOffer / nativeWebRtcAnswer / nativeWebRtcCandidate` 由服务端在房间内原样转发（`src/hooks/webrtc/remoteDesk.ts`、`src/utils/network/webRTC.ts`；TURN 地址可被 `localStorage` 覆盖）。
5. 控制端把鼠标/键盘操作封成 `billdDeskBehavior` WS 消息；被控端在 `views/remote` 收到后经 `use-ipcRendererSend.handleRtcBilldDeskBehavior`（按屏幕尺寸 × 缩放比换算坐标）→ `IPC_EVENT` → `electron-main/index.ts` 用 `@nut-tree-fork/nut-js` 注入系统。
6. 断线/离线判定依赖 Redis key 过期订阅（见服务端一节），客户端心跳 5s。

> `src/hooks/webrtc/` 下 `live.ts`、`srs.ts`、`tencentcloudCss.ts`、`meeting*.ts` 是 billd-live 直播域的遗留代码；远程桌面只用 `remoteDesk.ts` 与 `nativeWebRtc*` 消息。改公共 hook 前先确认是否仍有调用方。

配置入口：`src/spec-config.ts`（`prodDomain`，单点）→ `src/constant.ts` 派生 API/WS/TURN 地址（dev 固定 `ws://localhost:4300`、`/api` 走 Vite 代理）；运行时可在设置页用 `LS_KEY.axiosBaseUrl / wssUrl / coturnUrl` 覆盖（`src/utils/localStorage/app.ts`）。

### 服务端 `billd-desk-server/`

- 启动链：`src/index.ts`（先加载 `init/alias` 与 `init/initFile`）→ 连接 MySQL / Redis / Redis Pub-Sub → `src/setup.ts`：全局错误 → `koaBody`（300M 上传）→ 静态目录 → etag → CORS → `apiBeforeVerify` → **自动扫描 `src/router/*.router.ts` 挂载**（各 router 自带 `prefix`，新增文件即生效）→ HTTP listen 与 Socket.io 共用同一端口 → 定时任务 / FFmpeg 初始化。
- 分层：`controller`（Koa ctx 处理）→ `service`（业务）→ `model`（Sequelize，`loadAllModel` 自动加载 + `model/relation` 建立关联）；异常统一 `CustomError`，成功统一 `successHandler`。
  - 强约定：业务代码必须在 `next()` 前置 200 并走 `successHandler`，或直接 throw；否则被 `catchErrorMiddle` 当作 404/405 处理（非 200 请求会被写入 `log` 表，beta 环境不写）。
- 鉴权：`src/app/verify.middleware.ts` 的 `apiBeforeVerify` —— 所有 GET 与白名单接口免 token，其余要求 `Authorization: Bearer <JWT>`；附黑名单 IP 判定；`middleware/role`、`authJwt` 负责角色与权限。
- WebSocket：`src/config/websocket/index.ts` 注册的是**事件名即消息类型**（`WsMsgTypeEnum`，与客户端 `src/types/websocket.ts` 必须一致）；`handleMsg.ts` 承载全部处理逻辑，`live-redis.controller.ts` 维护 Redis 房间/在线态。
  - 远程桌面核心 handler：`handleWsBilldDeskJoin`、`handleWsBilldDeskStartRemote`（双端账号密码校验 + 在线校验，错误码 1~5）、`handleWsBilldDeskUpdateUser`；offer/answer/candidate 仅转发。
  - 在线判定：Redis key `${PROJECT_NAME}-${PROJECT_ENV}-deskUserUuid___<uuid>`（TTL 10s，心跳续期）；`src/config/redis/handleRedisKeyExpired.ts` 订阅 key 过期事件做离线清理。
- ⚠️ `PROJECT_ENV=beta` 时服务端**完全不初始化 Socket.io**（`connectWebSocket` 直接 return），beta 环境下远程桌面信令必然失败。
- 直播遗留域（`live` / `liveRoom` / `liveConfig` / `srs` / `tencentcloudCss` / `qiniu` / `alipay` / `wallet` / `order`）与远程桌面**共用同一套用户、房间与消息体系**，改动房间逻辑会同时影响两端。
- 配置：`src/secret/secret.ts` 被 gitignore；首次启动 `initFile` 会从 `secretTemp.ts` 复制一份占位文件，**必须手动填 mysql/redis/jwt/turn/srs/七牛/腾讯云等值**，否则连库失败后会 `process.exit(1)`。环境变量 `NODE_APP_RELEASE_PROJECT_ENV`（development/beta/prod）决定目录与 secret 分支；端口 development/beta=4300、prod=4200（见 package.json scripts）。
- 部署：Jenkins + `deploy/node-pm2.sh`（pm2）；构建 `tsc -P tsconfig.prod.json || true`（**类型错误不阻断产物**，故需单独跑 `typecheck`）。

## 常用命令

> 工具链：文档建议 Node 18.19 + pnpm 9.1.3；主仓 `.npmrc` 把 `@billd*` 指向私有镜像 `registry.hsslive.cn`、其余走 npmmirror——外网/镜像不可达时依赖装不上。

### 客户端 `billd-desk/`
```bash
pnpm i
npm run dev          # rimraf electron-dist dist && vite（同时起 renderer 与 Electron 主进程）
npm run lint         # eslint（vite-plugin-eslint2 + vue-tsc 在 dev 时也会报）
npm run lint:fix
npm run build:prod   # Web 部署产物 → dist/（build:beta 同脚本；electron-dist 也会一起生成）
npm run build:win    # 桌面包；build:mac / build:linux / build:ele / build:mw / build:deb
npm run gen-icons
```
- `build:win / build:mac / build:linux / build:ele / build:mw / build:deb` 均以 `standard-version` 开头：**会改 package.json 版本并发 tag**，要求当前 git 工作区干净，否则打包失败。
- `deploy:beta` / `deploy:prod` / `copy` 依赖 `deploy/` 目录，而该目录被 `.gitignore` 排除 → 干净 checkout 下不可用。

### 服务端 `billd-desk-server/`
```bash
pnpm i
npm run docker:dev   # 用 docker 拉起 mysql:8.0 / redis:7.0 / srs（rabbitmq 默认关闭）
cp src/secret/secretTemp.ts src/secret/secret.ts   # 然后填值（首次启动也会自动生成占位文件）
npm run mysql:dev    # 仅首次：CREATE DATABASE + 按模型对齐表结构(alter) + 灌默认数据，跑完即退出；库已存在会报错
npm run dev          # 开发模式，端口 4300（dev:beta 同上但无 Socket.io，dev:prod 走 4200 + prod secret）
npm run lint
npm run typecheck    # tsc --noEmit，构建时的 || true 会吞掉类型错误，提交前用它
npm run build        # → dist/（tsc，忽略错误）
npm run start:prod   # nodemon ./dist/index.js（pm2 之外的手动启动方式）
```

### 联调顺序
服务端先起（MySQL/Redis 必需）→ 客户端 `npm run dev`。Vite 已配好 `/api`、`/betaapi`、`/prodapi` 三个前缀分别代理到 4300/4300/4200 并去掉前缀。验证两端连通最省事的路径：同一浏览器开两个窗口（一个作为被控端领取设备码，另一个输入设备码发起连接）观察信令与画面。

### 测试
两个有代码的仓**都没有单元测试框架，也没有 `test` 脚本**；`test/` 目录是手工脚本与 autocannon 压测（如 `billd-desk-server/test/router/*.js`）。可用的"质量门禁"只有：`npm run lint`、服务端 `npm run typecheck`、客户端 dev 时的 `vite-plugin-checker`（vue-tsc + eslint），以及手工双端联调。

## 提交与发布
- 约定式提交（commitizen `npm run cz`；server 侧还有 commitlint + husky + lint-staged 跑 prettier）。
- 版本由 `standard-version` 依据提交生成 CHANGELOG 与 tag（客户端 `v0.x.y`，`release.md` 是更新日志）；**只需在具体子仓内操作**，不要跨仓批量提交。
