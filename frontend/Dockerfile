#
# SQLBot 前端镜像
#   前端构建（vue-tsc + vite build）需要 ~4GB 内存，2GB 服务器 OOM。
#   因此本镜像只托管预构建产物，dist/ 必须先在本地（Windows）通过
#   `cd frontend && npm install && npm run build` 生成，再 scp/rsync 到服务器。

FROM nginx:1.27-alpine AS runtime

# 静态产物（须先在本地 build 出来）
COPY frontend/dist/ /usr/share/nginx/html/

# nginx 配置以 template 形式放入：
# 官方 nginx 镜像启动时会用 envsubst 渲染 /etc/nginx/templates/*.template
# 到 /etc/nginx/conf.d/，且只替换环境变量（CONTEXT_PATH），
# 保留 $host / $uri / $remote_addr 等 nginx 内置变量。
COPY frontend/nginx.conf /etc/nginx/templates/default.conf.template

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD wget -qO- http://localhost/ >/dev/null 2>&1 || exit 1