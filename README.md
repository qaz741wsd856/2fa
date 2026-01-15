# 2FA Authenticator

[English](./README_EN.md)

云端 2FA 认证器，支持 Cloudflare Workers 和 Docker 两种部署方式。

## 功能特性

- **TOTP 生成**：兼容 Google Authenticator、Authy 等标准 TOTP 协议
- **云端同步**：数据存储在 Cloudflare KV，跨设备访问
- **端到端加密**：AES-256-GCM 加密，服务端只存储密文
- **零注册**：无需邮箱/手机号，用主密码即可创建账户
- **PWA 支持**：可安装到桌面/主屏幕，享受原生应用体验
- **离线使用**：首次登录后支持完全离线访问，数据自动缓存 7 天
- **二维码扫描**：支持摄像头扫描、图片上传、剪贴板粘贴识别二维码
- **导入导出**：支持 JSON 格式备份，方便数据迁移和本地备份

## 技术架构

支持两种部署方式：

**Cloudflare Workers 部署**:
```
浏览器 <--HTTPS--> Cloudflare Worker <--KV API--> KV 存储
```

**Docker 部署**:
```
浏览器 <--HTTP/HTTPS--> Express Server <--SQLite--> 本地数据库
```

**安全设计**：
| 方面 | 措施 |
|------|------|
| 数据加密 | AES-256-GCM，客户端加密后传输 |
| 密钥派生 | PBKDF2-SHA256，600,000 次迭代  |
| 用户标识 | 密码哈希 (PBKDF2) |

## 部署教程

### 方式一：Docker 部署（推荐）

前置条件：安装 [Docker](https://docs.docker.com/get-docker/)

#### 使用 Docker Run

```bash
docker run -d \
  --name 2fa-auth \
  -p 3000:3000 \
  -v 2fa-data:/app/data \
  l981244680/2fa:latest

# 访问 http://localhost:3000
```

#### 使用 Docker Compose

创建 `docker-compose.yml` 文件：

```yaml
services:
  2fa:
    image: l981244680/2fa:latest
    container_name: 2fa-authenticator
    ports:
      - "3000:3000"
    volumes:
      - ./data:/app/data
    restart: unless-stopped
```

然后运行：

```bash
docker compose up -d
```

#### 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `PORT` | 3000 | HTTP 服务端口 |
| `DB_PATH` | `/app/data/2fa.db` | SQLite 数据库路径 |

### 方式二：Cloudflare Workers 部署

#### 前置条件

- [Node.js](https://nodejs.org/) 18+
- [Cloudflare 账户](https://dash.cloudflare.com/sign-up)

#### 步骤 1: 安装 Wrangler CLI

```bash
npm install -g wrangler
```

#### 步骤 2: 登录 Cloudflare

```bash
wrangler login
```

#### 步骤 3: KV 命名空间（可选）

当 `wrangler.toml` 里只配置 `binding`（不填 `id`）时，wrangler 会在首次 `wrangler deploy` 时自动创建（或复用）KV 命名空间，重复部署也会绑定到同一个 KV。因此本仓库默认可以跳过本步骤。

如果你想手动创建（例如明确指定/复用已有 KV），执行：

```bash
# 进入项目目录
cd 2fa

# 创建生产环境 KV
wrangler kv namespace create DATA_KV
# 输出类似: { binding = "DATA_KV", id = "xxxxxxxxxxxx" }

# 创建预览环境 KV（可选）
wrangler kv namespace create DATA_KV --preview
# 输出类似: { binding = "DATA_KV", preview_id = "yyyyyyyyyyyy" }
```

#### 步骤 4: 配置 wrangler.toml

- 自动创建方式：保持 `wrangler.toml` 里 `[[kv_namespaces]]` 仅包含 `binding`，直接执行 `wrangler deploy`；Wrangler 会自动创建/复用 KV（不会修改 `wrangler.toml`）。
- 手动方式：将上一步输出的 `id` / `preview_id` 填入 `wrangler.toml`：

```toml
name = "2fa-sync"
main = "worker.js"
compatibility_date = "2024-01-01"
assets = { directory = "./public" }

[[kv_namespaces]]
binding = "DATA_KV"
id = "xxxxxxxxxxxx"        # 替换为你的 id
preview_id = "yyyyyyyyyyyy" # 替换为你的 preview_id
```

#### 步骤 5: 本地测试

```bash
wrangler dev
# 访问 http://localhost:8787
```

#### 步骤 6: 部署

```bash
wrangler deploy
# 输出类似: Published 2fa-sync (https://2fa-sync.xxx.workers.dev)
```

部署完成后，访问输出的 URL 即可使用。

### GitHub Actions 自动部署（可选）

本仓库包含一个用于自动部署 Cloudflare Worker 的工作流：

- Deploy Cloudflare Worker：`.github/workflows/deploy-worker.yml` — 在 push 到 `main` 或手动触发时部署 Worker。需要在仓库 Secrets 中设置：`CLOUDFLARE_API_TOKEN` 和 `CLOUDFLARE_ACCOUNT_ID`。

## 使用说明

### 首次使用（创建账户）

1. 访问部署后的 URL
2. 点击「首次使用? 创建账户」
3. 设置主密码（至少 4 个字符）
4. 确认密码后点击「设置密码」

### 登录

1. 输入主密码
2. 点击「解锁」

### 添加 2FA 密钥

点击右上角「+」按钮，支持三种方式：

**手动输入**：
1. 输入名称（如：GitHub）
2. 输入 Base32 格式的密钥
3. 点击「添加」

**扫描二维码**：
1. 切换到「扫描」标签
2. 点击「启动摄像头」
3. 将二维码对准摄像头，识别成功后自动填充

**上传图片**：
1. 切换到「上传」标签
2. 点击选择图片、拖拽图片或直接粘贴截图
3. 识别成功后自动填充

### 使用验证码

- 点击验证码可复制到剪贴板
- 右侧圆环显示剩余有效时间（30 秒周期）

### 退出登录

点击左上角退出按钮，清除当前会话并返回登录页面。

### 导入导出

**导出备份**：
1. 登录后点击页面底部「导出」按钮
2. 下载 JSON 格式的备份文件（明文存储，请妥善保管）

**导入备份**：
1. 点击页面底部「导入」按钮
2. 选择之前导出的 JSON 文件
3. 会跳过同名密钥，保留现有数据，仅导入新密钥

## 注意事项

1. **密码不可找回**：忘记密码将无法恢复数据，请牢记主密码
2. **密码即账户**：相同密码 = 相同账户，不同设备用相同密码登录可同步数据
3. **会话有效期**：关闭浏览器标签页后会话失效，需重新输入密码
4. **离线模式**：首次需联网登录，之后可离线使用（缓存有效期 7 天）
5. **数据同步**：离线期间的修改会在联网后自动同步，如有冲突会提示选择

## 项目结构

```
2fa/
├── .github/
│   └── workflows/
│       ├── deploy-worker.yml   # Worker 自动部署
│       └── docker-publish.yml  # Docker 镜像发布
├── public/
│   ├── icons/           # PWA 图标
│   ├── index.html       # 前端页面
│   ├── manifest.json    # PWA 清单
│   └── service-worker.js # Service Worker (离线缓存)
├── src/
│   └── server.js        # Docker 版本的 Express 服务器
├── worker.js            # Cloudflare Worker
├── wrangler.toml        # Wrangler 配置文件
├── Dockerfile           # Docker 镜像定义
├── docker-compose.yml   # Docker Compose 配置
├── package.json         # npm 依赖配置
└── README.md            # 本文档
```

## License

MIT
