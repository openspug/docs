---
id: open-api
slug: /open-api
title: 开放 API
sidebar_position: 10
---

## 介绍

除了页面操作，`Spug` 还开放了一组接口，便于在 `CI` 流程、自研平台或其他自动化脚本中直接触发 `Spug` 的能力：

| 能力 | 接口 | 说明 |
| --- | --- | --- |
| 触发批量执行模板 | `POST /api/apis/exec/<模板ID>/` | 在指定主机上执行 [执行模板](/docs/batch-exec#模板管理) 的命令 |
| 查询批量执行结果 | `GET /api/apis/exec/result/<token>/` | 查询上述执行的状态与输出 |
| 触发流水线 | `POST /api/apis/pipeline/<流水线ID>/` | 触发 [流水线](/docs/pipeline) 执行 |
| 查询流水线结果 | `GET /api/apis/pipeline/result/<token>/` | 查询上述流水线的状态与各节点输出 |
| 获取主机清单 | `GET /api/apis/host/` | 只读的主机列表，可直接作为 `Ansible` 动态 `inventory` |
| 获取应用配置 | `GET /api/apis/config/` | 详见 [配置中心 API](/docs/config-api) |
| 触发自动发布 | `POST /api/apis/deploy/<发布配置ID>/<类型>/` | 详见 [Webhook 自动发布](/docs/deploy-webhook) |

## 鉴权

所有开放接口共用 `系统管理/系统设置/开放服务设置` 中的 **访问凭据**，两种传递方式任选其一：

```shell script
# 方式一：请求头（推荐）
curl -H "X-Api-Key: JLV8IGO0DhoxcM7I" https://spug.example.com/api/apis/host/

# 方式二：查询参数
curl "https://spug.example.com/api/apis/host/?apiKey=JLV8IGO0DhoxcM7I"
```

凭据错误或未配置时返回 `401`，响应体统一为 `{"data": ..., "error": null}` 结构，出错时 `error` 为错误描述。

:::caution 注意
访问凭据等同于一把可以在你全部主机上执行命令的钥匙，**请勿写进公开仓库或前端代码**，建议放在 CI 的加密变量中，并定期更换。
:::

## 触发批量执行模板

- 请求地址：`/api/apis/exec/<模板ID>/`
- 请求方法：`POST`
- 模板 ID 在 `批量执行/模板管理` 中编辑模板时的地址栏或列表中查看
- 请求参数（`JSON` body，可整体省略）：

    | 参数名 | 类型 | 必填 | 说明 |
    | --- | :---: | :---: | --- |
    | host_ids | array | 否 | 指定执行的主机 ID 列表，省略时使用模板中配置的默认主机；模板也没有配置默认主机时返回 `400` |
    | params | object | 否 | 模板中定义的参数，键为参数的变量名。未传且参数有默认值时使用默认值，必填参数缺失返回 `400` |

- 返回：`202` 与一个 `token`，用于查询本次执行的结果

```shell script
curl -X POST -H "X-Api-Key: JLV8IGO0DhoxcM7I" -H "Content-Type: application/json" \
  -d '{"host_ids": [1, 2], "params": {"who": "nolan"}}' \
  https://spug.example.com/api/apis/exec/1/
```

```json
{"data": {"token": "8a35cc6f1a0043bab1964148c977176b"}, "error": null}
```

## 查询批量执行结果

- 请求地址：`/api/apis/exec/result/<token>/`
- 请求方法：`GET`
- 结果保留 1 小时，过期或 `token` 不存在返回 `404`

```shell script
curl -H "X-Api-Key: JLV8IGO0DhoxcM7I" \
  https://spug.example.com/api/apis/exec/result/8a35cc6f1a0043bab1964148c977176b/
```

```json
{
  "data": {
    "status": "running",
    "hosts": [
      {
        "id": 1,
        "title": "web-01(10.0.0.11:22)",
        "status": "success",
        "exit_code": 0,
        "output": "hello from web-01\r\n"
      },
      {
        "id": 2,
        "title": "web-02(10.0.0.12:22)",
        "status": "running",
        "exit_code": null,
        "output": ""
      }
    ]
  },
  "error": null
}
```

顶层 `status` 为本次执行的汇总状态：只要还有主机在执行即为 `running`，全部结束且无失败为 `success`，否则为 `failed`。单台主机的 `status` 同样是这三个值，`exit_code` 为命令的退出码（`Spug` 自身的异常使用 `130` 超时、`131` 异常）。

一个典型的轮询用法：

```bash
TOKEN=$(curl -s -X POST -H "X-Api-Key: $SPUG_API_KEY" -H "Content-Type: application/json" \
  -d '{"params": {"version": "v1.2.3"}}' "$SPUG_URL/api/apis/exec/1/" | jq -r .data.token)

while true; do
  STATUS=$(curl -s -H "X-Api-Key: $SPUG_API_KEY" "$SPUG_URL/api/apis/exec/result/$TOKEN/" | jq -r .data.status)
  [ "$STATUS" = "running" ] || break
  sleep 3
done
echo "执行结果：$STATUS"
[ "$STATUS" = "success" ] || exit 1
```

## 触发流水线

- 请求地址：`/api/apis/pipeline/<流水线ID>/`
- 请求方法：`POST`
- 请求参数（`JSON` body，可整体省略）：

    | 参数名 | 类型 | 必填 | 说明 |
    | --- | :---: | :---: | --- |
    | params | object | 否 | 流水线 **参数节点** 中定义的动态参数，以及构建节点需要的 `_spug_git_tag` / `_spug_git_commit`。值为数组时会自动用 `,` 连接。必填参数缺失返回 `400` |

- 返回：`202` 与一个 `token`

```shell script
curl -X POST -H "X-Api-Key: JLV8IGO0DhoxcM7I" -H "Content-Type: application/json" \
  -d '{"params": {"ver": "v1.2.3"}}' \
  https://spug.example.com/api/apis/pipeline/11/
```

:::info 说明
包含 **数据上传** 节点的流水线无法通过接口触发（该节点依赖页面上传文件），调用时会返回 `400`。接口触发的流水线执行记录归属于该流水线的创建人。
:::

## 查询流水线结果

- 请求地址：`/api/apis/pipeline/result/<token>/`
- 请求方法：`GET`
- 结果保留 1 小时

```json
{
  "data": {
    "pipeline_id": 11,
    "status": "success",
    "nodes": {
      "p1": {"status": "success", "output": "解析参数配置 ..."},
      "s1": {"status": "success", "output": ""},
      "s1.1": {"status": "success", "output": "开始执行 ... 执行结束"}
    }
  },
  "error": null
}
```

顶层 `status` 为 `running` / `success` / `failed`。`nodes` 的键为节点 ID，形如 `节点ID.主机ID` 的键是该节点在具体主机上的输出，节点自身的 `status` 取值与页面一致（`processing` / `success` / `error`）。

## 获取主机清单

- 请求地址：`/api/apis/host/`
- 请求方法：`GET`
- 请求参数：

    | 参数名 | 类型 | 必填 | 默认值 | 说明 |
    | --- | :---: | :---: | --- | --- |
    | format | string | 否 | ansible | `ansible` 输出 `Ansible` 动态 `inventory` 结构，`json` 输出主机列表数组 |

接口为 **只读**，仅包含主机名称、连接地址、端口、用户名、分组以及主机管理中采集到的资产信息，**不包含任何密钥、密码等凭据**。

```shell script
curl -H "X-Api-Key: JLV8IGO0DhoxcM7I" https://spug.example.com/api/apis/host/
```

```json
{
  "_meta": {
    "hostvars": {
      "web-01": {
        "ansible_host": "10.0.0.11",
        "ansible_port": 22,
        "ansible_user": "root",
        "spug_id": 1,
        "spug_groups": ["web"],
        "spug_os_name": "CentOS 7.9",
        "spug_private_ip_address": ["10.0.0.11"]
      }
    }
  },
  "生产": {"hosts": [], "children": ["web"]},
  "web": {"hosts": ["web-01"], "children": []},
  "all": {"children": ["ungrouped", "生产"]},
  "ungrouped": {"hosts": []}
}
```

### 作为 Ansible 动态 inventory

新建一个可执行脚本，把接口返回原样输出即可：

```bash title="spug_inventory.sh"
#!/bin/sh
case "$1" in
  --host) echo '{}' ;;
  *) curl -sf -H "X-Api-Key: ${SPUG_API_KEY}" "${SPUG_URL}/api/apis/host/" ;;
esac
```

```shell script
chmod +x spug_inventory.sh
export SPUG_API_KEY=JLV8IGO0DhoxcM7I SPUG_URL=https://spug.example.com

ansible-inventory -i ./spug_inventory.sh --graph
ansible -i ./spug_inventory.sh web -m ping
ansible-playbook -i ./spug_inventory.sh deploy.yml --limit 生产
```

主机分组的层级关系会映射为 `Ansible` 的组与子组，未分组的主机归入 `ungrouped`。主机名在 `Spug` 中可以重复而 `Ansible` 的组名不行，因此同名的分组会自动追加分组 ID 加以区分（如 `web_6`）。

:::tip 提示
`Ansible` 只接受字母、数字和下划线组成的组名，分组名为中文或含有 `-` 时会提示 `Invalid characters were found in group names`，这只是告警，定位分组依然可用。可在 `ansible.cfg` 中设置 `transform_invalid_group_chars = ignore` 关闭该提示。

`Ansible` 连接主机使用的是你本机的 `SSH` 密钥，`Spug` 不会下发自己的密钥，必要时通过 `--private-key` 指定。
:::
