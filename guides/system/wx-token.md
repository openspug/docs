---
id: wx-token
slug: /wx-token
title: 微信Token
sidebar_position: 6
---

## 介绍

微信 Token 是 Spug 提供内置服务的用户身份标识，通过该标识与微信用户建立关联，提供例如微信告警、登录两步验证等功能。

## 获取方法
1. 搜索（或扫描下方的二维码）关注微信公众号 <font color="#1890ff">Spug</font>。
2. 在公众号底部的 **我的** 菜单中获取微信 Token。

## 用途
- 作为 Spug 内置服务的调用凭据（`系统设置 / 基本设置`）。
- 告警模块中，作为 [报警联系人](/docs/alarm-contact) 的微信告警用户标识。
- 当启用登录 MFA（两步）认证时，作为发送验证码的用户标识（在 [账户管理](/docs/system-account) 或个人中心设置）。

## 小提示
- 同一个微信 Token 可以在不同的模块中使用。
- 由于微信的限制，你必须关注公众号才能收到相关信息。
- 普通用户可以在页面右上角 **个人中心 / 基本设置** 中自行填写微信 Token。

![about](https://cdn.spug.cc/img/spug-weixin.jpeg)
