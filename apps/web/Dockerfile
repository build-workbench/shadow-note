# ---- build ----
FROM node:20-slim AS build

WORKDIR /app

# Vite 在构建期注入环境变量。
# VITE_SOCKET_URL 默认为 "/"：socket.io-client 会解析为当前页面源，
# 由 nginx 容器把 /socket.io 反向代理到 api，实现同源零 CORS 部署。
# 独立域名部署时通过 --build-arg 覆盖，如 https://sync.example.com
ARG VITE_SOCKET_URL=/
ARG VITE_PBKDF2_ITERATIONS=10000
ENV VITE_SOCKET_URL=$VITE_SOCKET_URL \
    VITE_PBKDF2_ITERATIONS=$VITE_PBKDF2_ITERATIONS

COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build

# ---- serve ----
FROM nginx:1.27-alpine

COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
