# 回退 k8s 版本时是否需要还原 etcd 数据——调研报告

> **调研对象**：QKE v3.4.0 集群（k8s v1.27.10 → v1.33.4，经 1.29.10 → 1.31.2 → 1.33.4，离线环境）升级后，若需把 k8s 版本回退到低版本（如 1.33.4 → 1.31.2），是否需要**还原（恢复）etcd 备份数据**。
> **调研方式**：Kubernetes 官方文档（etcd 运维 / kubeadm 升级 / 版本偏差策略）+ etcd 官方文档（灾难恢复）+ KEP-4330（Compatibility Versions）+ GKE 官方博客（minor 版本回滚机制）+ 社区讨论。
> **结论一句话**：**回退 k8s minor 版本时，建议默认同时还原 etcd 数据（从升级前快照恢复）**——因为新版本 kube-apiserver 可能把数据以旧版本无法解析的格式/字段写入 etcd，Kubernetes 官方不支持跨 minor 版本降级，还原 etcd 是保证旧版本 apiserver 能读取全部数据的唯一可靠方式；本项目 etcd 版本不变（3.5.13），此处是**数据还原**而非 etcd 版本回退。

---

## 1. 调研结论（一句话）

**回退 k8s minor 版本（如 1.33.4 → 1.31.2）时，建议默认还原 etcd 数据**：Kubernetes 官方不支持跨 minor 版本降级，新版本 apiserver 可能把数据以旧版本无法解析的 storage version / 字段 / 对象写入 etcd，只有从升级前快照还原 etcd，才能保证回退后的旧版本 apiserver 读取全部数据；本项目 etcd 版本不变（3.5.13），因此这是**etcd 数据还原**（与 §8.6⑤ 的"etcd 版本回退"是两回事）。

---

## 2. 是否需要还原 etcd 数据的分析

### 2.1 核心机制：k8s API 对象存在 etcd 中，升级会改变 etcd 里的数据内容

- Kubernetes 官方文档明确："**All Kubernetes objects are stored in etcd**"（[Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)）。etcd 里存的是 k8s 的 API 对象（以 storage version 序列化）。
- 升级 k8s 时，新版本 kube-apiserver 启动后可能对 etcd 数据做以下改变：
  - **写入新字段**：对已有对象做默认值填充、status 更新、新增 annotation（如 `deployment.kubernetes.io/revision`、`kubectl.kubernetes.io/last-applied-configuration` 等）；
  - **迁移 storage version**：把某些 API 组的存储版本从旧版本（如 `v1beta1`）迁移到新版本（如 `v1`），对象以新 storage version 序列化后写入 etcd；
  - **创建新对象/新 CRD**：新版本引入的内置资源、新 CRD 实例、新控制器写入的数据；
  - **改变校验规则**：新版本放宽/收紧的字段校验。
- 本项目升级路径 1.27.10 → 1.29.10 → 1.31.2 → 1.33.4 中，**每一步升级都会用新版本 apiserver 读写 etcd**，因此 etcd 里的**数据内容**会随 k8s 升级而变化（尽管 etcd 二进制版本保持 3.5.13 不变）。

### 2.2 为什么回退 k8s 版本时旧版本可能读不了新数据

- Kubernetes **只支持升级、不支持降级**：官方 [Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/) 只定义各组件"升级"时的版本偏差范围，**没有任何降级支持**；kubeadm 升级文档也只讲升级，不讲降级。
- 官方/上游对"为什么不能降级"的权威表述：
  - **KEP-4330（Compatibility Versions）** 的 Motivation 明确：升级会"**introducing potentially corrupting data (i.e. data only present in N+1 and not in N)**"（引入只在 N+1 存在、N 不存在的潜在破坏性数据），并强调"**without writing any data to storage at newer API versions**"（不把数据以更新的 API 版本写入存储）是保证可回退的前提；其目标之一是"**A Kubernetes binary with emulation version set to N does not enable any changes (storage versions, CEL feature, features) that would prevent it from being rolled back to N-1**"（不启用任何会阻止回退到 N-1 的 storage version / 特性变更）。反过来说：**一旦新版本把数据以新 storage version / 新字段写入 etcd，旧版本就无法回退**。
  - **GKE 官方博客**（[Two-step control plane upgrades in GKE](https://dev.to/googlecloud/two-step-control-plane-upgrades-in-gke-how-minor-version-rollbacks-work-under-the-hood-i1l)）直接写明：
    > "In standard Kubernetes clusters, upgrading a control plane from one minor version to the next commits changes to storage schemas immediately. If an unexpected regression emerges after upgrading the API server, **rolling back to the previous minor version was impossible without restoring etcd snapshots**."
    > "**Because earlier binary versions cannot parse data stored in newer schemas, Kubernetes prohibits control plane downgrades across minor versions.**"
  - 即：**旧版本二进制无法解析新 schema 存储的数据，因此跨 minor 版本降级被禁止；要回退只能还原 etcd 快照**。GKE 的"两步升级"正是通过让新二进制以旧版本"模拟模式"运行、**不向 etcd 写入新格式数据**，才实现了"零数据丢失回退"——这从反面印证了：普通升级（写入新格式数据）后回退必须还原 etcd。

### 2.3 什么情况下需要还原 etcd 数据

**需要还原（推荐默认路径）**：

- 升级过程中 etcd 数据被新版本 k8s 修改/写入旧版本无法解析的内容，包括但不限于：
  - 新版本迁移了某 API 组的 **storage version**（旧版本 apiserver 不 serve 该 storage version，读不到对象）；
  - 新版本给已有对象写入**旧版本不认识的字段**（旧版本读取时忽略，但一旦旧版本回写对象会**裁剪掉这些字段，造成数据丢失**）；
  - 新版本创建了**新 CRD / 新 API 对象 / 新 annotation**，旧版本无法解析或校验失败；
  - 新版本改变了**校验规则**，旧版本对已存在对象校验报错。
- 回退目标是 **minor 版本回退**（本项目 1.33.4 → 1.31.2 即属此类，跨 2 个 minor）。
- 回退原因是**业务/平台/第三方组件/节点基础设施不兼容**，集群已处于异常状态，需要回到"已知良好"的升级前状态。

**可以不还原（谨慎、非默认）**：

- 仅当**确认升级未改变 etcd 数据结构**（只换了控制面组件二进制，未写入新字段/新 storage version/新对象/新 CRD），且回退后旧版本 apiserver 能正常启动并读取全部数据时，可不还原 etcd。
- 但此情况**难以事先确认**（升级过程中 apiserver/controller 几乎总会写入一些新数据），且存在"旧版本 apiserver 能启动、但回写对象时裁剪未知字段导致数据丢失"的隐性风险，因此**不建议作为默认路径**。
- 若回退只是 **patch 版本回退**（如 1.33.4 → 1.33.3），patch 版本不改变 storage schema，一般**不需要**还原 etcd；但本项目回退是 minor 版本回退，不适用。

### 2.4 还原 etcd 是重操作，需权衡

- 还原 etcd 需要**停止 kube-apiserver 与 etcd**（集群控制面短暂不可用），多成员集群需逐成员恢复，属于维护窗口内的重操作；
- 还原会**丢失快照之后的所有数据变更**（升级窗口内新建/修改的工作负载、事件等）——但对"因不兼容而回退"的场景，集群本就异常，回到升级前状态是可接受的；
- 还原后 etcd revision 回退，可能让依赖 watch/informer 的控制器缓存不一致（详见 §4 注意事项，可用 `--bump-revision --mark-compacted` 缓解）；
- 权衡结论：**回退 k8s minor 版本时，还原 etcd 的收益（保证旧版本可读、回到已知良好状态）大于代价（停机、丢快照后变更）**，建议作为回退的默认步骤。

### 2.5 区分：etcd 版本回退 vs etcd 数据还原（本项目关键）

| 项 | etcd 版本回退 | etcd 数据还原 |
|----|--------------|--------------|
| 对象 | etcd **二进制版本**（如 3.5.24 → 3.5.13） | etcd 里存的 **k8s 数据内容**（从快照恢复） |
| 本项目是否涉及 | **不涉及**（etcd 保持 v3.5.13 不升级，§8.6⑤"回退 etcd（若已升级）"即指此，本项目无需执行） | **可能涉及**（k8s 升级改变了 etcd 数据内容，回退 k8s 版本时需还原） |
| 操作 | 替换 etcd 二进制、逐节点滚动 | `etcdutl/etcdctl snapshot restore` + 替换数据目录 |

> 本项目 §8.6⑤ 的"回退 etcd（若已升级）"是**版本回退**（本项目 etcd 未升级，无需执行）；**缺少的正是"还原 etcd 数据"**——这是回退 k8s 版本时真正需要的内容，见 §5 表述建议。

---

## 3. 还原 etcd 数据的操作步骤（若需要）

> 以下为通用步骤（参考 [Kubernetes 官方 etcd 运维文档](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/) 与 [etcd 官方灾难恢复文档](https://etcd.io/docs/v3.5/op-guide/recovery/)），结合本项目环境（etcd 以 **systemd 服务**运行、开启 TLS、证书在 `/etc/ssl/etcd/ssl/`、数据目录默认 `/var/lib/etcd`、快照在 `/data/var/backups/`）整理。**具体命令以实际环境为准，执行前先确认 etcd 数据目录与成员拓扑。**

**前置**：确认已具备 §8.1① 的 etcd 快照（`/data/var/backups/etcd-snapshot-*.db`）。回退到哪个版本，就还原**该版本升级前**的快照（如回退 1.33.4 → 1.31.2，还原 1.31.2 → 1.33.4 升级前、即 1.31.2 状态时的快照）。

**① 停止 kube-apiserver（所有 master 节点）**

> 官方要求：**etcd 还原期间不得有 apiserver 在运行**（否则会继续写入，破坏还原）。QKE 中 apiserver 为静态 Pod，可临时移走 manifest 或停 kubelet 使静态 Pod 停止；还原完成后恢复。

```bash
# 方式一：临时移走 apiserver 静态 Pod manifest（还原后移回）
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/kube-apiserver.yaml.bak
# 方式二：停 kubelet（会连带停掉本节点所有静态 Pod，含 etcd，需注意顺序）
# systemctl stop kubelet
```

**② 停止 etcd（所有 master 节点）**

```bash
systemctl stop etcd
```

**③ 用快照还原 etcd 数据目录（每个 master 节点执行）**

> etcd v3.5.x 中 `etcdctl snapshot restore` 已**弃用**，官方推荐用 **`etcdutl`**（etcd v3.5.13 自带）。还原会覆盖 member ID / cluster ID，相当于**新建一个逻辑集群**，所有成员必须用**同一个快照**还原。

```bash
# 单成员/单节点（或先还原一个成员再让其他成员同步）：
etcdutl snapshot restore /data/var/backups/etcd-snapshot-<日期>.db \
  --data-dir /var/lib/etcd-restore

# 多成员集群：每个成员用同一快照 + 新成员信息还原（示例 3 成员 m1/m2/m3）
etcdutl snapshot restore /data/var/backups/etcd-snapshot-<日期>.db \
  --name m1 --data-dir /var/lib/etcd-restore \
  --initial-cluster m1=https://<m1-ip>:2380,m2=https://<m2-ip>:2380,m3=https://<m3-ip>:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls https://<m1-ip>:2380
# m2/m3 节点同理，--name/--data-dir/--initial-advertise-peer-urls 换成各自值
```

> 若快照是从数据目录 `member/snap/db` 直接拷贝的（无完整性 hash），需加 `--skip-hash-check`。
> 若担心还原后 revision 回退导致控制器 informer 缓存不一致，可加 `--bump-revision <大数> --mark-compacted`（见 §4）。

**④ 替换数据目录（每个 master 节点执行）**

```bash
# 备份旧数据目录（勿直接删除，便于回退失败时恢复）
mv /var/lib/etcd /var/lib/etcd.bak-$(date +%F-%H%M%S)
mv /var/lib/etcd-restore /var/lib/etcd
# 修正属主与权限（etcd 以 etcd 用户运行）
chown -R etcd:etcd /var/lib/etcd
chmod -R 700 /var/lib/etcd
```

**⑤ 启动 etcd 并验证（每个 master 节点执行）**

```bash
systemctl start etcd
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/node-<hostname>.pem \
  --key=/etc/ssl/etcd/ssl/node-<hostname>-key.pem \
  endpoint health
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/node-<hostname>.pem \
  --key=/etc/ssl/etcd/ssl/node-<hostname>-key.pem \
  member list
```

**⑥ 恢复 kube-apiserver 并重启控制面组件**

```bash
# 恢复 apiserver 静态 Pod manifest（若步骤①移走了）
mv /tmp/kube-apiserver.yaml.bak /etc/kubernetes/manifests/kube-apiserver.yaml
# 若步骤①停了 kubelet，则启动
# systemctl start kubelet
# 官方建议重启 kube-scheduler / kube-controller-manager / kubelet，避免依赖陈旧数据
systemctl restart kubelet
```

**⑦ 回退 k8s 控制面组件版本并验证**

- 还原 etcd 后，按 §8.6 ①~④ 回退 kube-apiserver / kube-controller-manager / kube-scheduler / kubelet / kube-proxy / coredns 到目标版本；
- 按 §8.6⑥ 还原数据库字段（qke_db_cluster 等）；
- 按 §8.7 / §6 验证：`kubectl get nodes -o wide`（全部 Ready 且为目标版本）、`kubectl get pod -A -owide | grep -v Running`（无异常 pod）、`kubectl version`。

---

## 4. 注意事项

1. **还原期间必须停 apiserver**：官方明确"如果集群中有 apiserver 在运行，不要尝试还原 etcd"——先停所有 apiserver → 还原所有 etcd 成员 → 再启动所有 apiserver，否则还原会被写入破坏。
2. **多成员集群**：所有成员必须用**同一个快照**还原，且还原会覆盖 member ID / cluster ID（相当于新建逻辑集群），需用 `--name / --initial-cluster / --initial-cluster-token / --initial-advertise-peer-urls` 指定新成员信息，避免与仍存活的旧成员误连。
3. **数据目录权限**：还原出的数据目录属主/权限需与 etcd 运行用户一致（本项目为 `etcd:etcd`、`700`），否则 etcd 启动失败；替换前先备份旧数据目录。
4. **TLS**：本项目 etcd 开启 TLS，etcdctl/etcdutl 操作需带 `--cacert/--cert/--key`（证书在 `/etc/ssl/etcd/ssl/`）；`etcdutl snapshot restore` 是本地文件操作，不需要证书，但 `endpoint health / member list` 需要。
5. **revision 回退与 informer 缓存**：还原到旧快照后 etcd revision 回退，依赖 watch 的控制器/operator（informer 本地缓存）可能不刷新、行为异常。官方建议对 Kubernetes 场景用 `--bump-revision <大数> --mark-compacted` 还原，使所有 watch 终止、缓存失效重建（[etcd 灾难恢复文档](https://etcd.io/docs/v3.5/op-guide/recovery/)）。
6. **数据丢失范围**：还原会丢失快照之后的所有变更（升级窗口内新建/修改的对象）。对"因不兼容而回退"场景可接受，但需在维护窗口与用户确认。
7. **快照选择**：回退到哪个版本，就还原**该版本升级前**的快照（§8.1 要求每步升级前都备份，正好对应）。还原后 etcd 里的 kubeadm-config / kubelet-config ConfigMap 也回到升级前状态，与回退后的 k8s 版本一致，有利于一致性；节点级文件（`/etc/kubernetes/kubeadm-config.yaml`、`/var/lib/kubelet/config.yaml`）按 §8.1② 备份一并还原。
8. **与数据库字段还原的配合**：etcd 数据还原与 §8.6⑥ 的 qke_db_cluster 版本字段还原要**同步执行**，保证平台状态与回退后的实际版本一致。
9. **etcd 版本不变**：本项目 etcd 保持 v3.5.13，还原是**数据还原**，不涉及 etcd 二进制版本回退；`etcdutl` 与 etcd 3.5.13 同版本即可，无需额外准备旧版本 etcd 二进制。

---

## 5. 对 §8.6 的表述建议

**现状**：§8.6 回退步骤有"⑤ 回退 etcd（若已升级）"（指 etcd 二进制版本回退，本项目 etcd 未升级、无需执行），但**缺少"还原 etcd 数据"**。

**建议**：在 §8.6 中新增一步"**还原 etcd 数据（回退 k8s 版本时建议执行）**"，并调整编号。建议表述如下：

> **⑤ 还原 etcd 数据（回退 k8s 版本时建议执行）**
>
> - **定位**：回退 k8s minor 版本时，新版本 apiserver 可能把数据以旧版本无法解析的格式/字段写入 etcd，Kubernetes 官方不支持跨 minor 版本降级；**从升级前快照还原 etcd 数据**是保证回退后旧版本 apiserver 能读取全部数据的可靠方式（详见 [etcd-restore-on-rollback-report.md](../refs/etcd-restore-on-rollback-report.md)）。
> - **与"回退 etcd 版本"的区别**：本项目 etcd 保持 v3.5.13 不升级，**无需回退 etcd 版本**；此处是**还原 etcd 数据**（从快照恢复），两者不同。
> - **操作**：停 kube-apiserver → 停 etcd → `etcdutl snapshot restore`（用回退目标版本升级前的快照，多成员用同一快照 + 新成员信息）→ 替换数据目录（`/var/lib/etcd`，注意属主权限）→ 起 etcd 并 `endpoint health` 验证 → 起 apiserver 并重启控制面组件 → 再执行 ①~④ 回退组件版本。
> - **权衡**：还原会丢失快照后的变更并需停机，但对"因不兼容而回退"场景可接受；若确认升级未改变 etcd 数据结构，可先尝试不还原直接回退组件版本，异常时再还原（不作为默认路径）。

**同时建议**：
- §8.4 回退原则中补充一句："回退 k8s minor 版本时，建议同时还原 etcd 数据（从升级前快照恢复），保证旧版本 apiserver 可读全部数据"；
- §8.1① 备份说明中注明"该快照同时用于回退时还原 etcd 数据"；
- §8.5 回退涉及组件中，把"etcd"拆分为"etcd 数据（还原）"与"etcd 版本（本项目不涉及）"。

---

## 6. 参考资料

| 来源 | 类型 | 说明 |
|------|------|------|
| [Kubernetes 官方文档：Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/) | 官方文档 | 所有 k8s 对象存于 etcd；还原流程（停 apiserver → 还原 etcd → 起 apiserver → 重启控制面组件）；`etcdctl snapshot restore` 用法与数据目录替换 |
| [Kubernetes 官方文档：Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) | 官方文档 | kubeadm 只讲升级不讲降级；`kubeadm-backup-etcd` 用于 etcd 升级失败时人工恢复 |
| [Kubernetes 官方文档：Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/) | 官方文档 | 只定义升级时组件版本偏差，无降级支持 |
| [KEP-4330：Compatibility Versions in Kubernetes](https://github.com/kubernetes/enhancements/tree/master/keps/sig-architecture/4330-compatibility-versions) | 上游 KEP | 升级会"引入只在 N+1 存在、N 不存在的潜在破坏性数据"；不把数据以新 API 版本写入存储是回退前提；storage version 变更会阻止回退到 N-1 |
| [GKE 官方博客：Two-step control plane upgrades in GKE](https://dev.to/googlecloud/two-step-control-plane-upgrades-in-gke-how-minor-version-rollbacks-work-under-the-hood-i1l) | 官方博客 | "回退到上一 minor 版本在**不还原 etcd 快照的情况下不可能**"；"旧版本二进制无法解析新 schema 存储的数据，因此禁止跨 minor 版本降级"；GKE 两步升级通过模拟旧版本、不写新格式数据实现零丢失回退 |
| [Google Cloud 博客：Kubernetes gets minor version rollback](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-gets-minor-version-rollback) | 官方博客 | GKE minor 版本回滚能力背景 |
| [etcd 官方文档：Disaster recovery](https://etcd.io/docs/v3.5/op-guide/recovery/) | 官方文档 | `etcdutl snapshot restore` 用法；还原覆盖 member/cluster ID、新建逻辑集群；多成员还原参数；`--bump-revision --mark-compacted` 对 Kubernetes 场景的建议 |
| [kubernetes-sigs/cluster-api Discussion #10942：Why version downgrade is not allowed?](https://github.com/kubernetes-sigs/cluster-api/discussions/10942) | 社区讨论 | 社区对"k8s 不支持降级"原因（数据 schema 兼容性）的讨论 |
