# ============================================================
# SoulTuner-Agent Next.js 前端 Dockerfile
# 多阶段构建：builder 编译 → runner 仅运行产物
# ============================================================

# ---- Stage 1: 构建 ----
FROM node:22-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

# 构建时需要的环境变量。
# NEXT_PUBLIC_API_URL 是浏览器侧用的，会被冻进客户端 JS bundle —— 只有在
# "docker compose up 跑在你自己电脑上，backend 端口发布到宿主机 localhost"
# 这一种拓扑下，浏览器直连 http://localhost:8501 才成立。换一个真实部署环境
# （例如把整个 compose 栈搬进同一个 K8s Pod，浏览器在别的机器上打开），
# 浏览器上的 localhost 永远指向"打开网页的那台电脑"，不是这个容器——
# 直连必然失败。留空则客户端代码里的 `${API_URL}/api/...` 退化成
# `/api/...`，同源相对路径，天然走下面这条已经在用的 rewrites 代理，
# 不用关心后端到底部署在哪。
#
# BACKEND_INTERNAL_URL 是服务端 rewrite（next.config.js 的 rewrites()）用的：
# 这段代码始终跑在这个容器里，从不在浏览器里执行，所以走 localhost 才是对的——
# 无论是本地 docker compose（backend 端口只在容器网络内可达）还是这个平台
# 把整个 compose 栈放进同一个 Pod（同一 Pod 内所有容器共享一个网络命名空间，
# 只有 localhost 能互通，compose 服务名 "backend" 这个主机名不会被解析）。
# **必须在这里设**——next.config 的 rewrites 会被 next build 冻进
# .next/routes-manifest.json，运行时再给环境变量已经晚了。
ENV NEXT_PUBLIC_API_URL=
ENV BACKEND_INTERNAL_URL=http://localhost:8501
RUN npm run build

# ---- Stage 2: 运行 ----
FROM node:22-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production

# 仅复制必要文件
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
COPY --from=builder /app/public ./public
COPY --from=builder /app/next.config.js ./

EXPOSE 3003

CMD ["npx", "next", "start", "-p", "3003"]
