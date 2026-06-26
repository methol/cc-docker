# 预装 Claude Code(官方原生安装) + 三个 plugin 的 CI 镜像
# ⚠️ 构建期需访问 downloads.claude.ai / github.com
# syntax=docker/dockerfile:1.7
FROM node:24-slim

# 0) 版本号入参:CI 传入具体 Claude Code 版本,本地构建默认 latest
#    镜像 tag 与此版本号统一(见 .github/workflows/build.yml)
ARG CC_VERSION=latest
LABEL org.opencontainers.image.title="cc-docker" \
      org.opencontainers.image.description="预装 Claude Code(原生安装) + 官方/karpathy plugins 的 CI 镜像" \
      org.opencontainers.image.source="https://github.com/methol/agent-docker" \
      org.opencontainers.image.version="${CC_VERSION}"

# 1) 基础工具:git + 常用开发工具 + curl(install.sh 必需)
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

# helper:安装可选代理 CA(仅本地构建注入;生产 CI 无需提供)
# 装好后 git/curl/npm 自动信任,不再需要额外 env var
RUN --mount=type=secret,id=proxy_ca,target=/tmp/proxy-ca.crt,required=false \
    if [ -f /tmp/proxy-ca.crt ]; then \
        cp /tmp/proxy-ca.crt /usr/local/share/ca-certificates/proxy-ca.crt && \
        update-ca-certificates; \
    fi

# 2) 固定 HOME,保证「构建期安装位置」与「运行期读取位置」一致(关键)
ENV HOME=/root
# 3) 原生安装器把二进制放在 ~/.local/bin,后续 RUN 必须能直接调用 claude
ENV PATH=/root/.local/bin:${PATH}

# 4) 官方原生安装(curl 方式,不依赖 Node;详见 https://code.claude.com/docs/en/setup)
#    版本号通过位置参数传入:latest|stable|x.y.z
RUN curl -fsSL https://claude.ai/install.sh | bash -s "${CC_VERSION}"

# 5) 关键坑:claude plugin 默认用 SSH clone GitHub,build 环境无 SSH key
#    用 git insteadOf 改写为 HTTPS,免密 clone 公开仓库
RUN git config --global url."https://github.com/".insteadOf "git@github.com:"

# 6) 添加 marketplace(官方 + karpathy 第三方)
RUN claude plugin marketplace add anthropics/claude-plugins-official \
 && claude plugin marketplace add forrestchang/andrej-karpathy-skills

# 7) 安装三个 plugin(user scope,落在 /root/.claude,固化进镜像层)
RUN claude plugin install code-review@claude-plugins-official        --scope user \
 && claude plugin install security-guidance@claude-plugins-official  --scope user \
 && claude plugin install andrej-karpathy-skills@karpathy-skills     --scope user

# 8) 构建期自检:装不上就让 build 失败,别等到 CI 才发现
RUN claude --version \
 && claude plugin list | grep -Eq "code-review.*enabled|✔" \
 && echo "✅ Claude Code + plugins ready"

WORKDIR /workspace
