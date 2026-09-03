---
id: system-set
slug: /system-set
title: 系统设置
sidebar_position: 5
---

## 介绍

`系统管理 / 系统设置` 包含平台级的配置，仅管理员可以访问，分为以下页签。

## 基本设置

![基本设置](/img/docs/4.0/zh/setting-basic.jpg)

- **调用凭据**：如需要使用 `Spug` 内置的邮件、微信报警和登录 MFA 等服务，请关注公众号 <font color="#1890ff">Spug</font> 获取 [微信 Token](/docs/wx-token) 填写在此处，否则留空。

## 安全设置

![安全设置](/img/docs/4.0/zh/setting-security.jpg)

- **访问IP校验**：建议开启，校验是否获取了真实的访问者 IP，防止因为增加的反向代理层导致基于 IP 的安全策略失效，当校验失败时会在登录时弹窗提醒。如果你在内网部署且仅在内网使用可以关闭该特性，详见 [实践指南](/docs/practice#security)。
- **登录IP绑定**：强烈建议开启，当开启后会把登录凭证与 IP 进行绑定，当该登录凭证通过其他 IP 访问时将自动失效。如非必要，切勿关闭该特性！
- **登录MFA（两步）认证**：开启后登录时需要输入发送到微信的验证码。开启前请确保 **调用凭据** 已配置且当前账户已设置微信 Token，开启时需要先通过一次验证码校验；所有账户都必须配置微信 Token 才能登录。如遇无法登录，可通过命令行临时关闭，见下文。

## LDAP设置 {#ldap}

![LDAP 设置](/img/docs/4.0/zh/setting-ldap.jpg)

- **LDAP服务地址 / 绑定DN / 密码**：用于查询用户的 LDAP 服务器地址（如 `ldap://ldap.example.com:389`）以及绑定账户。
- **用户OU**：用户所在的 OU，例如 `ou=users,dc=example,dc=com`。
- **用户过滤器**：用于定位用户的过滤条件，需要包含 `%(user)s` 占位符，例如 `(cn=%(user)s)`、`(uid=%(user)s)` 或 `(sAMAccountName=%(user)s)`。
- **登录名映射 / 姓名映射**：把 LDAP 用户的属性映射到 `Spug` 账户的登录名与姓名，例如 `cn` 与 `sn`。
- **测试**：填写一个 LDAP 用户的登录名和密码验证配置是否正确。
- **LDAP 用户导入**：从 LDAP 拉取用户列表并批量导入为 `Spug` 账户，导入后再为账户分配角色。

配置完成后，登录页选择 **LDAP登录** 即可使用 LDAP 账户登录。LDAP 账户首次登录会自动创建对应的 `Spug` 账户。

## 密钥设置

![密钥设置](/img/docs/4.0/zh/setting-key.jpg)

在这里你可以上传并使用已有的密钥对作为全局密钥，没有上传密钥的情况下，`Spug` 会在首次添加主机时自动生成密钥对。支持 RSA、Ed25519、ECDSA、DSA 格式，自动生成的为 2048 位 RSA 密钥。

:::caution 注意
更换全局密钥后，已经通过旧密钥验证的主机需要重新验证。
:::

## 报警服务设置

![报警服务设置](/img/docs/4.0/zh/setting-alarm.jpg)

- **邮件服务**：用于通过邮件方式发送报警信息，`内置` 模式使用 `Spug` 提供的邮件服务（需要调用凭据）；`自定义` 模式填写自己的 SMTP 服务器、端口、邮箱账号、密码 / 授权码与发件人昵称，支持 465（SSL）与 587 端口，可以点击 **测试** 发送测试邮件。

## 推送服务设置

![推送服务设置](/img/docs/4.0/zh/setting-push.jpg)

- **推送助手账户绑定**：登录 [推送助手](https://push.spug.cc)，在 **个人中心 / 个人设置** 中查看用户 ID 并填写在此处完成绑定，请注意保密该 ID。绑定后页面会显示短信、语音、邮件、微信公众号的余额，并可以在流水线的 **推送助手** 节点中选择推送助手的联系人发送消息，配置手册见 [推送助手文档](https://push.spug.cc/guide/spug)。

## 开放服务设置

![开放服务设置](/img/docs/4.0/zh/setting-service.jpg)

- **访问凭据**：用于访问平台的开放服务，例如 [配置中心 API](/docs/config-api) 的 `apiKey` 参数以及 [Webhook 自动发布](/docs/deploy-webhook) 的 Secret Token，请妥善保管。

## 关于

![关于](/img/docs/4.0/zh/setting-about.jpg)

展示操作系统、Python、Django 版本以及 `Spug API` 与 `Spug Web` 的版本号，两者不一致时会给出提示（通常刷新浏览器即可）。此处也提供官网文档与更新日志的入口，升级方法见 [版本升级](/docs/update-version)。

## 命令行管理

除了页面上对系统设置的管理，`Spug` 还提供了 `manage.py set` 命令可用于通过命令行进行管理操作。

### 禁用登录MFA

当某些特殊情况下（例如微信 Token 失效导致无法收到验证码）可通过 `manage.py set mfa disable` 命令禁用 MFA，用法示例如下
```bash
cd spug/spug_api
source venv/bin/activate
python manage.py set mfa disable
```
Docker 安装的可以执行如下命令
```bash
docker exec spug python3 /data/spug/spug_api/manage.py set mfa disable
```
