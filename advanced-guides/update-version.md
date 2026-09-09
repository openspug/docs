---
id: update-version
slug: /update-version
title: 版本升级
---

:::caution 注意
版本升级可能涉及数据库表结构的更新，表结构的更新依赖 Django 提供的 migrate 工具，
该工具的正常运行依赖安装时初始化生成的 migrations 目录和数据库的 django_migrations 表，
<font color="red">请不要删除或改动这些目录或表，否则当涉及表结构更新的版本升级后可能造成缺少表字段等问题。</font>
:::

:::danger 警告
如果你现在部署的是 `v2.x.x` 的版本（系统设置/关于），请不要通过升级工具直接升级到 `v3` 或 `v4` 版本，否则会造成运行异常！[#419](https://github.com/openspug/spug/issues/419)
:::

## 从 3.x 升级到 4.0 {#upgrade-4}

4.0 升级了后端的核心依赖（`Python 3.8+`、`Django 4.2`、`channels 4`、`paramiko 3.4`），旧的 3.x 运行环境与 Docker 镜像无法运行 4.0 代码，请按以下步骤升级：

### Docker 安装

:::danger 不要使用 manage.py update 从 3.x 升级到 4.0
`manage.py update` 会把 4.0 的 Python 依赖用 pip 安装进 3.x 容器，容器一旦重建这些依赖就会丢失，它只用于 4.x 之间的小版本更新。从 3.x 升级到 4.0 请按下面的步骤更换镜像。
:::

1. 备份数据库与数据目录：
   ```bash
   docker exec spug-db mysqldump -uroot -pspug.cc spug > spug-backup.sql
   cp -r /data/spug /data/spug.bak
   ```
2. 修改 `docker-compose.yml`，把 `spug` 服务的镜像改为 `openspug/spug-service:4.0`，并在 `environment` 中增加 `SPUG_DOCKER_VERSION=v4.0.1`（其余配置保持不变），然后重建容器：
   ```yaml
     spug:
       image: openspug/spug-service:4.0
       environment:
         - SPUG_DOCKER_VERSION=v4.0.1
         - MYSQL_DATABASE=spug
         # ……其余配置保持不变
   ```
   ```bash
   cd /data/spug
   docker compose up -d
   ```
3. 在容器内把代码切换到 `v4.0.1`、下载对应的前端包并升级表结构（原有的管理员账号、执行模板等数据以及 `overrides.py` 中的自定义配置都会保留）：
   ```bash
   docker exec spug bash -c 'cd /data/spug && git fetch origin --tags && git checkout v4.0.1 && curl -fo /tmp/web.tar.gz https://cdn.spug.cc/spug/web_v4.0.1.tar.gz && rm -rf spug_web/build && tar xf /tmp/web.tar.gz -C spug_web/ && python3 spug_api/manage.py updatedb'
   ```
4. 重启容器，在 `系统设置 / 关于` 中确认版本已是 `v4.0.1`，并强制刷新浏览器：
   ```bash
   docker restart spug
   ```

:::info 为什么不能只执行 docker compose pull
Docker Hub 上的 `latest` 标签目前仍是 3.3.3，只执行 `docker compose pull` 拿不到 4.0。即使换成 `4.0` 镜像，它启动时发现 `/data/spug/service` 中已有代码也不会覆盖，因此必须执行第 3 步手动切换代码，否则 3.x 的代码会运行在 Django 4.2 环境上，容器状态虽然是 running，但所有接口都会报错。
:::

### 手动部署

1. 备份数据库。
2. 安装 Python 3.8 及以上版本并重建虚拟环境：
   ```bash
   cd /data/spug/spug_api
   git fetch origin --tags && git checkout v4.0.1
   rm -rf venv && python3 -m venv venv
   source venv/bin/activate
   pip install -U pip setuptools
   pip install -r requirements.txt mysqlclient -i https://pypi.tuna.tsinghua.edu.cn/simple/
   python manage.py updatedb
   ```
3. 下载 4.0 的前端包 `https://cdn.spug.cc/spug/web_v4.0.1.tar.gz` 解压到 `spug_web` 目录（会生成 `build` 目录）。
4. 确认 `supervisor` 中的 `spug-ws` 使用 `tools/start-ws.sh`（`daphne`）启动，然后 `supervisorctl restart all`。

## 4.x 版本之间更新

以下方式仅用于 4.x 之间的小版本更新，从 3.x 升级到 4.0 请按上一节操作。

### Docker 安装，版本更新
```bash
# 默认更新到最新版本
# spug 是容器名称，也可以替换为自己的容器ID

docker exec -i spug python3 /data/spug/spug_api/manage.py update

# 更新完成后重启容器
docker restart spug
```

### 手动安装，版本更新
```bash
# 默认更新到最新版本

cd spug_api
source venv/bin/activate
python manage.py update

# 重启服务
supervisorctl restart all
```

`manage.py update` 会检查最新版本并提示更新内容，确认后自动下载前端包、切换后端代码到对应的 Tag、更新依赖并执行数据库结构更新；当前已是最新版本时可以选择进行修复性更新（重新拉取当前版本）。
