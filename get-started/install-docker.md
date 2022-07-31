---
id: install-docker
slug: /install-docker
title: Docker安装
sidebar_position: 1
---
## 依赖环境

- Docker
- 现代浏览器

## 安装步骤
以下安装步骤使用 `Centos7.x` 操作系统。

### 1. 安装docker

:::caution 注意
如已安装 docker 则忽略。

以下安装 docker 步骤适用于 `Centos`，其他系统安装请参考 [Docker官方文档](https://docs.docker.com/engine/install/#server)。
:::

```bash
yum install -y yum-utils
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install docker-ce docker-compose-plugin
systemctl enable docker
systemctl start docker
```

### 2. 创建docker-compose.yml

```bash
vi docker-compose.yml
```
```yaml
version: "3.9"
services:
  db:
    image: mariadb:10.8.2
    container_name: spug-db
    restart: always
    command: --port 3306 --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
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
      # 如果80端口被占用可替换为其他端口，例如: - "8000:80"
      - "80:80"
    environment:
      - SPUG_DOCKER_VERSION=v3.2.4
      - MYSQL_DATABASE=spug
      - MYSQL_USER=spug
      - MYSQL_PASSWORD=spug.cc
      - MYSQL_HOST=db
      - MYSQL_PORT=3306
    depends_on:
      - db
```

### 3. 启动容器

```bash
docker compose up -d
```

### 4. 初始化
以下操作会创建一个用户名为 `admin` 密码为 `spug.dev` 的管理员账户，可自行替换管理员账户/密码。

```bash
docker exec spug init_spug admin spug.dev
```

### 5. 访问测试
在浏览器中输入 `http://localhost:80` 访问（默认账户密码在第4步初始化时设置）。

### 6. 版本升级
你可以在 `系统管理/系统设置/关于` 中查看当前运行的 `Spug` 版本，可以在 [更新日志](/docs/change-log) 
查看当前最新版本，如果需要升级 `Spug` 请参考 [版本升级文档](/docs/update-version)。
