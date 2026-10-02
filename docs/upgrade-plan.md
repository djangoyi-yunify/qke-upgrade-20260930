# QKE v3.3.0 集群 Kubernetes 升级方案

> **状态**：迭代中（第 14 轮：子 agent 调研 k8s v1.33.4 对 coredns 的要求——**非"必须"但"强烈建议"升级到 v1.12.0**；§6.1⑦ 更新为调研结论；新增调研报告 refs/k8s-1.33-coredns-compatibility-report.md）
> **目标**：QKE v3.3.0 集群（k8s v1.27.10）升级到 k8s **v1.33.4**，含升级方法、错误处理、回退方法。
> **环境**：离线（无法访问互联网）。

---

## 1. 方案概述

### 1.1 背景与目标

- **背景**：已有 QKE 集群存在升级 Kubernetes 版本的需求。
- **目标**：将集群 k8s 从 v1.27.10 升级到 **v1.33.4**。
- **约束**：
  - 集群运行环境**无法访问互联网**（离线环境）；
  - QKE v3.3.0 最高支持 k8s **v1.27.10**，无法直接升级更高 k8s 版本；
  - QKE 底层使用 KubeKey（`kk`）升级 k8s。

**适用范围（两个测试环境）**：

- **测试环境 A：无 KSE** —— 当前方案处理对象（§2.2.1）；
- **测试环境 B：安装 KSE 4.1.2** —— 后续补充（§2.2.2），KSE 场景的差异点（组件兼容性、数据库字段、功能验证）在相关章节标注「KSE 4.1.2 场景」。

### 1.2 整体升级步骤与升级路径

**整体升级步骤（3 步）**：

1. **导入 QKE v3.4.0**（此工作由**青云专业人员**处理）；
2. **更新离线环境中 QKE v3.4.0 的 qke-installer 文件包**（参考 `/workspace/qke-artifacts/` 中的步骤，见 §3.3）；
3. **执行具体的升级步骤**（三段式，见下）。

> **为什么必须更新 qke-installer 包**：
> - QKE v3.3.0 使用的 kubekey 版本为 **3.1.0**；
> - QKE v3.4.0 使用的 kubekey 版本为 **3.1.7**；
> - 但 **kubekey 3.1.7 不支持升级 k8s 到 v1.33.4**，因此**必须更换 QKE v3.4.0 的 qke-installer 包中的 kubekey，使其版本为 3.1.12**；
> - 基于以上考虑，才有制作和更新 QKE v3.4.0 qke-installer 包的动作。

**三段式升级路径**：

```
① QKE 平台升级：v3.3.0 → v3.4.0（保持 k8s v1.27.10 不变）
   ↓
② 控制台升级 k8s：v1.27.10 → v1.29.10 → v1.31.2（QKE 3.4.0 控制台支持）
   ↓
③ 手动 kk 升级 k8s：v1.31.2 → v1.33.4（kubekey 3.1.12 支持的最高版本）
```

**各阶段要点**：

| 阶段 | 内容 | 关键约束 |
|------|------|----------|
| ① QKE 平台升级 | v3.3.0 → v3.4.0 | 保持 k8s v1.27.10 不变 |
| ② 控制台升级 k8s | v1.27.10 → v1.29.10 → v1.31.2 | QKE 3.4.0 控制台支持 1.27.10/1.29.10/1.31.2，升级顺序固定 |
| ③ 手动 kk 升级 k8s | v1.31.2 → v1.33.4 | kubekey 3.1.12 为 3.1.x 线最高 tag，支持到 v1.33.4 |

> **中间版本说明（已确认，2026-09-30）**：
> - **控制台升级路径**（1.27.10 → 1.29.10 → 1.31.2）是 QKE 控制台提供的路径，在 kubekey 支持的 k8s 版本内，无问题；
> - qke-artifacts 资源包准备的控制面镜像 **v1.29.15 / v1.32.8 / v1.33.4** 是 **kubekey 3.1.12 升级阶段**（1.31.2 → 1.33.4）所需的：kubekey 执行升级时**逐 minor 版本**进行，会拉取各中间版本镜像（这是 kubekey 本身的升级逻辑，与是否手动调用无关）；
> - 同时，资源包已**预下载 kubekey 升级过程中自动下载的各 k8s 版本所需文件**（etcd、calicoctl、kube 二进制、helm 等），避免离线环境因文件下载故障而停顿。

### 1.3 版本兼容性矩阵

| 项 | 当前 | 目标 | 说明 |
|----|------|------|------|
| QKE 平台 | v3.3.0 | v3.4.0 | 3.4.0 最高支持 k8s v1.31.2 |
| Kubernetes | v1.27.10 | v1.33.4 | 3.4.0 控制台支持 1.27.10/1.29.10/1.31.2；1.33.4 为 kubekey 3.1.12 上限 |
| kubekey | **真实环境**：QKE 3.3.0 → **3.1.0**；QKE 3.4.0 → **3.1.7**<br>**测试环境**：已调整为 3.1.7 / 3.1.12 | **3.1.12** | 3.1.7 不支持升级到 v1.33.4，**必须更换为 3.1.12**（3.1.x 线最高 tag） |
| etcd | v3.5.13 | v3.5.24（可选） | kubekey 3.1.12 默认 etcd v3.5.24；1.31+ 推荐 3.5.24；**实测未升级，3.5.13 满足要求**（见 §6.1③） |
| containerd | v1.7.13 | v1.7.13（不变） | 满足 1.27~1.33 全程要求 |
| coredns | 1.9.2（QKE 3.3.0） | 1.9.x（实测未变） | **coredns 版本由 kubekey 版本决定**（非 QKE）；**kk 升级时不自动升级 coredns**；是否需升级由 k8s 版本要求决定（见 §10.2） |
| OS | Ubuntu 20.04.5 | 不变 | |

---

## 2. 环境信息

### 2.1 集群现状

| 项 | 值 |
|----|-----|
| QKE 平台版本 | v3.3.0 |
| Kubernetes | v1.27.10 |
| containerd | v1.7.13 |
| etcd | v3.5.13 |
| OS | Ubuntu 20.04.5 |
| 网络插件 | 待确认（calico / hostnic？） |
| 存储插件 | 待确认（csi-qingcloud / hydrofs？） |
| KSE/KubeSphere | 待确认（是否启用、版本） |

### 2.2 测试环境

> 来源：2026-09-30 通过跳板机（dev-pek3）只读检查 master1（172.22.8.17）实测。
> **说明**：存在两个测试环境——**测试环境 A（无 KSE）** 与 **测试环境 B（安装 KSE 4.1.2）**。当前方案先处理**无 KSE** 的情况（测试环境 A），KSE 4.1.2 场景（测试环境 B）后续补充。

**访问方式**：

- 跳板机：`ssh dev-pek3`（root@139.198.21.102:50022）；
- 跳板机 → master1：`ssh -i /root/.ssh/id_qingcloud_pek3_djangoyi root@<master1-ip>`；
- 不同测试集群的 master1 IP 不同。

#### 2.2.1 测试环境 A：无 KSE（当前处理对象）

**访问方式**：

- 跳板机：`ssh dev-pek3`（root@139.198.21.102:50022）；
- 跳板机 → master1：`ssh -i /root/.ssh/id_qingcloud_pek3_djangoyi root@<master1-ip>`；
- 不同测试集群的 master1 IP 不同。

**当前测试集群（master1 = 172.22.8.23）实测信息**：

| 项 | 值 |
|----|-----|
| qke-cli | v3.3.0-patch.4（升级前）→ **v3.4.0-patch.1**（第一阶段升级后） |
| Kubernetes | v1.27.10（client/server/kubeadm/kubelet 一致） |
| containerd | v1.7.13 |
| crictl | v1.29.0 |
| etcd | v3.5.13 |
| kubekey（installer 内 kk） | 实测 v3.1.7（升级前）→ **v3.1.12**（第一阶段升级后） |
| OS | Ubuntu 20.04.5 LTS |
| 节点拓扑 | 3 master（172.22.8.22/23/24）+ 3 worker（172.22.8.19/20/21），全部 Ready |
| 网络插件 | calico（calico-node / calico-kube-controllers） |
| 存储插件 | 第一阶段升级后新增 csi-qingcloud（controller + node） |
| KSE/KubeSphere | 未安装（无相关 namespace） |
| 私有 OCI 注册服务 | `registry.qingcloud.com/kubesphere/`（kube-apiserver 镜像地址确认） |
| installer 下载源 | `https://qkeplus.pek3b.qingstor.com`（conf.ini [installer] url） |
| 系统组件 | kube-system 全部 Running，无异常 pod |

> **版本说明**：上表 `qke-cli` 版本为实测的 qke-cli 二进制版本。**QKE 平台**是一个整体，包含 qke-apiserver、qke-cli、qke 前端、qke-installer 等多个组件，各组件版本需分别确认，不能以 qke-cli 版本代表整个 QKE 平台版本。

> **kubekey 版本说明（已澄清，2026-09-30）**：
> - **测试环境**：用户**调整了 installer 包内的 kubekey 版本**——QKE v3.3.0 从 **3.1.0 → 3.1.7**，QKE v3.4.0 从 **3.1.7 → 3.1.12**；因此实测该实例升级前 kk 为 v3.1.7、第一阶段升级后为 v3.1.12；
> - **真实离线环境不同**：QKE v3.3.0 为 **3.1.0**、QKE v3.4.0 为 **3.1.7**（原始版本），需按 §3.3 更新为 **3.1.12**。

> **历史记录**：升级前检查在 master1 = 172.22.8.17 的实例上完成（节点 IP 段 .13~.18）；当前测试集群 master1 = 172.22.8.23（节点 IP 段 .19~.24）。

**测试结果**：第一阶段（QKE 平台升级）已完成并验证通过（见 §4.2）；第二、三阶段待用户执行后补充。

#### 2.2.2 测试环境 B：安装 KSE 4.1.2（后续处理）

- 待准备：QKE 实例（安装 KSE 4.1.2）的 master1 IP、拓扑、组件信息；
- 待确认：KSE 4.1.2 场景下升级 k8s 的差异点（KSE 组件兼容性、`qke_db_component.k8s_versions` 调整、KSE 功能验证等，见 §6.1⑧ 与 §9）；
- **当前方案先处理无 KSE 情况，KSE 4.1.2 场景后续补充**。

### 2.3 生产环境

- 待确认：集群拓扑（节点数、master/worker 分布）、业务影响、维护窗口、参与人员。

### 2.4 离线环境

> 来源：`/workspace/qke-artifacts/`（QKE 3.4.0 离线升级资源包）与 `/workspace/qke-cli/manual-upgrade-1.33.4-worklist.md`。

| 项 | 值 | 说明 |
|----|-----|------|
| 私有对象存储 | qingstor.com，bucket `qkeplus` | 存放 `qke-installer-v3.4.0.tar.gz` |
| installer 下载源 | `https://qkeplus.pek3b.qingstor.com/qke-installer-v3.4.0.tar.gz` | 即 `conf.ini [installer] url` |
| 私有 OCI 注册服务 | `registry.qingcloud.com`，project `kubesphere` | 存放控制面镜像（12 个 multi-arch） |
| 控制面镜像 | v1.29.15 / v1.32.8 / v1.33.4 × 4 组件 | 已推送至私有 OCI 注册服务（见 §3.3） |
| 镜像工具 | crane（multi-arch） | 保存/推送镜像，与 docker 无关 |

> **注意**：以上为青云公有云环境（qke-artifacts 资源包）的默认值；生产离线环境若不同，需按实际环境替换。

---

## 3. 升级前准备

### 3.1 变更通知

- 提前发送变更通知，告知业务方升级时间、影响范围与注意事项。
- 参考（jinke 方案）：升级完成并验证后，邮件/微信通知运维人员上线情况与后续注意事项。
- 待确认：通知对象、通知内容、通知时间。

### 3.2 备份

> 原则：**备份先行**。每个升级阶段/每步升级前都需备份，回退依赖备份。

参考（jinke 方案 + qke 手动升级方案草稿）备份清单：

| 备份项 | 方法 | 说明 |
|--------|------|------|
| etcd 快照 | `etcdctl snapshot save` | 任一健康成员执行即可，快照含全量集群状态 |
| kubeadm/kubelet 配置 | `kubectl get cm -n kube-system kubeadm-config / kubelet-config -o yaml` | 升级前导出 |
| 组件二进制 | `cp /usr/local/bin/kube*` | kubeadm/kubelet/kubectl |
| 组件清单 | `cp /etc/kubernetes/manifests/*` | master 静态 Pod 清单 |
| 数据库 | qkeplus 库备份 | 涉及平台版本/组件记录调整时 |
| 非 Running Pod 记录 | `kubectl get pod -A -owide \| grep -v Running` | 升级前基线 |

- 待确认：备份目录路径、etcd 备份脚本（如 `/usr/local/bin/kube-scripts/etcd-backup.sh`）是否可用。

### 3.3 镜像与升级包准备

> 离线环境无法从外网下载，需在互联网环境准备后导入离线环境。
> 来源：`/workspace/qke-artifacts/`（QKE 3.4.0 离线升级资源包，已按最终状态准备）。

**① qke-installer 文件包（已更新，kk 3.1.7 → 3.1.12）**

> **更新原因**：QKE v3.4.0 内置的 kubekey 版本为 **3.1.7**，**不支持升级 k8s 到 v1.33.4**，因此必须把 qke-installer 包中的 kubekey 更换为 **3.1.12**（3.1.x 线最高 tag，支持到 v1.33.4）。这是制作和更新 QKE v3.4.0 qke-installer 包的根本原因。

QKE 3.4.0 内置的 `qke-installer-v3.4.0.tar.gz` 已更新为支持 k8s v1.33.4 的最终版本，包内布局：

```
qke-installer/
├── kk                                          # kubekey 3.1.12 二进制
└── kubekey/                                    # KubeKey 依赖二进制
    ├── etcd/v3.5.24/amd64/etcd-v3.5.24-linux-amd64.tar.gz
    ├── cni/v3.27.4/amd64/calicoctl
    ├── kube/
    │   ├── v1.29.15/amd64/{kubeadm,kubelet,kubectl}
    │   ├── v1.32.8/amd64/{kubeadm,kubelet,kubectl}
    │   └── v1.33.4/amd64/{kubeadm,kubelet,kubectl}
    └── helm/v3.18.5/amd64/helm
```

- 更新方式：轻量处理（gunzip → tar 追加 → gzip），原始包已备份（`qke-installer-v3.4.0.tar.gz.20260923`，**必须保持不变**）；
- 校验：`bash scripts/verify-checksums.sh`（SHA256 与 kk 3.1.12 官方 `version/components.json` 一致）；
- 下载地址与 SHA256 见 `refs/README.md` 引用的 qke-artifacts 文档。

**② 控制面镜像（已推送，12 个 multi-arch）**

升级逐 minor 版本进行，涉及 **v1.29.15 / v1.32.8 / v1.33.4** 三个版本 × 4 组件（kube-apiserver / kube-controller-manager / kube-scheduler / kube-proxy），共 12 个镜像，已推送至私有 OCI 注册服务 `registry.qingcloud.com/kubesphere/`。

- 保存（联网机器）：`bash scripts/save-images.sh v1.29.15|v1.32.8|v1.33.4`（crane，multi-arch）；
- 推送（离线环境）：`bash scripts/load-push-images.sh <版本> offline-images/ '<用户名>' '<密码>'`（crane）；
- 校验：`crane manifest registry.qingcloud.com/kubesphere/kube-apiserver:v1.33.4`（mediaType 应为 manifest.list.v2+json）。

**③ 其他目标版本配套镜像**

- **coredns**：**coredns 版本由 kubekey 版本决定**（**非 QKE**）；**kk 升级时不会自动升级 coredns**（实测确认，QKE 3.3.0 实例为 **1.9.2**，实测升级后仍为 1.9.x）。**是否需升级 coredns 由 k8s 版本要求决定**（见 §10.2）；若需升级，需补充相应内容；
- 待确认：pause 3.10、nodelocaldns 1.25.0 等是否已推入私有仓库。

**④ 注意**

- 二进制与镜像注意架构（amd64/arm64）一致；
- 若生产离线环境与青云公有云环境不同，需按实际环境替换对象存储/OCI 注册服务地址。

### 3.4 前置检查

参考（jinke 方案）检查项：

```bash
# 节点状态与版本
kubectl get nodes -o wide

# 系统组件
kubectl get pods -n kube-system

# 全集群 Pod 状态（非 Running 需关注）
kubectl get pod -A -owide | grep -v Running

# 资源余量
kubectl top nodes
```

- 待填充：QKE 平台侧前置检查（控制台可访问、集群状态正常）。

**QKE 平台升级前检查（第一阶段，用户提供）**：

1. web 控制台显示正常状态；
2. k8s 状态正常：`kubectl version` 与 `kubectl get nodes` 显示的 k8s 版本一致，均为 1.27.10；
3. k8s 各 pod 无异常，特别是 kube-system 中的 pod。

---

## 4. 第一阶段：QKE 平台升级（v3.3.0 → v3.4.0）

> 保持 k8s 版本 v1.27.10 不变。

### 4.1 升级方法

**升级方式**：QKE 3.3.0 → 3.4.0 平台升级在 **QKE web 控制台**操作（用户在浏览器点击升级按钮）。

**升级入口与操作步骤（实测）**：

1. 进入 **集群运维 / 升级检查**；
2. 界面提示"**发现新集群版本**"；
3. 在**第一个下拉框**中选择"**QKE v3.4.0**"；
4. **不勾选**"**升级 K8s**"复选框（保持 k8s v1.27.10 不变）；
5. 点击"**立即升级**"。

**升级耗时（实测）**：**2 分 30 秒**。

**升级包导入**：参考 `/workspace/qke-artifacts/README.md`（qke-installer 文件包更新与导入，见 §3.3）。

**升级前检查（确保 QKE 3.3.0 实例状态正常）**：

1. **web 控制台显示正常状态**；
2. **k8s 状态正常**：`kubectl version` 与 `kubectl get nodes` 显示的 k8s 版本一致，均为 **1.27.10**；
3. **k8s 各 pod 无异常**，特别是 kube-system 中的 pod。

**已知事实**：

- QKE 3.4.0 的 `qke-installer-v3.4.0.tar.gz` 已准备就绪（kk 3.1.12 + 依赖，见 §3.3），存放于私有对象存储 `qkeplus` bucket；
- QKE 3.4.0 控制台支持 k8s 1.27.10 / 1.29.10 / 1.31.2。

**待确认**：

- 平台升级过程中是否有其他注意事项（如数据库字段、组件兼容性）。

### 4.2 升级后验证

**实测验证结果（2026-09-30，测试环境 A）**：

| 验证项 | 升级前 | 升级后 | 结论 |
|--------|--------|--------|------|
| qke-cli | v3.3.0-patch.4 | **v3.4.0-patch.1** | ✅ 平台已升级 |
| installer 内 kk | v3.1.7 | **v3.1.12** | ✅ 更新后的 installer 包已生效 |
| k8s 版本 | v1.27.10 | **v1.27.10** | ✅ 保持不变（未勾选"升级 K8s"） |
| 节点状态 | 全部 Ready | 全部 Ready | ✅ |
| kube-system pod | 全部 Running | 全部 Running | ✅ 无异常 |
| 平台组件 | — | cloud-controller-manager、csi-qingcloud-controller、qke-operator | ⚠️ **待确认**（见 issues-log #1：用户指出这些组件在 QKE 3.3.0 就已存在，非升级新增；待新建 QKE 3.3.0 实例核对） |

**installer 包内容变化（实测）**：

- `kubekey/kube/` 新增：v1.29.10、v1.29.15、v1.31.2、v1.32.8、v1.33.4；
- `kubekey/etcd/` 新增：v3.5.24。

**问题**：未遇到问题。

### 4.3 错误处理

- 待填充：常见错误与处理（结合测试问题清单）。

### 4.4 回退方法

- 待填充：QKE 平台版本回退。

---

## 5. 第二阶段：控制台升级 k8s（v1.27.10 → v1.29.10 → v1.31.2）

> 通过 QKE 3.4.0 web 控制台升级，升级顺序 1.27.10 → 1.29.10 → 1.31.2。

### 5.1 升级方法

**升级入口与操作步骤（实测）**：

1. 进入 **集群运维 / 升级检查**；
2. **勾选**"**升级 K8s**"复选框；
3. 在下拉框中选择对应的 k8s 版本；
4. 点击"**立即升级**"。

**升级顺序与耗时（实测）**：

| 步骤 | 升级前 → 升级后 | 耗时 | 问题 |
|------|----------------|------|------|
| 1 | v1.27.10 → **v1.29.10** | **15 分钟** | 未遇到问题 |
| 2 | v1.29.10 → **v1.31.2** | **15 分钟** | 未遇到问题 |

**已知事实**：

- QKE 3.4.0 控制台支持 k8s 1.27.10 / 1.29.10 / 1.31.2，升级顺序固定：1.27.10 → 1.29.10 → 1.31.2；
- 控制台升级走平台升级链路（qke-apiserver API）：`DescribeUpgradeVersions` → `UpgradeClusterPreCheck` → `UpgradeCluster` → `DescribeClusterUpgradeStatus`（异步，后台状态机自动推进，见 qke-docs/qke-apiserver-api.md §3.6）；
- 平台升级底层由 qke-cli 的 `upgradeClusterNew` action 驱动，最终调用 kubekey（`./kk upgrade`）执行（见 qke-docs/qke-cli.md §6）；
- QKE 3.4.0 内置 kubekey 3.1.7，最高支持 k8s 1.31.2。

**升级步骤（每步：备份 → 控制台升级 → 核对节点 Ready）**：

| 步骤 | 升级前（准备） | 升级后（核对） |
|------|---------------|----------------|
| 1.27.10 → 1.29.10 | 备份（见 §3.2） | 所有节点 Ready 且版本 v1.29.10 |
| 1.29.10 → 1.31.2 | 备份 | 所有节点 Ready 且版本 v1.31.2 |

- 待确认：控制台具体操作路径（菜单入口、升级按钮位置）、每步升级耗时、测试环境实测结果。

### 5.2 升级后验证

**实测验证结果（2026-09-30，测试环境 A）**：

| 验证项 | 1.27.10 → 1.29.10 后 | 1.29.10 → 1.31.2 后 | 结论 |
|--------|---------------------|---------------------|------|
| 节点版本 | v1.29.10 | **v1.31.2** | ✅ 全部节点 Ready |
| kube-system pod | 全部 Running | 全部 Running | ✅ 无异常 |
| 非 Running pod | 无 | 无 | ✅ |
| coredns 镜像 | — | `registry.qingcloud.com/coredns/coredns:1.9.3` | ✅ 实测未变（kk 不自动升级 coredns；版本由 kubekey 决定） |
| etcd | v3.5.13 | v3.5.13 | 待第三阶段升级到 v3.5.24 |
| containerd | v1.7.13 | v1.7.13 | 不变 |

**问题**：未遇到问题。

> **注意**：k8s 1.28+ 已移除 `kubectl version --short` 参数，升级到 1.29+ 后该命令会报错，改用 `kubectl version` 或 `kubectl version -o json`。

### 5.3 错误处理

- 待填充：常见错误与处理（结合测试问题清单；参考 qke-docs/qke-kse-version-support.md 中 KSE 安装失败根因链、calico 未就绪等案例）。

### 5.4 回退方法

- 参考（jinke 方案 + qke worklist）：控制台升级失败时，kubeadm 升级失败会自动回滚；升级完成后如需回退，通过替换组件镜像及二进制方式回退（见 §8.5）；
- 待确认：控制台升级失败后平台侧的处理方式（重试？回退？）。

---

## 6. 第三阶段：手动 kk 升级 k8s（v1.31.2 → v1.33.4）

> 手动执行 kk 命令，升级到 kubekey 3.1.12 支持的最高版本 v1.33.4。

### 6.1 升级方法

> **实测步骤（2026-09-30，测试环境 A，master1 172.22.8.23）**：升级耗时 **16 分钟**，未遇到问题。
> 参考来源：`/workspace/qke-cli/manual-upgrade-1.33.4-worklist.md`（QKE v3.4.0 / k8s 1.31.2 → 1.33.4 场景）。

**① 修改 kk 配置文件（升级前）**

修改 `/opt/qingcloud/qke/qke-installer/config-sample.yaml`，把 `kubernetes.version` 的值改为 `1.33.4`：

```yaml
spec:
  kubernetes:
    version: v1.33.4      # 由 v1.31.2 改为 v1.33.4
```

**② 执行 kk 升级命令**

在 master1 的 `/opt/qingcloud/qke/qke-installer` 目录中执行：

```bash
cd /opt/qingcloud/qke/qke-installer
./kk upgrade -f config-sample.yaml --skip-pull-images --skip-dependency-check
```

> 参数说明：
> - `--skip-pull-images`：跳过向节点预拉镜像（前提：目标镜像已推入私有仓库 `registry.qingcloud.com/kubesphere/`）；
> - `--skip-dependency-check`：跳过 KubeSphere↔k8s 依赖矩阵检查（QKE 的 KubeSphere 组合可能不在 kubekey 内置矩阵内）；**k8s 目标版本是否在支持列表的校验仍会执行**；
> - 本次**未使用** `--with-kubernetes` 参数，而是通过修改 `config-sample.yaml` 的 `kubernetes.version` 指定目标版本。

**③ etcd：本次未升级（保持 v3.5.13）**

- **原因**：k8s 1.33.4 可以使用 etcd 3.5.13，无需升级；
- 实测确认：升级后 `etcdctl version` 仍为 3.5.13，集群正常。

> 参考：kubekey 3.1.12 默认 etcd 为 v3.5.24，installer 包内已备 v3.5.24（`kubekey/etcd/`）；如需升级可逐节点滚动替换二进制（见 §10.3）。1.31+ 推荐 3.5.24，最低要求 3.5.11-0，**3.5.13 满足最低要求**。

**④ feature-gates：本次未清理**

- **原因**：测试环境已有的 `--feature-gates=RotateKubeletServerCertificate=true`，在 k8s 1.33.4 中是**默认开启**的，是否清理它**不会对升级造成影响**；
- 实测确认：升级顺利完成，未因 feature-gates 报错。

> ⚠️ **风险提示**：真实 QKE 实例的 feature-gates 可能与测试环境存在差异，**这部分内容目前无法获取**（见 issues-log #2）。若真实环境存在 k8s 1.33 已移除的 feature-gate（如 `TTLAfterFinished`、`ExpandCSIVolumes` 等），残留会导致新 kube-apiserver/kubelet 拒绝启动，**升级前必须清理**。清理位置：
> 1. configmap `kubeadm-config`：删除 `apiServer/controllerManager/scheduler.extraArgs.feature-gates`；
> 2. configmap `kubelet-config-1.31`：删除 `featureGates`；
> 3. 各节点 `/etc/kubernetes/manifests/kube-{apiserver,controller-manager,scheduler}.yaml` 的 `--feature-gates`；
> 4. 各节点 `/var/lib/kubelet/config.yaml` 的 `featureGates`。
>
> 检查命令：
> ```bash
> kubectl -n kube-system get cm kubeadm-config -o yaml | grep -A5 feature-gates
> kubectl -n kube-system get cm kubelet-config -o yaml | grep -A5 featureGates
> ```
> 另：qke-cli v3.4.0-patch.1 的 `removeDeprecatedArgs` 只处理 v1.25/v1.27，**不会自动清理 1.33 的 gate**。

**⑤ 升级后核对**

```bash
kubectl get nodes -o wide                      # 全部 Ready 且版本 v1.33.4
kubectl get pod -A -owide | grep -v Running    # 无异常 pod
```

**⑥ 平台自洽调整（升级后必做）**

> **目的**：使用户通过 QKE web 控制台管理 QKE 实例时能够自洽——控制台能正确显示实例使用的 k8s 版本；垂直/水平扩容、修改配置参数时，`config-sample.yaml` 中的 k8s 版本不会被错误渲染成 1.31.2 而造成未知风险。

**6.1 configmap 调整**：修改 kube-system 命名空间下 configmap `qke-cluster-config`：

| 字段 | 调整前 | 调整后 |
|------|--------|--------|
| `.kubernetes.version` | v1.31.2 | **v1.33.4** |
| `.kubernetes.pre_version` | v1.29.10 | **v1.31.2** |

**6.2 数据库元数据调整**（专业人员操作）：

| 项 | 值 |
|----|-----|
| 数据库 | `qkeplus`（PostgreSQL） |
| 表 | `qke_db_cluster` |
| 字段 | `kubernetes_version` |
| 过滤条件 | `cluster_id`（示例：`cl-vhm7nxnh`；测试环境实测为 `cl-tzc8j1gp`） |

```sql
update qke_db_cluster set kubernetes_version='v1.33.4' where cluster_id='cl-vhm7nxnh';
```

> 说明：`cluster_id` 可从 `kubectl -n kube-system get cm qke-cluster-config -o jsonpath='{.data.qkeCluster}'` 中的 `env.qingcloud.clusterID` 获取。

**6.3 平台自洽验证（实测，2026-09-30，另一实例 master1 172.22.8.7）**

> 该实例已完成整个升级过程，并在升级后**新增 1 个 worker 节点**（worker 数量 3 → 4），用于验证平台自洽性。

| 检查项 | 实测结果 | 结论 |
|--------|----------|------|
| 节点 | 3 master + **4 worker**（worker-s004 新增），全部 v1.33.4 | ✅ |
| `config-sample.yaml` 的 `kubernetes.version` | **v1.33.4** | ✅ 未被错误渲染成 1.31.2 |
| `qke-cluster-config` 的 `.kubernetes.version` | **v1.33.4** | ✅ |
| `qke-cluster-config` 的 `.kubernetes.pre_version` | **v1.31.2** | ✅ |
| `qke-cluster-config` 节点列表 | 含新增的 worker-s004 | ✅ 新增节点已纳入 |
| QKE web 控制台显示 k8s 版本 | **1.33.4** | ✅ 符合预期 |

**结论**：平台自洽目标达成——升级后新增 worker 节点，`config-sample.yaml` 仍正确显示 v1.33.4，垂直/水平扩容、修改配置参数时不会错误渲染 k8s 版本，无未知风险。

**⑦ coredns 升级（调研结论：非"必须"，但"强烈建议"）**

> **原则**：
> - **coredns 版本由 kubekey 版本决定**（**非 QKE**）；
> - **kk 升级时不会自动升级 coredns**（实测确认）；
> - **是否需升级 coredns 由 k8s 版本的要求决定**。

**调研结论（子 agent 调研 k8s v1.33.4 官方源码/文档，2026-09-30）**：

| 判断维度 | 结论 |
|----------|------|
| k8s 官方是否**强制要求** coredns ≥ 某版本 | **否**。k8s 1.33 无 coredns 最低版本要求，coredns 是 addon，控制面不校验其版本 |
| k8s 官方工具（kubeadm）在 1.33 中默认安装的 coredns | **v1.12.0**（`CoreDNSVersion = "v1.12.0"`） |
| kubeadm upgrade 是否自动升级 coredns | 是（addon 阶段 `CreateOrUpdate`）——**但 QKE 不走 kubeadm，不适用** |
| coredns 1.9.2 在 k8s 1.33.4 下是否可用 | **基本可用**（监听 v1 EndpointSlice，1.33 仍提供该 API） |
| 是否建议升级到 v1.12.0 | **强烈建议**（官方配套/测试版本） |

**一句话结论**：**不是"必须"，但属于"强烈建议"**。1.9.x 在 1.33.4 下大概率能继续工作，但不在 k8s 1.33 官方配套/测试范围内，存在功能与安全风险。

**kubekey 对各 k8s 版本的 coredns 配套版本**（来源：[kubekey components.md](https://raw.githubusercontent.com/kubesphere/kubekey/faf2739a7b4ed03f282fd0ef38479e35a49e0340/docs/en/installation/components.md)）：

| k8s 版本 | kubekey 默认 coredns |
|----------|---------------------|
| 1.27~1.28 | v1.10.1 |
| 1.29 | v1.11.1 |
| 1.30~1.32 | v1.11.3 |
| **1.33** | **v1.12.0** |

**本项目情况**：QKE 3.3.0 实例的 coredns 为 **1.9.2**（实测升级后仍为 1.9.x），低于 k8s 1.33 的配套版本 **v1.12.0**。

**风险（若保持 1.9.x）**：

1. **不在官方配套/测试范围内**：1.9.2 比 1.33 早约 3 年，属"未验证组合"；
2. **旧 coredns + 新 k8s 有失效先例**：华为云 CCE 记录旧版 coredns 升级后因监听被移除的 `v1beta1.EndpointSlice` 导致新建 Service 无法解析、Headless 返回旧 IP（[华为云 FAQ](https://support.huaweicloud.com/cce_faq/cce_faq_00490.html)）；1.9.2 当前不受影响，但需升级后验证 DNS；
3. **功能与安全缺失**：1.9.2 已停止维护，缺后续安全修复与新增能力。

**建议**：

1. **升级到 v1.12.0（推荐）**：k8s 1.33 官方配套版本（kubeadm 常量、coredns/deployment 记录、kubekey 三方一致）；离线环境提前导入 `coredns/coredns:v1.12.0` 镜像并保留 Corefile；
2. **若短期无法升级**：1.9.x 可继续运行，但升级后必须验证 DNS（新建 Service 解析、Headless 变更解析、coredns 日志无 watch 失败告警），并制定后续升级计划。

**升级方法**：

```bash
# 1) 准备 coredns v1.12.0 镜像并推入私有仓库（crane）
# 2) 更新 coredns deployment 镜像
kubectl -n kube-system set image deployment/coredns coredns=<registry>/coredns/coredns:v1.12.0
# 3) 核对 coredns pod 状态与 DNS 解析
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl run -it --rm dns-test --image=<registry>/busybox:latest -- nslookup kubernetes.default
```

> **无法确认项**：QKE 官方是否将 coredns 1.9.x 列为 k8s 1.33.4 的受支持组合（QKE 私有兼容性声明，公开资料无法确认，需查 QKE 官方文档或咨询青云）。
> 详细调研报告见 [refs/k8s-1.33-coredns-compatibility-report.md](refs/k8s-1.33-coredns-compatibility-report.md)。

**参考：备份（建议，本次实测未明确记录）**

> 原则：**备份先行**。以下为参考备份清单（来源：qke-cli worklist），建议在真实环境升级前执行。

```bash
mkdir -p /data/var/backups
# etcd 快照（etcd 已开 TLS，用证书）
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/ssl/etcd/ssl/ca.pem --cert=/etc/ssl/etcd/ssl/node-master1.pem \
  --key=/etc/ssl/etcd/ssl/node-master1-key.pem \
  snapshot save /data/var/backups/etcd-snapshot-$(date +%F-%H%M%S).db
# kubeadm / kubelet 配置
kubectl -n kube-system get cm kubeadm-config -o yaml > /data/var/backups/kubeadm-config.yaml
kubectl -n kube-system get cm kubelet-config -o yaml > /data/var/backups/kubelet-config.yaml
# 节点 manifests / 二进制
cp /etc/kubernetes/manifests/* /data/var/backups/
cp /usr/local/bin/kube* /data/var/backups/
```

**参考：installer 包同步（若 installer 未更新）**

```bash
# 下载并解包（<version> 为集群版本）
mkdir -p /opt/qingcloud/qke/ && cd /data
wget "<url>/qke-installer-<version>.tar.gz" --no-check-certificate
tar zxvf qke-installer-<version>.tar.gz && rm -rf qke-installer-<version>.tar.gz
# 定向同步（与 qke-cli DownloadInstaller 一致）
cd /data && rsync -aur qke-installer/yamls qke-installer/charts qke-installer/kk qke-installer/kubekey \
  --exclude=kubekey/images/*  /opt/qingcloud/qke/qke-installer
```

> 本次实测中，installer 包已在第一阶段（QKE 平台升级）随平台升级更新为 kk 3.1.12，故无需单独同步。

**参考：etcd 升级（本次未做，如需可参考）**

```bash
# 每节点：停 etcd → 换二进制 → 起 etcd → 查健康
systemctl stop etcd
tar -zxf /opt/qingcloud/qke/qke-installer/kubekey/etcd/v3.5.24/amd64/etcd-v3.5.24-linux-amd64.tar.gz -C /tmp
cp -f /tmp/etcd-v3.5.24-linux-amd64/etcd* /usr/local/bin/ && chmod +x /usr/local/bin/etcd*
systemctl start etcd
etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/node-master1.pem --key=/etc/ssl/etcd/ssl/node-master1-key.pem endpoint health
```

**参考：手动调整（可选）**

- **pause → 3.10**：改各节点 `/etc/containerd/config.toml` 的 `sandbox_image` 为 `<registry>/pause:3.10`，重启 containerd（会中断业务，需维护窗口）；
- **crictl → 1.33.0**（可选）：替换 `/usr/bin/crictl`。

**参考：数据库其他字段调整（KSE 场景）**

| 表 | 字段 | 调整内容 |
|----|------|---------|
| `qke_db_component` | `k8s_versions` | 给 KSE 组件记录（`component_name='ks-console'`, `component_type='consolePlugin'`）追加 `v1.33.4`，否则 KSE 安装/升级被拦（`CurrentKSENoUpgradeable`） |
| `qke_db_version` | `compatibility_k8s` | 追加 `v1.33.4`（若后续走平台升级流程，控制台才显示该选项） |

```sql
-- KSE 组件补 k8s 兼容版本
UPDATE qke_db_component
SET k8s_versions = k8s_versions || '["v1.33.4"]'
WHERE component_name = 'ks-console' AND component_type = 'consolePlugin';
-- 平台版本记录补兼容 k8s（若走平台流程）
UPDATE qke_db_version SET compatibility_k8s = compatibility_k8s || ',v1.33.4' WHERE qke_version = 'v3.4.0';
```

> 注意：`qke_db_version` 存的是 **QKE 平台版本**（决定 installer 包），k8s 兼容性主要在 `qke_db_component.k8s_versions` 与 installer 包内 `kubekey/kube/`。

> **KSE 4.1.2 场景（测试环境 B，后续补充）**：若集群安装了 KSE 4.1.2，升级 k8s 后需重点核对 `qke_db_component` 中 KSE 组件记录（`component_name='ks-console'`, `component_type='consolePlugin'`）的 `k8s_versions` 是否包含目标 k8s 版本，否则 KSE 安装/升级被拦（`CurrentKSENoUpgradeable`）；同时需验证 KSE 4.1.2 在目标 k8s 版本下的功能正常（详见 qke-docs/qke-kse-version-support.md）。

### 6.2 升级后验证

**实测验证结果（2026-09-30，测试环境 A）**：

| 验证项 | 升级前 | 升级后 | 结论 |
|--------|--------|--------|------|
| 节点版本 | v1.31.2 | **v1.33.4** | ✅ 全部节点 Ready |
| kube-system pod | 全部 Running | 全部 Running | ✅ 无异常 |
| 非 Running pod | 无 | 无 | ✅ |
| etcd | v3.5.13 | v3.5.13 | ✅ 未升级（满足 1.33.4 要求） |
| containerd | v1.7.13 | v1.7.13 | ✅ 不变 |
| coredns 镜像 | `coredns:1.9.3` | `coredns:1.9.3` | ✅ 实测未变（**kk 不自动升级 coredns**；版本由 kubekey 决定；QKE 3.3.0 实例为 1.9.2） |
| `qke-cluster-config` | version=v1.31.2, pre_version=v1.29.10 | version=**v1.33.4**, pre_version=**v1.31.2** | ✅ 已调整 |
| 数据库 `qke_db_cluster.kubernetes_version` | v1.31.2 | **v1.33.4** | ✅ 已调整（专业人员操作） |

**问题**：未遇到问题。

**验证命令**：

```bash
kubectl get nodes -o wide                      # 全部 Ready 且版本 v1.33.4
kubectl get pods -n kube-system                # 系统组件正常
kubectl get pod -A -owide | grep -v Running    # 无异常 pod
kubectl -n kube-system get deploy coredns -o jsonpath='{.spec.template.spec.containers[0].image}'
kubectl -n kube-system get cm qke-cluster-config -o jsonpath='{.data.qkeCluster}' | grep -A2 kubernetes
```

**待验证（需用户确认）**：

- 平台功能：控制台可登录、集群列表/详情显示正确版本（依赖 §6.1⑥ 数据库字段调整）；
- 各组件：存储（csi-qingcloud / hydrofs）、网络（calico / hostnic）、KSE（若启用）功能验证。

### 6.3 错误处理

- 待填充：常见错误与处理（结合测试问题清单；参考 qke-artifacts README §九 常见问题：crane push 权限、kk 联网下载依赖、pod 拉取镜像失败等）。

### 6.4 回退方法

- 参考（jinke 方案 + qke worklist §6）：kubeadm 升级失败会自动回滚；升级完成后通过替换组件镜像及二进制回退，**回退版本不超过 3 个大版本**；
- 回退步骤见 §8.5（master 组件镜像、kubelet 二进制、kube-proxy、coredns、etcd、数据库字段）；
- 待确认：本项目第三阶段回退的具体触发条件与耗时。

---

## 7. 升级后验证（整体）

> 参考（jinke 方案 + qke 手动升级方案草稿）验证清单，各阶段升级后按需执行。

### 7.1 节点与系统组件

```bash
# 节点版本与状态（全部 Ready 且版本为目标版本）
kubectl get nodes -o wide

# 系统组件
kubectl get pods -n kube-system

# 全集群 Pod 状态
kubectl get pod -A -owide | grep -v Running
```

### 7.2 平台基础功能

**实测验证结果（2026-09-30，实例 master1 172.22.8.7）**：

- ✅ **控制台显示正确版本**：QKE web 控制台显示 k8s 版本为 **1.33.4**，符合预期；
- ✅ **扩容自洽**：升级后新增 1 个 worker 节点（3 → 4），`config-sample.yaml` 与 `qke-cluster-config` 均正确显示 v1.33.4，未被错误渲染成 1.31.2（详见 §6.1⑥ 6.3）。

**待验证**：

- 使用平台管理员账号、普通账号分别登录，测试创建工作负载、svc 等基础功能；
- 业务空间网关启用/禁用等功能验证。

### 7.3 各组件验证

- 存储（csi-qingcloud / hydrofs）、网络（calico / hostnic）、KSE（若启用）功能验证；
- 参考（jinke 方案）组件命名空间：`kubesphere-system`、`kubesphere-logging-system`、`kubesphere-monitoring-system`、`kubesphere-chaosmesh-system`、`kubeeye-system`、`kruise-system`、`fluid-system` 等。

### 7.4 巡检

- 通过平台巡检功能对集群巡检，确认巡检报告无异常。

---

## 8. 回退方案（整体）

> 各阶段回退细节见 §4.4 / §5.4 / §6.4；本节为整体回退原则与通用步骤。

### 8.1 回退条件

- 参考（jinke 方案）：升级过程中遇到严重问题，到计划时间仍未解决，则回退。
- 待确认：本项目回退触发条件与时间点。

### 8.2 回退计划

- 参考（jinke 方案）：回退预计耗时 2 小时。
- 待确认：本项目回退耗时预估。

### 8.3 回退原则

- kubeadm 升级失败时会**自动回滚**（kubeadm 内部机制）；
- 升级完成后如需回退，通过**替换组件镜像及二进制**方式将 k8s 版本回退到低版本；
- **回退版本不超过 3 个大版本**（kubeadm 支持范围）；
- 回退前提：升级前备份（etcd 快照、kubeadm/kubelet 配置、组件镜像、数据库）完整。

### 8.4 回退涉及组件

kube-apiserver、kube-controller-manager、kube-scheduler、kube-proxy、kubelet、CoreDNS、kubeadm、etcd、数据库字段。

### 8.5 回退步骤（参考 jinke 方案，需按本项目各阶段确认）

**① 回退 master 组件版本（所有 master 节点执行）**

进入 `/etc/kubernetes/manifests` 目录，将 kube-apiserver / kube-controller-manager / kube-scheduler 三个组件的镜像替换为待回退版本，kubelet 会自动重启对应静态 Pod。

**② 回退 kubelet 版本（所有 worker 节点执行）**

```bash
systemctl stop kubelet
# 将 /usr/local/bin/kubelet 替换为待回退版本
systemctl start kubelet
```

**③ 回退 kube-proxy 版本**

kube-proxy 以 DaemonSet 方式运行，使用 kubectl 修改镜像版本：

```bash
kubectl edit ds -n kube-system kube-proxy
```

**④ 回退 coredns**

```bash
kubectl -n kube-system set image deployment coredns coredns=<registry>/coredns/coredns:<旧版本>
```

**⑤ 回退 etcd**（若已升级）

从 installer 包取旧版本压缩包解包替换，逐节点滚动回退。

**⑥ 回退数据库字段**（若已调整）

还原 qke_db_cluster 的 k8s 版本字段、qke_db_component.k8s_versions 等，保证平台状态与回退后的实际版本一致。

### 8.6 回退后检查

按 §7 验证清单检查（节点版本应为回退目标版本）。

---

## 9. 风险与注意事项

> 基于参考资料与已知事实整理，随升级问题清单补充完善。

1. **kubekey 版本兼容**：QKE 3.4.0 使用 kubekey 3.1.7，手动升级到 1.33.4 需使用 kubekey 3.1.12；需确认 3.1.12 能识别现有集群的 kubeadm-config，升级前先在测试集群验证。
2. **etcd 版本**：kubekey 3.1.12 默认 etcd v3.5.24；当前 etcd v3.5.13 满足 1.33.4 最低要求（3.5.11-0），但为保持一致建议升级到 3.5.24（逐节点滚动，见 §6.1③）。
3. **feature-gates 残留**：k8s 1.33 移除了 `TTLAfterFinished`、`ExpandCSIVolumes` 等 feature-gate；升到 1.33.4 前必须从配置中清理，否则新组件启动失败。qke-cli 的 `removeDeprecatedArgs` 只处理 v1.25/v1.27，**不会自动清理 1.33 的 gate**，必须手动做。
4. **镜像可用性**：控制面镜像需补齐 v1.29.15 / v1.32.8 / v1.33.4 全部版本（逐 minor 升级会拉取中间版本），**必须先把对应镜像推入私有仓库**，否则升级失败。**注意：coredns 版本由 kubekey 版本决定**（非 QKE），**kk 升级时不会自动升级 coredns**（实测确认）；是否需升级 coredns 由 k8s 版本要求决定（见 §10.2）。
5. **数据库一致性**：QKE 平台升级与 k8s 升级后，需保证平台数据库（qke_db_cluster.kubernetes_version、qke_db_component.k8s_versions、qke_db_version.compatibility_k8s）中的版本字段与实际一致，控制台才显示正确版本；只改版本字段，不动状态机字段。
6. **维护窗口**：部分操作（如 pause 升级需重启 containerd）会中断节点业务，需安排在维护窗口执行。
7. **备份先行**：每个升级阶段前都需备份（etcd 快照、kubeadm/kubelet 配置、组件镜像、数据库）。
8. **离线环境**：所有二进制与镜像需在互联网环境准备后导入离线环境，注意架构（amd64/arm64）一致；镜像保存/推送使用 crane（multi-arch），勿用 docker save/load（会丢失多架构）。
9. **installer 包备份**：更新 `qke-installer-v3.4.0.tar.gz` 前必须备份原始包（`.20260923` 备份**必须保持不变**），避免覆盖。
10. **KSE 4.1.2 场景（测试环境 B，后续补充）**：若集群安装 KSE 4.1.2，升级 k8s 后需核对 KSE 组件 `k8s_versions` 兼容性并验证 KSE 功能；KSE 安装/升级可能被 k8s 版本兼容性拦截（`CurrentKSENoUpgradeable`）。当前方案先处理无 KSE 情况。

---

## 10. 附录

### 10.1 参考资料

- 本项目内：`refs/upgrade-plan-of-jinke.md`、`refs/README.md`。
- 项目外（只读）：`/workspace/qke-docs/` 系列文档（installer 包、qke-cli、qke-apiserver、KSE 版本支持等）。

### 10.2 版本兼容性参考表

> 来源：`refs/upgrade-plan-of-jinke.md`（其他项目 k8s 升级方案，v1.23.17 → v1.33.7）。

**etcd 与 k8s 版本对应**：

| k8s 版本 | etcd 最低要求 | etcd 推荐 |
|----------|--------------|-----------|
| 1.27.0~1.27.16 | 3.2.18 | 3.5.12 |
| 1.28.0~1.28.15 | 3.4.13-4 | 3.5.15 |
| 1.29.0~1.29.15 | 3.4.13-4 | 3.5.15 |
| 1.30.0~1.30.14 | 3.4.13-4 | 3.5.15 |
| 1.31.0~1.31.13 | 3.5.11-0 | 3.5.24 |
| 1.32.0~1.32.9 | 3.5.11-0 | 3.5.24 |
| v1.33.0~v1.33.5 | 3.5.11-0 | 3.5.24 |

**containerd 与 k8s 版本对应**：

| k8s 版本 | 推荐 containerd |
|----------|-----------------|
| 1.27 | v1.7.13 |
| 1.28 | v1.7.13 |
| 1.29 | v1.7.13 |
| 1.30 | v1.7.13 |
| 1.31 | v1.7.13 |
| 1.32 | v1.7.13 |
| 1.33 | v1.7.13 |

**pause / coredns / nodelocaldns 与 k8s 版本对应**：

> ⚠️ **重要**：下表为 **kubekey/kubeadm 的默认配套值**。**coredns 版本由 kubekey 版本决定**（**非 QKE**）；**实测确认 kk 升级时不会自动升级 coredns**（QKE 3.3.0 实例为 **1.9.2**，升级后仍为 1.9.x）。**是否需升级 coredns 由 k8s 版本要求决定**——若需升级，需补充相应内容（见下）。

| k8s 版本 | pause | coredns（kubekey 默认，QKE 不采用） | nodelocaldns |
|----------|-------|---------|--------------|
| 1.27 | 3.9 | v1.10.1 | 1.22.20 |
| 1.28 | 3.9 | v1.10.1 | 1.22.23 |
| 1.29 | 3.9 | v1.11.1 | 1.22.23 |
| 1.30 | 3.9 | v1.11.3 | 1.22.28 |
| 1.31 | 3.10 | v1.11.3 | 1.23.1 |
| 1.32 | 3.10 | v1.11.3 | 1.23.1 |
| 1.33 | 3.10 | v1.12.0 | 1.25.0 |

**实际 coredns 版本**：由 **kubekey 版本**决定；QKE 3.3.0 实例为 **1.9.2**，实测升级到 1.33.4 后仍为 1.9.x（未变）。

**coredns 是否需要升级**：由 **k8s 版本的要求**决定。若目标 k8s 版本要求更高的 coredns 版本，则需单独升级 coredns（补充内容待确认，见 issues-log #3）。

### 10.3 常用命令速查

> 参考（jinke 方案 + qke 手动升级方案草稿），具体命令以实际环境为准。

**备份类**：

```bash
# etcd 快照（etcd 开启 TLS 时需带证书参数）
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=<ca.pem> --cert=<node.pem> --key=<node-key.pem> \
  snapshot save /data/var/backups/etcd-snapshot-$(date +%F-%H%M%S).db

# kubeadm / kubelet 配置
kubectl -n kube-system get cm kubeadm-config -o yaml > kubeadm-config.yaml
kubectl -n kube-system get cm kubelet-config -o yaml > kubelet-config.yaml

# 组件二进制与清单
cp /usr/local/bin/kube* .
cp /etc/kubernetes/manifests/* .
```

**检查类**：

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system
kubectl get pod -A -owide | grep -v Running
kubectl -n kube-system get deploy coredns -o yaml | grep -E "image:"
```

**kk 升级类**（第三阶段，来源：qke-cli worklist + qke-artifacts）：

```bash
# 在 master1、installer 目录内执行（./kk 为 kubekey 3.1.12）
cd /opt/qingcloud/qke/qke-installer
./kk upgrade -f config-sample.yaml --with-kubernetes 1.33.4 --skip-pull-images --skip-dependency-check
# 或（qke-artifacts 资源包写法）
kk upgrade -f <your-config-file> --skip-pull-images --skip-dependency-check
```

**镜像保存/推送类**（离线环境，crane，来源：qke-artifacts）：

```bash
# 联网机器：保存 multi-arch 镜像为 OCI tar.gz
bash scripts/save-images.sh v1.29.15
bash scripts/save-images.sh v1.32.8
bash scripts/save-images.sh v1.33.4

# 离线环境：推送镜像到私有 OCI 注册服务
bash scripts/load-push-images.sh v1.33.4 offline-images/ '<用户名>' '<密码>'

# 校验 multi-arch manifest list
crane manifest registry.qingcloud.com/kubesphere/kube-apiserver:v1.33.4
```

**installer 文件包更新类**（离线环境，来源：qke-artifacts）：

```bash
# 备份私有对象存储中的 qke-installer
rclone copy qingstor:qkeplus/qke-installer-v3.4.0.tar.gz installer-work/origin/
rclone copyto qingstor:qkeplus/qke-installer-v3.4.0.tar.gz \
  qingstor:qkeplus/qke-installer-v3.4.0.tar.gz.$(date +%Y%m%d)

# 校验下载的文件
bash scripts/verify-checksums.sh

# 制作新的 qke-installer 文件包（轻量处理：tar 追加）
bash scripts/append-to-installer.sh \
  installer-work/staging/qke-installer-v3.4.0.tar.gz \
  installer-work/downloads \
  installer-work/staging/qke-installer-v3.4.0-new.tar.gz

# 推送新的文件包到私有对象存储
rclone copyto installer-work/staging/qke-installer-v3.4.0-new.tar.gz \
  qingstor:qkeplus/qke-installer-v3.4.0.tar.gz
```

---

> **待确认项汇总**：见 [status/project-status.md](status/project-status.md)「待用户提供的信息」。
