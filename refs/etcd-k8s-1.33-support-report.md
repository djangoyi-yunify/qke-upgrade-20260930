# k8s v1.33.4 对 etcd 版本要求的调研报告（etcd v3.5.13 兼容性确认）

> 调研对象：QKE（青云 Kubernetes 引擎）v3.4.0 集群升级到 **k8s v1.33.4**（离线环境），当前 etcd 为 **v3.5.13**（本项目**不升级** etcd）。
> 调研日期：基于 k8s v1.33.4 官方源码 / 官方文档 / 官方配套组件记录。

---

## 1. 调研结论（一句话）

**etcd v3.5.13 完全满足 k8s v1.33.4 对 etcd 版本的要求，不存在兼容性风险，本项目可继续使用 etcd v3.5.13 而不升级。**

- k8s v1.33.4 官方要求 etcd **≥ 3.5.11**（官方文档最低推荐版本 3.5.11+；kubeadm v1.33.4 源码 `MinExternalEtcdVersion = "3.5.11-0"`）。
- etcd **v3.5.13 ≥ 3.5.11**，满足要求。
- k8s v1.33.4 官方"配套/默认"的 etcd 版本为 **3.5.21**（kubeadm `DefaultEtcdVersion = "3.5.21-0"`），但这是"默认部署版本"而非"最低要求"；运行 3.5.x 系列中更高的补丁版本（3.5.13）完全在官方支持范围内。

---

## 2. k8s v1.33.4 对 etcd 版本的要求（官方依据）

### 2.1 官方文档：最低推荐版本 3.5.11+

Kubernetes 官方文档《Operating etcd clusters for Kubernetes》（v1.33 分支）在"前置条件"中明确：

> The minimum recommended etcd versions to run in production are `3.4.29+` and `3.5.11+`.

即：生产环境最低推荐的 etcd 版本为 **3.4.29+ 或 3.5.11+**。

来源：[Kubernetes v1.33 官方文档 Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)（[release-1.33 分支原文](https://raw.githubusercontent.com/kubernetes/website/release-1.33/content/en/docs/tasks/administer-cluster/configure-upgrade-etcd.md)）

### 2.2 kubeadm v1.33.4 源码：最低外部 etcd 版本 3.5.11-0

kubeadm v1.33.4 源码 `cmd/kubeadm/app/constants/constants.go`（v1.33.4 tag）中：

```go
// MinExternalEtcdVersion indicates minimum external etcd version which kubeadm supports
MinExternalEtcdVersion = "3.5.11-0"

// DefaultEtcdVersion indicates the default etcd version that kubeadm uses
DefaultEtcdVersion = "3.5.21-0"

// SupportedEtcdVersion lists officially supported etcd versions with corresponding Kubernetes releases
SupportedEtcdVersion = map[uint8]string{
    30: "3.5.21-0",
    31: "3.5.21-0",
    32: "3.5.21-0",
    33: "3.5.21-0",
}
```

- `MinExternalEtcdVersion = "3.5.11-0"`：kubeadm 支持的外部 etcd **最低版本为 3.5.11**（低于此版本 kubeadm 会拒绝）。
- `DefaultEtcdVersion = "3.5.21-0"`：kubeadm 在 k8s 1.33 中**默认部署**的 etcd 版本为 3.5.21。
- `SupportedEtcdVersion[33] = "3.5.21-0"`：k8s 1.33 官方**配套/测试**的 etcd 版本为 3.5.21。

来源：[kubernetes/kubernetes v1.33.4 `cmd/kubeadm/app/constants/constants.go`](https://raw.githubusercontent.com/kubernetes/kubernetes/v1.33.4/cmd/kubeadm/app/constants/constants.go)

### 2.3 说明：k8s 1.33 官方配套 etcd 版本（3.5.21 → 后续 3.5.24）

- k8s v1.33.0 起官方配套 etcd 为 **3.5.21**（见 [kubernetes/kubernetes issue #131101](https://github.com/kubernetes/kubernetes/issues/131101)）。
- 2025-11-03，k8s 将 1.31/1.32/1.33 的配套 etcd 版本提升到 **3.5.24**（[PR #134861](https://github.com/kubernetes/kubernetes/pull/134861)、[cherry-pick #135018](https://github.com/kubernetes/kubernetes/pull/135018)）。该提升**晚于 v1.33.4 发布**，只影响后续发布的 1.33.x 补丁版本（已核实 v1.33.4、v1.33.5 的 kubeadm 源码仍为 3.5.11 / 3.5.21）。
- 无论 3.5.21 还是 3.5.24，都是"官方配套/默认部署"版本，**不是最低要求**；官方最低要求始终是 3.5.11+。

---

## 3. etcd v3.5.13 是否满足（结论）

| 判断维度 | 结论 |
|---|---|
| k8s v1.33.4 官方最低推荐 etcd 版本 | **3.5.11+**（官方文档） |
| kubeadm v1.33.4 最低外部 etcd 版本 | **3.5.11-0**（源码常量） |
| 本项目 etcd 版本 | **v3.5.13** |
| v3.5.13 是否 ≥ 3.5.11 | **是**（3.5.13 > 3.5.11） |
| 结论 | **满足要求，无兼容性风险** |

**明确结论：etcd v3.5.13 用于 k8s v1.33.4 不存在兼容性风险。**

- v3.5.13 高于官方最低推荐版本 3.5.11+，也高于 kubeadm v1.33.4 的最低外部 etcd 版本 3.5.11-0。
- v3.5.13 属于 etcd 3.5.x 系列，与 k8s 1.33 官方配套的 3.5.21 / 3.5.24 同属一个 minor 系列，API / 存储格式兼容。
- k8s 官方对 etcd 的版本要求是"最低版本"而非"精确版本"，运行高于最低要求的 3.5.x 补丁版本完全在支持范围内。

### 3.1 注意事项（供后续参考，不影响 v1.33.4 结论）

- 若本项目**后续**将 k8s 升级到 **2025-11-03 之后发布的 1.33.x 补丁**（约 v1.33.9 及之后），kubeadm 的最低外部 etcd 版本已提升到 **3.5.24-0**，届时 etcd v3.5.13 将不满足 kubeadm 的最低要求，需考虑升级 etcd。
- 本项目当前目标版本为 **v1.33.4**，不受此影响。

---

## 4. 对文档的意义（供 upgrade-plan 修订参考）

1. **"kubekey 默认 etcd v3.5.24 而本项目用 v3.5.13"不构成风险**：k8s v1.33.4 对 etcd 的最低要求是 3.5.11，v3.5.13 满足要求；kubekey 默认的 3.5.24 只是"默认部署版本"，不是"最低要求"。**相关风险提示可去掉**。
2. **附录中 etcd 升级命令应标注为"可选升级项"（非必须）**：本项目保持 etcd v3.5.13 即可满足 k8s v1.33.4 要求，etcd 升级不是本次 k8s 升级的前置条件。

---

## 5. 参考资料

- [Kubernetes 官方文档：Operating etcd clusters for Kubernetes（v1.33）](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/) —— 最低推荐 etcd 版本 3.4.29+ / 3.5.11+
- [Kubernetes v1.33 分支该文档原文（release-1.33）](https://raw.githubusercontent.com/kubernetes/website/release-1.33/content/en/docs/tasks/administer-cluster/configure-upgrade-etcd.md)
- [kubernetes/kubernetes v1.33.4 `cmd/kubeadm/app/constants/constants.go`](https://raw.githubusercontent.com/kubernetes/kubernetes/v1.33.4/cmd/kubeadm/app/constants/constants.go) —— MinExternalEtcdVersion=3.5.11-0、DefaultEtcdVersion=3.5.21-0、SupportedEtcdVersion[33]=3.5.21-0
- [kubernetes/kubernetes v1.33.5 `cmd/kubeadm/app/constants/constants.go`](https://raw.githubusercontent.com/kubernetes/kubernetes/v1.33.5/cmd/kubeadm/app/constants/constants.go) —— 核实 v1.33.5 仍为 3.5.11 / 3.5.21
- [kubernetes/kubernetes issue #131101：Land etcd v3.5.21 in K8s 1.33.0](https://github.com/kubernetes/kubernetes/issues/131101)
- [kubernetes/kubernetes PR #134861：Bump supported etcd version to v3.5.24 for v1.31/v1.32/v1.33](https://github.com/kubernetes/kubernetes/pull/134861)
- [kubernetes/kubernetes PR #135018：cherry-pick 到 release-1.33（2025-11-03 合并）](https://github.com/kubernetes/kubernetes/pull/135018)
- [Kubernetes Releases（版本支持周期）](https://kubernetes.io/releases/)
