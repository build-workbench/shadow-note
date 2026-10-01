# 两阶段构建：sqlite3 等原生模块若无预编译二进制需要现场编译，
# 编译工具链只留在 builder 阶段，runtime 镜像保持精简。
# 两个阶段使用同一 node:20 基础镜像，保证 ABI 一致。
FROM node:20-slim AS deps

RUN apt-get update \
    && apt-get install -y --no-install-recommends python3 make g++ \
    && rm -rf /var/lib/apt/lists/*

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
