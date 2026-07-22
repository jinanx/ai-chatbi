# syntax=docker/dockerfile:1.7
#
# SQLBot 后端镜像（多阶段构建）
# 复用内置了 Python 3.11 + uv + Oracle/达梦 客户端的自定义 base 镜像。
#
# 构建：docker build -f backend/Dockerfile -t sqlbot-backend:latest .
# （上下文为仓库根目录，以便 COPY backend/ 整个源码目录）

ARG BASE_IMAGE=registry.cn-qingdao.aliyuncs.com/dataease/sqlbot-base:latest

############################
# Stage 1: builder
############################
FROM ${BASE_IMAGE} AS builder

ENV PYTHONUNBUFFERED=1 \
    UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy \
    SQLBOT_HOME=/opt/sqlbot \
    APP_HOME=/opt/sqlbot/app \
    PYTHONPATH=/opt/sqlbot/app \
    PATH="/opt/sqlbot/app/.venv/bin:${PATH}" \
    DEBIAN_FRONTEND=noninteractive

WORKDIR ${APP_HOME}

# 1) 先只装依赖，命中 Docker 层缓存（pyproject/uv.lock 不变时跳过重装）
#    bind mount 不污染最终镜像；uv cache 跨构建复用
RUN --mount=type=bind,source=backend/uv.lock,target=uv.lock \
    --mount=type=bind,source=backend/pyproject.toml,target=pyproject.toml \
    --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-install-project || uv sync --no-install-project

# 2) 拷入全部源码并完成最终安装（含 cpu 版 torch）
COPY backend/ ${APP_HOME}/

RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --extra cpu

############################
# Stage 2: runtime
############################
FROM ${BASE_IMAGE} AS runtime

# 继承 base 镜像的 PATH，在头部追加 venv
ENV PYTHONUNBUFFERED=1 \
    SQLBOT_HOME=/opt/sqlbot \
    APP_HOME=/opt/sqlbot/app \
    PYTHONPATH=/opt/sqlbot/app \
    PATH="/opt/sqlbot/app/.venv/bin:${PATH}" \
    TZ=Asia/Shanghai

RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone

WORKDIR ${APP_HOME}

# 仅拷入构建产物（含 .venv + 源码），其余构建工具链留在 builder 层
COPY --from=builder /opt/sqlbot /opt/sqlbot

# 预建运行期目录（运行时由 volume 覆盖，避免首次启动因目录缺失报错）
RUN mkdir -p /opt/sqlbot/images \
             /opt/sqlbot/data/excel \
             /opt/sqlbot/data/file \
             /opt/sqlbot/app/logs

EXPOSE 8000

# lifespan 会先跑 alembic 迁移 + 初始化 embedding，start_period 留足时间
HEALTHCHECK --interval=30s --timeout=10s --start-period=120s --retries=5 \
    CMD python -c "import os,urllib.request;urllib.request.urlopen(f'http://localhost:8000/{os.environ.get(\"CONTEXT_PATH\",\"\").strip(\"/\")}',timeout=3)" || exit 1

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "1", "--proxy-headers", "--forwarded-allow-ips=*"]
