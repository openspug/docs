---
id: batch-exec
slug: /batch-exec
title: 批量执行
sidebar_position: 2
---

## 介绍

包含维护命令模版和批量远程执行命令两部分功能，常用来执行一些临时的任务例如，批量安装/卸载某个依赖包等。

## 执行任务
可以选择一到多个在主机管理中添加的主机作为执行的目标主机，命令内容可以直接写也支持从模板中读取已保存的命令。

:::tip 提示
[使用 `nohup` 或 `&` 启动后台进程页面一直在转圈不会结束？](/docs/use-problem#nohup)
:::

## 全局变量
在 Shell 脚本中通过 `$SPUG_HOST_HOSTNAME` 来引用，在 Python 脚本中可以通过 `os.getenv('SPUG_HOST_HOSTNAME')` 来引用。

| 变量名 | 示例 | 说明 |
|--|--|--|
| SPUG_HOST_ID | 1 | 主机ID |
| SPUG_HOST_HOSTNAME | 172.10.26.5 | 主机IP/域名 |
| SPUG_SSH_PORT | 22 | SSH连接端口 |
| SPUG_SSH_USERNAME | root | SSH连接用户 |
| SPUG_INTERPRETER | python | 命令解释器 |

## 模板管理
用于存储复杂、常用的命令集合，以便后期可随时使用。