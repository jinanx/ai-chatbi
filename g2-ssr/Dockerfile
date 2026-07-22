# syntax=docker/dockerfile:1.7
#
# SQLBot g2-ssr 图表渲染服务镜像（多阶段构建）
#   依赖 node-canvas 的原生库（libcairo2/libpango 等）已内置在 sqlbot-base 中。
#
# 构建：docker build -f g2-ssr/Dockerfile -t sqlbot-g2-ssr:latest .

ARG BASE_IMAGE=registry.cn-qingdao.aliyuncs.com/dataease/sqlbot-base:latest

############################
# Stage 1: builder
############################
FROM ${BASE_IMAGE} AS builder

ENV DEBIAN_FRONTEND=noninteractive

WORKDIR /app

# 依赖单独一层
COPY g2-ssr/package.json g2-ssr/package-lock.json* ./

RUN npm config set fund false \
    && npm config set audit false \
    && npm config set progress false \
    && npm install

COPY g2-ssr/ ./

############################
# Stage 2: runtime
############################
FROM ${BASE_IMAGE} AS runtime

ENV TZ=Asia/Shanghai

RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone

WORKDIR /app

# 拷入 node_modules + 应用源码（canvas 原生运行库在 base 中已具备）
COPY --from=builder /app /app

# 字体（图表中文/特殊字符渲染）
COPY g2-ssr/*.ttf /usr/share/fonts/truetype/liberation/

EXPOSE 3000

# 单容器由 Docker restart 策略托管，直接用 node 运行（无需 PM2）
CMD ["node", "app.js"]
