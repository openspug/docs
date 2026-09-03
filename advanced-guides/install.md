---
id: install
slug: /install
title: 二次开发
---

> 此安装文档适合具有一定编程能力基础的人员进行二次开发时的环境搭建，如果你是在生产环境部署，推荐 [Docker安装](/docs/install-docker)，
> 如有必要你也可以考虑 [手动部署](/docs/deploy-product/)。

## 依赖环境

- Python 3.8 及以上
- Node.js 18 及以上（推荐 20 LTS；前端的 `npm start` / `npm run build` 脚本带有 `--openssl-legacy-provider` 参数，Node 16 及以下无法运行）
- Redis 5.0 及以上
- Git 2.17.0 及以上（常规发布需要）
- 现代浏览器

## 项目结构

| 目录 | 说明 |
| --- | --- |
| `spug_api` | 后端，Django 4.2 + channels 4，各功能模块位于 `apps/` 目录（host、exec、deploy、pipeline、credential、schedule、monitor、alarm、config、setting 等） |
| `spug_web` | 前端，React 16 + antd 4 + mobx，页面位于 `src/pages`，英文词典位于 `src/locales/en` |
| `spug_web2` | 基于 Vite + React 18 + antd 5 的新版前端雏形，尚未启用，4.0 使用的仍是 `spug_web` |
| `docs/docker` | 官方 Docker 镜像的构建文件 |

## 安装步骤
以下安装步骤假设项目安装在一台 `macOS` 系统的 `/data/spug` 目录下。

### 1. Clone项目代码

```bash
git clone https://github.com/openspug/spug /data/spug
```

### 2. 创建运行环境
```bash
cd /data/spug/spug_api
python3 -m venv venv
source venv/bin/activate
pip install -U pip setuptools
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple/
```

### 3. 初始化数据库
默认使用的 `SQLite` 数据库。
```bash
python manage.py updatedb
````
### 4. 创建默认管理员账户
```bash
python manage.py user add -u admin -p spug.cc -s -n 管理员

# -u 用户名
# -p 密码
# -s 超级管理员
# -n 用户昵称
```

### 5. 启动 api 开发环境服务
```bash
python manage.py runserver
```

`runserver` 只提供 HTTP 接口（默认 `http://127.0.0.1:8000`），不提供 WebSocket。Web 终端、批量执行 / 发布 / 流水线的实时输出等功能依赖 WebSocket，调试这些功能时请改用 `daphne` 启动后端，它同时提供 HTTP 与 WebSocket，但修改代码后需要手动重启：
```bash
daphne -b 127.0.0.1 -p 8000 spug.asgi:application
```

### 6. 安装前端依赖
可以把 `npm` 用 `yarn` 或 `pnpm` 代替。
```bash
cd /data/spug/spug_web
npm install --registry=https://registry.npmmirror.com
```

### 7. 启动前端
```bash
npm start
```

前端通过 `src/setupProxy.js` 把 `/api` 请求代理到后端的 8000 端口。

:::caution 注意
`src/setupProxy.js` 中 `ws` 配置项出现了两次，目前生效的是末尾的 `ws: false`，开发服务器不会转发 WebSocket。调试 Web 终端、实时输出等功能时，请把末尾的 `ws: false` 改为 `ws: true` 并重启 `npm start`，同时后端需用 `daphne` 启动（见第 5 步）。
:::

### 8. 访问测试
正常情况下 `npm start` 会自动在浏览器中打开项目，如果未打开可以在浏览器中输入 `http://localhost:3000` 访问。
如果你按照上边的文档执行的话，在第 4 步创建了默认的管理员账户：
```
用户名：admin
密码：spug.cc
```

### 9. 其他可选服务
通过以上步骤已经可以正常访问 `Spug` 了，但批量执行、发布、流水线依赖 `runworker`，任务计划依赖 `runscheduler`，监控中心依赖 `runmonitor`，请参考 [二开相关问题](/docs/dev-problem) 按需启动。

### 10. 打包前端
```bash
cd /data/spug/spug_web
npm run build
```
产物位于 `spug_web/build` 目录，手动部署时由 Nginx 直接提供静态文件。

## 常见安装问题
请参考 [安装问题](/docs/install-problem)
