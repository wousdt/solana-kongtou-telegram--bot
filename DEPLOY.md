# Solana 批量空投 Telegram Bot｜部署教程

这是 Solana 批量钱包 Telegram Bot 的**部署仓库**。

本仓库只提供 Docker Compose、环境变量模板和部署说明；机器人程序通过 GitHub Container Registry（GHCR）公开镜像交付，用户无需GitHub账号或Token。

> 安全提示：这是热钱包程序。请仅存放可承受损失的资产，并先在测试环境和少量钱包中验证。燃烧交易不可逆。

## 功能概览

- Telegram 用户自动注册，新用户默认禁用
- 管理员启用或禁用用户
- 用户数据和子钱包按 Telegram ID 隔离
- 导入并加密保存主钱包
- 批量生成、导出和清空子钱包
- 独立设置空投 RPC 与回收 RPC
- 批量 SPL Token 空投
- 按 Mint 批量燃烧并关闭已清空的代币账户
- 空投与回收使用独立 Redis 队列和 Worker
- 管理员全局关闭或开启批量空投
- 管理员维护全局禁止空投 Mint 列表
- 严格远程授权：未知实例、已关闭实例或授权服务不可用时停止执行

## 仓库文件

```text
docker-compose.yml
env.example
README.md
```

本仓库不应包含：

```text
src/
Dockerfile
package.json
dist/
.env
数据库
钱包私钥
授权管理端私钥
```

## 部署要求

- Ubuntu 22.04/24.04 或 Debian 12
- 至少 2 核 CPU、2 GB 内存
- Docker 与 Docker Compose
- Telegram Bot Token
- 可公开拉取的GHCR机器人镜像
- 授权管理端地址和公钥
- 可用的 Solana RPC

## 一、安装 Docker

宝塔用户可在以下位置安装：

```text
宝塔面板 → Docker → 安装 Docker 和 Docker Compose
```

验证：

```bash
docker --version
docker compose version
```

## 二、从 GitHub 下载部署文件

### 方法 A：Git Clone

```bash
mkdir -p /www/wwwroot
cd /www/wwwroot

git clone \
  https://github.com/wousdt/solana-kongtou-telegram--bot.git \
  solana-telegram-bot

cd /www/wwwroot/solana-telegram-bot
ls -la
```

### 方法 B：下载 ZIP

```bash
cd /tmp

curl -L \
  -o solana-bot-deploy.zip \
  "https://github.com/wousdt/solana-kongtou-telegram--bot/archive/refs/heads/main.zip"

unzip solana-bot-deploy.zip
mkdir -p /www/wwwroot/solana-telegram-bot

cp -a \
  /tmp/solana-kongtou-telegram--bot-main/. \
  /www/wwwroot/solana-telegram-bot/

cd /www/wwwroot/solana-telegram-bot
```

目录中应看到：

```text
docker-compose.yml
env.example
README.md
```

## 三、验证公开GHCR镜像

公开镜像不需要登录，直接执行：

```bash
docker manifest inspect \
  ghcr.io/wousdt/solana-kongtou-bot:2026.07.30
```

能够返回JSON即表示镜像公开且版本存在。

## 四、创建配置

```bash
cd /www/wwwroot/solana-telegram-bot
cp env.example .env
chmod 600 .env
```

生成四个随机值：

```bash
openssl rand -hex 32
openssl rand -hex 32
openssl rand -hex 16
openssl rand -hex 32
```

四行输出依次用于：

```text
MASTER_KEY
REDIS_PASSWORD
LICENSE_INSTANCE_ID
LICENSE_INSTANCE_SECRET
```

编辑：

```bash
nano .env
```

完整配置格式：

```env
BOT_IMAGE=ghcr.io/wousdt/solana-kongtou-bot:2026.07.30

BOT_TOKEN=BotFather提供的机器人Token
ADMIN_IDS=Telegram管理员数字ID
MASTER_KEY=第一条命令生成的64位十六进制字符串

DEFAULT_RPC_URL=https://api.mainnet-beta.solana.com

REDIS_PASSWORD=第二条命令生成的64位十六进制字符串
REDIS_URL=redis://:与REDIS_PASSWORD完全相同的密码@redis:6379

DATABASE_PATH=./data/bot.db

TX_BATCH_SIZE=8
RPC_CONCURRENCY=8
MAX_CONCURRENT_CHAIN_JOBS=10

AIRDROP_WORKER_CONCURRENCY=10
RECOVERY_WORKER_CONCURRENCY=10
AIRDROP_RPC_CONCURRENCY=12
RECOVERY_RPC_CONCURRENCY=8
AIRDROP_PER_JOB_CONCURRENCY=4

RPC_REQUEST_TIMEOUT_MS=30000
TX_CONFIRM_TIMEOUT_MS=90000

LICENSE_SERVER_URL=https://bot.apibot.xin
LICENSE_INSTANCE_ID=第三条命令生成的32位十六进制字符串
LICENSE_INSTANCE_SECRET=第四条命令生成的64位十六进制字符串
LICENSE_PUBLIC_KEY=授权管理员提供的完整Ed25519公钥
LICENSE_TIMEOUT_MS=5000
LICENSE_CHECK_INTERVAL_MS=3000
```

注意：

- `REDIS_URL` 中的密码必须与 `REDIS_PASSWORD` 完全一致。
- `MASTER_KEY` 部署后不能更换；更换后原数据库中的钱包私钥无法解密。
- `LICENSE_PUBLIC_KEY` 不能换行或遗漏末尾的 `=`。
- `.env` 中不能保留中文占位符。
- 多个管理员 ID 使用英文逗号分隔。

检查关键字段长度：

```bash
awk -F= '
/^MASTER_KEY=/{print $1,length($2)}
/^REDIS_PASSWORD=/{print $1,length($2)}
/^LICENSE_INSTANCE_ID=/{print $1,length($2)}
/^LICENSE_INSTANCE_SECRET=/{print $1,length($2)}
/^LICENSE_PUBLIC_KEY=/{print $1,length($2)}
' .env
```

预期：

```text
MASTER_KEY 64
REDIS_PASSWORD 64
LICENSE_INSTANCE_ID 32
LICENSE_INSTANCE_SECRET 64
LICENSE_PUBLIC_KEY 大于等于40
```

## 五、创建数据目录

```bash
cd /www/wwwroot/solana-telegram-bot

mkdir -p data redis-data
chown -R 1000:1000 data
chmod 700 data
chmod 600 .env
```

`data/` 保存机器人 SQLite 数据库，`redis-data/` 保存任务队列。更新和重启时不得删除。

## 六、拉取并启动

检查 Compose：

```bash
docker compose config >/dev/null
```

拉取镜像：

```bash
docker compose pull
```

启动：

```bash
docker compose up -d --force-recreate
docker compose ps
```

正常应看到：

```text
bot                 Up
airdrop-worker      Up
recovery-worker     Up
redis               Up (healthy)
```

本版本不提供外部 HTTP 接收接口，没有开放 3000 端口属于正常情况。

## 七、查看日志

```bash
docker compose logs --tail=150 bot
docker compose logs --tail=100 airdrop-worker recovery-worker
```

实时日志：

```bash
docker compose logs -f --tail=100 \
  bot airdrop-worker recovery-worker
```

首次部署未登记时应显示：

```text
未授权停用：实例未登记
```

这属于正常状态，容器应保持运行。

## 八、申请实例授权

查看实例信息：

```bash
grep '^LICENSE_INSTANCE_ID=' .env
grep '^LICENSE_INSTANCE_SECRET=' .env
```

通过安全方式将这两项交给授权管理员。管理员需要在授权后台：

```text
填写实例ID
填写实例密钥
添加并开启实例
开启全局授权
```

成功后日志显示：

```text
[license] 远程授权已启用
```

授权管理端无法访问、全局关闭或实例被关闭时，机器人会按严格模式停止交互、任务提交和后续链上批次。

## 九、Telegram 初始化

向机器人发送：

```text
/start
```

新用户默认禁用，需要管理员在机器人“管理员功能”中启用。

用户启用后依次完成：

```text
设置空投RPC
设置回收RPC
导入主钱包
生成子钱包
```

## 十、更新镜像

收到新版本号后，只修改 `.env` 中的：

```env
BOT_IMAGE=ghcr.io/wousdt/solana-kongtou-bot:新版本号
```

然后：

```bash
docker compose pull
docker compose up -d --force-recreate
docker compose ps
docker compose logs --tail=100 \
  bot airdrop-worker recovery-worker
```

更新时不能覆盖：

```text
.env
data/
redis-data/
```

不要重新生成 `MASTER_KEY`。

## 常见问题

### `pull access denied`

公开镜像出现该错误，通常是Package仍为Private或镜像地址错误。镜像所有者需要将Package设为Public，并确认版本标签存在。

### `manifest unknown`

填写的镜像版本不存在。向镜像提供方确认准确的 `BOT_IMAGE`。

### `LICENSE_PUBLIC_KEY` 太短

```bash
grep '^LICENSE_PUBLIC_KEY=' .env |
awk -F= '{print length($2)}'
```

长度必须至少 40，不能填写中文占位符。

### `unable to open database file`

```bash
chown -R 1000:1000 data
chmod 700 data
docker compose up -d --force-recreate
```

并确认：

```env
DATABASE_PATH=./data/bot.db
```

### Redis 认证失败

确认：

```env
REDIS_PASSWORD=同一个密码
REDIS_URL=redis://:同一个密码@redis:6379
```

### 实例密钥认证失败

实例密钥与机器人 `.env` 不一致。使用相同实例ID和正确实例密钥重新登记。

### 签名无效

机器人中的 `LICENSE_PUBLIC_KEY` 与当前授权管理端私钥不匹配。向授权管理员索取最新公钥并重新创建容器。


## 备份

必须备份：

```text
.env
data/
redis-data/
```

绝不能公开：

```text
BOT_TOKEN
MASTER_KEY
REDIS_PASSWORD
LICENSE_INSTANCE_SECRET
钱包私钥
```

## 免责声明

本项目涉及热钱包、代币转账和不可逆燃烧操作。部署者应自行评估服务器、RPC、Telegram、链上交易和密钥管理风险。请先使用测试环境及小额资产验证，任何链上损失由实际操作与部署方自行承担。
