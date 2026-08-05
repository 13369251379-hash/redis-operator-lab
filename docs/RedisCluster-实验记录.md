# RedisCluster（分片模式）实验记录

> 实验日期：2026-08-05 13:40（预约执行）
> 环境：同第 1 弹（Docker Desktop K8s 单节点 4CPU/16GB，`redis-op-lab` 命名空间，operator v0.25.0）
> CR：`../manifests/redis-cluster.yaml`

---

## 1. 一句话结论

> **RedisCluster = 无 sentinel 的自动分片集群。** 3 分片自动建好、槽位自动分配、故障自动切换，
> 但**扩容会被故障切换遗留的死节点卡死**，需要人工 `CLUSTER FORGET` 才能自愈。

---

## 2. 架构对照（第 1 弹 RedisReplication vs 第 2 弹 RedisCluster）

| 维度 | RedisReplication（第 1 弹） | RedisCluster（本弹） |
|---|---|---|
| 高可用机制 | Sentinel 投票切换 | **集群自身**（gossip + 槽位选举），无 sentinel |
| 数据分布 | 单 master 全量复制 | **16384 槽位分片**，key 哈希路由 |
| 故障切换 | ~18s（实测） | ~23s（实测，含轮询粒度） |
| 故障后 Service | operator 更新 master/replica SVC | 客户端走集群路由，无 SVC 切换 |
| 扩容 | 手写=改 STS 副本数 | **改 CR `redisFollower.replicas`** |
| 扩容的坑 | 无 | **需先清理失败节点**（见下） |
| 失败节点清理 | sentinel 自动处理 | **不会自动清理**，需 `CLUSTER FORGET` |

---

## 3. 部署验证

### 3.1 从 1 个 CR 生成

```
statefulset/redis-cluster-leader   3/3   （3 个分片主节点）
statefulset/redis-cluster-follower 6/6   （6 个从节点 = 每分片 2 副本）
service：leader/follower/master + additional + headless 共 7 个
pvc：9 个全 Bound（每 Pod 1Gi local-path）
```

> 注意：follower.replicas 默认 = clusterSize（3），本实验显式设为 6 演示扩容。

### 3.2 槽位分配（自动完成）

```
cluster_state: ok，slots_assigned: 16384，cluster_size: 3，known_nodes: 6
leader-0: 0-5460        leader-1: 5461-10922        leader-2: 10923-16383
follower-0→leader-0     follower-1→leader-1         follower-2→leader-2
```

### 3.3 读写（分片生效）

```
SET cluster:key hello-cluster  → OK（-c 集群模式自动路由）
GET cluster:key                → hello-cluster
两个不同 key 落在不同分片（leader-0 与 leader-1 各 1 个 key）→ 分片真正生效
```

---

## 4. 故障切换（13:47 实验）

```
13:47:26  kill redis-cluster-leader-1（槽位 5461-10922）
13:47:49  follower-1 已自主提升为 master，接管 5461-10922   ← ~23s
          被杀的 leader-1 由 STS 重建（新 Pod），自动挂到新 master 下成为 slave
          旧节点在 CLUSTER NODES 中保留 master,fail 标记    ← 不自动清理
```

验证：`cluster_state:ok`、故障分片旧数据可读、新 master 可写。

---

## 5. 扩容实验（13:50，最有价值的发现）

### 5.1 操作

```
kubectl patch rediscluster redis-cluster --type=json \
  -p '[{"op":"replace","path":"/spec/redisFollower/replicas","value":6}]'
```

### 5.2 现象：扩容被卡死

- operator 创建了 3 个新 follower Pod（follower-3/4/5）
- 但 **CLUSTER NODES 里没有它们**，新 Pod 是孤立节点（`cluster_state:fail`）
- operator 日志反复：**`Leaders.Count: 4, Instance.Size: 3`** → 卡在"创建集群"循环

### 5.3 根因

故障切换遗留的死节点（`master,fail`，10.244.0.46）仍被 operator 计为 leader，
导致 **Leaders.Count(4) > clusterSize(3)**，operator 判定"leader 未全部入集群"而拒绝挂载从节点。

### 5.4 解决：手动 CLUSTER FORGET

```
kubectl exec <每个 leader> -- redis-cli -a $PASS CLUSTER FORGET 8a7076d6...（死节点 ID）
```

FORGET 后下一轮 reconcile：日志变为 **`All leader are part of the cluster, adding follower/replicas`**
（Leaders.Count:3 == Instance.Size:3），**6 个 follower 全部自动挂上分片**，每分片 2 副本。

```
最终拓扑（healthy，known_nodes:9，cluster_state:ok）
  分片0: leader-0 + follower-0, follower-3
  分片1: follower-1(提升) + leader-1(重建), follower-4
  分片2: leader-2 + follower-2, follower-5
```

---

## 6. 踩坑与生产启示

| # | 现象 | 原因 | 生产建议 |
|---|---|---|---|
| 1 | 故障切换后死节点不自动清理 | Redis Cluster 特性，operator 不执行 FORGET | 定期巡检 `CLUSTER NODES` 中 `fail` 节点并 FORGET |
| 2 | 扩容被卡死（Leaders 4 vs 3） | 死节点仍被计为 leader | **先清理失败节点，再扩容** |
| 3 | 扩容只建 Pod 不挂分片 | 同 #2，operator 拒绝挂载 | 扩容前确认集群无 fail 节点 |
| 4 | follower.replicas 默认=clusterSize | 官方文档 | 显式写明更清晰 |

**对 prod 的启示：**
- RedisCluster 适合**需要分片/横向扩容**的场景；RedisReplication 适合单写节点 + sentinel HA
- 如果 prod 只是缓存/会话（单节点可装下），用 RedisReplication/sentinel 足够，不需要分片复杂度
- 若用 RedisCluster，**必须把"失败节点清理"纳入运维 SOP**，否则扩容会在最需要时卡住

---

## 7. 复现

```bash
# 前提：operator 已装（见第 1 弹），redis-secret 已建
kubectl apply -f ../manifests/redis-cluster.yaml

# 扩容
kubectl patch rediscluster redis-cluster -n redis-op-lab --type=json \
  -p '[{"op":"replace","path":"/spec/redisFollower/replicas","value":6}]'

# 清理失败节点（故障切换后扩容前必做）
# ID 从 CLUSTER NODES 中 `master,fail` 行取
kubectl exec -n redis-op-lab redis-cluster-leader-0 -- redis-cli -a $PASS --no-auth-warning CLUSTER FORGET <dead-node-id>
```
