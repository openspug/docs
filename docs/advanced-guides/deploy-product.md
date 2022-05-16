---
id: deploy-product
slug: /deploy-product
title: 手动部署
---

:::caution 注意
我们推荐你使用 [Docker安装](/docs/install-docker) 来确保体验的一致性。
:::

## 准备环境

- Python 3.6及以上
- Mysql 5.6及以上
- 现代浏览器
- 自 `v2.3.9` 开始 `Git` 版本需要 `2.17.0+` （影响新建常规发布申请单）

## 安装步骤
以下安装步骤假设项目部署在一台 `Centos7` 系统的 `/data/spug` 目录下。

### 1. Clone项目代码
```bash
git clone https://github.com/openspug/spug /data/spug
cd /data/spug
git checkout x.x.x   # x.x.x 为指定的发行版本，例如 git checkout v2.2.2 
```

### 2. [下载](https://github.com/openspug/spug/releases) 已编译打包后的前端项目
将下载好的前端压缩包解压到指定目录，假设`web_x.y.z.tar.gz` 是的你下载好的压缩包

:::tip 提示
访问 [`Github`](https://github.com/openspug/spug/releases) 比较慢的情况下，可以在尝试在 [`Gitee`](https://gitee.com/openspug/spug/releases) 下载。
:::

```bash
tar xf web_x.y.z.tar.gz -C /data/spug/spug_web/;
```

### 3. 创建运行环境
如需要使用常规发布功能，则需要安装 <font color="red">git v2.17.0+</font>。
```bash
# 安装依赖
yum install mariadb-devel python3-devel gcc openldap-devel redis nginx supervisor

# 创建虚拟环境
cd /data/spug/spug_api
python3 -m venv venv
source venv/bin/activate

# 安装python包
pip install -U pip -i https://pypi.tuna.tsinghua.edu.cn/simple/
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple/
pip install gunicorn mysqlclient -i https://pypi.tuna.tsinghua.edu.cn/simple/
```

### 4. 修改后端配置
后端默认使用的 `Sqlite` 数据库，通过修改配置使用 `MYSQL` 作为后端数据库，[如何使用SqlServer数据库？](/docs/install-problem#use-sqlserver)

:::caution 注意
在 `spug_api/spug/` 目录下创建 `overrides.py` 文件，启动后端服务后会自动覆盖默认的配置，避免直接修改 `settings.py` 以便于后期获取新版本。
:::

```python title="spug_api/spug/overrides.py"
DEBUG = False

DATABASES = {
    'default': {
        'ATOMIC_REQUESTS': True,
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'spug',             # 替换为自己的数据库名，请预先创建好编码为utf8mb4的数据库
        'USER': 'spug_user',        # 数据库用户名
        'PASSWORD': 'spug_passwd',  # 数据库密码
        'HOST': '127.0.0.1',        # 数据库地址
        'OPTIONS': {
            'charset': 'utf8mb4',
            'sql_mode': 'STRICT_TRANS_TABLES',
            #'unix_socket': '/opt/mysql/mysql.sock' # 如果是本机数据库,且不是默认安装的Mysql,需要指定Mysql的socket文件路径
        }
    }
}
```

### 5. 初始化数据库
```bash
cd /data/spug/spug_api
python manage.py updatedb
````
### 6. 创建默认管理员账户
```bash
python manage.py user add -u admin -p spug.dev -s -n 管理员

# -u 用户名
# -p 密码
# -s 超级管理员
# -n 用户昵称
```

### 7. 创建启动服务脚本
```bash title="/etc/supervisord.d/spug.ini"
[program:spug-api]
command = bash /data/spug/spug_api/tools/start-api.sh
autostart = true
stdout_logfile = /data/spug/spug_api/logs/api.log
redirect_stderr = true

[program:spug-ws]
command = bash /data/spug/spug_api/tools/start-ws.sh
autostart = true
stdout_logfile = /data/spug/spug_api/logs/ws.log
redirect_stderr = true

[program:spug-worker]
command = bash /data/spug/spug_api/tools/start-worker.sh
autostart = true
stdout_logfile = /data/spug/spug_api/logs/worker.log
redirect_stderr = true

[program:spug-monitor]
command = bash /data/spug/spug_api/tools/start-monitor.sh
autostart = true
stdout_logfile = /data/spug/spug_api/logs/monitor.log
redirect_stderr = true

[program:spug-scheduler]
command = bash /data/spug/spug_api/tools/start-scheduler.sh
autostart = true
stdout_logfile = /data/spug/spug_api/logs/scheduler.log
redirect_stderr = true
```

### 8. 创建前端nginx配置文件
```bash title="/etc/nginx/conf.d/spug.conf"
server {
        listen 80;
        server_name _;     # 修改为自定义的访问域名
        root /data/spug/spug_web/build/;
        client_max_body_size 20m;   # 该值会影响文件管理器可上传文件的大小限制，请合理调整

        gzip  on;
	    gzip_min_length  1k;
	    gzip_buffers     4 16k;
	    gzip_http_version 1.1;
	    gzip_comp_level 7;
	    gzip_types       text/plain text/css text/javascript application/javascript application/json;
        gzip_vary on;

        location ^~ /api/ {
                rewrite ^/api(.*) $1 break;
                proxy_pass http://127.0.0.1:9001;
                proxy_read_timeout 180s;
                proxy_redirect off;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location ^~ /api/ws/ {
                rewrite ^/api(.*) $1 break;
                proxy_pass http://127.0.0.1:9002;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "Upgrade";
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location / {
                try_files $uri /index.html;
        }
}
```

:::caution 注意
注意：如果你没有在 `spug.conf` 中指定 `server_name` 则需要把 `/etc/nginx/nginx.conf` 中默认的 `server` 块注释或删除后才能正常访问，
否则会打开 Nginx 默认页面。
:::

### 9. 启动服务
```bash
# 设置开机启动
systemctl enable nginx
systemctl enable redis
systemctl enable supervisord

# 启动服务
systemctl restart nginx
systemctl restart redis
systemctl restart supervisord
```

### 10. 访问测试
通过浏览器访问测试。


### 11. 安全建议
- 请确保安装的 `Redis` 仅监听在 `127.0.0.1`。如果需要使用密码认证的 `Redis` 请参考 [如何配置使用带密码的 Redis 服务？](/docs/install-problem/#use-redis)
- 确保服务端接收到请求 `HTTP Header` 的 ` X-Real-IP` 为真实的客户端地址，`Spug` 会使用该 IP 提高安全性（当登用户的 IP 发生变化时 Token 自动失效），[参考文档](/docs/practice)。
