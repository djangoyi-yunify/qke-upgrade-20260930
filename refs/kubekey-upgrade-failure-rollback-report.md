# kubekey（kk）升级失败是否自动回滚——调研报告

> **调研对象**：kubekey（kk）v3.1.12 执行 `kk upgrade` 时，若某阶段失败（镜像拉取失败、kubeadm 升级失败、节点升级失败等），是否会**自动回滚**到升级前版本。
> **调研方式**：kubekey 官方文档 + kubekey v3.1.12 源码（升级 pipeline / 任务框架 / 升级任务实现）+ kubeadm 官方文档 + kubekey 社区 issue。
> **结论一句话**：**kubekey（kk）升级失败不会自动回滚**——任一阶段失败时 kk 会**立即停止**在失败步骤，集群停留在"部分升级"的中间状态，需要**人工排查、重试或按回退步骤人工回退**。

---

## 1. 调研结论（一句话）

**kubekey（kk）执行 `kk upgrade` 时不会自动回滚**：升级是"多阶段、多节点"的线性编排流程，任一阶段（镜像拉取、kubeadm 升级、节点升级、coredns 部署等）失败，kk 立即报错退出，**不会**把集群回退到升级前版本；集群可能处于"部分节点已升级、部分未升级"的中间状态，需人工介入处理（排查 → 重试 → 必要时按回退步骤人工回退）。

---

## 2. kubekey 升级失败行为分析

### 2.1 升级流程是"线性模块序列"，无回滚模块（源码证据，v3.1.12）

kubekey v3.1.12 的升级入口 `NewUpgradeClusterPipeline`（[cmd/kk/pkg/pipelines/upgrade_cluster.go](https://github.com/kubesphere/kubekey/blob/v3.1.12/cmd/kk/pkg/pipelines/upgrade_cluster.go)）把升级组织为**一串顺序执行的模块**：

```
precheck（Greetings/NodePreCheck/ClusterPreCheck）
→ confirm（升级确认）
→ artifact 解包
→ binaries（同步二进制）
→ container（cri-dockerd）
→ etcd（PreCheck/Certs/InstallBinary/Configure/Backup）
→ kubernetes.SetUpgradePlanModule(ToV121) + ProgressiveUpgradeModule(ToV121)
→ loadbalancer
→ kubesphere（Clean/Convert/Deploy/CheckResult）
→ kubernetes.SetUpgradePlanModule(ToV122) + ProgressiveUpgradeModule(ToV122)
→ filesystem.Chown
→ certs.AutoRenewCerts
```

**整个 pipeline 中没有任何"回滚模块"**。其中 `ProgressiveUpgradeModule`（[cmd/kk/pkg/kubernetes/module.go](https://github.com/kubesphere/kubekey/blob/v3.1.12/cmd/kk/pkg/kubernetes/module.go)）内部的任务为：`CalculateNextVersion → DownloadBinaries → PullImages → SyncKubeBinary → UpgradeClusterOnMaster → GetClusterStatus → UpgradeClusterOnWorker → GenerateCoreDNS → DeployCoreDNS → GenerateNodeLocalDNS → DeployNodeLocalDNS → SetCurrentK8sVersion`，**这些任务均未注册任何回滚处理器**。

### 2.2 失败即停：pipeline 在失败模块处直接返回错误（源码证据）

- `pipeline.Start()`（[cmd/kk/pkg/core/pipeline/pipeline.go](https://github.com/kubesphere/kubekey/blob/v3.1.12/cmd/kk/pkg/core/pipeline/pipeline.go)）：遍历模块，一旦某模块结果 `res.IsFailed()`，**立即 `return error`**，整个 `kk upgrade` 命令以失败退出，后续模块不再执行。
- `BaseTaskModule.Run()`（[cmd/kk/pkg/core/module/task_module.go](https://github.com/kubesphere/kubekey/blob/v3.1.12/cmd/kk/pkg/core/module/task_module.go)）：任务失败时虽会调用 `t.ExecuteRollback()`，但 `RemoteTask.ExecuteRollback()`（[cmd/kk/pkg/core/task/remote_task.go](https://github.com/kubesphere/kubekey/blob/v3.1.12/cmd/kk/pkg/core/task/remote_task.go)）**仅在 `t.Rollback != nil` 时才执行**；升级相关任务均未设置 `Rollback` 字段，因此该调用是**空操作**。
- `cmd/kk/pkg/core/rollback/`（[interface.go](https://github.com/kubesphere/kubekey/blob/v3.1.12/cmd/kk/pkg/core/rollback/interface.go)、[base.go](https://github.com/kubesphere/kubekey/blob/v3.1.12/cmd/kk/pkg/core/rollback/base.go)）只是任务框架预留的**通用回滚接口**（`BaseRollback.Execute` 默认返回 `nil`，即什么都不做），**升级流程并未使用它**。

### 2.3 升级任务本身无回滚逻辑（源码证据）

`UpgradeKubeMaster` / `UpgradeKubeWorker` / `KubeadmUpgrade`（[cmd/kk/pkg/kubernetes/tasks.go](https://github.com/kubesphere/kubekey/blob/v3.1.12/cmd/kk/pkg/kubernetes/tasks.go)）执行的是：

- 第一个 master：`kubeadm upgrade apply <version> -y --ignore-preflight-errors=all ...`
- 其他 master / worker：`kubeadm upgrade node`
- 随后重启 kubelet

这些动作**失败时仅返回错误**，没有任何"回退到旧版本"的逻辑。kk 在升级前**不备份** kubeadm/kubelet 二进制与配置（v3.1.12 中仅 etcd 模块有 `BackupModule`，用于 etcd 数据备份，非回滚机制）。

### 2.4 官方文档：只要求"升级前备份以便失败后恢复"，无自动回滚

kubekey 官方升级文档 [Upgrade Cluster](https://github.com/kubesphere/kubekey/blob/main/docs/en/installation/upgrade-cluster.md) 的 Prerequisites 与 Important Notes 中，对失败的处理只有一句：

> **Back up the cluster before upgrading (especially etcd data) so you can recover if the upgrade fails.**

即"升级前备份（尤其 etcd 数据），以便升级失败后**恢复**"——恢复是**人工**动作，文档通篇**未提及任何自动回滚**。文档还明确说明：worker 节点**并行升级**、**不自动 cordon/drain**，如需严格滚动需人工 drain——进一步说明 kk 不承担失败后的自动恢复职责。

### 2.5 社区 issue：升级失败即停，需人工处理（实例证据）

[kubekey Issue #2511 "upgradge K8S cluster error"](https://github.com/kubesphere/kubekey/issues/2511)（kk v3.1.7，k8s 1.30.6→1.31.2）：`kubeadm upgrade apply` 因 CRI socket 错误（`dockershim.sock: connect: connection refused`）失败，kk 日志停在失败步骤并报错退出；用户需**人工排查**（发现 pause 镜像版本缺失、手动上传镜像等）后**重试**。该 issue 是"kk 升级失败 → 停在失败状态 → 人工处理"的典型实例，全程无自动回滚。

---

## 3. kubeadm 与 kubekey 的区别（重要）

原 §8.4 表述"kubeadm 升级失败时会自动回滚（kubeadm 内部机制）"**部分正确但易误导**，需区分两个层面：

### 3.1 kubeadm 自身：单节点内、单组件、尽力而为的"自动回滚"

kubeadm 官方文档 [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) 的 "How it works" 一节写明：

> Upgrades the control plane components **or rollbacks if any of them fails to come up**.

即 kubeadm 在**单个控制面节点内**升级控制面静态 Pod 组件时，若某组件**起不来**，会回滚该组件的 manifest。但这是**有限**的：

- 仅针对**单节点内**的控制面静态 Pod 组件（kube-apiserver / controller-manager / scheduler / etcd 等 manifest）；
- 是**尽力而为**的：同一文档 "Recovering from a failure state" 明确写 **"If `kubeadm upgrade` fails and does not roll back ... you can run `kubeadm upgrade` again"**（失败且未回滚时，可**重跑** `kubeadm upgrade`，命令幂等），并说明 `/etc/kubernetes/tmp` 下的 `kubeadm-backup-manifests-*` / `kubeadm-backup-etcd-*` 备份在 **"if the automatic rollback does not work"（自动回滚不生效）** 时需**人工恢复**到 `/etc/kubernetes/manifests` / `/var/lib/etcd`；
- **不**回滚 kubelet/kubeadm 二进制、kubeadm-config ConfigMap、多节点整体状态。

### 3.2 kubekey 作为编排工具：多节点/多阶段，无集群级自动回滚

kubekey 在 kubeadm 之上做**多节点、多阶段**编排（先 etcd、再控制面逐节点、再 worker 并行、再 coredns 等）。即便 kubeadm 在单节点内有上述"组件级尽力回滚"，kubekey **整体**升级失败时：

- 不会把**整个集群**回退到升级前版本；
- 不会自动恢复已升级节点的 kubelet/kubeadm 二进制与配置；
- 集群可能处于"部分节点已升级、部分未升级"的中间状态（kk 的幂等设计允许**重跑** `kk upgrade` 继续完成升级，而非回退）。

**结论**：kubekey 升级失败**不会自动回滚**；kubeadm 的"自动回滚"仅限单节点内控制面组件的尽力而为机制，且官方文档也承认可能不生效。

---

## 4. 依据与来源

| 来源 | 类型 | 说明 |
|------|------|------|
| [kubekey Upgrade Cluster 官方文档](https://github.com/kubesphere/kubekey/blob/main/docs/en/installation/upgrade-cluster.md) | 官方文档 | 升级流程说明；失败处理仅"升级前备份以便失败后恢复"，无自动回滚；worker 并行升级、不自动 drain |
| [kubekey v3.1.12 upgrade_cluster.go（升级 pipeline）](https://github.com/kubesphere/kubekey/blob/v3.1.12/cmd/kk/pkg/pipelines/upgrade_cluster.go) | 源码 | 升级为线性模块序列，无回滚模块 |
| [kubekey v3.1.12 pipeline.go（pipeline 执行）](https://github.com/kubesphere/kubekey/blob/v3.1.12/cmd/kk/pkg/core/pipeline/pipeline.go) | 源码 | 模块失败立即返回错误，整个命令失败退出 |
| [kubekey v3.1.12 task_module.go（任务执行）](https://github.com/kubesphere/kubekey/blob/v3.1.12/cmd/kk/pkg/core/module/task_module.go) | 源码 | 任务失败调用 ExecuteRollback，但升级任务未注册 Rollback |
| [kubekey v3.1.12 remote_task.go（任务回滚钩子）](https://github.com/kubesphere/kubekey/blob/v3.1.12/cmd/kk/pkg/core/task/remote_task.go) | 源码 | ExecuteRollback 仅在 Rollback != nil 时执行；升级任务均为 nil |
| [kubekey v3.1.12 rollback/interface.go、base.go](https://github.com/kubesphere/kubekey/blob/v3.1.12/cmd/kk/pkg/core/rollback/interface.go) | 源码 | rollback 仅为框架预留通用接口，默认 Execute 返回 nil，升级未使用 |
| [kubekey v3.1.12 kubernetes/module.go（ProgressiveUpgradeModule）](https://github.com/kubesphere/kubekey/blob/v3.1.12/cmd/kk/pkg/kubernetes/module.go) | 源码 | 升级任务清单（download/pull/syncBinary/upgradeMaster/upgradeWorker/coredns 等），均无 Rollback |
| [kubekey v3.1.12 kubernetes/tasks.go（升级任务实现）](https://github.com/kubesphere/kubekey/blob/v3.1.12/cmd/kk/pkg/kubernetes/tasks.go) | 源码 | UpgradeKubeMaster/UpgradeKubeWorker/KubeadmUpgrade 失败仅返回错误，无回滚逻辑 |
| [kubeadm 官方文档 Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/) | 官方文档 | "How it works"：控制面组件起不来时回滚该组件；"Recovering from a failure state"：失败且未回滚时可重跑，备份需人工恢复 |
| [kubekey Issue #2511（升级失败实例）](https://github.com/kubesphere/kubekey/issues/2511) | 社区 issue | kk v3.1.7 升级失败停在失败步骤，用户人工排查（补镜像）后重试，无自动回滚 |

---

## 5. 对回退方案 §8.4 的表述建议

原 §8.4 第一条：

> - kubeadm 升级失败时会**自动回滚**（kubeadm 内部机制）；

**建议改为**（或拆为两条）：

> - **kubekey 升级失败不会自动回滚**：任一阶段失败时 kk 会停在失败步骤，集群可能处于"部分节点已升级"的中间状态，需**人工排查并重试** `kk upgrade`（kk 幂等，可继续完成升级）；
> - kubeadm 仅在**单节点内**对控制面静态 Pod 组件有**尽力而为**的自动回滚（组件起不来时回滚该组件 manifest），且官方文档承认可能不生效；kubeadm **不提供集群级回滚**；
> - 若需回退到升级前版本，**按 §8.6 回退步骤人工执行**（以 §8.1 升级前备份为基础）。

**要点**：
1. 删除"kubeadm 升级失败会自动回滚"这一笼统表述，避免误导读者以为升级失败后集群会自动回到旧版本；
2. 明确"升级失败 → 人工排查/重试"是默认处理路径（与 §8.2 回退条件"仅业务不兼容才回退"呼应）；
3. 明确"回退是人工动作"，依赖 §8.1 备份（etcd 快照、kubeadm/kubelet 配置、组件镜像/二进制、数据库），与 §8.6 回退步骤衔接。
