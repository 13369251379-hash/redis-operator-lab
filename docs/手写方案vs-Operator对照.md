# Redis 主从集群：手写 Kustomize vs Redis Operator 对照学习

> 实验日期：2026-08-05
> 环境：本地 Docker Desktop K8s 单节点（4 CPU / 16GB，v1.36.1）
> Operator：OT-CONTAINER-KIT redis-operator v0.25.0（helm 安装）

---

## 1. 实验目标

在本地集群用 **1 个 RedisReplication CR** 复刻 `D:\项目\redis主从集群\overlays\prod\` 手写清单的
「1 Master + 2 Slave + 3 Sentinel（quorum=2）」架构，对比两种做法的差异。

**不动 prod 正式代码，纯学习。** 实验命名空间 `redis-op-lab`，与已有 `redis-ha` 实验隔离。

---

## 2. 一句话结论

> **你手写的 StatefulSet/Service/ConfigMap/PVC，正是 Operator 从一个 CR 里自动生成的全部东西。**
> Operator 多做了一件纯 YAML 做不到的事：**做决策**——谁当 master、故障时怎么切、Service 指向谁。

---

## 3. 架构对照

| 维度 | prod 手写方案（Kustomize） | Operator 方案（RedisReplication CR） |
|---|---|---|
| 声明方式 | 13 个 YAML 文件（3 STS + 3 SVC + 2 ConfigMap + Secret + NodePort…） | **1 个 CR** |
| Redis 节点 | 独立 master STS(1) + slave STS(2) | **1 个 STS `redis-replication`，3 副本**，主从角色动态确定 |
| Sentinel | 独立 sentinel STS(3) | 独立 STS `redis-replication-s`，3 副本 |
| 密码 | `--requirepass`/`--masterauth` 命令行参数 | 环境变量 `REDIS_PASSWORD` + 镜像 entrypoint 写 conf |
| 配置下发 | ConfigMap 挂载 `redis.conf`/`sentinel.conf` | **无 ConfigMap**，entrypoint 从 env 渲染 conf |
| 主从建立 | slave 静态 `--replicaof redis-master` | **operator 查询 Sentinel 动态配置** |
| 故障切换 | 纯 Sentinel 投票 + 手动检查 | Sentinel 检测 + **operator 更新 Service/label** |
| 存储 | 20Gi ESSD | 1Gi local-path（学习调小） |
| 镜像 | `gitlab-docker.racobit.com/.../redis:7.4.3`（同镜像三角色） | redis 用 `opstree/redis`，sentinel 用 `opstree/redis-sentinel`（**专用镜像**） |

### prod 手写资源 → Operator 自动生成资源的映射

| prod 手写文件 | Operator 自动生成的等价物 |
|---|---|
| `redis-master-sts.yaml`（replicas=1） | `redis-replication` STS 中的 1 个 master 角色 |
| `redis-slave-sts.yaml`（replicas=2） | `redis-replication` STS 中的 2 个 slave 角色 |
| `redis-sentinel-sts.yaml`（replicas=3） | `redis-replication-s` STS（3 副本） |
| `redis-master-svc.yaml` / `redis-slave-svc.yaml` | `redis-replication-master` / `redis-replication-replica`（**自动切换指向**） |
| `redis-configmap.yaml` / `sentinel-configmap.yaml` | **不需要**，entrypoint 从 env 渲染 |
| `nodeport.yaml` / `imagepullsecret.yaml` | CR 的 `kubernetesConfig` 配置（本实验未用 NodePort） |
| `secret.yaml` | `redisSecret` 指向外部 Secret（本实验自建 `redis-secret`） |
| init container（sentinel 等 DNS） | **不需要**——operator 用 env 注入 master 信息 |

---

## 4. 关键学习点（本次实验踩出来的）

### 4.1 Operator 有「镜像契约」

这是本实验**最大的坑**，也是最有价值的发现：

```
operator 注入环境变量 ──▶ 镜像 entrypoint 消费 ──▶ 写进 redis.conf / 启动正确进程
```

- redis 节点注入：`REDIS_PASSWORD`、`SERVER_MODE`、`SETUP_MODE`、`REDIS_PORT` 等
- sentinel 注入：`QUORUM`、`MASTER_PASSWORD` 等
- **官方 redis 镜像的 entrypoint 不认这些变量** → requirepass 不生效、sentinel 跑成 redis-server
- 必须用 `quay.io/opstree/redis` 和 `quay.io/opstree/redis-sentinel`（entrypoint 专门消费这些变量）

> prod 手写方案不受此限制，因为密码用命令行参数 `--requirepass` 显式传入。

### 4.2 Helm chart 的 values ≠ CRD 字段

官方文档/Helm chart 是「抽象层」，和 CRD 字段名不一致（实测踩坑）：

| Helm values 写法 | CRD 实际字段 |
|---|---|
| `redisSecret.secretName` / `secretKey` | `redisSecret.name` / `key` |
| `sentinel.enabled: true` | **不存在**，sentinel 块存在即启用 |
| `quorum: 2`（数字） | 必须写成字符串 `"2"`（同 `parallelSyncs` 等） |
| `sentinel.image` | 同名字段，但须指向 opstree sentinel 镜像 |

### 4.3 主从架构的差异

- prod：master、slave 是**两个独立 StatefulSet**，slave 靠静态 `--replicaof` 绑定
- operator：master+slave 是**同一个 STS 的 3 副本**，角色由 operator 查 Sentinel 动态决定
  → 故障切换时不需要重建 STS，operator 把 slave 提升为 master 并重配其他副本

### 4.4 Service 的「决策」能力

operator 生成了 6 个 Service，其中：
- `redis-replication-master`：永远指向**当前 master**
- `redis-replication-replica`：指向所有 slave

故障切换后 operator **自动更新**这些 Service 的端点——这是纯 YAML 手写方案做不到的。

---

## 5. 验证结果

### 5.1 部署状态

```
pod/redis-replication-0   1/1 Running   (master)
pod/redis-replication-1   1/1 Running   (slave)
pod/redis-replication-2   1/1 Running   (slave)
pod/redis-replication-s-0 1/1 Running   (sentinel)
pod/redis-replication-s-1 1/1 Running   (sentinel)
pod/redis-replication-s-2 1/1 Running   (sentinel)
pvc/redis-replication-redis-replication-{0,1,2}  Bound（1Gi standard）
```

### 5.2 主从 + Sentinel

```
redis-replication-0: role=master, connected_slaves=2
redis-replication-1: role=slave,  master_host=10.244.0.34
redis-replication-2: role=slave,  master_host=10.244.0.34
SENTINEL master mymaster: ip=10.244.0.34 port=6379, num-slaves=2, quorum=2
```

### 5.3 读写验证

```
SET lab:key hello-operator  → OK
GET lab:key (slave-1)       → hello-operator   # 复制传播正常
```

### 5.4 故障切换实验（核心验证）

```
11:18:13  kubectl delete pod redis-replication-0（模拟 master 宕机）
11:18:31  新 master 选举完成 = 10.244.0.36（redis-replication-1）  ← ~18 秒 RTO
          redis-replication-master Service 端点 → 10.244.0.36（operator 自动更新）
          redis-replication-replica 端点 → .38, .39
          被删的 redis-replication-0 重建后自动变成新 master 的 slave
          旧数据 lab:key 在新 master 可读（数据连续性 OK）
          新 master 写入 lab:failover → slave 可读（复制 OK）
```

> prod 手写方案的故障切换：同样由 Sentinel 完成（quorum=2），但 **Service 指向不会自动跟随**，
> 需要人工/额外脚本更新 `redis-master-svc` 指向，这是两者的本质差别。

---

## 6. 踩坑记录（照做会省时间）

| # | 现象 | 原因 | 解法 |
|---|---|---|---|
| 1 | CR 应用报 `unknown field` | Helm values 字段名 ≠ CRD | 用 `kubectl get crd -o json` 查真实 schema |
| 2 | `sentinel.enabled` 报错 | CRD 没有该字段 | 删掉，sentinel 块存在即启用 |
| 3 | `quorum: 2` 报 must be string | CRD 类型是 string | 写成 `"2"` |
| 4 | sentinel Pod Running 但 26379 拒连 | 用了官方镜像，entrypoint 跑的是 redis-server | sentinel 用 `opstree/redis-sentinel` |
| 5 | master `requirepass` 为空，`-a` 认证失败 | 官方镜像 entrypoint 不读 `REDIS_PASSWORD` | redis 节点用 `opstree/redis` |
| 6 | 部署初期 3 节点全是 `role:master` | 复制关系由 operator 调谐建立，需要几十秒 | 等 operator reconcile 完成 |

---

## 7. 对 prod 的启示（要不要迁移）

**不建议迁移**（当前状态），理由：

1. prod 手写方案已经过验证，且**与私有镜像 `gitlab...redis:7.4.3` 深度绑定**——该镜像是官方镜像，
   entrypoint 不消费 operator 的环境变量，要迁移得换镜像或改造
2. prod 无分片需求，sentinel 模式的复杂度不高，手写维护成本可接受
3. operator 引入的**镜像契约、CRD 版本升级、Argo CD 双循环**是新的运维负担

**如果未来要迁移**，优先考虑：把私有 redis 镜像按 opstree entrypoint 契约改造（读 env 写 conf），
即可同时保持 prod 镜像版本 + operator 自动运维。

---

## 8. 复现步骤

```bash
# 1. 安装 operator（helm）
helm repo add ot-container-kit https://ot-container-kit.github.io/helm-charts
helm install redis-operator ot-container-kit/redis-operator -n redis-op-lab --create-namespace

# 2. 建密码 Secret
kubectl create secret generic redis-secret -n redis-op-lab \
  --from-literal=redis-password='LabRedis2026'

# 3. 应用 CR
kubectl apply -f ../manifests/redis-replication.yaml

# 4. 验证
kubectl get pods -n redis-op-lab
```

CR 文件见 `../manifests/redis-replication.yaml`（含注释，标注每处与 prod 的对照）。
