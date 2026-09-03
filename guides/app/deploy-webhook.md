---
id: deploy-webhook
slug: /deploy-webhook
title: Webhook自动发布
sidebar_position: 6
---

## 介绍

常规发布配置可以开启 Webhook，与代码托管平台的 Webhook 结合实现 **推送代码或打 Tag 后自动创建并执行发布**。目前支持 `Gitee`、`GitHub`、`GitLab`、`Gogs`、`Coding` 和 `Codeup`（阿里云）。

## 配置步骤

1. 在 `应用发布 / 发布配置` 页面展开应用，在对应的发布配置行点击 **Webhook**。

   ![Webhook 设置](/img/docs/4.0/zh/deploy-webhook.jpg)

2. 选择 **触发方式**：
   - `Branch`：指定分支，当该分支有新的推送时触发发布，发布的代码为推送的最新提交，生成的 Webhook URL 会带上该分支的 `name` 参数，其他分支的推送不会触发；
   - `Tag`：当仓库有新的 Tag 推送时触发，发布该 Tag 对应的代码。
3. 复制页面上生成的 **Webhook URL** 与 **Secret Token**。
4. 到代码托管平台的仓库设置中添加 Webhook：
   - URL 填写复制的 Webhook URL；
   - 密钥 / Secret 填写 Secret Token（Gitee 中为 WebHook 密码，Gogs 中为密钥文本，GitHub 中为 Secret，GitLab 中为 Secret Token）；
   - 事件选择 `Push` 事件（Tag 触发方式选择 `Tag Push`）。
5. 提交一次推送测试，成功后会在 [发布申请](/docs/deploy-request) 中看到类型为 `Webhook触发` 的申请单，标题自动取自提交信息的前 20 个字符（Tag 触发时标题为 Tag 名）。

:::tip 提示
- 老版本 GitLab 等没有 Secret Token 设置项的平台，可以在 Webhook URL 后附加 `&token=<Secret Token>` 参数。
- 如果发布配置开启了 **发布审核**，Webhook 只会创建一个 `待审核` 的发布申请，构建与发布都要等审核通过后再手动执行。
- 请确保代码托管平台可以访问到 `Spug` 的地址，内网部署时可通过公网代理转发 `/api/apis/deploy/` 路径。
:::

## 请求校验

`Spug` 会根据请求头识别平台并校验 Secret Token：

| 平台 | 请求头 | 校验方式 |
| --- | --- | --- |
| GitLab | `X-Gitlab-Token` | 与 Secret Token 明文比较 |
| Gitee | `X-Gitee-Token` | 与 Secret Token 明文比较 |
| Codeup | `X-Codeup-Token` | 与 Secret Token 明文比较 |
| Gogs | `X-Gogs-Signature` | HMAC-SHA256 签名 |
| GitHub | `X-Hub-Signature-256` | HMAC-SHA256 签名 |
| Coding | `X-Coding-Signature` | HMAC-SHA1 签名 |

也可以在自己的 CI 系统中直接调用 Webhook URL（分支触发：`POST /api/apis/deploy/<发布配置ID>/branch/?name=<分支名>&token=<Secret Token>`；Tag 触发：`POST /api/apis/deploy/<发布配置ID>/tag/?token=<Secret Token>`），请求体按上述任一平台的格式携带 `ref` 与提交信息即可。

更多实践请参考博客 [Webhook 自动发布](/blog/webhook-deploy)。
