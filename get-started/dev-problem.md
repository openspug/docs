---
id: dev-problem
slug: /dev-problem
title: 二开相关问题
sidebar_position: 5
---

### 标准安装批量执行、发布、流水线的任务卡住无法看到执行输出
批量执行、文件分发、应用发布、流水线等功能依赖 worker 服务，通过以下命令启动，以下操作命令基于 [二次开发](/docs/install) 文档的环境

```bash
cd /data/spug/spug_api
source venv/bin/activate
python manage.py runworker
```

实时输出还依赖 WebSocket 服务，二次开发环境下 `python manage.py runserver` 已通过 `daphne` 同时提供 HTTP 与 WebSocket 服务，无需额外启动。

### 标准安装任务计划模块添加的任务不会执行
任务计划功能需要启动额外的服务，通过以下命令启动

```bash
cd /data/spug/spug_api
source venv/bin/activate
python manage.py runscheduler
python manage.py runworker
```

### 标准安装监控中心模块添加的监控任务不会执行
监控中心功能需要启动额外的服务，通过以下命令启动

```bash
cd /data/spug/spug_api
source venv/bin/activate
python manage.py runmonitor
python manage.py runworker
```

### macOS 如何使用 MySQL 替代默认的 SQLite 数据库？
需要安装 `mysqlclient` 数据库驱动库，可通过以下步骤安装

```bash
brew install mysql-client
export PATH="/usr/local/opt/mysql-client/bin:$PATH"
export LDFLAGS="-I/usr/local/opt/openssl/include -L/usr/local/opt/openssl/lib"
pip install mysqlclient
```

### 如何新增界面文案的翻译？
`Spug` 以中文文案作为翻译 key，英文词典位于 `spug_web/src/locales/en/` 目录，按模块拆分并在 `index.js` 中合并，`base.js` 为公共词汇且优先级最高。
在页面代码中用 `t('中文')` 包裹需要翻译的文案，然后在对应模块的词典中补充英文即可；后端接口返回的错误信息与选项展示值在 `spug_api/libs/locale.py` 中翻译。
