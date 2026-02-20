# Shiroi Remote Deploy

使用 GitHub Actions 构建 Shiroi Docker 镜像，推送到 GHCR 私有仓库，通过 SSH 连接服务器拉取镜像并重启容器。

## 为什么？

Shiroi 是 [Shiro](https://github.com/Innei/Shiro) 的闭源开发版本。

由于 Next.js 构建需要大量内存，很多服务器无法承受这样的开销。此项目利用 GitHub Actions 构建 Docker 镜像，推送到 GitHub Container Registry (GHCR) 私有仓库，然后通过 SSH 连接服务器执行 `docker compose pull` 拉取新镜像并重启容器。

## 工作流程

```
Shiroi 仓库 push
       ↓
GitHub Actions 构建镜像
       ↓
推送到 GHCR (私有)
       ↓
SSH 连接服务器
       ↓
docker compose pull & up -d
```

## 配置步骤

### 1. 服务器配置

#### 1Panel 添加 GHCR 私有镜像仓库

在 1Panel 面板 → 容器 → 仓库 → 添加仓库：

| 字段 | 值 |
|------|-----|
| 名称 | `ghcr` |
| 地址 | `ghcr.io` |
| 用户名 | 你的 GitHub 用户名 |
| 密码 | GitHub PAT (需要 `read:packages` 权限) |

#### docker-compose.yml

在服务器上准备好 `docker-compose.yml`与`.env`，例如：

```yaml
version: '3'

services:
  shiroi:
    container_name: shiroi
    image: ghcr.io/<your-username>/shiroi:latest
    env_file:
      - .env
    restart: always
    ports:
      - 2323:2323
```

```
# Env from https://github.com/innei-dev/Shiroi/blob/main/.env.template
NEXT_PUBLIC_API_URL=
NEXT_PUBLIC_GATEWAY_URL=

NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=

## Clerk
CLERK_SECRET_KEY=

NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up
NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL=/
NEXT_PUBLIC_CLERK_AFTER_SIGN_UP_URL=/

TMDB_API_KEY=

GH_TOKEN=
```

### 2. GitHub Secrets 配置

在此仓库的 Settings → Secrets and variables → Actions 中添加：

| Secret | 说明 |
|--------|------|
| `GH_PAT` | 可访问 Shiroi 私有仓库的 GitHub Token |
| `SSH_HOST` | 服务器 IP 或域名 |
| `SSH_PORT` | SSH 端口（可选，默认 `22`） |
| `SSH_USERNAME` | SSH 登录用户名 |
| `SSH_KEY` | SSH 私钥（与 `SSH_PASSWORD` 二选一） |
| `SSH_PASSWORD` | SSH 密码（与 `SSH_KEY` 二选一） |
| `DEPLOY_COMPOSE_PATH` | 服务器上 `docker-compose.yml` 所在目录的绝对路径 |
| `DEPLOY_SERVICE_NAME` | docker compose 中的服务名（如 `shiroi`） |
| `AFTER_DEPLOY_SCRIPT` | (可选) 部署后在服务器上执行的脚本 |

> SSH 认证支持 Key 和 Password 两种方式，配置其中一个即可。如果两个都配置了，优先使用 Key。

### 3. 创建 GitHub PAT

1. 进入 [GitHub Settings → Tokens](https://github.com/settings/tokens)
2. Generate new token (classic)
3. 选择权限：
   - `repo` - 访问私有仓库
   - `read:packages` - 读取 packages (服务器拉取镜像用)
   - `write:packages` - 写入 packages (CI 推送镜像用，会自动获得)

## 触发部署

- **自动触发**：Push 到 `main` 分支
- **手动触发**：Actions 页面手动 Dispatch，支持 `force_build` 参数强制重建
- **外部触发**：使用 `repository_dispatch` 事件

## 镜像管理

镜像存储在 GHCR 私有仓库：`ghcr.io/<username>/shiroi`

每次构建会生成两个 tag：
- `latest` - 最新版本
- `<commit-sha>` - 对应的 commit hash

## 故障排除

### 镜像推送失败

1. 检查 `GITHUB_TOKEN` 权限是否包含 `packages:write`
2. 确认仓库 Settings → Actions → General → Workflow permissions 设置为 "Read and write permissions"

### SSH 连接失败

1. 确认 `SSH_HOST`、`SSH_PORT`、`SSH_USERNAME` 配置正确
2. 如果使用 Key 认证，确认 `SSH_KEY` 是完整的私钥（包含 `-----BEGIN` 和 `-----END`）
3. 如果使用 Password 认证，确认 `SSH_PASSWORD` 正确
4. 确认服务器防火墙允许 SSH 端口访问
5. 检查 GitHub Actions 的 IP 是否被服务器安全组放行

### 镜像拉取失败

1. 确认 1Panel 中配置的 GHCR 仓库凭证正确
2. 确认 PAT 有 `read:packages` 权限
3. 检查镜像地址是否正确：`ghcr.io/<username>/shiroi:latest`
4. 在服务器上手动运行 `docker pull ghcr.io/<username>/shiroi:latest` 测试

### 容器启动失败

1. 检查 `DEPLOY_COMPOSE_PATH` 路径是否正确
2. 检查 `DEPLOY_SERVICE_NAME` 是否与 `docker-compose.yml` 中的服务名一致
3. 在服务器上运行 `docker compose logs <service-name>` 查看日志
