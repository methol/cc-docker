# 预装 OpenAI Codex + codex-security plugin 的 CI 镜像
# 用途:在 CI runner 中执行 security-diff-scan,无需每次安装依赖
# ⚠️ 构建时需要能访问 npm registry 和 api.openai.com
FROM node:24-slim

# 0) 版本号入参:CI 传入具体 Codex 版本,本地构建默认 latest
ARG CODEX_VERSION=latest
LABEL org.opencontainers.image.title="codex-docker" \
      org.opencontainers.image.description="预装 OpenAI Codex + codex-security plugin 的 CI 镜像" \
      org.opencontainers.image.source="https://github.com/methol/agent-docker" \
      org.opencontainers.image.version="${CODEX_VERSION}"

# 1) 基础工具:git(含完整 history 供 diff-scan 使用)+ 常用开发工具
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

# 2) 安装 Codex CLI(钉死到 CI 传入的版本,保证镜像内版本 = 镜像 tag)
RUN npm install -g @openai/codex@${CODEX_VERSION}

# 3) 固定 HOME,保证构建期安装位置与运行期读取位置一致
ENV HOME=/root

# 4) CODEX_HOME:plugin 默认安装到此目录,固化进镜像层后容器启动即可用
ENV CODEX_HOME=/root/.codex

# 5) TMPDIR:security-diff-scan 会把 scan bundle 写到这里,需要可写
ENV TMPDIR=/tmp

# 6) git SSH → HTTPS,免密 clone 公开仓库(构建环境无 SSH key)
RUN git config --global url."https://github.com/".insteadOf "git@github.com:"

# 7) 安装 codex-security plugin(user scope,落在 CODEX_HOME,固化进镜像层)
#    plugin 提供 $codex-security:security-diff-scan skill,供 CI 调用
RUN codex plugin install codex-security

# 8) 构建期自检:装不上就让 build 失败
RUN codex plugin list | grep -q "codex-security" && echo "✅ codex-security plugin ready"

# 工作目录:CI 会把被扫描仓库 checkout 到这里
WORKDIR /workspace
