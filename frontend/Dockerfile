# syntax=docker/dockerfile:1.7
#
# SQLBot 前端镜像（多阶段构建）
#   Stage 1: 用 sqlbot-base（自带 Node 18）构建 Vue 静态产物
#   Stage 2: nginx:alpine 托管静态文件并反向代理后端 API
#
# 构建：docker build -f frontend/Dockerfile -t sqlbot-frontend:latest .

ARG BASE_IMAGE=registry.cn-qingdao.aliyuncs.com/dataease/sqlbot-base:latest

############################
# Stage 1: builder
############################
FROM ${BASE_IMAGE} AS builder

ENV UI_HOME=/opt/sqlbot/frontend \
    DEBIAN_FRONTEND=noninteractive

WORKDIR /build

# 依赖单独一层，加速重复构建
COPY frontend/package.json frontend/package-lock.json* ./

RUN npm config set fund false \
    && npm config set audit false \
    && npm config set progress false \
    && npm install

COPY frontend/ ./

# 与 package.json 的 build 脚本对齐：vue-tsc + vite build（4G 堆内存）
RUN NODE_OPTIONS=--max_old_space_size=4096 npm run build

############################
# Stage 2: runtime (nginx)
############################
FROM nginx:1.27-alpine AS runtime

# 静态产物
COPY --from=builder /build/dist /usr/share/nginx/html

# nginx 配置以 template 形式放入：
# 官方 nginx 镜像启动时会用 envsubst 渲染 /etc/nginx/templates/*.template
# 到 /etc/nginx/conf.d/，且只替换环境变量（CONTEXT_PATH），
# 保留 $host / $uri / $remote_addr 等 nginx 内置变量。
COPY frontend/nginx.conf /etc/nginx/templates/default.conf.template

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD wget -qO- http://localhost/ >/dev/null 2>&1 || exit 1
