---
id: deploy-product-script
slug: /deploy-product-script
title: 一键部署
---

## 操作系统
> 我们推荐你使用 [Docker安装](/docs/install-docker) 来确保体验的一致性，一键安装脚本在Centos7/8和Ubuntu 18.04验证通过，其他版本暂未验证。

## 安装命令
```shell script
$ curl https://spug.dev/installer/spug-installer | bash
```

## 默认设置
```
# 默认代码安装路径：
/data/spug

# 默认创建的数据库账号
用户：spug   
密码：spug.dev


# 默认创建的系统管理员
账户：admin  
密码：spug.dev
```
> 如果访问时出现 `403 Forbidden` 的错误页面，大部分情况下是 `selinux` 的问题。你可以执行 `setenforce 0` 来临时关闭 `selinux` 测试能否正常访问。 

## 安全建议
- 默认安装的 `Redis` 服务监听在 `127.0.0.1` 但未设置密码，如需启用密码认证，请参考 [如何配置使用带密码的 Redis 服务？](/docs/install-problem/#use-redis)
- 默认安装了 `Mariadb` 作为数据存储服务，监听在 `127.0.0.1` 创建了用户名为 `spug` 密码为 `spug.dev` 的用户，安全起见请自行更改该密码，并修改 `/data/spug/spug_api/spug/overrides.py` 使用新密码。
- 确保服务端接收到请求 `HTTP Header` 的 ` X-Real-IP` 为真实的客户端地址，`Spug` 会使用该IP提高安全性（当登用户的 IP 发生变化时 Token 自动失效）。
- 如果想更多的控制安装过程，请参考 [生产环境部署文档](/docs/deploy-product) 进行手动部署配置。