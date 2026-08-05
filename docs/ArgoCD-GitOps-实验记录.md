# Redis-Operator × Argo CD GitOps 实验记录（第二阶段）

> 实验日期：2026-08-05
> 环境：Docker Desktop K8s 单节点（4 CPU / 16GB，v1.36.1），Argo CD v3.4.5
> 目标：把 operator 管理的 Redis 纳入 Argo CD GitOps，实测「两个循环」如何配合、
>       CRD/CR 时序、selfHeal 冲突、更新/回滚。
> 前置：第一阶段 RedisReplication/RedisCluster 实验（见 `手写方案vs-Operator对照.md`）

---

## 1. 一句话结论

> **Argo CD 只管理 git 里声明的东西（CRD/operator/CR）；operator 派生的
> StatefulSet/Service/PVC 属于第二个循环，Argo CD 不碰 —— 两个循环各管各的，天然不冲突。**
> 但 operator **v0.25.0 有一个严重的 STS 更新 panic bug**，会在更新派生 STS 时把 operator 打崩，
> 这是当前版本**绝对不能上 prod** 的硬证据。

---

## 2. 架构：App-of-Apps + sync-wave 双 App

```
gitops/apps/                    ← 父 App(redis-gitops) 源，kubectl 引导一次后自管理
├── redis-operator.yaml         ← 子 App 1（sync-wave 0）：gitops/operator/
└── redis-replication.yaml      ← 子 App 2（sync-wave 2）：gitops/redis/
```

| App | 内容 | 作用 |
|---|---|---|
| 父 App `redis-gitops` | 两个子 Application | App-of-Apps：按子 App 的 sync-wave 顺序创建并驱动子 App |
| 子 App `redis-operator`（wave 0） | CRD ×4 + SA + ClusterRole/Binding + Deployment | 先装 operator + CRD |
| 子 App `redis-replication`（wave 2） | RedisReplication CR + redis-secret | operator 就绪后才应用 CR |

**为什么能保证「先 CRD/operator，后 CR」**：
- 父 App sync 时按子 App 的 sync-wave 顺序创建子 App；wave 0 的子 App **Healthy 后**才进入 wave 2。
- 子 App `redis-operator` 内，Argo CD 内置排序保证 **CRD 先于 Deployment**。
- 于是 CR 应用时 CRD 一定已存在、operator 已就绪。

**目录源递归坑（实测踩到）**：Argo CD 的 directory 源默认**不递归子目录**。
`gitops/operator/crds/*.yaml` 一开始没被 App 收录（子 App 只同步了 4 个资源、CRD 没接管），
必须加 `source.directory.recurse: true` 才让 CRD 也进 GitOps 管理范围。

---

## 3. 两个循环的配合（核心验证）

```
Argo CD 循环： git ──同步──▶ CR + operator 清单 ──▶ 集群
Operator 循环：CR ──调谐──▶ StatefulSet/Service/PVC ──▶ Redis 集群
```

验证点（`kubectl get app redis-replication -o jsonpath='{.status.resources}'`）：

```
redis-replication App 管理：Secret: redis-secret, RedisReplication: redis-replication
redis-operator    App 管理：ServiceAccount/ClusterRole/Binding/Deployment + 4×CustomResourceDefinition
```

- operator 派生的 `redis-replication`/`redis-replication-s` STS、6 个 Service、3 个 PVC
  **都不在**任何 App 的管理列表里 → **Argo CD 全程不干预它们**。
- Argo CD 只对比 git 里声明的资源；operator 对派生资源的改动不会让任何 App 变 OutOfSync。

---

## 4. selfHeal 验证结果

| 测试 | 结果 |
|---|---|
| 手动改 CR `spec.clusterSize` 3→1 | ✅ Argo CD **~20s 内自动回滚**到 3。tracked 资源变更会触发即时 reconcile，不用等 3 分钟轮询。App 事件可见 `automated sync` + `OutOfSync→Synced` |
| 手动给 CR 加 `metadata.labels.drift-test` | ⚠️ **未被视为漂移**，App 一直 Synced，label 残留没被清。Argo CD 对 metadata label/annotation 的增量添加容忍（与 spec 字段漂移行为不同）——记录观察结果，版本相关 |
| 手动把 `redis-replication` STS 缩到 1 | ⚠️ **operator panic + CrashLoopBackOff**（详见 §6）。Argo CD 完全没反应（STS 不在它管理范围）——这就是「两个循环不冲突」的正向证据，但暴露出 operator 健壮性问题 |
| 手动改派生 STS 期间 App 状态 | ✅ 3 个 App 全程 Synced/Healthy，Argo CD 从不介入派生资源 |

---

## 5. 更新 / 回滚演练（GitOps 闭环）

改动：`gitops/redis/redis-replication.yaml` 的 `requests.cpu` 250m→400m（更新）再回滚。

```
更新： git commit 250m→400m ──▶ Argo CD 轮询同步(~3min) ──▶ CR cpu=400m
       ──▶ operator 调谐 ──▶ redis STS pod template=400m ──▶ Pod 滚动重启
回滚： git revert ──▶ Argo CD 同步 ──▶ CR cpu=250m ──▶ STS 回滚 ──▶ Pod 重启 ──▶ 集群恢复
```

验证：更新/回滚后 master+2 slave 复制正常（master SET → slave GET 一致）。

> Argo CD 的 git 轮询默认 ~3 分钟（无 webhook 时）。更新过程中 operator 又触发了一次
> sentinel STS panic（见 §6），回滚时因 sentinel STS 刚重建、无需更新而未触发。

---

## 6. ⚠️ 重大发现：operator v0.25.0 的 STS 更新 panic bug

**现象**：operator 在 reconcile 中需要 **更新 sentinel StatefulSet（redis-replication-s）** 时，
`reconciler.Reconcile` 的 `client.Update` 失败 → 返回 `nil, err` → `statefulset_reconcile.go:53`
直接 `*reconciled.(*appsv1.StatefulSet)` **不做 nil 检查** → panic
`interface conversion: client.Object is nil, not *v1.StatefulSet`。
controller-runtime 捕获后**重新 panic**（防缓存不一致）→ 进程退出 → **CrashLoopBackOff**。

**触发场景**（本实验实测两次）：
1. 手动把 `redis-replication` STS 缩到 1（派生资源副本漂移）。
2. GitOps 更新 CR 触发 redis STS 滚动更新后。

**为什么持续 panic**：sentinel STS 处于「需要更新」状态（如 labels 与 operator 期望不一致），
每次 reconcile 都走 Update 路径、Update 持续失败 → 每次重启后再次 panic，无法自愈。

**恢复方法**（已验证）：删除 sentinel STS（`kubectl delete sts redis-replication-s`）+ 重启 operator
→ Get 变 NotFound → 走 create 路径重建 → 恢复。但若再触发 Update 路径可能复发。

**对 prod 的意义**：operator v0.25.0 不能优雅处理派生 STS 变更，会把 operator 打崩。
加上镜像契约、CRD 升级负担，**当前版本不建议上 prod**（强化第一阶段结论）。

---

## 7. 环境搭建要点（复现用）

```bash
# Argo CD 安装（quay.io 大镜像拉不动，用本机缓存 v3.4.5；dex 无用禁用）
helm install argocd argo/argo-cd -n argocd --create-namespace \
  --set global.image.tag=v3.4.5 --set dex.enabled=false

# repo-server 走宿主 Clash 代理（集群内访问 host.docker.internal）
kubectl -n argocd patch deployment argocd-repo-server --type=json -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/env/-","value":{"name":"HTTPS_PROXY","value":"http://host.docker.internal:7897"}},
  {"op":"add","path":"/spec/template/spec/containers/0/env/-","value":{"name":"HTTP_PROXY","value":"http://host.docker.internal:7897"}},
  {"op":"add","path":"/spec/template/spec/containers/0/env/-","value":{"name":"NO_PROXY","value":"kubernetes.default.svc,localhost,127.0.0.1,.svc"}}
]'

# GitHub 私有仓库凭据（HTTPS + fine-grained PAT，只读）
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: redis-operator-lab
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/13369251379-hash/redis-operator-lab.git
  username: <你的用户名>
  password: <github_pat_...>
EOF

# 父 App 引导（只引导一次，子 App 自 git 管理）
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: redis-gitops
  namespace: argocd
  finalizers: [resources-finalizer.argocd.argoproj.io]
spec:
  project: default
  source:
    repoURL: https://github.com/13369251379-hash/redis-operator-lab.git
    targetRevision: main
    path: gitops/apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated: { prune: true, selfHeal: true }
EOF
```

**admin 密码**：`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`

---

## 8. 踩坑清单

| # | 现象 | 原因 | 解法 |
|---|---|---|---|
| 1 | CRD 不被 App 管理 | Argo CD directory 源默认不递归子目录 | `source.directory.recurse: true` |
| 2 | quay.io 大镜像拉取卡死 | 无镜像源、直连慢 | 用本机缓存版本 `global.image.tag=v3.4.5` |
| 3 | dex 镜像 ghcr.io 拉不到 | ghcr.io 未走通代理 | `--set dex.enabled=false`（无 SSO 不需要） |
| 4 | GitHub 仓库连不上 | 需代理 | repo-server 设 `HTTPS_PROXY=host.docker.internal:7897` |
| 5 | 缩容派生 STS 后 operator 崩 | v0.25.0 panic bug（§6） | 删 sentinel STS 强制 create 路径恢复 |
| 6 | 更新后全 master、复制中断 | 滚动重启后复制关系需 operator 重建 | 等 operator reconcile（几十秒） |

---

## 9. 对 prod 的最终建议

- **operator v0.25.0 不建议上 prod**：除镜像契约、CRD 升级外，新增 **STS 更新 panic bug**，
  运维一旦碰到派生 STS 变更就可能把 operator 打崩，且不会自愈。
- GitOps 双循环本身没有问题：**Argo CD 管 git 声明物、operator 管派生物，互不打架**。
  这个模式对 operator 迁移后依然成立，但前提是 operator 版本够稳。
- 若未来要迁移，优先升级 operator 到修复了 `statefulset_reconcile.go:53` nil 解引用的版本，
  并补一条运维 SOP：**不要手动动 operator 管理的 StatefulSet**。
