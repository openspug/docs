---
id: deploy-rollback
slug: /deploy-rollback
title: 回滚机制
---

## 介绍
发布目前支持两种方式 `常规发布` 和 `自定义发布`，这里来分别讲解下这两种发布方式的回滚机制。

## 常规发布
常规发布由发布配置中的 `保留历史版本数量` 来决定是否可以回滚，超过保留上限的版本将无法执行回滚操作。回滚将经历以下几个步骤：
1. 当用户在发布申请中点击 `回滚` 按钮时，将会自动基于本次发布的上个版本（例如：有 `v1.1.3` `v1.1.2` `v1.1.1` 三个发布记录，当点击 `v1.1.3`
的回滚按钮时将基于 `v1.1.2`，同理点击`v1.1.2` 时将基于 `v1.1.1`）创建新的发布申请。
2. 找到第一步所创建的带有 `回滚` 字样标题的发布申请，如果开启了审核则需要先通过审核，否则可直接点击发布按钮。
3. 执行回滚发布，**回滚发布将跳过检出前后动作，直接执行发布前和发布后任务并使用存储的历史版本发布**。

如果你想控制回滚时行为执行不同的操作，可以在 `发布前任务` 和 `发布后任务` 中使用内置的全局变量 `SPUG_DEPLOY_TYPE`， 通过判断其值来做不同的操作。


## 自定义发布
因为自定义发布更加的灵活自由，`Spug` 也无法对回滚进行任何限制或干涉，完全依靠用户来控制。回滚将经历以下几个步骤：
1. 与常规发布相同，当用户在发布申请中点击 `回滚` 按钮时，将会自动基于本次发布的上个版本（例如：有 `v1.1.3` `v1.1.2` `v1.1.1` 三个发布记录，当点击 `v1.1.3`
的回滚按钮时将基于 `v1.1.2`，同理点击`v1.1.2` 时将基于 `v1.1.1`）创建新的发布申请。
2. 找到第一步所创建的带有 `回滚` 字样标题的发布申请，如果开启了审核则需要先通过审核，否则可直接点击发布按钮。
3. 执行回滚发布，默认行为与发布一致，会根据定义的顺序完整执行所有动作。

如果你想控制回滚时行为执行不同的操作，在自定义的动作中使用内置的全局变量 `SPUG_DEPLOY_TYPE`， 通过判断其值来做不同的操作。


## 控制行为
内置的全局变量 `SPUG_DEPLOY_TYPE` 可用于区分回滚和正常发布，`1` 为正常发布， `2` 为回滚发布。 用户可通过判断其值来实现不同情况下的不同操作，
例如：

```shell script
if [ "$SPUG_DEPLOY_TYPE" = "1" ]; then
    echo "执行正常发布的操作，巴啦啦..."
else
    echo "这里只有回滚时才会执行"
fi
```