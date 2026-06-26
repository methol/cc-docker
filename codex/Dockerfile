# 预装 OpenAI Codex CLI 的 CI 镜像,支持 codex-security 安全扫描
# ⚠️ 构建时需要能访问 npm registry
#
# codex-security 插件是 OpenAI 企业级商业插件,须通过 Codex Web App 授权安装。
# 运行时通过 -v 挂载或环境变量注入已激活插件的 CODEX_HOME:
#
#   docker run --rm \
#     -v "$HOME/.codex:/root/.codex:ro" \
#     -v "$(pwd):/workspace" \
#     -e OPENAI_API_KEY \
#     methol/codex-docker \
#     codex exec --sandbox workspace-write \
#       --output-last-message /tmp/security-review.md \
#       'Use $codex-security:security-diff-scan to review changes from origin/main to HEAD.'
# syntax=docker/dockerfile:1.7
FROM node:24-slim

# 0) 版本号入参:CI 传入具体 Codex 版本,本地构建默认 latest
ARG CODEX_VERSION=latest
LABEL org.opencontainers.image.title="codex-docker" \
      org.opencontainers.image.description="预装 OpenAI Codex CLI + 开发工具的 CI 镜像,支持 codex-security 安全扫描" \
      org.opencontainers.image.source="https://github.com/methol/agent-docker" \
      org.opencontainers.image.version="${CODEX_VERSION}"

# 1) 基础工具:git(保留完整 history 供 diff-scan 使用)+ 常用开发工具
#    与 cc-docker 保持一致,方便在同一 pipeline 中混用
RUN apt-get update && apt-get install -y --no-install-recommends \
        git git-lfs \
        curl wget ca-certificates bash \
        openssh-client \
        jq \
        ripgrep \
        fd-find \
        tree \
        make \
        python3 python3-pip \
        unzip \
        less \
        procps \
    && ln -s /usr/bin/fdfind /usr/local/bin/fd \
    && rm -rf /var/lib/apt/lists/*

# helper:安装可选代理 CA 证书到系统 trust store
# 仅在本地代理环境构建时注入,生产 CI 无需提供(required=false)
# 安装后 git/curl/npm 均自动信任该 CA,无需额外 env var
RUN --mount=type=secret,id=proxy_ca,target=/tmp/proxy-ca.crt,required=false \
    if [ -f /tmp/proxy-ca.crt ]; then \
        cp /tmp/proxy-ca.crt /usr/local/share/ca-certificates/proxy-ca.crt && \
        update-ca-certificates; \
    fi

# 2) 安装 Codex CLI(钉死到 CI 传入的版本,保证镜像内版本 = 镜像 tag)
RUN npm install -g @openai/codex@${CODEX_VERSION}

# 3) 固定 HOME,保证构建期与运行期路径一致
ENV HOME=/root

# 4) CODEX_HOME:Codex 配置/插件目录骨架
#    codex-security 是商业插件,运行时通过 volume 挂载已激活的 CODEX_HOME
ENV CODEX_HOME=/root/.codex
RUN mkdir -p /root/.codex

# 5) git SSH → HTTPS,免密 clone 公开仓库(build 环境无 SSH key)
RUN git config --global url."https://github.com/".insteadOf "git@github.com:"

# 6) 安装 codex-security 插件
#    openai/plugins 是官方 openai-curated marketplace,但名称被 CLI 内置保留,
#    需 sparse clone 后将 marketplace.json 中的名称改为 openai-security 才能注册。
#    marketplace 注册路径写死在配置里,需保留到镜像中(不能 rm -rf)。
RUN git clone --depth 1 --filter=blob:none --sparse \
        https://github.com/openai/plugins.git /root/.codex/openai-plugins \
    && cd /root/.codex/openai-plugins \
    && git sparse-checkout set '.agents/plugins' 'plugins/codex-security' \
    && sed -i 's/"openai-curated"/"openai-security"/g' .agents/plugins/marketplace.json \
    && codex plugin marketplace add /root/.codex/openai-plugins \
    && codex plugin add codex-security@openai-security

# 7) 构建期自检:CLI 可调用 + codex-security 插件已安装启用
RUN codex --version && \
    codex plugin list | grep -q "codex-security" && \
    echo "✅ codex CLI + codex-security plugin ready"

# TMPDIR:security-diff-scan 的 scan bundle 写入位置
ENV TMPDIR=/tmp

WORKDIR /workspace
