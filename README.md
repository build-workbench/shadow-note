**English** | [中文](#chinese)

<a id="top"></a>

# ShadowNote / Shadow Notes

A zero-knowledge real-time note synchronization system — end-to-end encrypted, the server only relays ciphertext and cannot see the content; lightweight self-hosting, no account required.

![Browser window frame](./docs/screenshots/browser.png)

## Features

- **End-to-end encryption**: AES-256-GCM, the server only relays ciphertext
- **12-word mnemonic recovery**: BIP39, sync keys across devices
- **Real-time sync**: WebSocket + Socket.IO, offline-first, multi-device collaboration
- **Conflict resolution**: three-way merge with a manual resolution UI

## Quick Start (Self-Hosting)

One command brings up the full stack — frontend, sync API, and Redis with AOF persistence:

```bash
docker compose up -d --build
# open http://localhost:8080
```

The built-in nginx serves the SPA and reverse-proxies `/socket.io` to the API on the same origin, so there is no CORS setup and no second port to expose. Put your TLS reverse proxy in front for production (one-line Caddy config, or an nginx snippet — see the [self-hosting guide](./docs/self-hosting.md)).

```text
browser ──HTTPS──> your reverse proxy ──> web (nginx: SPA + /socket.io proxy)
                                              └──> api (Express + Socket.IO) ──> redis (AOF) / sqlite
```

Key knobs live in `.env` (see [.env.example](./.env.example)): public origin for CORS, published port, log level.

> Remember: the 12-word mnemonic shown when you create a note chain is the only key to the notes. It never leaves your device — the server cannot recover it, and neither can we.

## Development

```bash
# 后端（http://localhost:3002）
cd apps/api && npm ci && npm start

# 前端（http://localhost:5173）
cd apps/web && npm ci && npm run dev
```

```bash
npm test    # 全部测试
npm run lint
npm run build
```

See [docs/self-hosting.md](./docs/self-hosting.md) for deployment, [ARCHITECTURE.md](./ARCHITECTURE.md) for the architecture, and [CHANGELOG.md](./CHANGELOG.md) for the changelog.

## License

[MIT](./LICENSE)

---

<a id="chinese"></a>
[English](#top) | **中文**

# ShadowNote / 影子笔记

零知识的实时笔记同步系统——端到端加密，服务器只转发密文、看不见内容；轻量自托管，无需账号。

![浏览器窗口框](./docs/screenshots/browser.png)

## 特性

- **端到端加密**：AES-256-GCM，服务器只转发密文
- **12 词助记词恢复**：BIP39，跨设备同步密钥
- **实时同步**：WebSocket + Socket.IO，离线优先，多设备协作
- **冲突解决**：三路合并，手动解决 UI

## 快速开始（自部署）

一条命令拉起完整技术栈——前端、同步 API、Redis（AOF 持久化）：

```bash
docker compose up -d --build
# 打开 http://localhost:8080
```

内置 nginx 托管前端静态文件，并把 `/socket.io` 同源反向代理到 API：无需配置 CORS，也无需暴露第二个端口。生产环境把你的 TLS 反向代理放在前面即可（Caddy 一行配置、nginx 若干行，见[自部署指南](./docs/self-hosting.md)）。

```text
浏览器 ──HTTPS──> 你的反向代理 ──> web（nginx：SPA + /socket.io 反代）
                                        └──> api（Express + Socket.IO）──> redis（AOF）/ sqlite
```

主要配置在 `.env`（见 [.env.example](./.env.example)）：CORS 用的站点来源、对外端口、日志级别。

> 提醒：创建笔记链时显示的 12 词助记词是笔记的唯一钥匙，它只存在于你的设备上——服务器无法找回，我们也一样。

## 本地开发

```bash
# 后端（http://localhost:3002）
cd apps/api && npm ci && npm start

# 前端（http://localhost:5173）
cd apps/web && npm ci && npm run dev
```

```bash
npm test    # 全部测试
npm run lint
npm run build
```

部署详见 [docs/self-hosting.md](./docs/self-hosting.md)，架构见 [ARCHITECTURE.md](./ARCHITECTURE.md)，变更记录见 [CHANGELOG.md](./CHANGELOG.md)。

## 许可

[MIT](./LICENSE)
