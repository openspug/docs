---
id: install-problem
slug: /install-problem
title: 安装相关问题
sidebar_position: 2
---

### 执行数据初始化命令 python manage.py updatedb 报错
一般有以下几种情况：

- Python 版本过低，`Spug 4.0` 要求 `Python 3.8` 及以上。
- 依赖包版本不匹配，请使用文档中的 `pip install -r requirements.txt` 安装依赖（`Django 4.2`、`channels 4` 等），不要单独升降级某个包。
- 使用 SQLite 时系统自带的 SQLite 版本太低，`Django 4.2` 要求 SQLite `3.21.0` 及以上，生产环境推荐使用 MySQL / MariaDB。

### Nginx 访问前端文件提示无权限问题
确认系统是否开启了 `selinux`。如果开启可通过执行 `setenforce 0` 来临时关闭后重试。

### 登录报错 请求失败: 504 Gateway Timeout
请确认 api 服务已启动。手动部署时 api 监听在 `127.0.0.1:9001`（见 `spug_api/tools/start-api.sh`），Nginx 配置中 `/api/` 的 `proxy_pass` 需要指向该地址；二次开发时前端通过 `spug_web/src/setupProxy.js` 代理到 `http://127.0.0.1:8000`，请确保 `python manage.py runserver` 监听在该端口。

### 登录报错 请求失败: 502 Bad Gateway
请确保 api 服务已正常启动且 `nginx` 配置正确。另可查看 `nginx` 日志，如有发现 `13: Permission denied` 字样的报错则可尝试关闭 `selinux` 后再测试。

### 登录报错 Exception: Error 61 connecting to 127.0.0.1:6379. Connection refused.
需要安装 `Redis`（5.0 及以上），如果安装的 `Redis` 不是监听在 `127.0.0.1` 需要修改配置文件 `spug_api/spug/overrides.py`
指定 `Redis` 的地址，配置中的 `CACHES` 和 `CHANNEL_LAYERS` 均使用了 `Redis`，[参考下文](#use-redis)。

### 主机 Console、发布页面或流水线控制台一直没有输出
这些功能的实时输出通过 WebSocket 推送，请确认 `spug-ws` 服务（`daphne`）已启动，并且 Nginx 中 `/api/ws/` 路径配置了 WebSocket 转发，[参考文档](/docs/use-problem#websocket)。

### 添加主机报错 Exception: not a vaild RSA private key file
当 `Spug` 生成的密钥对无法通过验证时会抛出该错误，请检查 `系统管理 / 系统设置 / 密钥设置` 中上传的密钥是否为有效的私钥（支持 RSA、Ed25519、ECDSA、DSA 格式，`Spug` 自动生成的是 2048 位 RSA 密钥），或者清空后由 `Spug` 自动生成。

### 如何配置使用带密码的 Redis 服务？ {#use-redis}
假设 `Redis` 密码为 `foo123`，则需要更改配置文件 `spug_api/spug/overrides.py`（推荐）或者 `settings.py`（影响后续版本升级）如下内容，修改完成后记得重启服务。

:::tip 提示
自定义的配置可以在 `spug_api/spug/` 目录下创建 `overrides.py` 文件来覆盖默认的配置，Docker 安装的该文件位于宿主机 `/data/spug/service/spug_api/spug/overrides.py`。`overrides.py` 中的配置项是整体覆盖而不是合并，请保留 `settings.py` 中该字典的其余键（如 `CACHES` 的 `KEY_PREFIX`，`CHANNEL_LAYERS` 的 `prefix` / `capacity` / `expiry`），只修改地址与密码。
:::

```bash title="vi spug_api/spug/overrides.py"
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://:foo123@127.0.0.1:6379/1",
        "KEY_PREFIX": "spug",
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
        }
    }
}

CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {
            "hosts": ["redis://:foo123@127.0.0.1:6379/0"],
            "prefix": "spug:channel",
            "capacity": 1000,
            "expiry": 120,
        },
    },
}
```

### Docker 部署使用外部 MySQL {#use-mysql}
Docker 安装方式的数据库连接信息完全来自 `docker-compose.yml` 中 `spug` 服务的环境变量，使用外部数据库只需要：

1. 在外部 MySQL / MariaDB 上创建编码为 `utf8mb4` 的数据库和账户。
2. 删除 `docker-compose.yml` 中的 `db` 服务以及 `spug` 服务的 `depends_on`。
3. 修改 `spug` 服务的 `MYSQL_HOST`、`MYSQL_PORT`、`MYSQL_DATABASE`、`MYSQL_USER`、`MYSQL_PASSWORD` 为外部数据库的信息。
4. 执行 `docker compose up -d` 启动后再执行 [初始化命令](/docs/install-docker#4-初始化)。

:::caution 注意
容器首次启动时会根据环境变量生成 `/data/spug/spug_api/spug/overrides.py`，之后修改环境变量不会再自动覆盖该文件。如果容器已经初始化过，请直接修改宿主机上的 `/data/spug/service/spug_api/spug/overrides.py` 中的 `DATABASES` 配置后重启容器。如需迁移数据，请查看 [版本升级注意事项](/docs/update-version)，以免造成后期无法升级新版本。
:::

### 使用 SqlServer 数据库 {#use-sqlserver}
感谢 @xiongwu1 提供的支持，请参考 [#38](https://github.com/openspug/spug/issues/38)。
