---
id: install-problem
slug: /install-problem
title: 安装相关问题
sidebar_position: 2
---

### 执行数据初始化命令 python manage.py updatedb 报错
一般有以下两种情况
    
- `Django` 版本使用了 3.x 的版本，我们仅支持 2.2.x 版本，安装依赖推荐使用文档中的 `pip install -r requirements.txt` 来安装
- 系统的 Sqlite 版本太低，`Django 2.2` Sqlite 的版本最低要求为 3.8.3 [参见文档](https://docs.djangoproject.com/en/2.2/releases/2.2/)。

### Nginx 访问前端文件提示无权限问题
确认系统是否开启了 `selinux`。如果开启可通过执行 `setenforce 0` 来临时关闭后重试。
 
### 登录报错 请求失败: 504 Gateway Timeout
请确保 api 服务是否启动，如果已启动则可以通过控制台查看是否监听在 `8000` 端口，如果不是 `8000` 端口可以改为 `8000` 端口或者修改前端项目的
`spug/spug_web/src/setupProxy.js` 文件中的 `target` 值为你的 api 服务的监听地址和端口。

### 登录报错 请求失败: 502 Bad Gateway
请确保 api 服务已正常启动且 `nginx` 配置正确。另可查看 `nginx` 日志如有发现 `13: Permission denied` 字样的报错则可尝试关闭 `selinux` 后再测试。

### 登录报错 Exception: Error 61 connecting to 127.0.0.1:6379. Connection refused.
需要安装 `Redis`，如果安装的 `Redis` 不是监听在 `127.0.0.1` 需要修改配置文件 `spug_api/spug/settings.py`
指定 `Redis` 的 Host，配置中的 `CACHES` 和 `CHANNEL_LAYERS` 均使用了 `Redis`。

### 添加主机报错 Exception: not a vaild RSA private key file
当 `Spug` 生成的密钥对无法通过验证时，会尝试读取系统的 `~/.ssh/` 目录下的密钥进行验证，这个报错一般是在读取系统密钥时出错。 可以尝试先移除系统
的密钥，然后再操作添加主机，等添加完成后再恢复原有的密钥。

### 如何配置使用带密码的 Redis 服务？ {#use-redis}
假设 `Redis` 密码为 `foo123`，则需要更改以配置文件 `spug_api/spug/overrides.py`（推荐） 或者 `settings.py`（影响后续版本升级） 如下内容，修改完成后记得重启服务。

:::tip 提示
 自定义的配置可以在 `spug_api/spug/` 目录下创建 `overrides.py` 文件来覆盖默认的配置。
:::

```bash title="vi spug_api/spug/overrides.py"
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://:foo123@127.0.0.1:6379/1",
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
        },
    },
}
```

### Docker 部署使用外部 Mysql {#use-mysql}
官方 Docker 镜像内置了数据库服务，如果你想使用自己的外部数据库，可以通过如下方法：

:::caution 注意
如果需要迁移数据，请查看 [版本升级注意事项](/docs/update-version)，以免造成后期无法升级新版本。
:::

```bash
# 1. 进入容器
docker exec -it spug bash

# 2. 修改配置文件使访问外部数据库
vi /data/spug/spug_api/spug/overrides.py

DATABASES = {
    'default': {
        'ATOMIC_REQUESTS': True,
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'spug',
        'USER': 'spug',  # 修改为外部数据库的用户
        'PASSWORD': 'spug.dev',  # 修改为外部数据的用户密码
        'HOST': 'localhost',    # 修改为外部数据的ip
        'OPTIONS': {
            'unix_socket': '/var/lib/mysql/mysql.sock',   # ！！！删除该行
            'charset': 'utf8mb4',
            'sql_mode': 'STRICT_TRANS_TABLES',
        }
    }
}

# 3. 停止容器内的数据库服务
vi /etc/supervisord.d/spug.ini

# 找到如下行并删除
[program:mariadb]
command = /usr/libexec/mysqld --user=mysql
autostart = true

# 4. 退出并重启容器
exit
docker restart spug
```

### 使用 SqlServer 数据库 {#use-sqlserver}
感谢 @xiongwu1 提供的支持，请参考 [#38](https://github.com/openspug/spug/issues/38)
