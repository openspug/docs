---
id: system-account
slug: /system-account
title: 账户管理
sidebar_position: 3
---

## 介绍

管理登录平台的账户，账户通过关联一个或多个 [角色](/docs/system-role) 获得权限，管理员账户不受权限限制。

![账户管理](/img/docs/4.0/zh/account-form.jpg)

## 配置项

- **登录名**：登录使用的用户名，全局唯一。
- **姓名**：展示在页面右上角的昵称。
- **密码**：至少 8 位且包含数字、小写和大写字母，仅新建时设置。
- **角色**：可选择多个角色，遵循权限最大化原则，组合多个角色的权限。
- **微信Token**：用于微信报警与登录 MFA 验证码，详见 [微信 Token](/docs/wx-token)，用户也可以在个人中心自行设置。

## 操作说明

- **编辑**：修改姓名、角色、微信 Token；**重置密码** 为账户设置新密码。
- **禁用 / 启用**：连续 3 次登录失败的账户会自动禁用，可在此重新启用；LDAP 账户不允许删除，可以使用禁用代替。
- **删除**：删除账户，已删除账户创建的数据仍会保留。
- LDAP 用户可以在 `系统设置 / LDAP设置` 中批量导入，详见 [系统设置](/docs/system-set#ldap)。

## 命令行管理

除了页面上对普通用户的管理，`Spug` 还提供了 `manage.py user` 命令可用于管理员账户的管理操作。

### 创建账户

创建账户使用 `manage.py user add` 命令，用法示例如下
```bash
cd spug/spug_api
source venv/bin/activate
python manage.py user add -u admin -p 123 -n 张三丰 -s
```
Docker 安装的可以执行如下命令
```bash
docker exec spug python3 /data/spug/spug_api/manage.py user add -u admin -p 123 -n 张三丰 -s
```
以上命令会创建个登录名为 `admin` 密码为 `123` 昵称为 `张三丰` 的管理员账户，注意最后的 `-s` 参数，如果携带了这个参数意味着该账户为管理员账户，
管理员账户可以不受任何限制的访问所有功能模块。

### 重置密码

使用 `manage.py user reset` 命令来重置账户密码，用法示例如下
```bash
cd spug/spug_api
source venv/bin/activate
python manage.py user reset -u admin -p abc
```
Docker 安装的可以执行如下命令
```bash
docker exec spug python3 /data/spug/spug_api/manage.py user reset -u admin -p abc
```
上述操作会重置登录名为 `admin` 的账户的密码为 `abc`。

### 启用账户

当页面上登录连续错误数次超过 3 次后账户自动转为禁用状态，普通用户可以通过 `系统管理 / 账户管理` 在页面上启用账户即可，但管理员账户需要使用如下命令来启用
```bash
cd spug/spug_api
source venv/bin/activate
python manage.py user enable -u admin
```
Docker 安装的可以执行如下命令
```bash
docker exec spug python3 /data/spug/spug_api/manage.py user enable -u admin
```
