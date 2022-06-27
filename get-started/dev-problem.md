---
id: dev-problem
slug: /dev-problem
title: 二开相关问题
sidebar_position: 3
---

### 标准安装批量执行的任务卡住无法看到执行输出
批量执行功能需要启动额外服务，通过以下命令启动，以下操作命令基于 [标准安装](/docs/install) 文档的环境

```bash
cd /data/spug/spug_api
source venv/bin/activate
python manage.py runworker
```

### 标准安装任务计划模块添加的任务不会执行
任务计划功能需要启动额外的服务，通过以下命令启动，以下操作命令基于 [标准安装](/docs/install) 文档的环境

```bash
cd /data/spug/spug_api
source venv/bin/activate
python manage.py runscheduler
python manage.py runworker
```

### 标准安装监控中心模块添加的监控任务不会执行
监控中心功能需要启动额外的服务，通过以下命令启动，以下操作命令基于 [标准安装](/docs/install) 文档的环境

```bash
cd /data/spug/spug_api
source venv/bin/activate
python manage.py runmonitor
python manage.py runworker
```

### macOS 如何使用 Mysql 替代默认的 Sqlite 数据库？
需要安装 `mysqlclient` 数据库驱动库，可通过以下步骤安装

```bash
brew install mysql-client
export PATH="/usr/local/opt/mysql-client/bin:$PATH"
export LDFLAGS="-I/usr/local/opt/openssl/include -L/usr/local/opt/openssl/lib"
pip install mysqlclient
```