---
id: install-k8s
slug: /install-k8s
title: Kubernetes 部署
sidebar_position: 2
---

如果你已经有 Kubernetes 集群，可以直接使用官方 Docker 镜像 `openspug/spug-service` 把 `Spug` 部署到集群中。Kubernetes 部署与 [Docker 安装](/docs/install-docker) 使用同一个镜像，镜像内已内置 Nginx、Redis 以及全部后台服务，数据库使用独立的 MariaDB，官方提供了一份完整的部署清单，`kubectl apply` 一条命令即可完成部署。

## 依赖环境

- Kubernetes 1.19 及以上，本机已安装 `kubectl` 并能访问集群
- 集群有可用的默认 `StorageClass`（清单会创建 3 个 `PersistentVolumeClaim`），可通过 `kubectl get storageclass` 查看
- 集群节点可以拉取 Docker Hub 镜像，Pod 可以访问 `gitee.com` 与 `cdn.spug.cc`（首次启动拉取代码与前端包）
- 命名空间允许运行特权容器（`privileged: true`，文件分发与流水线的 `sshfs` 挂载需要），Pod Security 标准不能是 `restricted`

:::info 支持的 CPU 架构
`openspug/spug-service:4.0.0` 镜像同时提供 `amd64` 与 `arm64` 架构，ARM64 节点可直接使用，暂不支持 32 位 ARM（armv7）。阿里云镜像仓库没有 4.0 镜像，请不要替换清单中的镜像地址，拉取 Docker Hub 困难时请为节点的容器运行时配置镜像加速器。
:::

## 部署架构

清单在 `spug` 命名空间中创建以下资源：

| 资源 | 名称 | 说明 |
| --- | --- | --- |
| Secret | `spug-db` | 数据库名、账号与密码，`spug-db` 与 `spug` 两个 Deployment 共用 |
| PersistentVolumeClaim | `spug-db-data` | MariaDB 数据目录（10Gi） |
| PersistentVolumeClaim | `spug-service` | Spug 程序目录，对应容器内 `/data/spug`（10Gi） |
| PersistentVolumeClaim | `spug-repos` | 常规发布的代码仓库与构建产物，对应容器内 `/data/repos`（20Gi） |
| Deployment / Service | `spug-db` | MariaDB 10.8，集群内通过 `spug-db:3306` 访问 |
| Deployment | `spug` | Spug 主服务，单副本，特权容器 |
| Service | `spug` | `NodePort` 类型，默认映射节点的 `30080` 端口 |

## 快速部署

清单文件托管在官网，三条命令即可完成部署与初始化：

```bash
# 1. 部署全部资源
kubectl apply -f https://ops.spug.cc/k8s/spug.yaml

# 2. 等待 Pod 就绪（首次启动需要拉取代码与前端包，通常 1～3 分钟）
kubectl -n spug rollout status deploy/spug

# 3. 初始化数据库并创建管理员账户（用户名 admin、密码 spug.cc，可自行替换）
kubectl -n spug exec deploy/spug -- init_spug admin spug.cc
```

完成后在浏览器中打开 `http://<任意节点IP>:30080` 即可登录。生产环境请先按下面的说明修改数据库密码与存储配置，再执行部署。

## 清单文件说明

以下为 `https://ops.spug.cc/k8s/spug.yaml` 的完整内容，可以保存到本地按需修改后再 `kubectl apply -f spug.yaml`：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: spug
---
# 数据库账号（请修改默认密码）
apiVersion: v1
kind: Secret
metadata:
  name: spug-db
  namespace: spug
type: Opaque
stringData:
  MYSQL_DATABASE: spug
  MYSQL_USER: spug
  MYSQL_PASSWORD: spug.cc
  MYSQL_ROOT_PASSWORD: spug.cc
---
# 数据库数据
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: spug-db-data
  namespace: spug
spec:
  accessModes:
    - ReadWriteOnce
  # storageClassName: <your-storageclass>
  resources:
    requests:
      storage: 10Gi
---
# Spug 程序目录（容器内 /data/spug）
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: spug-service
  namespace: spug
spec:
  accessModes:
    - ReadWriteOnce
  # storageClassName: <your-storageclass>
  resources:
    requests:
      storage: 10Gi
---
# 发布用代码仓库与构建产物（容器内 /data/repos）
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: spug-repos
  namespace: spug
spec:
  accessModes:
    - ReadWriteOnce
  # storageClassName: <your-storageclass>
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spug-db
  namespace: spug
  labels:
    app: spug-db
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: spug-db
  template:
    metadata:
      labels:
        app: spug-db
    spec:
      containers:
        - name: mariadb
          image: mariadb:10.8
          args:
            - --character-set-server=utf8mb4
            - --collation-server=utf8mb4_unicode_ci
          envFrom:
            - secretRef:
                name: spug-db
          ports:
            - name: mysql
              containerPort: 3306
          readinessProbe:
            tcpSocket:
              port: mysql
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            tcpSocket:
              port: mysql
            initialDelaySeconds: 60
            periodSeconds: 20
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: spug-db-data
---
apiVersion: v1
kind: Service
metadata:
  name: spug-db
  namespace: spug
spec:
  selector:
    app: spug-db
  ports:
    - name: mysql
      port: 3306
      targetPort: mysql
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spug
  namespace: spug
  labels:
    app: spug
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: spug
  template:
    metadata:
      labels:
        app: spug
    spec:
      containers:
        - name: spug
          image: openspug/spug-service:4.0.0
          securityContext:
            privileged: true
          env:
            # 首次启动拉取的代码版本，必须与镜像版本配套
            - name: SPUG_DOCKER_VERSION
              value: v4.0.0
            - name: MYSQL_HOST
              value: spug-db
            - name: MYSQL_PORT
              value: "3306"
          envFrom:
            - secretRef:
                name: spug-db
          ports:
            - name: http
              containerPort: 80
          # 首次启动需要拉取代码与前端包，给 10 分钟
          startupProbe:
            httpGet:
              path: /
              port: http
            periodSeconds: 10
            failureThreshold: 60
          readinessProbe:
            httpGet:
              path: /
              port: http
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: http
            periodSeconds: 30
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
          volumeMounts:
            - name: service
              mountPath: /data/spug
            - name: repos
              mountPath: /data/repos
      volumes:
        - name: service
          persistentVolumeClaim:
            claimName: spug-service
        - name: repos
          persistentVolumeClaim:
            claimName: spug-repos
---
# 默认通过 NodePort 30080 访问；使用 Ingress 时可改为 ClusterIP
apiVersion: v1
kind: Service
metadata:
  name: spug
  namespace: spug
spec:
  type: NodePort
  selector:
    app: spug
  ports:
    - name: http
      port: 80
      targetPort: http
      nodePort: 30080
```

需要关注的配置：

| 配置 | 说明 |
| --- | --- |
| `Secret spug-db` | 数据库账号密码，生产环境请修改 `MYSQL_PASSWORD` 与 `MYSQL_ROOT_PASSWORD`；也可以删除 `spug-db` 的 Deployment / Service，把 `MYSQL_HOST` / `MYSQL_PORT` 改为外部 MySQL 地址，[参考文档](/docs/install-problem#use-mysql) |
| `storageClassName` | 三个 PVC 默认使用集群的默认 `StorageClass`，没有默认 `StorageClass` 时请取消注释并填写；`spug-service` 与 `spug-repos` 中的数据丢失后需要重新初始化，请务必使用持久化存储 |
| `SPUG_DOCKER_VERSION` | 首次启动时拉取的代码版本（git 标签），必须与镜像版本对应，`4.0.0` 镜像请填写 `v4.0.0`；该变量没有默认值，缺少时容器会在拉取代码阶段退出并反复重启 |
| `privileged: true` | 文件分发与流水线数据传输依赖 `sshfs` 挂载，需要特权容器；如集群策略不允许，可以删除该项，但对应功能将不可用 |
| `nodePort: 30080` | 访问端口，可改为 `30000～32767` 之间的其他端口；使用 Ingress 时可把 `type` 改为 `ClusterIP` 并删除 `nodePort` |
| `resources` | 清单只设置了资源请求，请按集群情况补充 `limits`，建议至少 1 核 2G 内存 |
| `replicas: 1` | 镜像内置 Redis，任务计划、监控等后台服务为单实例，`Spug` 不支持多副本，请勿调大 |

## 部署步骤

### 1. 部署资源

```bash
kubectl apply -f https://ops.spug.cc/k8s/spug.yaml
```

如果修改了清单，则改为 `kubectl apply -f spug.yaml`。执行后可以查看 Pod 状态：

```bash
kubectl -n spug get pod,pvc,svc
```

### 2. 等待启动完成

`spug-db` 首次启动会初始化数据库，`spug` 首次启动会从 `gitee.com` 克隆 `SPUG_DOCKER_VERSION` 指定版本的代码、从 `cdn.spug.cc` 下载前端包并生成配置文件 `spug_api/spug/overrides.py`，之后才会启动各服务，整个过程通常需要 1～3 分钟，清单中的 `startupProbe` 允许最长 10 分钟。

```bash
# 等待 spug Deployment 就绪
kubectl -n spug rollout status deploy/spug

# 查看启动日志
kubectl -n spug logs -f deploy/spug
```

:::info 首次启动需要访问外网
`4.0.0` 镜像不内置代码，Pod 首次启动时需要访问 `gitee.com` 与 `cdn.spug.cc`，内网 / 离线集群无法直接完成首次启动。如果 Pod 一直处于 `CrashLoopBackOff`，请查看日志确认是否为拉取代码失败或缺少 `SPUG_DOCKER_VERSION`。
:::

### 3. 初始化

Pod 就绪后执行以下命令初始化数据库表结构，并创建一个用户名为 `admin`、密码为 `spug.cc` 的管理员账户，可自行替换账户 / 密码。

```bash
kubectl -n spug exec deploy/spug -- init_spug admin spug.cc
```

:::tip 提示
如果报错 `can't open file '/data/spug/spug_api/manage.py'`（代码尚未拉取完成）或 `Can't connect to MySQL server on 'spug-db'`（数据库还未就绪）导致初始化失败，等待几秒后重新执行即可。
:::

### 4. 访问测试

在浏览器中输入 `http://<任意节点IP>:30080` 访问（默认账户密码在第 3 步初始化时设置），页面右上角可切换中英文界面。如果节点有安全组 / 防火墙，请放行 `30080` 端口。

:::tip 登录后提示未能获取到访问者的真实 IP
通过 `NodePort` 访问时，Pod 看到的来源 IP 是节点转换后的地址，`Spug` 会提示无法获取真实 IP，内网使用可以忽略，或在 `系统管理 / 系统设置 / 安全设置` 中关闭 **访问IP校验**。需要真实 IP 请通过下面的 Ingress 方式访问，或给 `spug` Service 设置 `externalTrafficPolicy: Local`。
:::

## 通过 Ingress 访问 {#ingress}

生产环境推荐通过 Ingress 用域名访问。`Spug` 的 Web 终端、发布控制台等功能使用 WebSocket 长连接，Ingress 需要支持 WebSocket 并适当加大超时时间，以下为 `ingress-nginx` 的示例（文件地址 `https://ops.spug.cc/k8s/spug-ingress.yaml`）：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: spug
  namespace: spug
  annotations:
    # 允许上传大文件
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    # Web 终端 / 发布控制台使用 WebSocket 长连接，避免空闲 60 秒被断开
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
spec:
  ingressClassName: nginx
  rules:
    - host: spug.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: spug
                port:
                  name: http
  # 启用 HTTPS 时取消注释并准备好证书 Secret
  # tls:
  #   - hosts:
  #       - spug.example.com
  #     secretName: spug-tls
```

把 `spug.example.com` 改为你的域名后执行 `kubectl apply -f spug-ingress.yaml`，Ingress 生效后可以把 `spug` Service 的 `type` 改为 `ClusterIP` 不再暴露 NodePort。使用其他 Ingress 控制器（Traefik、云厂商负载均衡等）时，请确保其转发 WebSocket 并传递 `X-Forwarded-For` 请求头，[参考文档](/docs/use-problem#websocket)。

## 常用运维命令

```bash
# 查看 Pod 状态
kubectl -n spug get pod

# 查看 Spug 日志（容器内各服务日志在 /data/spug/spug_api/logs）
kubectl -n spug logs -f deploy/spug

# 进入容器
kubectl -n spug exec -it deploy/spug -- bash

# 重启 Spug（数据在 PVC 中，重启不会丢失）
kubectl -n spug rollout restart deploy/spug

# 备份数据库
kubectl -n spug exec deploy/spug-db -- mysqldump -uroot -pspug.cc spug > spug-backup.sql
```

容器内的目录约定与 Docker 安装相同，请参考 [Docker 安装](/docs/install-docker#ops-commands)。

## 版本升级

4.x 之间的小版本更新在容器内执行更新命令后重启 Deployment 即可，代码目录在 `spug-service` PVC 中，Pod 重建后更新仍然保留：

```bash
kubectl -n spug exec deploy/spug -- python3 /data/spug/spug_api/manage.py update
kubectl -n spug rollout restart deploy/spug
```

发布新的镜像版本时，请同时修改清单中的 `image` 标签与 `SPUG_DOCKER_VERSION` 后重新 `kubectl apply`，并按 [版本升级文档](/docs/update-version) 中 Docker 安装的步骤在容器内切换代码（把 `docker exec spug` 替换为 `kubectl -n spug exec deploy/spug --` 即可）。

## 卸载

```bash
kubectl delete -f https://ops.spug.cc/k8s/spug.yaml
```

:::caution 注意
删除清单会连同 `spug` 命名空间与其中的 PVC 一起删除，数据库与代码目录中的数据将随之丢失（取决于 `StorageClass` 的回收策略），卸载前请先备份数据库。
:::

## 安全建议

- 修改 `Secret spug-db` 中的数据库默认密码，不要把 `NodePort` 直接暴露在公网，如必须公网访问请通过 Ingress 启用 HTTPS，并在 `系统管理 / 系统设置 / 安全设置` 中开启登录 MFA。
- `spug` Pod 需要通过 SSH 访问被管理的主机，请确保集群到目标主机的网络策略放行 22 端口（或主机自定义的 SSH 端口）。
