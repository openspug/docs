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
如果你现在部署的 `v2.x.x` 的版本（系统设置/关于），请不要通过该升级工具升级 `v3` 版本，否则会造成运行异常！。[#419](https://github.com/openspug/spug/issues/419)
:::

## Docker安装，版本更新
```bash
# 默认更新到最新版本
# spug 是容器名称，也可以替换为自己的容器ID

docker exec -i spug python3 /data/spug/spug_api/manage.py update 

# 更新完成后重启容器
docker restart spug
```

## 手动安装，版本更新
```bash
# 默认更新到最新版本

cd spug_api
source venv/bin/activate
python manage.py update

# 重启服务
supervisorctl restart all
```
