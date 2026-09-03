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

1. 备份数据库与数据目录：
   ```bash
   docker exec spug-db mysqldump -uroot -pspug.cc spug > spug-backup.sql
   cp -r /data/spug /data/spug.bak
   ```
2. 拉取新版镜像并重建容器（`docker-compose.yml` 无需修改）：
   ```bash
   cd /data/spug
   docker compose pull
   docker compose up -d
   ```
3. 新镜像启动时会自动释放 4.0 的代码，执行数据库结构更新：
   ```bash
   docker exec spug python3 /data/spug/spug_api/manage.py updatedb
   docker restart spug
   ```

:::info 说明
3.x 的容器把代码释放在 `/data/spug/service` 目录，4.0 镜像检测到该目录已存在时不会覆盖。如果升级后 `系统设置 / 关于` 中的版本仍为 3.x，请把旧代码目录移走后重新启动容器（数据库不受影响）：
```bash
docker compose down
mv /data/spug/service /data/spug/service.3x
docker compose up -d
docker exec spug python3 /data/spug/spug_api/manage.py updatedb
```
如在 `service/spug_api/spug/overrides.py` 中有自定义配置（如外部数据库、Redis 密码），请复制到新的目录中。
:::

### 手动部署

1. 备份数据库。
2. 安装 Python 3.8 及以上版本并重建虚拟环境：
   ```bash
   cd /data/spug/spug_api
   git fetch origin && git checkout 4.0
   rm -rf venv && python3 -m venv venv
   source venv/bin/activate
   pip install -U pip setuptools
   pip install -r requirements.txt gunicorn mysqlclient -i https://pypi.tuna.tsinghua.edu.cn/simple/
   python manage.py updatedb
   ```
3. 下载 4.0 的前端包 `https://cdn.spug.cc/spug/web_v4.0.0.tar.gz` 解压到 `spug_web` 目录（会生成 `build` 目录）。
4. 确认 `supervisor` 中的 `spug-ws` 使用 `tools/start-ws.sh`（`daphne`）启动，然后 `supervisorctl restart all`。

## 4.x 版本之间更新

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
