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
`openspug/spug-service` 镜像同时提供 `amd64` 与 `arm64` 两种架构，鲲鹏、飞腾、树莓派、Apple Silicon 等 ARM64 主机可直接使用 `latest` 标签，Docker 会自动拉取对应架构的镜像。暂不支持 32 位 ARM（armv7）。
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
    image: openspug/spug-service
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
| `/data/spug/service` | Spug 程序目录（容器内 `/data/spug`），首次启动时自动释放代码并生成配置文件 `spug_api/spug/overrides.py` |
| `/data/spug/repos` | 常规发布的代码仓库与构建产物存储目录 |
| `MYSQL_*` | 数据库连接信息，与 `db` 服务保持一致；也可改为你自己的外部 MySQL，[参考文档](/docs/install-problem#use-mysql) |
| `privileged: true` | 文件分发功能依赖 `sshfs` 挂载，需要特权模式 |

:::caution 镜像加速
受网络影响可能会导致镜像下载失败，可以替换 `docker-compose.yml` 中的镜像地址为阿里云镜像仓库：
```yaml
db:
    image: registry.cn-hangzhou.aliyuncs.com/openspug/mariadb:10.8
spug:
    image: registry.cn-hangzhou.aliyuncs.com/openspug/spug-service
```
:::

### 3. 启动容器

```bash
docker compose up -d
```

:::caution 注意
旧版本的 `docker-compose` 独立安装方式请执行 `docker-compose up -d`。
:::

### 4. 初始化

以下命令会初始化数据库表结构，并创建一个用户名为 `admin`、密码为 `spug.cc` 的管理员账户，可自行替换账户 / 密码。

```bash
docker exec spug init_spug admin spug.cc
```

:::tip 提示
如果首次启动时数据库容器还未就绪导致初始化失败，等待几秒后重新执行上述命令即可。
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

# 容器内查看各服务状态（api / ws / worker / monitor / scheduler / nginx / redis）
docker exec spug supervisorctl status

# 重启容器
docker restart spug
```

容器内的目录约定：

| 路径 | 说明 |
| --- | --- |
| `/data/spug/spug_api` | 后端代码，配置覆盖文件为 `spug/overrides.py` |
| `/data/spug/spug_api/logs` | 各后台服务日志 |
| `/data/spug/env` | 可选的环境变量文件，容器启动时自动 `source`，[参考文档](/docs/use-problem#command-not-found) |
| `/etc/nginx/nginx.conf` | Nginx 配置 |
| `/etc/supervisor/conf.d/spug.conf` | 后台服务进程配置 |

## 安全建议

- 不要把 `Spug` 直接暴露在公网，如必须公网访问请通过防火墙限制来源 IP，并在 `系统管理 / 系统设置 / 安全设置` 中开启登录 MFA。
- 如在容器 80 端口之前再增加一层反向代理（Nginx 等），请正确转发 `X-Forwarded-For` 请求头并支持 WebSocket，[参考文档](/docs/use-problem#websocket)。
- 修改 `docker-compose.yml` 中的数据库默认密码。
