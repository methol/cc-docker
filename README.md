# agent-docker

预装 Claude Code 和 OpenAI Codex CLI 的 CI 镜像,各自配套官方/社区插件,可直接在 GitHub Actions / 其他 pipeline 中拉起使用。

两个镜像均使用 **官方推荐的 curl 原生安装** 方式(不依赖 Node.js),版本号与镜像 tag 严格对齐。

## 镜像

| Image                  | CLI                     | 内置 plugins                                                  |
| ---------------------- | ----------------------- | ------------------------------------------------------------- |
| `methol/cc-docker`     | `claude` (Claude Code)  | `code-review`, `security-guidance`, `andrej-karpathy-skills`  |
| `methol/codex-docker`  | `codex` (OpenAI Codex)  | `codex-security`, `andrej-karpathy-skills`                    |

镜像 tag 与镜像内 CLI 版本一致(`methol/cc-docker:2.1.89` 内的 `claude --version` 必为 `2.1.89`);`:latest` 跟随最新发布版本。

## 目录布局

```
.
├── claude/
│   └── Dockerfile          # cc-docker 构建上下文
├── codex/
│   └── Dockerfile          # codex-docker 构建上下文
├── .github/workflows/
│   ├── build.yml           # 构建并推送 cc-docker
│   └── build-codex.yml     # 构建并推送 codex-docker
├── CLAUDE.md               # 在本仓库工作时给 Claude Code 的项目指南
└── README.md
```

## 快速使用

### cc-docker

```bash
docker run --rm -it \
  -v "$(pwd):/workspace" \
  -e ANTHROPIC_API_KEY \
  methol/cc-docker \
  claude
```

> 首次运行需登录 Anthropic 账号,或通过 `ANTHROPIC_API_KEY` / 其它认证方式注入凭证。详见 <https://code.claude.com/docs/en/setup>。

### codex-docker

`codex-security` 是 OpenAI 商业插件,必须通过 Codex Web App 完成授权后才能使用。运行时把宿主机已激活的 `CODEX_HOME` 挂载进容器:

```bash
docker run --rm \
  -v "$HOME/.codex:/root/.codex:ro" \
  -v "$(pwd):/workspace" \
  -e OPENAI_API_KEY \
  methol/codex-docker \
  codex exec --sandbox workspace-write \
    --output-last-message /tmp/security-review.md \
    'Use $codex-security:security-diff-scan to review changes from origin/main to HEAD.'
```

## 本地构建

```bash
# cc-docker:版本号即镜像 tag
docker build -t cc-docker:dev --build-arg CC_VERSION=latest claude/

# codex-docker
docker build -t codex-docker:dev --build-arg CODEX_VERSION=latest codex/
```

如果你的本地构建走代理(自签 CA),可通过 BuildKit secret 注入 CA:

```bash
DOCKER_BUILDKIT=1 docker build \
  --secret id=proxy_ca,src=/path/to/proxy-ca.crt \
  -t codex-docker:dev codex/
```

## CI/CD

`.github/workflows/build.yml` 和 `build-codex.yml` 在以下情形触发:

- **每日定时**(02:00 / 03:00 UTC):检查 npm 是否发布新版,有则构建并推送 `methol/<image>:<version>` + `:latest`。
- **`workflow_dispatch`**:可手动指定版本 / 强制重建已有 tag。
- **`push` 到 `main`**:Dockerfile 或对应 workflow 改动时强制重建当前最新版 tag。

构建结束有版本一致性自检:`docker run --rm $IMAGE:$VERSION <cli> --version | grep -q $VERSION`,版本不对就直接 fail。

需要在仓库 Secrets 配置:

| Secret              | 用途                                                 |
| ------------------- | ---------------------------------------------------- |
| `DOCKERHUB_USERNAME`| Docker Hub 用户名(`methol`)                         |
| `DOCKERHUB_TOKEN`   | Docker Hub Personal Access Token(Read & Write 权限) |

## 安装方式

两个 Dockerfile 都用官方 curl 安装脚本:

- Claude Code:`curl -fsSL https://claude.ai/install.sh | bash -s "$CC_VERSION"`
- Codex CLI:`curl -fsSL https://chatgpt.com/codex/install.sh | CODEX_RELEASE="$CODEX_VERSION" CODEX_NON_INTERACTIVE=1 sh`

两者均把二进制装到 `$HOME/.local/bin`,因此 Dockerfile 里 `ENV PATH=/root/.local/bin:${PATH}` 必须保留,否则后续 RUN 找不到 CLI。

参考:

- <https://code.claude.com/docs/en/setup>
- <https://developers.openai.com/codex/cli>
