# 项目指南(给 Claude Code)

本仓库构建并发布两个 Docker 镜像,分别预装 Claude Code 和 OpenAI Codex CLI,供 CI / 自动化 pipeline 直接使用。

## 目录布局

```
claude/Dockerfile           # methol/cc-docker
codex/Dockerfile            # methol/codex-docker
.github/workflows/build.yml        # 构建并推送 cc-docker
.github/workflows/build-codex.yml  # 构建并推送 codex-docker
```

每个镜像独立一个目录,docker build context 也对应这个目录(`docker build claude/` / `docker build codex/`)。不要把共享文件放到根目录然后 `context: .`,会破坏 build cache 边界。

## 核心约定

- **安装方式**:两个 CLI 都通过官方 curl 安装脚本拉原生二进制(不走 npm)。详见各 Dockerfile 顶部注释。
- **版本 = 镜像 tag**:`CC_VERSION` / `CODEX_VERSION` 是 build-arg,值就是发布到 Docker Hub 的镜像 tag。CI 在构建尾部用 `<cli> --version | grep -q $VERSION` 做版本一致性自检。
- **`PATH`**:原生安装把 binary 放到 `~/.local/bin`,Dockerfile 必须显式 `ENV PATH=/root/.local/bin:${PATH}`,否则后续 RUN(`claude plugin install` / `codex plugin add`)找不到命令。
- **`HOME=/root`**:固定 HOME,保证构建期写入的配置 / 插件路径与运行期一致(容器默认以 root 运行)。
- **CN 注释**:Dockerfile 和 workflow 用中文行内注释解释「为什么」,新增内容沿用同样风格。

## 不要做的事

- 不要把 `Dockerfile` 放回仓库根目录 —— GitHub workflow 的 `paths` watchlist 和 `context` 都已经指向 `claude/` 和 `codex/` 子目录。
- 不要在 Dockerfile 里写死版本号。CI 通过 `npm view` 解析最新版后用 `--build-arg` 注入,本地构建走 ARG 默认 `latest`。
- 不要新增 `npm install -g @anthropic-ai/claude-code` 或 `npm install -g @openai/codex`。已经迁到 curl 原生安装,混用会装两份。
- 不要在 plugin install 步骤后 `rm -rf /root/.codex/openai-plugins` —— `codex` 的 marketplace 注册写死了路径,删了下次启动会找不到。

## 改动后怎么验证

```bash
# 本地构建 + smoke test
docker build -t cc-docker:dev --build-arg CC_VERSION=latest claude/
docker run --rm cc-docker:dev claude --version
docker run --rm cc-docker:dev claude plugin list

docker build -t codex-docker:dev --build-arg CODEX_VERSION=latest codex/
docker run --rm codex-docker:dev codex --version
docker run --rm codex-docker:dev codex plugin list
```

如果有自签 CA(本地代理),`docker build` 加 `--secret id=proxy_ca,src=...`,Dockerfile 里的 secret mount 会自动 trust。

## CI 触发逻辑速记

- 定时(02:00 / 03:00 UTC):npm 有新版 → 构建并推 `:$VERSION` + `:latest`,没新版直接跳过。
- `workflow_dispatch`:可手动指定版本和 `force`(已有 tag 也重建)。
- `push` 到 `main` 且改动 Dockerfile / 对应 workflow:强制重建当前最新版 tag。
