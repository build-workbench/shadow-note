# 两阶段构建：
# - builder 使用完整版 node:20：自带 python3/gcc/make，可现场编译 sqlite3
#   等原生模块，无需 apt-get，受限网络环境下只要基础镜像可拉取即可构建；
# - runtime 使用 node:20-slim：只复制编译产物。两阶段同为 Debian bookworm
#   + Node 20 ABI，原生模块直接兼容。
FROM node:20 AS deps

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

# ---- runtime ----
FROM node:20-slim

ENV NODE_ENV=production

WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY index.js ./
COPY src ./src

# 运行时以 node 用户身份写 SQLite 数据文件
RUN mkdir -p /data && chown node:node /data
ENV SQLITE_DB_PATH=/data/notesync.db
VOLUME /data

USER node

EXPOSE 3002

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:'+(process.env.PORT||3002)+'/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"

CMD ["node", "index.js"]
