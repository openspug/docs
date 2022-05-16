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
yum install docker-ce docker-ce-cli containerd.io
systemctl start docker
```

### 2. 拉取镜像

:::tip 提示
阿里云的镜像与 [Docker hub](https://hub.docker.com/r/openspug/spug/tags) 同步更新，国内用户建议使用阿里云的镜像。
:::

```bash
docker pull registry.aliyuncs.com/openspug/spug
```

### 3. 启动容器
如果需要持久化存储代码和数据，可以添加：-v 映射容器内/data路径。
:::tip 提示
官方镜像内置了 `Mysql` 数据库，如果需要使用外部已有数据库（Mysql 5.6+），可以参考 [此文档](/docs/install-problem#use-mysql) 
设置后再进行下一步的初始化操作。
:::

:::caution 注意
根据需要，以下两种启动方式任选其一即可。
:::

```bash
# 持久化存储启动命令：
# /spug 指的是映射本地的磁盘路径，也可以是其他目录，/data是容器内代码和数据初始化存储的路径
docker run -d --restart=always --name=spug -p 80:80 -v /spug:/data registry.aliyuncs.com/openspug/spug

# 如果你需要在spug内使用docker命令则需要添加额外的参数
docker run -d --restart=always --name=spug -p 80:80 -v /spug/:/data -v /var/run/docker.sock:/var/run/docker.sock -v /usr/bin/docker:/usr/bin/docker registry.aliyuncs.com/openspug/spug
```

### 4. 初始化
以下操作会创建一个用户名为 `admin` 密码为 `spug.dev` 的管理员账户，可自行替换管理员账户。
:::caution 注意
如果提示连接数据失败，再次执行尝试就可以了。
:::

```bash
docker exec spug init_spug admin spug.dev
```

### 5. 访问测试
在浏览器中输入 `http://localhost:80` 访问。  

```
用户名： admin  
密码： spug.dev
```

### 6. 版本升级
你可以在 `系统管理/系统设置/关于` 中查看当前运行的 `Spug` 版本，可以在 [更新日志](/docs/change-log) 
查看当前最新版本，如果需要升级 `Spug` 请参考 [版本升级文档](/docs/update-version)。
