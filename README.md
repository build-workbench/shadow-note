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

## Quick Start

```bash
# 后端（http://localhost:3002）
cd apps/api && npm ci && npm start

# 前端（http://localhost:5173）
cd apps/web && npm ci && npm run dev
```

## Development

```bash
npm test    # 全部测试
npm run lint
npm run build
```

See [ARCHITECTURE.md](./ARCHITECTURE.md) for the architecture, and [CHANGELOG.md](./CHANGELOG.md) for the changelog.

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

## 快速开始

```bash
# 后端（http://localhost:3002）
cd apps/api && npm ci && npm start

# 前端（http://localhost:5173）
cd apps/web && npm ci && npm run dev
```

## 开发

```bash
npm test    # 全部测试
npm run lint
npm run build
```

架构见 [ARCHITECTURE.md](./ARCHITECTURE.md)，变更记录见 [CHANGELOG.md](./CHANGELOG.md)。

## 许可

[MIT](./LICENSE)
