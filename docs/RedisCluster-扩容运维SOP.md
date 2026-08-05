# RedisCluster 扩容运维 SOP（第 4 弹实验）

> 实验日期：2026-08-05
> 环境：Docker Desktop K8s 单节点（4 CPU/16GB，context=docker-desktop），`redis-op-lab` 命名空间
> operator：quay.io/opstree/redis-operator:v0.25.0，CR：`../manifests/redis-cluster.yaml`
> 前置：已读第 2 弹 `RedisCluster-实验记录.md`（本 SOP 是其"扩容运维"主题的完整版）

---

## 1. 一句话结论

> **v0.25.0 的 RedisCluster 扩/缩容绝大部分是自动的**（加分片=add-node+rebalance、减分片=迁槽+del-node），
> 但**operator 对三种"扩容卡住"没有自愈能力，全部需要人工介入**：
> ① 死节点顶高 master 计数；② 节点资源不足新 Pod 无法调度；③ 新节点残留数据导致 add-node 被拒。
> 本 SOP 就是这三类故障的排查与处置流程。

---

## 2. 前置认知：operator 到底怎么"数"节点（源码级，决定一切）

RedisCluster 的扩缩容判断全部依赖 `CheckRedisNodeCount`，它的实现决定了 operator 的边界：

```go
// internal/k8sutils/redis.go
func nodeIsOfType(node clusterNodesResponse, nodeType string) bool {
    return strings.Contains(node[2], nodeType)   // flags 子串匹配
}
```

**关键结论：`master,fail` 死节点的 flags 含 `master` 子串 → 被计为 master。**

- `Leaders.Count`（= `CheckRedisNodeCount("leader")`）= 活主 + 死主
- 集群内节点数 `CheckRedisNodeCount("")` = CLUSTER NODES 里所有行（含死节点、fail 节点）
- 扩/缩容分支：
  - `Leaders.Count == desired` → 走"加 follower / 稳定检查 / rebalance"正常路径
  - `Leaders.Count < desired` → `add-node`（一次加一个）→ 收敛后再 rebalance 分槽
  - `Leaders.Count > desired`（死节点导致）→ **死循环，什么都不做** ← 卡死根因

### operator 自动化的扩缩容流程（v0.25.0）

| 操作 | operator 自动执行序列 |
|---|---|
| 加分片 `clusterSize↑` | `redis-cli --cluster fix` → `add-node`（新 leader 入集群）→ 空 master 存在时 `rebalance --cluster-use-empty-masters`（渐进迁移，最终每 master 槽位均衡）→ 挂 follower |
| 减分片 `clusterSize↓` | 前置校验转移目标均为 master（不是则 failover）→ 移除该分片全部 follower → `reshard` 迁槽（round-robin：`shardIdx % 剩余master数` 选目标）→ `del-node` → `rebalance` |
| 加副本 `follower.replicas↑` | 集群健康时新 follower Pod 就绪后自动 `cluster meet + replicate` 挂载 |

> 实测：加分片从 3→4，最终 4 个 master **各自恰好 4096 槽**（16384/4），数据跨分片自动重分布。
> 注意 rebalance 是**渐进迁移**，中间态槽位不均属正常，最终会收敛（本地约 1~3 分钟）。

---

## 3. 扩容前检查清单（三查）

**在做任何扩容/缩容操作前，先执行：**

```bash
PASS=<redis-password>
# ① 集群状态健康
kubectl exec -n redis-op-lab redis-cluster-leader-0 -- redis-cli -a $PASS --no-auth-warning CLUSTER INFO | grep -E "cluster_state|slots_fail"
# 期望: cluster_state:ok   cluster_slots_fail:0

# ② 无死节点（fail/pfail 行数应为 0）
kubectl exec -n redis-op-lab redis-cluster-leader-0 -- redis-cli -a $PASS --no-auth-warning CLUSTER NODES | grep -cE "fail|disconnected"

# ③ 节点可调度资源充足（扩的 Pod 数 × 单 Pod request CPU ≤ 节点剩余）
kubectl describe node <node> | grep -A5 "Allocatable"
# 本实验环境：节点 4 CPU，kube-system 已占 ~950m，operator 500m，
#   每 Redis Pod 请求 200m → 10 Pod=2.0 CPU 是上限附近，11+ Pod 就会 Pending
```

**三个红灯 → 分别对应三类卡住：**
- ② 有死节点 → 见 §5.1 FORGET
- ③ 资源不足 → 先腾资源或降低扩容目标（新 Pod 会 Pending，集群状态卡在 Bootstrap/不 Ready）
- ①②③ 全绿仍卡 → 见 §5.3 残留数据

---

## 4. 标准操作流程

### 4.1 水平扩容（加副本）：改 `redisFollower.replicas`

```bash
kubectl patch rediscluster redis-cluster -n redis-op-lab --type=json \
  -p '[{"op":"replace","path":"/spec/redisFollower/replicas","value":6}]'   # 3 → 6
```

- 预期：新 follower Pod 逐序创建（STS 顺序），operator 自动挂载到对应分片
- 验证：`CLUSTER NODES` 里新 Pod 以 `slave` 角色出现，known_nodes 相应增加
- 实测：3→6 成功后每分片 1 主 2 从，自动完成，无人工干预

### 4.2 横向扩容（加分片）：改 `clusterSize` + `redisLeader.replicas`

```bash
kubectl patch rediscluster redis-cluster -n redis-op-lab --type=json \
  -p '[{"op":"replace","path":"/spec/clusterSize","value":4},
       {"op":"replace","path":"/spec/redisLeader/replicas","value":4}]'   # 3 → 4
```

- 预期：新 leader Pod 创建 → `add-node` 入集群 → 自动 rebalance 分槽（新分片从 0 槽渐增至 4096）→ follower 挂载
- 验证：最终每个 master 槽位约 16384/clusterSize；新分片可读写
- **观察点**：rebalance 迁移期间槽位不均属正常，等 1~3 分钟看最终收敛

### 4.3 缩容（减分片）：改 `clusterSize` + `redisLeader.replicas`

```bash
kubectl patch rediscluster redis-cluster -n redis-op-lab --type=json \
  -p '[{"op":"replace","path":"/spec/clusterSize","value":3},
       {"op":"replace","path":"/spec/redisLeader/replicas","value":3}]'   # 4 → 3
```

- 预期流程（operator 自动）：校验剩余 master 均为转移目标 → 移除被删分片的所有 follower → 迁槽 → del-node → rebalance → STS 缩副本
- 验证：`CLUSTER NODES` 死节点与额外 master 消失、槽位仍 16384 全覆盖、**迁移 key 全部可读**
- **前置条件（硬性）**：`CheckRedisNodeCount("leader") == leader STS 副本数`，否则直接 skip downscale
  - 若此前发生过 failover 把某个 leader pod 降成 slave，master 数与 STS 副本数不相等 → **缩容不触发**（见 §5.4）

---

## 5. 故障处置 SOP

### 5.1 死节点清理（最常遇到，故障切换后的标配）

**症状**：故障切换后 `CLUSTER NODES` 里旧节点保留为 `master,fail`；此时任何扩容都卡在
operator 日志 `Not all leader are part of the cluster... Leaders.Count: N > Instance.Size: M`。

**处置**：在**所有存活 master** 上 FORGET 死节点：

```bash
PASS=<redis-password>
DEAD=<死节点ID，从 CLUSTER NODES 的 fail 行取第 1 列>
# 找出所有存活 master 的 Pod（这里以 4 主为例，按实际填）
kubectl exec -n redis-op-lab <master-pod> -- redis-cli -a $PASS --no-auth-warning CLUSTER FORGET $DEAD
# 每个 master 都要执行一次
```

FORGET 后 operator 下一轮 reconcile 即恢复（`Leaders.Count` 回落 == desired → 继续挂载 follower）。

**为什么必须人工**：operator 的 `UnhealthyNodesInCluster`/`RepairDisconnectedMasters` 只处理
**master 失联/断连**，从**不执行 `CLUSTER FORGET`**。死节点是 Redis Cluster 特性，operator 选择不做。

### 5.2 资源不足导致扩容卡住

**症状**：新扩容的 Pod `Pending`，`kubectl describe pod` 见 `0/1 nodes are available: 1 Insufficient cpu`；
集群状态长期不 Ready（如 Bootstrap）。

**处置**：
1. 核算：`kubectl describe node` 的 Allocatable vs 已有请求（扩容数 × 单 Pod request）
2. 腾资源（缩其他负载/临时删除已结束实验的组件）或 **降低扩容目标到节点可承载量**
3. 释放后 Pod 自动调度，operator 继续收敛

> 本实验实测：4 CPU 节点上 RedisCluster 最多约 10 Pod（含 operator 500m）就贴顶，11+ Pod 必 Pending。

### 5.3 新节点残留数据 → add-node 被拒

**症状**：operator 日志 `[ERR] Node <new-pod> is not empty. Either the node already knows other
nodes ... or contains some key in database 0`；`add-node` 反复失败，状态卡 Bootstrap。

**成因**：缩容时被删的 pod 其 **PVC 未清**。若该 pod 之前是 master 且其 key 未及迁移（缩容中断），
扩容重建同一序号 pod 会挂回旧 PVC → redis 加载残留 key → `add-node` 拒绝"非空"节点。

**处置**：确认残留 key 已另有副本（先验证数据可读）后清空：

```bash
kubectl exec -n redis-op-lab <new-leader-pod> -- redis-cli -a $PASS --no-auth-warning FLUSHALL
```

清空后 operator 下一轮 reconcile 的 `add-node` 即成功。
> 更稳妥的生产做法：缩容前确认"先迁槽、再 del-node、再删 STS"，避免 pod 带数据消失留下脏 PVC。

### 5.4 缩容不触发（leader 角色错乱）

**症状**：改小 clusterSize 后 operator 日志 `masterCount is not equal to leader statefulset replicas, skip downscale`。

**成因**：缩容前置要求 `CheckRedisNodeCount("leader") == leader STS 副本数`。若此前 failover 把某 leader
pod 降成 slave、而 follower pod 反而成了 master，master 数与 STS 副本数不匹配 → 缩容分支永不进入。

**处置**：人工把角色理顺——把"应该是 leader pod 却还是 slave"的节点 failover 提升为 master，
使 master 数 == STS 副本数，operator 才会接管缩容。若拓扑已乱得无法理顺，走 §6 重建。

### 5.5 缩容中断的恢复（本实验踩过的坑，含数据保全）

**场景**：缩容进行中 operator 突然失能（如共享 Secret 被误删、operator 重启），STS 已缩但集群层迁槽未完成。

**后果链**：leader pod 被 STS 删除 → 其 slave 自动提升为 master 接管槽位（failover 级联）→ 原节点变 `master,fail`，
且**多出一个"孤儿 master"**（原分片的 slave 提升而成），集群变成 4 主 3 从之类的错乱。

**恢复步骤（按序执行，已验证数据无损）：**
```bash
# 1) 记下孤儿 master 的 nodeID 与其持槽位（%s 会显示 slave,fail 死节点，先忽略）
kubectl exec -n redis-op-lab redis-cluster-leader-0 -- redis-cli -a $PASS --no-auth-warning CLUSTER NODES

# 2) FORGET 死节点（所有存活 master 上执行）
kubectl exec -n redis-op-lab <each-master> -- redis-cli -a $PASS --no-auth-warning CLUSTER FORGET <dead-id>

# 3) 把孤儿 master 的槽位迁到保留 master（本例迁到 leader-0，4096 槽）
kubectl exec -n redis-op-lab <master-pod> -- redis-cli -a $PASS --no-auth-warning \
  --cluster reshard <master-addr> \
  --cluster-from <orphan-master-id> --cluster-to <keep-master-id> \
  --cluster-slots <槽位数> --cluster-yes
# reshard 完成后空槽 master 会自动降级为 slave，无需手动 replicate

# 4) 手动 rebalance 均衡（因为迁槽全堆到一个 master 上）
kubectl exec -n redis-op-lab redis-cluster-leader-0 -- redis-cli -a $PASS --no-auth-warning \
  --cluster rebalance <addr> --cluster-yes
```

**红线提醒**：中途**不要删共享 Secret**（本实验的 `redis-secret` 同时被 RedisReplication 和 RedisCluster 引用，
且被 Argo CD 的某个子 App 管理）。删掉后 operator 拿不到密码，`CheckRedisNodeCount` 全返回 0，
operator 会误判"集群不存在"而试图 `ExecuteRedisClusterCommand` 重建。恢复=按原定义重建同名同值 Secret。

---

## 6. 兜底：重建 RedisCluster

拓扑错乱且人工无法理顺时的最终手段（数据会丢，需先备份）：

```bash
kubectl delete rediscluster redis-cluster -n redis-op-lab
# 确认 Pod/PVC 清理后，重新 apply 干净 CR（见 manifests/redis-cluster.yaml）
kubectl apply -f ../manifests/redis-cluster.yaml
```

---

## 7. 生产建议汇总

| # | 建议 | 依据 |
|---|---|---|
| 1 | **扩容前必查死节点 + 可调度资源** | §3 三个红灯，三类卡住 |
| 2 | 故障切换后把"FORGET 死节点"纳入巡检 SOP | §5.1 operator 不做 FORGET |
| 3 | 缩容必须先迁槽再删节点，杜绝带数据 pod 消失 | §5.3 脏 PVC 卡扩容 |
| 4 | 扩容目标×单 Pod request 要低于节点余量 | §5.2 资源卡住 |
| 5 | 共享 Secret 别随意删，operator 密码缺失=双目失明 | §5.5 红线 |
| 6 | rebalance 是渐进迁移，验证要等最终收敛 | §2 实测 1~3 分钟 |
| 7 | operator 版本升级后再测一遍扩缩容（当前 v0.25.0 无自愈） | 三类卡住全靠人工 |

---

## 8. 复现速查（命令集）

```bash
# 前置：operator 已装（见第 1 弹），redis-secret 已建

# 部署 3 分片集群
kubectl apply -f ../manifests/redis-cluster.yaml

# 扩容 A：加副本 3 → 6
kubectl patch rediscluster redis-cluster -n redis-op-lab --type=json \
  -p '[{"op":"replace","path":"/spec/redisFollower/replicas","value":6}]'

# 扩容 B：加分片 3 → 4
kubectl patch rediscluster redis-cluster -n redis-op-lab --type=json \
  -p '[{"op":"replace","path":"/spec/clusterSize","value":4},
       {"op":"replace","path":"/spec/redisLeader/replicas","value":4}]'

# 缩容：减分片 4 → 3
kubectl patch rediscluster redis-cluster -n redis-op-lab --type=json \
  -p '[{"op":"replace","path":"/spec/clusterSize","value":3},
       {"op":"replace","path":"/spec/redisLeader/replicas","value":3}]'

# 故障注入（复现死节点卡扩容）：
kubectl delete pod redis-cluster-leader-1 -n redis-op-lab --grace-period=0 --force

# 巡检/验证命令
kubectl exec -n redis-op-lab redis-cluster-leader-0 -- redis-cli -a $PASS --no-auth-warning CLUSTER INFO
kubectl exec -n redis-op-lab redis-cluster-leader-0 -- redis-cli -a $PASS --no-auth-warning CLUSTER NODES
kubectl get rediscluster redis-cluster -n redis-op-lab -o jsonpath='{.status.state}'
```
