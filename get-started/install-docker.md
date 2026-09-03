---
id: install-docker
slug: /install-docker
title: Docker安装
sidebar_position: 1
---

推荐使用 Docker 方式部署 `Spug`，官方镜像已内置 Nginx、Redis 以及全部后台服务，数据库使用独立的 MariaDB 容器，通过 `docker compose` 一键启动。

## 依赖环境

- Docker 20.10 及以上（含 `docker compose` 插件）
- 现代浏览器（Chrome / Edge / Firefox / Safari 最新版本）

:::info 支持的 CPU 架构
Docker Hub 上的 `openspug/spug-service` 镜像，`4.0.0`（本文档对应的 4.0 版本）与 `latest`（3.x，不是 4.0）两个标签都同时提供 `amd64` 与 `arm64` 架构，Docker 会按宿主机架构自动选择，鲲鹏、飞腾、树莓派、Apple Silicon 等 ARM64 主机可直接使用。ARM64 主机请从 Docker Hub 拉取，阿里云镜像仓库目前只有 x86 架构的 3.x 镜像；暂不支持 32 位 ARM（armv7）。
:::

## 安装步骤

以下步骤以 `Ubuntu 22.04` / `CentOS 7` 为例，其他 Linux 发行版仅安装 Docker 的命令不同。

### 1. 安装 Docker

:::caution 注意
如已安装 Docker 则跳过此步骤。其他系统的安装方法请参考 [Docker 官方文档](https://docs.docker.com/engine/install/#server)。
:::

```bash
# CentOS / RHEL
yum install -y yum-utils
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce docker-compose-plugin
systemctl enable docker
systemctl start docker
```

```bash
# Ubuntu / Debian
curl -fsSL https://get.docker.com | bash
systemctl enable docker
systemctl start docker
```

### 2. 创建 docker-compose.yml

```bash
mkdir -p /data/spug && cd /data/spug
vi docker-compose.yml
```

```yaml
version: "3.3"
services:
  db:
    image: mariadb:10.8
    container_name: spug-db
    restart: always
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    volumes:
      - /data/spug/mysql:/var/lib/mysql
    environment:
      - MYSQL_DATABASE=spug
      - MYSQL_USER=spug
      - MYSQL_PASSWORD=spug.cc
      - MYSQL_ROOT_PASSWORD=spug.cc
  spug:
    image: openspug/spug-service:4.0.0
    container_name: spug
    privileged: true
    restart: always
    volumes:
      - /data/spug/service:/data/spug
      - /data/spug/repos:/data/repos
    ports:
      # 如果 80 端口被占用可替换为其他端口，例如: - "8000:80"
      - "80:80"
    environment:
      - SPUG_DOCKER_VERSION=v4.0.0
      - MYSQL_DATABASE=spug
      - MYSQL_USER=spug
      - MYSQL_PASSWORD=spug.cc
      - MYSQL_HOST=db
      - MYSQL_PORT=3306
    depends_on:
      - db
```

配置说明：

| 路径 / 参数 | 说明 |
| --- | --- |
| `/data/spug/mysql` | 数据库数据目录，请务必持久化 |
| `/data/spug/service` | Spug 程序目录（容器内 `/data/spug`），首次启动时从 gitee 拉取 `SPUG_DOCKER_VERSION` 指定版本的代码并下载前端包，同时生成配置文件 `spug_api/spug/overrides.py` |
| `/data/spug/repos` | 常规发布的代码仓库与构建产物存储目录 |
| `SPUG_DOCKER_VERSION` | 首次启动时拉取的代码版本（git 标签），必须与镜像版本对应，`4.0.0` 镜像请填写 `v4.0.0`；该变量没有默认值，缺少时容器会在拉取代码阶段退出并反复重启 |
| `MYSQL_*` | 数据库连接信息，与 `db` 服务保持一致；也可改为你自己的外部 MySQL，[参考文档](/docs/install-problem#use-mysql) |
| `privileged: true` | 文件分发功能依赖 `sshfs` 挂载，需要特权模式 |

:::caution 镜像加速
阿里云镜像仓库 `registry.cn-hangzhou.aliyuncs.com/openspug/spug-service` 目前仅提供 x86 架构的 3.x 镜像，没有 `4.0.0` 标签，也没有 ARM64 版本，请不要把上面的镜像地址替换为阿里云地址。受网络影响拉取 Docker Hub 镜像失败时，请为 Docker 配置 Docker Hub 镜像加速器（在 `/etc/docker/daemon.json` 中设置 `registry-mirrors` 后重启 Docker），无需修改 `docker-compose.yml` 中的镜像地址。
:::

### 3. 启动容器

```bash
docker compose up -d
```

:::caution 注意
旧版本的 `docker-compose` 独立安装方式请执行 `docker-compose up -d`。
:::

:::info 首次启动需要访问外网
与内置代码的 3.x 镜像不同，`4.0.0` 镜像不内置代码：容器首次启动时会从 `gitee.com` 克隆 `SPUG_DOCKER_VERSION` 指定版本的代码，并从 `cdn.spug.cc` 下载对应的前端包，请确保服务器可以访问这两个地址，内网 / 离线环境无法直接完成首次启动。拉取代码与初始化数据库通常需要 10～30 秒（可通过 `docker logs -f spug` 查看容器日志），完成后再执行第 4 步。
:::

### 4. 初始化

以下命令会初始化数据库表结构，并创建一个用户名为 `admin`、密码为 `spug.cc` 的管理员账户，可自行替换账户 / 密码。

```bash
docker exec spug init_spug admin spug.cc
```

:::tip 提示
首次启动时请等待代码拉取完成、数据库初始化就绪（通常 10～30 秒）再执行上述命令。如果报错 `can't open file '/data/spug/spug_api/manage.py'`（代码尚未拉取完成）或 `Can't connect to MySQL server on 'db:3306'`（数据库还未就绪）导致初始化失败，等待几秒后重新执行即可。
:::

### 5. 访问测试

在浏览器中输入 `http://<服务器IP>:80` 访问（默认账户密码在第 4 步初始化时设置），页面右上角可切换中英文界面。

### 6. 版本升级

可以在 `系统管理 / 系统设置 / 关于` 中查看当前运行的 `Spug` 版本，在 [更新日志](/docs/change-log) 查看最新版本，升级方法请参考 [版本升级文档](/docs/update-version)。

## 常用运维命令

```bash
# 查看容器日志
docker logs -f spug

# 进入容器
docker exec -it spug bash

# 查看容器内各服务进程（gunicorn 为 api、daphne 为 ws，以及 runworker / runmonitor / runscheduler、nginx、redis）
# 4.0.0 镜像内未开启 supervisor 的 RPC 接口，supervisorctl 命令不可用，重启服务请直接重启容器
docker exec spug ps -ef

# 重启容器
docker restart spug
```

容器内的目录约定：

| 路径 | 说明 |
| --- | --- |
| `/data/spug/spug_api` | 后端代码，配置覆盖文件为 `spug/overrides.py` |
| `/data/spug/spug_api/logs` | 各后台服务日志 |
| `/data/spug/env` | 3.x 镜像的环境变量文件，`4.0.0` 镜像启动时不再读取；需要全局生效的环境变量请写在 `docker-compose.yml` 的 `environment` 中，[参考文档](/docs/use-problem#command-not-found) |
| `/etc/nginx/nginx.conf` | Nginx 配置 |
| `/etc/supervisord.conf` | 后台服务进程配置，supervisord 直接使用此文件启动 nginx、redis 与各 Spug 服务 |

## 安全建议

- 不要把 `Spug` 直接暴露在公网，如必须公网访问请通过防火墙限制来源 IP，并在 `系统管理 / 系统设置 / 安全设置` 中开启登录 MFA。
- 如在容器 80 端口之前再增加一层反向代理（Nginx 等），请正确转发 `X-Forwarded-For` 请求头并支持 WebSocket，[参考文档](/docs/use-problem#websocket)。
- 修改 `docker-compose.yml` 中的数据库默认密码。
