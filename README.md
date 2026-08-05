# Redis-Operator 学习实验

用 **OT-CONTAINER-KIT redis-operator** 在本地 K8s 复刻 prod Redis 主从集群（1 主 + 2 从 + 3 Sentinel），
与手写 Kustomize 方案（`D:\项目\redis主从集群\overlays\prod\`）做对照学习。

## 目录

```
Redis-Operator学习/
├── manifests/
│   └── redis-replication.yaml    # 核心：1 个 CR 复刻 prod 架构（含逐字段对照注释）
├── docs/
│   └── 手写方案vs-Operator对照.md # 完整学习文档（架构对照/踩坑/验证/启示）
└── README.md
```

## 关键结论

- 手写方案的 13 个 YAML ≈ operator 的 1 个 CR + 2 个专用镜像
- operator 有「镜像契约」：密码/模式靠 env + entrypoint，官方 redis 镜像不兼容
- 故障切换 RTO ~18s，且 **operator 自动更新 master/replica Service 指向**（手写方案做不到）
- 当前 prod **不建议迁移**（镜像契约/CRD 升级/Argo CD 双循环是新增负担）

## 环境状态（2026-08-05）

- 本地集群：Docker Desktop K8s 单节点（4 CPU / 16GB）
- 命名空间 `redis-op-lab`：operator + RedisReplication 均已部署并验证通过
- 实验数据不保留价值，可直接 `kubectl delete -n redis-op-lab redisreplication redis-replication`

## 复现

见 `docs/手写方案vs-Operator对照.md` 第 8 节。
