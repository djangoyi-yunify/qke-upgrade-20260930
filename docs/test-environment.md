# 测试环境情况

> 本文档记录 QKE 升级方案的**测试环境情况**（实测数据、验证结果、观察项），与 [upgrade-plan.md](upgrade-plan.md)（可操作步骤）分离。
> 决策思路见 [decisions-rationale.md](decisions-rationale.md)。
> 更新日期：2026-09-30

## 1. 测试环境概览

存在两个测试环境：

- **测试环境 A：无 KSE** —— 当前方案处理对象；
- **测试环境 B：安装 KSE 4.1.2** —— 后续补充。

**访问方式**：

- 跳板机：`ssh dev-pek3`（root@139.198.21.102:50022）；
- 跳板机 → master1：`ssh -i /root/.ssh/id_qingcloud_pek3_djangoyi root@<master1-ip>`；
- 不同测试集群的 master1 IP 不同。

## 2. 测试环境 A：无 KSE

### 2.1 实例信息

| 项 | 值 |
|----|-----|
| qke-cli | v3.3.0-patch.4（升级前）→ **v3.4.0-patch.1**（第一阶段升级后） |
| Kubernetes | v1.27.10 → **v1.33.4** |
| containerd | v1.7.13 |
| crictl | v1.29.0 |
| etcd | v3.5.13（未升级） |
| kubekey（installer 内 kk） | v3.1.7（升级前）→ **v3.1.12**（第一阶段升级后） |
| OS | Ubuntu 20.04.5 LTS |
| 节点拓扑 | 3 master + 3 worker，全部 Ready |
| 网络插件 | calico（calico-node / calico-kube-controllers） |
| 存储插件 | csi-qingcloud（controller + node）——**QKE 3.3.0 原生组件** |
| KSE/KubeSphere | 未安装（无相关 namespace） |
| 私有 OCI 注册服务 | `registry.qingcloud.com/kubesphere/` |
| installer 下载源 | `https://qkeplus.pek3b.qingstor.com` |

> **版本说明**：上表 `qke-cli` 版本为实测的 qke-cli 二进制版本。**QKE 平台**是一个整体，包含 qke-apiserver、qke-cli、qke 前端、qke-installer 等多个组件，各组件版本需分别确认，不能以 qke-cli 版本代表整个 QKE 平台版本。

> **kubekey 版本说明**：测试环境中用户**调整了 installer 包内的 kubekey 版本**——QKE v3.3.0 从 **3.1.0 → 3.1.7**，QKE v3.4.0 从 **3.1.7 → 3.1.12**；因此实测该实例升级前 kk 为 v3.1.7、第一阶段升级后为 v3.1.12。**真实离线环境不同**（见 decisions-rationale.md §2）。

> **历史记录**：升级前检查在 master1 = 172.22.8.17 的实例上完成（节点 IP 段 .13~.18）；当前测试集群 master1 = 172.22.8.23（节点 IP 段 .19~.24）。

### 2.2 第一阶段验证结果（QKE 平台升级 3.3.0 → 3.4.0）

| 验证项 | 升级前 | 升级后 | 结论 |
|--------|--------|--------|------|
| qke-cli | v3.3.0-patch.4 | **v3.4.0-patch.1** | ✅ 平台已升级 |
| installer 内 kk | v3.1.7 | **v3.1.12** | ✅ 更新后的 installer 包已生效 |
| k8s 版本 | v1.27.10 | **v1.27.10** | ✅ 保持不变（未勾选"升级 K8s"） |
| 节点状态 | 全部 Ready | 全部 Ready | ✅ |
| kube-system pod | 全部 Running | 全部 Running | ✅ 无异常 |
| 平台组件 | — | cloud-controller-manager、csi-qingcloud-controller、qke-operator | ✅ **QKE 3.3.0 原生组件**（新建实例 172.22.8.7 核对确认，非升级新增） |

**installer 包内容变化（实测）**：

- `kubekey/kube/` 新增：v1.29.10、v1.29.15、v1.31.2、v1.32.8、v1.33.4；
- `kubekey/etcd/` 新增：v3.5.24。

**问题**：未遇到问题。

### 2.3 第二阶段验证结果（控制台升级 k8s 1.27.10 → 1.29.10 → 1.31.2）

| 验证项 | 1.27.10 → 1.29.10 后 | 1.29.10 → 1.31.2 后 | 结论 |
|--------|---------------------|---------------------|------|
| 节点版本 | v1.29.10 | **v1.31.2** | ✅ 全部节点 Ready |
| kube-system pod | 全部 Running | 全部 Running | ✅ 无异常 |
| 非 Running pod | 无 | 无 | ✅ |
| coredns 镜像 | — | `coredns:1.9.3` | ✅ 实测未变（kk 不自动升级 coredns） |
| etcd | v3.5.13 | v3.5.13 | 未升级 |
| containerd | v1.7.13 | v1.7.13 | 不变 |

**问题**：未遇到问题。

> **注意**：k8s 1.28+ 已移除 `kubectl version --short` 参数，升级到 1.29+ 后该命令会报错，改用 `kubectl version` 或 `kubectl version -o json`。

### 2.4 第三阶段验证结果（手动 kk 升级 1.31.2 → 1.33.4）

| 验证项 | 升级前 | 升级后 | 结论 |
|--------|--------|--------|------|
| 节点版本 | v1.31.2 | **v1.33.4** | ✅ 全部节点 Ready |
| kube-system pod | 全部 Running | 全部 Running | ✅ 无异常 |
| 非 Running pod | 无 | 无 | ✅ |
| etcd | v3.5.13 | v3.5.13 | ✅ 未升级（满足 1.33.4 要求） |
| containerd | v1.7.13 | v1.7.13 | ✅ 不变 |
| coredns 镜像 | `coredns:1.9.3` | `coredns:1.9.3` | ✅ 实测未变（kk 不自动升级 coredns） |
| `qke-cluster-config` | version=v1.31.2, pre_version=v1.29.10 | version=**v1.33.4**, pre_version=**v1.31.2** | ✅ 已调整 |
| 数据库 `qke_db_cluster.kubernetes_version` | v1.31.2 | **v1.33.4** | ✅ 已调整（专业人员操作） |

**问题**：未遇到问题。

### 2.5 平台自洽验证（实例 master1 172.22.8.7）

该实例已完成整个升级过程，并在升级后**新增 1 个 worker 节点**（worker 数量 3 → 4），用于验证平台自洽性。

| 检查项 | 实测结果 | 结论 |
|--------|----------|------|
| 节点 | 3 master + **4 worker**（worker-s004 新增），全部 v1.33.4 | ✅ |
| `config-sample.yaml` 的 `kubernetes.version` | **v1.33.4** | ✅ 未被错误渲染成 1.31.2 |
| `qke-cluster-config` 的 `.kubernetes.version` | **v1.33.4** | ✅ |
| `qke-cluster-config` 的 `.kubernetes.pre_version` | **v1.31.2** | ✅ |
| `qke-cluster-config` 节点列表 | 含新增的 worker-s004 | ✅ 新增节点已纳入 |
| QKE web 控制台显示 k8s 版本 | **1.33.4** | ✅ 符合预期 |

**结论**：平台自洽目标达成——升级后新增 worker 节点，`config-sample.yaml` 仍正确显示 v1.33.4，垂直/水平扩容、修改配置参数时不会错误渲染 k8s 版本，无未知风险。

### 2.6 新建 QKE 3.3.0 实例核对（master1 172.22.8.7）

用于核对问题 #1（平台组件是否原生存在）：

| 组件 | 状态 | 结论 |
|------|------|------|
| cloud-controller-manager | 1/1 Running | ✅ **QKE 3.3.0 原生存在** |
| csi-qingcloud-controller | 1/1 Running | ✅ **QKE 3.3.0 原生存在** |
| qke-operator | 1/1 Running | ✅ **QKE 3.3.0 原生存在** |

**其他核对项**：

| 项 | 实测值 | 说明 |
|----|--------|------|
| k8s 版本 | v1.27.10 | ✅ 新建实例正确 |
| installer 内 kk | v3.1.7 | 由已调整（3.1.0→3.1.7）的 installer 创建 |
| coredns 镜像 | `coredns:1.9.3` | 1.9.x，与"kk 不自动升级 coredns"一致 |
| `config-sample.yaml` | `kubernetes.version: v1.27.10` | ✅ 正确 |
| `qke-cluster-config` | `version: v1.27.10` | ✅ 正确 |
| 节点 | 3 master + 3 worker，全部 Ready | ✅ |

## 3. 测试环境 B：安装 KSE 4.1.2（后续处理）

- 待准备：QKE 实例（安装 KSE 4.1.2）的 master1 IP、拓扑、组件信息；
- 待确认：KSE 4.1.2 场景下升级 k8s 的差异点（KSE 组件兼容性、`qke_db_component.k8s_versions` 调整、KSE 功能验证等）；
- **当前方案先处理无 KSE 情况，KSE 4.1.2 场景后续补充**。
