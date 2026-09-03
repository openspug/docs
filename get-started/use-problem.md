---
id: use-problem
slug: /use-problem
title: 使用常见问题
sidebar_position: 3
---

### 使用 nohup 启动后台进程页面一直在转圈不会结束？ {#nohup}
在 [批量执行](/docs/batch-exec/)、[发布配置](/docs/deploy-config/) 或流水线等的执行脚本中以 `nohup` 或 `&` 的方式启动后台子进程时需要
把命令的标准输出重定向至 `/dev/null`，例如以下启动 `Tomcat` 的命令：

```bash
cd web/WEB-INF/
nohup ./startup.sh &
```
把上述命令改为：

```bash
cd web/WEB-INF/
nohup ./startup.sh > /dev/null 2>&1 &
```

### 能否使用自己的密钥对？
可以，可以在 `系统管理 / 系统设置 / 密钥设置` 中上传自己的密钥对作为全局密钥；也可以在新建 / 编辑主机时为单台主机上传独立的私钥，独立密钥优先级高于全局密钥。

### 新建常规发布申请 git clone 错误 {#clone}
`Spug` 无法提供交互式的输入账户密码登录 git 仓库的能力，如果是公开的仓库 `http/https/ssh` 任何一种协议都可以，但如果是私有仓库推荐在发布配置的 `Git仓库地址` 旁点击 **私有仓库？** 按照提示配置：

- **密钥** 认证：把 `Spug` 的公钥添加到代码托管平台的部署公钥（Deploy Key）中，仓库地址使用 `ssh` 协议。
- **账户密码** 认证：仓库地址使用 `http/https` 协议，填写账户和密码（或 Access Token），`Spug` 会自动拼接为 `https://yourname:password@gitee.com/openspug/spug.git` 的形式，账户名或密码中的 `@`、`/` 等特殊字符会被自动转义，无需手工替换。

特别要注意的是，如果你是通过 Docker 方式部署的则需要确保在容器内可以访问仓库，而不是在宿主机上。流水线的构建节点则通过 [凭据管理](/docs/system-credential) 中的凭据访问私有仓库。

### 主机 Console、发布页面或流水线控制台无内容 {#websocket}
这种情况大部分都是 `WebSocket` 连接建立失败了，一般出现在部署时自己加了一层 Nginx 之类的代理工具，这些代理工具默认无法处理 `WebSocket` 请求，
这就需要你配置其支持转发 `WebSocket` 请求，下边给个 `Nginx` 的例子，这里假设你用 Docker 部署的 `Spug`，映射了宿主机的 8000 端口：

```bash
server {
  listen 80;
  server_name xxx.xxx.xxx;
  client_max_body_size 100m;

  location / {
    proxy_pass http://127.0.0.1:8000;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }

  location ^~ /api/ws/ {
    proxy_pass http://127.0.0.1:8000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }

  error_page 404 /index.html;
}
```

### 文件管理器上传文件报错 413 Request Entity Too Large
文件上传大小受 `Nginx` 的 `client_max_body_size` 影响。官方 Docker 镜像内的 Nginx 已设置为不限制大小，出现该错误通常是外层反向代理的限制，请在代理层增加如下配置：

```bash
server {
  listen 80;
  server_name xxx.xxx.xxx;
  client_max_body_size 100m;

  ...
}
```

:::caution 注意
如需修改容器内的 Nginx 配置，文件位于容器内的 `/etc/nginx/nginx.conf`，可以在容器外部编辑后通过 `docker cp` 复制至容器内，也可以在容器内执行
`vi` 直接修改，最后别忘了重启容器。
:::

### 钉钉收不到通知？ {#use-dd}
钉钉机器人安全设置中 **IP地址** 添加部署 `Spug` 的服务器的公网地址，或者使用 **自定义关键词** 填写 `通知`，如下图。
如果机器人选择了 **加签**，请把签名密钥填写到报警联系人的 `钉钉 Secret`（或流水线钉钉推送节点的 `加签密钥`）中。

![about](https://cdn.spug.cc/v2/install-error-dd.png)

### 飞书、企业微信收不到通知？
- 飞书机器人如果开启了 **签名校验**，请把密钥填写到报警联系人的 `飞书 Secret` 中；如果设置了 **自定义关键词** 请包含 `通知` 或 `告警`。
- 企业微信群机器人无安全设置，请确认 Webhook 地址完整，且部署 `Spug` 的服务器可以访问 `qyapi.weixin.qq.com`。

### 发布任务执行时命令找不到提示 `xx: not found`？ {#command-not-found}
`Spug` 执行发布任务时，可能会在主机上执行一些命令，如果提示命令找不到，可以尝试在发布配置中的 `应用发布前执行` 或 `应用发布后执行` 中添加 `export PATH=$PATH:/xxx` 以加载环境变量。

Docker 部署时，`4.0.0` 镜像启动时不再读取 3.x 的 `/data/spug/env` 文件，需要在容器内全局生效的环境变量请写在 `docker-compose.yml` 中 `spug` 服务的 `environment` 里，然后执行 `docker compose up -d` 重建容器：

```yaml
    environment:
      - JAVA_HOME=/usr/local/jdk
      - PATH=/usr/local/jdk/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
      # ……SPUG_DOCKER_VERSION、MYSQL_* 等原有变量保持不变
```

:::caution 注意
- `PATH` 会整体覆盖镜像的默认值，请保留上面的系统目录。
- 这些变量只对在 Spug 容器内执行的步骤生效（如常规发布的 `代码检出前 / 后执行`），`应用发布前 / 后执行` 在目标主机上执行，使用目标主机自身的环境。
- 容器重建后镜像层内手动安装的软件会丢失，JDK、Node 等工具建议放在挂载目录 `/data/spug`（宿主机 `/data/spug/service`）下。
:::

### 文件分发或流水线数据传输提示 sshfs / rsync 相关错误
文件分发与流水线的数据传输、数据上传节点依赖部署 `Spug` 的主机（或容器）上的 `rsync` 与 `sshfs`，同时目标主机需要安装 `rsync`。官方 Docker 镜像已内置，手动部署请自行安装；Docker 部署时容器需要 `privileged: true` 才能执行 `sshfs` 挂载。

### 监控 Ping 检测提示未安装 ping 命令
Ping 检测在部署 `Spug` 的主机（或容器）上执行 `ping` 命令，请安装 `iputils-ping`（Debian / Ubuntu）或 `iputils`（CentOS）。官方 Docker 镜像已内置。

### 登录时提示未能获取到访问者的真实 IP
`Spug` 会依据请求来源 IP 做登录凭证绑定，当经过反向代理且未正确传递 `X-Forwarded-For` 时会出现该提示，请参考 [实践指南](/docs/practice#security) 配置代理，或在内网环境下于 `系统管理 / 系统设置 / 安全设置` 中关闭 **访问IP校验**。
