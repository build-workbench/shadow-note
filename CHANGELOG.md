# 更新日志

本文件记录了项目的所有重要变更。

格式基于 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，
版本号遵循 [语义化版本](https://semver.org/lang/zh-CN/)。

## [Unreleased]

_(暂无)_

## [v2.3.0] - 2026-09-28
### Added
- README 新增主界面产品截图（`docs/screenshots/main.png`）

### Fixed
- 修复跨房间数据泄漏：切换同步链后旧链的离线操作会以新链密钥加密推送到新链；
  现在按 roomId 校验并丢弃，防止旧链内容进入新链房间
- 修复推送误报已同步：`emitEncryptedUpdate` 此前 fire-and-forget，
  服务端拒绝（限流/超限/未加入）时客户端仍认为已送达并删除离线队列
- 修复 `join-chain` 首次调用不受速率限制（meta 未初始化时跳过计数），
  换连接可免费加入一次
- 修复 SQLite 连接未等待打开回调：PRAGMA 可能在连接就绪前执行，
  打开失败时异常无法被捕获
- 修复 Redis 只读活跃房间被误删：`cleanupExpired` 依据"最后推送时间"判断，
  改为访问续命（读取时刷新 timestamp，与 SQLite 语义一致）
- 修复 Redis `reconnectStrategy` 返回 Error 对象导致重连行为不可预期，
  改为返回 false 停止重连并交由健康检查故障转移
- 修复 `unhandledRejection` 仅记录不退出，进程可能带病运行
- 修复优雅关闭未断开 Socket.IO 长连接，WebSocket 使 `server.close` 挂起
- 修复服务端接收任意垃圾密文并广播：`push-update` 现在校验 base64 格式、
  最小长度与 v2 信封字段（deviceId/seq），非法的直接拒绝
- 修复 CI `npm ci` 失败：`apps/web/package-lock.json` 缺少 esbuild 0.28.2 条目
  （vitest 嵌套的 vite 8 将其声明为 optional peer），重新生成 lock 补齐
- 修复 `crypto.test.js` 校验和测试偶发失败：BIP39 校验和仅 4 位，替换单个词有
  1/16 概率校验仍通过；改为遍历字表构造确定性无效助记词
- 修复浏览器中创建同步链失败：bip39 依赖 Node `Buffer`，入口现在注入 polyfill
- 修复侧边栏助记词断词：`break-all` 改为 `break-words`，单词不再被硬截断
- 修复离线队列失败重试逻辑失效：`processQueue` 的 processor 现在正确解析 `{ success: boolean }`
  返回值，此前对象恒为 truthy 导致发送失败的离线编辑被静默删除
- 修复内容哈希只覆盖前 1000 字符导致的静默数据丢失：尾部编辑被误判为"未变更"后遭远程覆盖；
  `hashContent` 改为覆盖全部内容
- 修复 `joinChain` 后 `lastSyncedHashRef` 哨兵值依赖旧弱哈希巧合的问题
- 修复冲突检测中失效的版本比较分支：服务端 version（毫秒时间戳）与本地（小计数器）量纲不同，
  该分支此前恒真/恒假方向错误；现改为"内容不同 + 时间窗外 = offline_divergence"，双向一致
- 修复历史记录弱去重与同毫秒 id 撞号：去重改为全文比较，id 使用加密安全随机串
- 修复服务端 `request-sync` 无成员校验、无限流：任意连接可拉取任意房间密文；
  现与 `push-update` 一致要求房间成员身份
- 修复 Redis 客户端使用 v3 API 连接 v4 依赖：顶层 `host/port/db/retry_strategy`
  被 v4 静默忽略（配置 `REDIS_HOST` 实际无效）；改用 v4 的 `socket:{}` / `database` /
  `reconnectStrategy`
- 修复 SQLite 持久层无过期清理导致的磁盘无限增长：周期任务现在调用 `cleanupExpired`
- 修复 `LOG_LEVEL` 环境变量无效：logger 现在读取该配置；服务入口的裸 `console.*` 统一替换为结构化 logger

### Changed
- 同步协议引入**发送确认机制**：`push-update` 带 ack 回调，服务端确认后才更新
  本地同步水位/出队，未确认（超时、限流、被拒）的操作保留重试，不再误报已同步
- `join-chain` 成功后新增 `join-ack` 事件：客户端收到确认后才冲刷离线队列，
  避免推送因尚未成为房间成员而被拒绝
- 服务端错误统一为结构化 `{ type, message, code, recoverable }`；
  `/health` 持久化错误脱敏，不再向任意访问者暴露底层错误细节
- 客户端仅接受 v2 加密信封（legacy 无 AAD 分支移除），防止恶意服务器
  省略 `v` 字段降级到无 AAD 路径
- 内容哈希升级为 64 位双哈希（DJB2 + FNV-1a），碰撞概率降至 2^-64
- 离线队列增加指数退避（1s/2s/4s…），失败操作不再毫秒级连试后被丢弃
- 跨房间防护：离线操作与当前同步链不匹配时丢弃并提示，
  防止切链后旧链内容被推送到新链
- 加密后预校验 5MB 上限，超限明确提示而不是误报断线
- 项目更名为 **ShadowNote / 影子笔记**（原 brave-sync-notes / Secure Note Chain）：
  产品显示名、网页标题、包名（shadow-note / shadow-note-client / shadow-note-server）
  与文档同步更新，体现"零知识 + 实时 + 轻量自托管"定位
- CI 升级至 Node 24，`actions/checkout`/`actions/setup-node` 升级 v4 → v5，
  消除 Node 20 运行时弃用警告（此前 actions 被强制跑在 Node 24 上）
- 前端 vite 5 → 6、react-syntax-highlighter 15 → 16（PrismJS 修复版）
- 后端 sqlite3 5 → 6（N-API 预编译二进制，安装不再依赖 node-gyp 编译链）
- PBKDF2 迭代次数默认值提升至 310,000（OWASP 建议），并强制下限 100,000——
  此前可经 `VITE_PBKDF2_ITERATIONS` 配置为任意低值甚至 1
- 密钥缓存改以 roomId（公开哈希）为键，避免明文助记词长期驻留内存；
  新增 `clearKeyCache()` 并在断开连接时调用
- 服务端 `/stats` 脱敏：不再暴露进程内存、Redis host/port/db 等基础设施细节
- 移除无效的 `express.json({ limit: '50mb' })`（服务无 JSON body 端点）；
  Socket.IO `maxHttpBufferSize` 与密文上限（5MB）对齐
- socket 重连不再重复发送 `join-chain`（socket.io 重连必先触发 connect）
- socket 事件文案惰性解析当前语言，切换语言后提示不再使用旧快照
- store.isOnline 由 socket 连接事件实时驱动，修复挂载后永不更新的死状态
- `joinChain` 增加防重入保护，消除 StrictMode/依赖抖动下的并发连接竞态
- 移除未使用的 `useOffline` hook 及其测试（与 useSocket 内离线逻辑完全重复）

### Docs
- README 项目名修正为 Secure Note Chain 并精简（移除仓库结构、合并测试命令）；
  AGENTS.md 项目名同步修正
- ARCHITECTURE/SECURITY 补充已知安全局限披露：助记词明文本地持久化、无重放防护、
  无前向保密、salt 由密码派生、服务器可见元数据范围
- Landing 页"无追踪、无数据收集"文案修正为如实描述

### Security
- 依赖漏洞清零（两端 `npm audit` 0 漏洞，Dependabot 3 项告警全部消除）：
  - 后端：socket.io 升级修复 engine.io WebTransport SID DoS（GHSA-gr94-w7qr-f4j3）；
    js-yaml 升级修复 omap 解析二次复杂度（GHSA-5p4m-2wfm-xmqj）；
    sqlite3 5 → 6 移除 node-gyp 安装链（tar 任意文件写 critical、http-proxy-agent 等 7 项）
  - 前端：postcss 升级修复 sourceMappingURL 任意 .map 读取（GHSA-fxqj-rqcc-2cmp）；
    vite 5 → 6 携 esbuild 0.25 修复 dev server 任意请求（GHSA-67mh-4wv8-2f99）；
    react-syntax-highlighter 15 → 16 携 PrismJS 1.30 修复 DOM Clobbering（GHSA-x7hr-w5r2-h6wg）
  - 两端：socket.io-parser 内存耗尽（GHSA-2m8v-j782-fhvr）、ws 未初始化内存泄露/
    分片 DoS、body-parser、path-to-regexp、browserslist、brace-expansion 等传递依赖同步升级

### Removed
- 移除 VitePress 文档站（37 文件）及全部构建配置，核心架构信息合并至 `ARCHITECTURE.md`
- 移除 Docker 部署基础设施（`docker-compose.yml`、两个 `Dockerfile`、`nginx.conf`）
- 移除 GitHub 社区文件（issue 模板、PR 模板、CODEOWNERS、copilot-instructions、WORKFLOWS.md）
- 移除 4 个 CI workflow（docs、release、codeql、dependency-review），CI 简化为单 job
- 移除 `CONTRIBUTING.md`（归档项目不再接受贡献）
- 移除 `netlify.toml`（路径错误且归档不需要）
- 移除 `start-local.sh`（功能等同于 `npm start`）
- 移除前端 9 个未使用/冗余依赖：`yjs`、`y-indexeddb`、`idb-keyval`、`events`、`process`、`rehype-highlight`、`crypto-js`、`buffer`、`vite-plugin-node-polyfills`
- 移除 `vite-plugin-node-polyfills` 及其 vite 配置

### Changed
- 加密模块改用浏览器原生 Web Crypto API（PBKDF2 + AES-256-GCM），不再依赖 crypto-js
- `crypto.js` 内嵌同步 SHA-256 实现，使 `createNotebook` 保持同步（Zustand store action 兼容）
- `deriveKeys` / `encryptData` / `decryptData` 改为 async（Web Crypto API 异步特性）
- 移除 notebook 对象中的 `encryptionKey` 字段（冗余，实际加密密钥仅在内存中派生）
- 合并重复配置：根目录统一 `.prettierrc` 和 `.gitignore`，删除子项目副本
- eslint 降级至 ^8.57.1（修复与 eslint-plugin-react 的 peer dependency 冲突）
- 简化 `SECURITY.md` 为归档项目最小说明
- 简化 `.vscode/settings.json` 移除 copilot 配置
- 精简顶层 `package.json`：移除 docs/docker/install 相关脚本

### Changed
- 助记词校验启用 BIP39 校验和验证：拼写错误（恰好小写）的助记词在入口即被拒绝，不再进入错误的同步房间

### Fixed
- 修复大笔记（>50KB）分块传输导致新加入设备无法恢复内容的问题：取消客户端分块，改为单条密文直推（加密后 5MB 上限内），服务端移除未实现的分块重组死代码
- 修复 react-dom 版本不匹配（`^19.2.5` -> `^18.2.0`，与 react ^18.2.0 对齐）
- 修复 `@types/react-dom` 版本不匹配（`^19.2.3` -> `^18.2.0`）

---

## [2.2.0] - 2026-03-22

### Added
- **文档站**：完整导航闭环和深度页面
  - 添加 `changelog/index.md` 作为可浏览归档页
  - 添加深度文档页面：`architecture.md`、`deployment.md`、`security-sync.md`
  - 为所有文档文件添加 front matter 和固定链接

- **CI 流水线**：增强工作流门禁
  - 拆分 CI 为 `client` 和 `server` 两个 job
  - 集成客户端和服务端测试执行

- **测试**：同步链路测试覆盖
  - 客户端：`useSocket.test.js` - 连接、sync-update、request-sync、错误处理
  - 服务端：`index.test.js` - 健康检查、join-chain、push-update、request-sync

### Fixed
- 服务端定时器调用 `unref()` 防止测试进程悬挂
- `gracefulShutdown` 和 `startServer` 在测试环境不再调用 `process.exit()`
- 提取 `handleSocketConnection` 便于直接测试 Socket 事件处理

---

## [2.1.0] - 2026-03-13

### Changed
- **文档架构**：重构入口职责
  - `README.md` / `README.zh-CN.md`：仅作为仓库入口（定位、快速启动、链接）
  - `index.md`：文档门户，包含阅读路径指引
  - `CONTRIBUTING.md`：集成到文档站点

- **配置修复**：
  - 修复 `_config.yml` 仓库名从 `LessUp/sync-notes` 到 `LessUp/brave-sync-notes`
  - 修复 `baseurl` 从 `/sync-notes` 到 `/brave-sync-notes`
  - 添加 `CONTRIBUTING.md` 到 Pages 工作流路径

---

## [2.0.2] - 2026-03-10

### Added
- **工作流标准化**：
  - 统一 CI 权限：`contents: read`
  - 添加并发配置防止重复运行
  - 添加 `actions/configure-pages@v5` 步骤
  - 添加 `paths` 触发过滤减少无效构建

### Changed
- **GitHub Pages 优化**：
  - 添加 SEO 元数据到 `_config.yml`
  - 添加 `exclude` 列表跳过非文档文件的 Jekyll 构建
  - 优化 `paths` 触发器为特定文件
  - 修复 `sparse-checkout` 配置
  - 添加部署状态徽章到 `index.md`

---

## [2.0.1] - 2026-03-09 [SECURITY]

### Security
- **[严重] 服务端输入验证**：
  - 添加 `encryptedData` 验证：必须为字符串，最大 5MB
  - 添加每 socket 速率限制：每分钟最多 30 次更新
  - 添加 `timestamp` 类型验证

- **[破坏性] PBKDF2 盐值改进**：
  - 从硬编码盐值改为助记词派生盐值：`SHA256("notesync-salt:" + mnemonic)`
  - PBKDF2 迭代次数从 1,000 增加到 10,000
  - ⚠️ **破坏性变更**：现有同步链需要重新创建

### Fixed
- **内存泄漏**：房间清理增强
  - 添加 `MAX_MEMORY_ROOMS` 硬上限（默认 10,000）
  - 两阶段清理：先清理过期房间，超容量时清理最旧房间
  - 清理间隔从 60 分钟减少到 30 分钟

- **Socket 事件监听器泄漏**：修复快速调用 `joinChain` 时的竞态条件
  - 在移除监听器前设置 `socketRef.current = null`
  - 取消前一个会话的待处理防抖推送
  - 清除待处理的分块重组状态

---

## [2.0.0] - 2025-11-25

### Added
- **编辑器**：CodeMirror 集成
  - 语法高亮：Markdown、JavaScript、Python、HTML、CSS、JSON
  - 自动补全、括号匹配、代码折叠
  - 可自定义字体大小、Tab 大小、行号、自动换行
  - 深色/浅色主题自动切换

- **预览**：增强 Markdown 渲染
  - Prism 代码块高亮
  - GFM 支持：表格、任务列表、删除线
  - 分屏视图：编辑/预览/并排

- **历史记录**：版本管理
  - 每次同步自动保存
  - 一键恢复任意版本
  - 单条删除或清空全部
  - LocalStorage 持久化

- **移动端**：二维码加入
  - 自动生成同步链二维码
  - 手机扫码即时加入
  - URL 参数自动填充

- **导入导出**：文件操作
  - 导入 .txt、.md、.markdown 文件
  - 导出为 Markdown 或纯文本

### Changed
- **性能**：
  - 大于 50KB 内容分块传输
  - 可配置防抖同步（默认 300ms）
  - Zustand 状态管理
  - 组件懒加载
  - useMemo 记忆化

- **UI/UX**：
  - Framer Motion 动画的毛玻璃设计
  - 完全响应式，移动端优先布局
  - 系统深色模式偏好支持
  - 双语界面（中/英）

- **服务端**：
  - 连接池：WebSocket + Polling 双通道
  - 自动清理过期房间
  - 健康检查：`GET /health`、`GET /stats`
  - 优雅关闭：SIGTERM/SIGINT
  - 配置：maxHttpBufferSize 10MB、pingTimeout 60s

---

## [1.2.0] - 2025-12-19

### Added
- **冲突管理**：客户端冲突检测与解决
  - 添加 `noteVersion`、`noteTimestamp`、`noteDeviceId` 到 store 状态
  - 在 `sync-update` 流程中集成 `ConflictManager`
  - `ConflictIndicator` 和 `ConflictDialog` UI 组件
  - 手动解决：保留本地/远程或自定义合并

### Changed
- `setNote(note, meta?)` 支持远程元数据
- `restoreFromHistory` 维护版本元数据
- 从 hook 暴露 `conflictCount`、`pendingConflicts`、`resolveConflict`、`clearConflicts`

---

## [1.1.0] - 2025-12-18

### Fixed
- **[严重] 异步处理器**：修复 `join-chain` 回调未声明为 `async`
- **配置**：
  - 服务端启动时添加 `require('dotenv').config()`
  - 使用 `DataValidator.isValidRoomId()` 统一房间 ID 验证
  - CORS Origin 可通过 `CORS_ORIGIN` 环境变量配置
  - 房间 TTL 可通过 `ROOM_TTL_MS` 环境变量配置

---

## [1.0.1] - 2025-11-24

### Added
- **部署**：生产环境准备
  - 添加客户端 `.gitignore`（node_modules、构建产物、环境文件）
  - 规划使用 Netlify CLI 部署 Vite + React 前端

---

## [1.0.0] - 2025-02-13

### Added
- **项目基础设施**：
  - 添加 `.editorconfig` 统一代码格式
  - 添加标准徽章到 README（License、React、Express、Socket.IO、Vite）

---

[Unreleased]: https://github.com/build-workbench/shadow-note/compare/v2.2.0...HEAD
[2.2.0]: https://github.com/build-workbench/shadow-note/compare/v2.1.0...v2.2.0
[2.1.0]: https://github.com/build-workbench/shadow-note/compare/v2.0.2...v2.1.0
[2.0.2]: https://github.com/build-workbench/shadow-note/compare/v2.0.1...v2.0.2
[2.0.1]: https://github.com/build-workbench/shadow-note/compare/v2.0.0...v2.0.1
[2.0.0]: https://github.com/build-workbench/shadow-note/compare/v1.2.0...v2.0.0
[1.2.0]: https://github.com/build-workbench/shadow-note/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/build-workbench/shadow-note/compare/v1.0.1...v1.1.0
[1.0.1]: https://github.com/build-workbench/shadow-note/compare/v1.0.0...v1.0.1
[1.0.0]: https://github.com/build-workbench/shadow-note/releases/tag/v1.0.0
