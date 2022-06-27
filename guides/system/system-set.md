---
id: system-set
slug: /system-set
title: 系统设置
---

## 介绍

除了页面上对系统设置的管理，`Spug` 还提供了 `manage.py set` 命令可用于通过命令行进行管理操作。

## 禁用登录MFA
当某些特殊情况下可通过 `manage.py set mfa disable` 命令禁用MFA，用法示例如下
```bash
cd spug/spug_api
source venv/bin/activate
python manage.py set mfa disable
```
Docker 安装的可以执行如下命令
```bash
docker exec spug python3 /data/spug/spug_api/manage.py set mfa disable
```
