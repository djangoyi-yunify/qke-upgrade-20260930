# k8s feature-gates 获取与删除方法调研报告

> 调研对象：QKE v3.4.0 集群，k8s **v1.31.2 → v1.33.4** 升级（离线环境）。
> 用途：为 [docs/upgrade-plan.md](../docs/upgrade-plan.md) §5.1 ⑤ feature-gates 提供详细参考（"为什么"、背景、来源、完整命令）；主文档只写"怎么做"，本报告承载"为什么"与细节。
> 调研日期：基于 k8s v1.33 官方文档（`release-1.33` 分支）与用户提供的文档片段整理核实。

---

## 1. 概述

**feature-gate（特性门控）** 是 k8s 各组件（kube-apiserver / kube-controller-manager / kube-scheduler / kubelet）启动时通过 `--feature-gates=Key=Value,...` 参数或配置文件中的 `featureGates` 字段显式开启/关闭的特性开关。

- **显式配置**：指在 kubeadm 配置、组件启动参数、kubelet 配置文件中**明确写出**的 feature-gate（区别于组件内置的默认值）。
- **为什么升级前要清理**：k8s 1.33 已移除 `TTLAfterFinished`、`ExpandCSIVolumes` 等 feature-gate。若这些已移除的 gate 仍以 `--feature-gates=...` 显式传入，**新版本组件启动时会因"未知 feature-gate"报错而拒绝启动**（kube-apiserver / kubelet 等对未知 gate 是 fatal 错误）。因此必须在 `kk upgrade` 之前把已移除的 gate 从所有显式配置位置删除。
- **例外**：如果显式配置的 feature-gate 只有 `RotateKubeletServerCertificate`（1.33 中仍存在且默认 true），则**不需要做任何操作**（见 §4）。

---

## 2. 获取显式配置的 feature-gate

### 2.1 可能存在的所有位置

| # | 位置 | 类型 | 字段 / 参数 |
|---|------|------|-------------|
| 1 | configmap `kubeadm-config`（kube-system 命名空间） | 集群内 | `data.ClusterConfiguration` 中：`apiServer.extraArgs.feature-gates`、`controllerManager.extraArgs.feature-gates`、`scheduler.extraArgs.feature-gates` |
| 2 | configmap `kubelet-config-1.31`（kube-system 命名空间） | 集群内 | `data.kubelet` 中：`featureGates` |
| 3 | 第一个 master 节点 `/etc/kubernetes/kubeadm-config.yaml` | 节点文件 | `apiServer.extraArgs.feature-gates`、`controllerManager.extraArgs.feature-gates`、`scheduler.extraArgs.feature-gates`、`KubeletConfiguration.featureGates` |
| 4 | 各节点 `/etc/kubernetes/manifests/kube-{apiserver,controller-manager,scheduler}.yaml` | 节点文件 | `spec.containers[0].command` 中的 `--feature-gates=xxx` |
| 5 | 各节点 `/var/lib/kubelet/config.yaml` | 节点文件 | `featureGates` |

> **说明**：
> - 位置 1、2 是**集群内 kubeadm 的"源配置"**（configmap），升级工具（kk/kubeadm）在升级时会据此重新生成组件配置；位置 3、4、5 是**节点上当前生效的配置**。两者必须保持一致，否则下一次 kubeadm 操作会用 configmap 覆盖节点文件（或反之）。
> - 位置 2 的 configmap 名带**小版本号**：本项目 k8s 为 **1.31.2**，故为 `kubelet-config-1.31`（用户片段中的 `kubelet-config-1.23` 是 k8s 1.23 场景的旧名，本项目不适用）。k8s 1.25 起 kubeadm 还会额外生成一个**无版本号**的 `kubelet-config` configmap（`UnversionedKubeletConfigMap` 已 GA），若存在也建议一并检查。
> - 位置 4 的 static pod manifest 由 kubelet 直接 watch，改动后 kubelet 会自动重启对应组件；位置 5 是 kubelet 实际读取的配置文件。

### 2.2 检查命令

**① configmap `kubeadm-config`（控制面组件 feature-gates）**

```bash
kubectl -n kube-system get cm kubeadm-config -o yaml | grep -A5 feature-gates
# 或只看 ClusterConfiguration 字段：
kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' | grep -A5 feature-gates
```

**② configmap `kubelet-config-1.31`（kubelet featureGates）**

```bash
kubectl -n kube-system get cm kubelet-config-1.31 -o yaml | grep -A5 featureGates
# 或只看 kubelet 字段：
kubectl -n kube-system get cm kubelet-config-1.31 -o jsonpath='{.data.kubelet}' | grep -A5 featureGates
# 若存在无版本号 configmap，一并检查：
kubectl -n kube-system get cm kubelet-config -o yaml | grep -A5 featureGates
```

**③ 第一个 master 节点 `/etc/kubernetes/kubeadm-config.yaml`**

```bash
grep -A5 feature-gates /etc/kubernetes/kubeadm-config.yaml   # 控制面组件
grep -A5 featureGates  /etc/kubernetes/kubeadm-config.yaml   # kubelet
```

**④ 各节点 static pod manifest（控制面组件）**

```bash
grep -- '--feature-gates' /etc/kubernetes/manifests/kube-apiserver.yaml
grep -- '--feature-gates' /etc/kubernetes/manifests/kube-controller-manager.yaml
grep -- '--feature-gates' /etc/kubernetes/manifests/kube-scheduler.yaml
```

**⑤ 各节点 kubelet 配置文件**

```bash
grep -A5 featureGates /var/lib/kubelet/config.yaml
```

> **补充检查点**（可选）：kubelet 的 systemd 单元 `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf` 通常只设置 `KUBELET_KUBECONFIG_ARGS` 与 `KUBELET_CONFIG_ARGS`（指向 `/var/lib/kubelet/config.yaml`），一般**不会**直接写 `--feature-gates`；若该文件出现 `--feature-gates` 也需一并清理。

---

## 3. 删除 feature-gate 的操作步骤

> 以下步骤基于用户提供的文档片段整理，并针对本项目（k8s 1.31.2 → 1.33.4）做了修正与补充。**在 `kk upgrade` 之前完成**，且**所有 master 节点**都要改（不只是第一个 master）。

### 步骤 1：修改 configmap `kubeadm-config`

```bash
kubectl edit cm -n kube-system kubeadm-config
```

删除 `data.ClusterConfiguration` 中的字段：

- `apiServer.extraArgs.feature-gates`
- `controllerManager.extraArgs.feature-gates`
- `scheduler.extraArgs.feature-gates`

### 步骤 2：修改 configmap `kubelet-config-1.31`

> ⚠️ 片段中写的是 `kubelet-config-1.23`，本项目 k8s 为 **1.31.2**，实际应为 **`kubelet-config-1.31`**。

```bash
kubectl edit cm -n kube-system kubelet-config-1.31
```

删除 `data.kubelet` 中的字段：`featureGates`。

> ⚠️ **注意**：片段第 2 步中"添加部分配置"（`containerLogMaxSize: "100Mi"`、`containerLogMaxFiles: 5`、`imageGCHighThresholdPercent: 70`、`imageGCLowThresholdPercent: 50`、`tlsCipherSuites: [...]`）**与 feature-gate 删除无关**，属于可选的 kubelet 其他配置调整，**不要**与 feature-gate 清理混为一谈。若本次升级只做 feature-gate 清理，**不要添加**这些字段。

### 步骤 3：删除第一个 master 节点 `/etc/kubernetes/kubeadm-config.yaml` 中的字段

- `apiServer.extraArgs.feature-gates`
- `controllerManager.extraArgs.feature-gates`
- `scheduler.extraArgs.feature-gates`
- `KubeletConfiguration.featureGates`

### 步骤 4~6：修改各节点 static pod manifest

删除各节点 `/etc/kubernetes/manifests/` 下三个文件中 `spec.containers[0].command` 里的 `--feature-gates=xxx` 参数：

```bash
# 每个 master 节点上执行
sed -i '/--feature-gates=/d' /etc/kubernetes/manifests/kube-apiserver.yaml
sed -i '/--feature-gates=/d' /etc/kubernetes/manifests/kube-controller-manager.yaml
sed -i '/--feature-gates=/d' /etc/kubernetes/manifests/kube-scheduler.yaml
```

> 若希望保留部分仍有效的 gate，请用 `kubectl edit` / `vi` 手动删除 `--feature-gates=` 中**已移除的项**，而不是整行删除。

### 步骤 7：删除各节点 `/var/lib/kubelet/config.yaml` 中的 `featureGates`

```bash
# 每个节点上执行，删除 featureGates 字段（含其下的键值）
vi /var/lib/kubelet/config.yaml
```

### 重启 / 生效说明

| 改动位置 | 是否需要重启 | 说明 |
|----------|--------------|------|
| configmap `kubeadm-config` / `kubelet-config-1.31` | 不需要立即重启 | configmap 是 kubeadm 的**源配置**，kubeadm 不会自动应用；其作用是在 `kk upgrade` 时**不再重新写入**已删除的 gate。 |
| `/etc/kubernetes/manifests/kube-*.yaml` | **自动重启** | kubelet 持续 watch 该目录，文件变化后**自动重启**对应 static pod（kube-apiserver / kube-controller-manager / kube-scheduler）。注意：删除 apiserver 的 `--feature-gates` 会触发 apiserver 重启，期间 API 短暂不可用。 |
| `/var/lib/kubelet/config.yaml` | **需要手动重启 kubelet** | kubelet 启动时读取该文件，改动后执行：`systemctl daemon-reload && systemctl restart kubelet`。 |

**推荐顺序**：先改 configmap（源配置，步骤 1~2）→ 再改节点文件（当前生效配置，步骤 3~7），全部在 `kk upgrade` 之前完成，保证升级工具与当前运行配置一致。

### 注意事项

1. **只删已移除的 gate**：1.33 中仍存在的 gate（如 `RotateKubeletServerCertificate`）无需删除；若显式配置的只有它，则**整个步骤可跳过**。
2. **残留后果**：已移除的 gate（`TTLAfterFinished`、`ExpandCSIVolumes` 等）残留会导致新版本 kube-apiserver / kubelet 启动失败（未知 feature-gate 为 fatal 错误）。
3. **备份**：修改前建议备份 configmap 与节点文件（`kubectl get cm -n kube-system kubeadm-config -o yaml > kubeadm-config.bak.yaml` 等），便于回退。
4. **所有 master 节点都要改**：static pod manifest 与 kubelet 配置是逐节点生效的，不能只改第一个 master。
5. **qke-cli 不会自动清理**：qke-cli 的 `removeDeprecatedArgs` 只处理 v1.25/v1.27 的旧参数，**不会自动清理 1.33 移除的 gate**，必须手动执行上述步骤。

---

## 4. 相关 feature-gate 在 k8s 1.33 的状态

以下状态均以 k8s 官方文档 `release-1.33` 分支的 feature-gate 页面为准（见 §5 链接）。

| feature-gate | 演进 | 1.33 状态 | 是否需要处理 |
|--------------|------|-----------|--------------|
| **RotateKubeletServerCertificate** | 1.7 alpha（默认 false）→ **1.12 beta（默认 true）**，此后一直存在 | **仍存在，默认 true** | **不需要删除**。若显式配置的只有它，整个清理步骤可跳过 |
| **TTLAfterFinished** | 1.12 alpha → 1.21 beta（默认 true）→ 1.23 stable（默认 true）→ 之后移除 | **已移除**（`removed: true`） | **必须删除**，残留会导致组件启动失败 |
| **ExpandCSIVolumes** | 1.14 alpha → 1.16 beta（默认 true）→ 1.24 stable（默认 true）→ 之后移除 | **已移除**（`removed: true`） | **必须删除**，残留会导致组件启动失败 |

**要点**：

- `RotateKubeletServerCertificate`：控制 kubelet 服务端 TLS 证书轮换，1.12 起默认开启，1.33 中仍是有效 gate，**保留即可**。
- `TTLAfterFinished`：控制 TTL 控制器清理已完成的 Job 等资源，功能已 GA 并固化，gate 本身在 1.33 中**不再存在**。
- `ExpandCSIVolumes`：控制 CSI 卷扩容，功能已 GA 并固化，gate 本身在 1.33 中**不再存在**。
- 除上述三个外，k8s 1.33 还移除了 `APIListChunking`、`AdmissionWebhookMatchConditions`、`AggregatedDiscoveryEndpoint`、`EfficientWatchResumption`、`KubeProxyDrainingTerminatingNodes`、`RemainingItemCount`、`ValidatingAdmissionPolicy`、`WatchBookmark` 等 gate（官方在 v1.33/1.32 文档中统一标记 `removed: true`）。**若集群显式配置了这些 gate，同样需要删除**——因此务必先按 §2 检查实际配置，再决定删哪些。

---

## 5. 参考资料

- [Feature Gates | Kubernetes（v1.33 官方文档）](https://v1-33.docs.kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
- [TTLAfterFinished feature gate（k8s website release-1.33，`removed: true`）](https://raw.githubusercontent.com/kubernetes/website/release-1.33/content/en/docs/reference/command-line-tools-reference/feature-gates/TTLAfterFinished.md)
- [ExpandCSIVolumes feature gate（k8s website release-1.33，`removed: true`）](https://raw.githubusercontent.com/kubernetes/website/release-1.33/content/en/docs/reference/command-line-tools-reference/feature-gates/ExpandCSIVolumes.md)
- [RotateKubeletServerCertificate feature gate（k8s website release-1.33，1.12 起默认 true）](https://raw.githubusercontent.com/kubernetes/website/release-1.33/content/en/docs/reference/command-line-tools-reference/feature-gates/RotateKubeletServerCertificate.md)
- [kubernetes/website commit：Update removed feature gates in v1.33/1.32（1.33 移除 gate 清单）](https://github.com/kubernetes/website/commit/912378c766f9d9be91a1a1e8c42c66111942a73a)
- [Configuring each kubelet in your cluster using kubeadm（v1.33，kubelet-config configmap 机制）](https://v1-33.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/)
- [Customizing components with the kubeadm API（v1.33，extraArgs 机制）](https://v1-33.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/control-plane-flags/)
- 用户提供文档片段：`6.2.3 删除 feature-gate 参数`（本项目调研基础，已按 1.31.2 → 1.33.4 场景修正）
