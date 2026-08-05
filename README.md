# Redis-Operator 学习实验

用 **OT-CONTAINER-KIT redis-operator** 在本地 K8s 复刻 prod Redis 主从集群（1 主 + 2 从 + 3 Sentinel），
与手写 Kustomize 方案（`D:\项目\redis主从集群\overlays\prod\`）做对照学习。

## 目录

```
Redis-Operator学习/
├── manifests/                    # 前两弹实验 CR（历史参考）
│   ├── redis-replication.yaml    # 第 1 弹：1 个 CR 复刻 prod 主从+sentinel 架构
│   └── redis-cluster.yaml        # 第 2 弹：RedisCluster 分片模式 CR
├── gitops/                       # 第二阶段：Argo CD GitOps 结构
│   ├── apps/                     # 父 App(redis-gitops) 源：两个子 App（sync-wave 排序）
│   │   ├── redis-operator.yaml   #   子 App 1（wave 0）：operator（CRD + Deployment）
│   │   └── redis-replication.yaml#   子 App 2（wave 2）：RedisReplication CR + Secret
│   ├── operator/                 # 子 App 1 源：helm 渲染的 operator 清单
│   │   ├── crds/*.yaml           #   4 个 CRD（先于 CR 应用）
│   │   └── operator.yaml         #   SA + ClusterRole/Binding + Deployment
│   └── redis/                    # 子 App 2 源：RedisReplication CR + redis-secret
│       ├── redis-replication.yaml
│       └── redis-secret.yaml
├── docs/
│   ├── 手写方案vs-Operator对照.md # 第 1 弹完整学习文档
│   ├── RedisCluster-实验记录.md   # 第 2 弹实验记录（故障切换/扩容/死节点坑）
│   └── ArgoCD-GitOps-实验记录.md  # 第二阶段实验记录
└── README.md
```

## 关键结论

- 手写方案的 13 个 YAML ≈ operator 的 1 个 CR + 2 个专用镜像
- operator 有「镜像契约」：密码/模式靠 env + entrypoint，官方 redis 镜像不兼容
- 第 1 弹（RedisReplication）：故障切换 RTO ~18s，operator 自动更新 Service 指向
- 第 2 弹（RedisCluster）：3 分片自动建好，故障自动切换 ~23s；**扩容会被失败节点卡死，需 CLUSTER FORGET 自愈**
- 当前 prod **不建议迁移**（镜像契约/CRD 升级/Argo CD 双循环是新增负担）

## 环境状态（2026-08-05）

- 本地集群：Docker Desktop K8s 单节点（4 CPU / 16GB），context=docker-desktop
- 命名空间 `redis-op-lab`：operator + Redis 由 **Argo CD** 统一管理（GitOps 第二阶段）
- Argo CD：`argocd` 命名空间，父 App `redis-gitops`（App-of-Apps）
- 仓库凭据：GitHub 私有仓库 `redis-operator-lab`（HTTPS + PAT），repo-server 走宿主 Clash 代理
- 清场命令：`kubectl delete -n redis-op-lab rediscluster redis-cluster`；`helm uninstall redis-operator -n redis-op-lab`

## 复现

见 `docs/手写方案vs-Operator对照.md` 第 8 节；GitOps 接入见 `docs/ArgoCD-GitOps-实验记录.md`。
