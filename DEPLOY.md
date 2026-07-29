# 私有镜像部署教程（部署者不接触源码）

## 一、镜像所有者构建并推送

以下步骤只由源码所有者执行。先在GitHub创建私有仓库并创建具有`write:packages`权限的Personal Access Token，然后登录GHCR：

```bash
echo '你的GitHub令牌' | docker login ghcr.io -u 你的GitHub账号 --password-stdin
```

在机器人源码目录构建并推送固定版本号，不建议只使用`latest`：

```bash
cd /你的机器人源码目录
docker build --pull -t ghcr.io/你的GitHub账号/solana-kongtou-bot:2026.07.30 .
docker push ghcr.io/你的GitHub账号/solana-kongtou-bot:2026.07.30
```

在GitHub Packages中确认该容器包为Private。只向指定部署者发放只读`read:packages`令牌，不要发源码仓库权限、管理端私钥或可写令牌。

## 二、部署者登录私有镜像仓库

服务器只上传本部署包，解压到：

```text
/www/wwwroot/solana-telegram-bot
```

使用镜像所有者发放的只读令牌登录：

```bash
echo '只读镜像令牌' | docker login ghcr.io -u 镜像读取账号 --password-stdin
```

复制配置：

```bash
cd /www/wwwroot/solana-telegram-bot
cp .env.example .env
chmod 600 .env
```

分别生成需要的随机值：

```bash
openssl rand -hex 32
openssl rand -hex 32
openssl rand -hex 16
openssl rand -hex 32
```

依次用于`MASTER_KEY`、`REDIS_PASSWORD`、`LICENSE_INSTANCE_ID`和`LICENSE_INSTANCE_SECRET`。修改`.env`，并将`BOT_IMAGE`固定到实际版本：

```env
BOT_IMAGE=ghcr.io/你的GitHub账号/solana-kongtou-bot:2026.07.30
```

不要把`LICENSE_PRIVATE_KEY`放进机器人`.env`。

## 三、准备数据目录并启动

```bash
mkdir -p data redis-data
chown -R 1000:1000 data
chmod 700 data
docker compose pull
docker compose up -d --force-recreate
docker compose ps
docker compose logs --tail=100 bot airdrop-worker recovery-worker
```

该编排没有`build:`字段，因此部署服务器不会获得Dockerfile或源代码，也不会在本机编译机器人。

## 四、授权

把机器人`.env`中的`LICENSE_INSTANCE_ID`和`LICENSE_INSTANCE_SECRET`登记到授权后台并开启，同时确保全局授权开启。机器人只保存管理端公钥，管理端私钥永远不交付。

## 五、更新镜像

源码所有者推送新版本标签后，部署者只修改`.env`中的`BOT_IMAGE`，然后执行：

```bash
docker compose pull
docker compose up -d --force-recreate
docker image prune -f
```

更新前必须备份`.env`、`data/`和`redis-data/`。绝不能重新生成`MASTER_KEY`，否则既有钱包私钥无法解密。

## 安全边界

私有镜像能避免直接交付源码仓库，但拥有Docker与root权限的部署者仍可能导出镜像并分析其中的JavaScript。远程授权必须继续采用失败关闭、短时签名、实例密钥和每批链上操作复检。更高强度需要把关键链上构建或签名逻辑迁移到你控制的远程服务，而不只是客户端容器。
