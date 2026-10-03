# QKE v3.3.0 集群 Kubernetes 升级方案

> **状态**：迭代中（第 48 轮：§7.1 增加"核对/定位方法"列，用于核对和定位问题（如 `kubectl get nodes` 查看节点 k8s 版本））
> **目标**：QKE v3.3.0 集群（k8s v1.27.10）升级到 k8s **v1.33.4**，含升级方法、问题处理、回退方案。
> **环境**：离线（无法访问互联网）。
> **相关文档**：测试环境情况见 [test-environment.md](test-environment.md)；决策思路见 [decisions-rationale.md](decisions-rationale.md)。

---

## 1. 背景

### 1.1 升级目标

- 将集群 k8s 从 v1.27.10 升级到 **v1.33.4**；
- 约束：集群运行环境**无法访问互联网**（离线环境）；QKE v3.3.0 最高支持 k8s v1.27.10；QKE 底层使用 KubeKey（`kk`）升级 k8s。

### 1.2 整体升级步骤（3 步）

1. **导入 QKE v3.4.0**（此工作由**青云专业人员**处理）；
2. **更新 QKE v3.4.0 的 qke-installer 文件包**（参考 `/workspace/qke-artifacts/` 中的步骤，见 §2.1）；
3. **执行具体的升级步骤**（三段式，见下）。

### 1.3 三段式升级路径

```
① QKE 平台升级：v3.3.0 → v3.4.0（保持 k8s v1.27.10 不变）
   ↓
② 控制台升级 k8s：v1.27.10 → v1.29.10 → v1.31.2（QKE 3.4.0 控制台支持）
   ↓
③ 手动 kk 升级 k8s：v1.31.2 → v1.33.4（kubekey 3.1.12 支持的最高版本）
```

| 阶段 | 内容 | 关键约束 |
|------|------|----------|
| ① QKE 平台升级 | v3.3.0 → v3.4.0 | 保持 k8s v1.27.10 不变 |
| ② 控制台升级 k8s | v1.27.10 → v1.29.10 → v1.31.2 | QKE 3.4.0 控制台支持 1.27.10/1.29.10/1.31.2，升级顺序固定 |
| ③ 手动 kk 升级 k8s | v1.31.2 → v1.33.4 | kubekey 3.1.12 为 3.1.x 线最高 tag，支持到 v1.33.4 |

### 1.4 版本兼容性矩阵

| 项 | 当前 | 目标 | 说明 |
|----|------|------|------|
| QKE 平台 | v3.3.0 | v3.4.0 | 3.4.0 最高支持 k8s v1.31.2 |
| Kubernetes | v1.27.10 | v1.33.4 | 3.4.0 控制台支持 1.27.10/1.29.10/1.31.2；1.33.4 为 kubekey 3.1.12 上限 |
| kubekey | QKE 3.3.0 → **3.1.0**；QKE 3.4.0 → **3.1.7** | **3.1.12** | 3.1.7 不支持升级到 v1.33.4，**必须更换为 3.1.12** |
| etcd | v3.5.13 | v3.5.13（不升级） | 3.5.13 满足 1.33.4 要求 |
| containerd | v1.7.13 | v1.7.13（不变） | 满足 1.27~1.33 全程要求 |
| coredns | 1.9.x | 1.9.x（建议项升级到 v1.12.0） | kk 不自动升级 coredns |
| OS | Ubuntu 20.04.5 | 不变 | |

### 1.5 环境信息

> **真实环境缺失的信息，暂时以测试环境的信息为准**（测试环境详情见 [test-environment.md](test-environment.md)）。

**集群现状**：

| 项 | 值 |
|----|-----|
| QKE 平台版本 | v3.3.0 |
| Kubernetes | v1.27.10 |
| containerd | v1.7.13 |
| etcd | v3.5.13 |
| OS | Ubuntu 20.04.5 |
| 网络插件 | calico（暂以测试环境为准） |
| 存储插件 | csi-qingcloud（暂以测试环境为准） |
| KSE/KubeSphere | 未安装（暂以测试环境为准） |

**qke-cli、qke-installer 源和 OCI registry**：

| 项 | 值 | 说明 |
|----|-----|------|
| qke-cli | 随 QKE 平台安装，位于 `/opt/qingcloud/qke/bin/qke-cli` | QKE 平台组件之一 |
| qke-installer 下载源 | `https://qkeplus.pek3b.qingstor.com/qke-installer-v3.4.0.tar.gz` | 即 `conf.ini [installer] url`；存放于私有对象存储 qingstor.com bucket `qkeplus` |
| 私有 OCI 注册服务 | `registry.qingcloud.com`，project `kubesphere` | 存放控制面镜像（v1.29.15 / v1.32.8 / v1.33.4 × 4 组件，共 12 个） |

> **注意**：以上为青云公有云环境（qke-artifacts 资源包）的默认值；真实环境若不同，需按实际环境替换。

**生产环境**：待确认（集群拓扑、业务影响、维护窗口、参与人员）。

---

## 2. qke v3.4.0 导入与 qke-installer 文件包更新

> 本节为整体升级步骤（§1.2）第 1、2 步，在**三段式升级（§3/§4/§5）之前**执行，由**青云专业人员**处理；详细操作见专门文档。

**① QKE v3.4.0 导入**：由青云专业人员导入 QKE v3.4.0。

**② qke-installer 包更新**：QKE v3.4.0 内置的 `qke-installer-v3.4.0.tar.gz` 已更新为支持 k8s v1.33.4 的最终版本（kk 3.1.7 → 3.1.12 + 依赖 + helm v3.18.5），原始包已备份（`qke-installer-v3.4.0.tar.gz.20260923`，**必须保持不变**）。详细操作见 `/workspace/qke-artifacts/更新qke-installer-v3.4.0文件包操作文档.md`。

**③ 控制面镜像**：v1.29.15 / v1.32.8 / v1.33.4 × 4 组件（kube-apiserver / kube-controller-manager / kube-scheduler / kube-proxy）共 12 个 multi-arch 镜像已推送至私有 OCI 注册服务 `registry.qingcloud.com/kubesphere/`。升级前需确认镜像已就绪（kk 升级时逐 minor 拉取中间版本镜像）。详细操作见 `/workspace/qke-artifacts/镜像离线保存与推送操作文档.md`。

> **注意**：
> - 二进制与镜像注意架构（amd64/arm64）一致；
> - 若真实环境与青云公有云环境不同，需按实际环境替换对象存储/OCI 注册服务地址；
> - **coredns**：kk 升级时**不会自动升级** coredns（实测确认），无需为 k8s 升级准备 coredns v1.12.0 镜像；coredns 升级为**建议项**（见 §5.1⑧）；
> - 待确认：pause 3.10、nodelocaldns 1.25.0 等是否已推入私有仓库。

> **备份与前置检查**：为**每步升级前**的通用动作（非一次性准备），已并入各阶段升级步骤（§3.1 / §4.1 / §5.1），命令见 §10.3 备份类 / 检查类。

### 2.1 快速验证（真实环境的测试集群验证升级链路）

> **目的**：正式升级前，验证 QKE 3.4.0 导入、qke-installer 更新包、控制面镜像的正确性，以及对 QKE v3.3.0 升级的支持。
> **执行方式**：在**真实环境的测试集群**（真实用户环境中的测试集群，与青云测试环境 [test-environment.md](test-environment.md) 不同）执行升级测试，具体测试方式由测试人员把握。

**验证方法**：在真实环境的测试集群上执行一次完整升级链路测试：

1. **QKE v3.3.0 → v3.4.0 升级测试**：验证 QKE 3.4.0 导入与 qke-installer 更新包对 QKE v3.3.0 升级的支持（k8s 保持 v1.27.10）；
2. **QKE v3.4.0 中 k8s 版本升级测试**：验证控制面镜像与 kk 3.1.12 对 k8s 升级的支持，v1.27.10 → v1.29.10 → v1.31.2 → v1.33.4。

**验证项与通过标准**：

| 验证项 | 验证内容 | 通过标准 |
|--------|----------|----------|
| QKE 3.4.0 导入 | web 控制台"发现新集群版本"；qke-cli 版本 v3.4.0-patch.1 | 控制台可识别并选择 QKE v3.4.0 |
| qke-installer 更新包 | kk 3.1.12；kubekey/kube/ 含 v1.29.10/v1.29.15/v1.31.2/v1.32.8/v1.33.4；kubekey/etcd/ 含 v3.4.13/v3.5.13/v3.5.24；SHA256 校验通过 | 包内依赖齐全、校验通过 |
| 控制面镜像 | 12 个 multi-arch 镜像已推送至 `registry.qingcloud.com/kubesphere/`；manifest list 校验通过 | kk 升级时逐 minor 拉取成功 |
| 对 QKE v3.3.0 升级的支持 | 真实环境的测试集群 QKE v3.3.0 → v3.4.0 升级成功；k8s 1.27.10 → 1.29.10 → 1.31.2 → 1.33.4 升级成功 | 三段式升级链路全部通过 |

> **说明**：青云测试环境（[test-environment.md](test-environment.md)）已实测验证过三段式升级链路，可作为参考；但**真实环境的测试集群**是真实用户环境中的独立测试集群，需在其上单独执行本快速验证。

---

## 3. 第一阶段：QKE 平台升级（v3.3.0 → v3.4.0）

> 保持 k8s 版本 v1.27.10 不变。

### 3.1 升级步骤

**升级方式**：在 **QKE web 控制台**操作（用户在浏览器点击升级按钮）。

**操作步骤**：

1. **备份**（见 §10.3 备份类）；
2. **前置检查**（见 §10.3 检查类）：节点状态、系统组件、pod 状态、资源余量；
3. **控制台升级前检查**（QKE 平台升级与 k8s 版本升级通用）：web 控制台显示正常状态；`kubectl version` 与 `kubectl get nodes` 显示的 k8s 版本一致；**所有节点 k8s 版本一致**；kube-system pod 无异常；
4. 进入 **集群运维 / 升级检查**；
5. 界面提示"**发现新集群版本**"；
6. 在**第一个下拉框**中选择"**QKE v3.4.0**"；
7. **不勾选**"**升级 K8s**"复选框（保持 k8s v1.27.10 不变）；
8. 点击"**立即升级**"，等待升级完成（实测耗时 **2 分 30 秒**）；
9. **升级后验证**（命令见 §10.3 验证类）：
   - qke-cli 版本升级为 **v3.4.0-patch.1**；
   - installer 内 kk 版本为 **v3.1.12**（更新后的 installer 包已生效）；
   - k8s 版本保持 **v1.27.10** 不变；
   - 节点全部 Ready，kube-system pod 全部 Running；
   - 平台组件（cloud-controller-manager / csi-qingcloud-controller / qke-operator）正常。

> 测试环境实测结果见 [test-environment.md](test-environment.md) §2.2。

**已知事实**：

- QKE 3.4.0 的 `qke-installer-v3.4.0.tar.gz` 已准备就绪（kk 3.1.12 + 依赖，见 §2.1），存放于私有对象存储 `qkeplus` bucket；
- QKE 3.4.0 控制台支持 k8s 1.27.10 / 1.29.10 / 1.31.2。

**注意事项**：

- 已完成 §2.1 快速验证（真实环境的测试集群验证升级链路）；
- 此操作（QKE 平台升级）无法回退。

### 3.2 回退

- QKE 平台升级（第一阶段）**无法回退**。

---

## 4. 第二阶段：控制台升级 k8s（v1.27.10 → v1.29.10 → v1.31.2）

> 通过 QKE 3.4.0 web 控制台升级，升级顺序 1.27.10 → 1.29.10 → 1.31.2。

### 4.1 升级步骤

**每步升级流程：备份 → 前置检查 → 控制台升级 → 核对节点 Ready**。

**操作步骤**（每步）：

1. **备份**（见 §10.3 备份类）；
2. **前置检查**（见 §10.3 检查类）：节点状态、系统组件、pod 状态、资源余量；**控制台升级前检查**（见 §3.1 第 3 步）：所有节点 k8s 版本一致；
3. 进入 **集群运维 / 升级检查**；
4. **勾选**"**升级 K8s**"复选框；
5. 在下拉框中选择对应的 k8s 版本；
6. 点击"**立即升级**"，等待升级完成（实测每步耗时 **15 分钟**）；
7. **升级后验证**（命令见 §10.3 验证类）：
   - 所有节点 Ready 且版本为 v1.29.10（第 1 步后）/ v1.31.2（第 2 步后）；
   - kube-system pod 全部 Running，无异常 pod。

> **注意**：k8s 1.28+ 已移除 `kubectl version --short` 参数，升级到 1.29+ 后该命令会报错，改用 `kubectl version` 或 `kubectl version -o json`。
> 测试环境实测结果见 [test-environment.md](test-environment.md) §2.3。

**升级顺序**：

| 步骤 | 升级前 → 升级后 | 耗时（实测） |
|------|----------------|--------------|
| 1 | v1.27.10 → **v1.29.10** | 15 分钟 |
| 2 | v1.29.10 → **v1.31.2** | 15 分钟 |

**已知事实**：

- QKE 3.4.0 控制台支持 k8s 1.27.10 / 1.29.10 / 1.31.2，升级顺序固定：1.27.10 → 1.29.10 → 1.31.2；
- QKE 3.4.0 内置 kubekey 3.1.7，最高支持 k8s 1.31.2。

**注意事项**：

- 必须按照 1.27 → 1.29 → 1.31 的顺序升级，不能直接从 1.27 → 1.31 升级。

### 4.2 问题处理

- 排查问题并进行修复：可能的原因是离线文件缺失、oci registry 缺少镜像等；
- 再次执行 kk upgrade 命令进行升级，并进行验证：

  ```bash
  cd /opt/qingcloud/qke/qke-installer
  ./kk upgrade -f config-sample.yaml --skip-pull-images --skip-dependency-check
  ```

- 修改 PostgreSQL 数据库 `qkeplus` 中表 `qke_db_cluster` 的字段，让 qke 实例的状态为正常（而不是异常）。

---

## 5. 第三阶段：手动 kk 升级 k8s（v1.31.2 → v1.33.4）

> 手动执行 kk 命令，升级到 kubekey 3.1.12 支持的最高版本 v1.33.4。
> 参考来源：`/workspace/qke-cli/manual-upgrade-1.33.4-worklist.md`（QKE v3.4.0 / k8s 1.31.2 → 1.33.4 场景）。

### 5.1 升级步骤

**① 备份与前置检查**（备份见 §10.3 备份类；前置检查见 §10.3 检查类）。

**② 修改 kk 配置文件（升级前）**

修改 `/opt/qingcloud/qke/qke-installer/config-sample.yaml`，把 `kubernetes.version` 的值改为 `1.33.4`：

```yaml
spec:
  kubernetes:
    version: v1.33.4      # 由 v1.31.2 改为 v1.33.4
```

**③ 执行 kk 升级命令**

在 master1 的 `/opt/qingcloud/qke/qke-installer` 目录中执行：

```bash
cd /opt/qingcloud/qke/qke-installer
./kk upgrade -f config-sample.yaml --skip-pull-images --skip-dependency-check
```

> 参数说明：
> - `--skip-pull-images`：跳过向节点预拉镜像（前提：目标镜像已推入私有仓库 `registry.qingcloud.com/kubesphere/`）；
> - `--skip-dependency-check`：跳过 KubeSphere↔k8s 依赖矩阵检查（QKE 的 KubeSphere 组合可能不在 kubekey 内置矩阵内）；**k8s 目标版本是否在支持列表的校验仍会执行**；
> - 本次**未使用** `--with-kubernetes` 参数，而是通过修改 `config-sample.yaml` 的 `kubernetes.version` 指定目标版本。

**④ etcd：不升级（保持 v3.5.13）**

- 保持 etcd v3.5.13 不变，不执行 etcd 升级步骤；
- 如需升级到 v3.5.24，参考 §10.3（逐节点滚动替换二进制）。

**⑤ feature-gates**

- 获取当前显式配置的 feature-gate（检查位置与命令见 [feature-gates-cleanup-report.md](../refs/feature-gates-cleanup-report.md) §2）；
- 删除或保留 feature-gate（根据实际情况而定）。若需删除，按 [feature-gates-cleanup-report.md](../refs/feature-gates-cleanup-report.md) §3 执行：
  1. 修改 configmap `kubeadm-config`：删除 `apiServer/controllerManager/scheduler.extraArgs.feature-gates`；
  2. 修改 configmap `kubelet-config-1.31`：删除 `featureGates`；
  3. 删除第一个 master 节点 `/etc/kubernetes/kubeadm-config.yaml` 中的 feature-gates 字段；
  4. 修改各节点 `/etc/kubernetes/manifests/kube-{apiserver,controller-manager,scheduler}.yaml`：删除 `--feature-gates=xxx`；
  5. 删除各节点 `/var/lib/kubelet/config.yaml` 中的 `featureGates`。

> **注意事项**：如果显式配置的 feature-gate 只有 `RotateKubeletServerCertificate`，则不需要做任何操作。

**⑥ 升级后核对**

```bash
kubectl version                                # 服务端版本 v1.33.4（与节点版本一致）
kubectl get nodes -o wide                      # 全部 Ready 且版本 v1.33.4
kubectl get pod -A -owide | grep -v Running    # 无异常 pod
```

**⑦ 平台自洽调整（升级后必做）**

**configmap 调整**：修改 kube-system 命名空间下 configmap `qke-cluster-config`：

| 字段 | 调整前 | 调整后 |
|------|--------|--------|
| `.kubernetes.version` | v1.31.2 | **v1.33.4** |
| `.kubernetes.pre_version` | v1.29.10 | **v1.31.2** |

**数据库元数据调整**（专业人员操作）：

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

**⑧ coredns 升级（建议项，非必做）**

- **定位**：建议项，不阻塞 k8s 升级流程，是否执行由用户/运维决定；若执行，建议在第三阶段升级后、业务验证前进行；
- **目标版本**：v1.12.0（k8s 1.33 官方配套版本）；
- **升级方法**：

```bash
# 1) 准备 coredns v1.12.0 镜像并推入私有仓库
# 2) 更新 coredns deployment 镜像
kubectl -n kube-system set image deployment/coredns coredns=<registry>/coredns/coredns:v1.12.0
# 3) 核对 coredns 镜像与 pod 状态、DNS 解析
kubectl -n kube-system get deploy coredns -o jsonpath='{.spec.template.spec.containers[0].image}'
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl run -it --rm dns-test --image=<registry>/busybox:latest -- nslookup kubernetes.default
```

- **若不升级**：升级后必须验证 DNS（新建 Service 解析、Headless 变更解析、coredns 日志无 watch 失败告警）。

### 5.2 升级后验证

**通用验证命令**见 §10.3 验证类（节点版本与状态、系统组件、无异常 pod）。

**阶段特有检查**：

```bash
kubectl version                              # 服务端版本 v1.33.4（与节点版本一致）
kubectl -n kube-system get cm qke-cluster-config -o jsonpath='{.data.qkeCluster}' | grep -A2 kubernetes
```

**检查项**：

- 所有节点 Ready 且版本 v1.33.4；
- `kubectl version` 服务端版本为 v1.33.4，与节点版本一致；
- kube-system pod 全部 Running，无异常 pod；
- `qke-cluster-config` 的 `kubernetes.version` 为 v1.33.4、`pre_version` 为 v1.31.2；
- 数据库 `qke_db_cluster.kubernetes_version` 为 v1.33.4（专业人员操作）。

**待验证（需用户确认）**：

- 平台功能：控制台可登录、集群列表/详情显示正确版本（依赖 §5.1⑦ 数据库字段调整）；
- 各组件：存储（csi-qingcloud / hydrofs）、网络（calico / hostnic）、KSE（若启用）功能验证。

> 测试环境实测结果见 [test-environment.md](test-environment.md) §2.4。

### 5.3 问题处理

- 排查问题并进行修复：可能的原因是离线文件缺失、oci registry 缺少镜像等；
- 再次执行 kk upgrade 命令进行升级，并进行验证：

  ```bash
  cd /opt/qingcloud/qke/qke-installer
  ./kk upgrade -f config-sample.yaml --skip-pull-images --skip-dependency-check
  ```

---

## 6. 升级后验证（整体）

### 6.1 节点与系统组件

```bash
# k8s 版本（服务端版本与节点版本一致）
kubectl version

# 节点版本与状态（全部 Ready 且版本为目标版本）
kubectl get nodes -o wide

# 系统组件
kubectl get pods -n kube-system

# 全集群 Pod 状态
kubectl get pod -A -owide | grep -v Running
```

### 6.2 平台基础功能

- 控制台可登录，集群列表/详情显示正确版本；
- 使用平台管理员账号、普通账号分别登录，测试创建工作负载、svc 等基础功能；
- 业务空间网关启用/禁用等功能验证。

> 测试环境实测结果（控制台显示 1.33.4、扩容自洽）见 [test-environment.md](test-environment.md) §2.5。

### 6.3 各组件验证

- 存储（csi-qingcloud / hydrofs）、网络（calico / hostnic）、KSE（若启用）功能验证；
- 参考（jinke 方案）组件命名空间：`kubesphere-system`、`kubesphere-logging-system`、`kubesphere-monitoring-system`、`kubesphere-chaosmesh-system`、`kubeeye-system`、`kruise-system`、`fluid-system` 等。

### 6.4 巡检

- 通过平台巡检功能对集群巡检，确认巡检报告无异常。

---

## 7. 问题处理（汇总）

> 本节只收录**升级过程中引发故障的问题**（故障类问题）及其处理方法。以下问题不在此列：
> - 决策类问题（如是否升级某组件）；
> - 验证类问题（QKE 平台固有特性，与 k8s 升级关系不大）。
> 已知问题与处理登记见 [status/issues-log.md](../status/issues-log.md)（issues-log 记录的问题不一定是真实环境的问题，本节仅收录升级故障类问题）。

### 7.1 已知问题与处理

| # | 问题 | 核对/定位方法 | 处理方法 |
|---|------|--------------|---------|
| 1 | 节点 k8s 版本不一致时，QKE 控制台会阻止升级 | `kubectl get nodes` 查看各节点 k8s 版本是否一致 | 等待集群状态稳定、所有节点 k8s 版本一致后再升级（控制台升级前检查项之一，见 §3.1 第 3 步） |

### 7.2 待补充

- 用户提供测试环境升级过程中遇到的问题清单后，筛选出**升级过程中引发故障的问题**，登记到 issues-log，并同步到本节（§7 问题处理）。

---

## 8. 回退方案（整体）

> 本节为整体回退原则与通用步骤。

### 8.1 回退条件

- 升级过程中遇到严重问题，到计划时间仍未解决，则回退；
- 待确认：本项目回退触发条件与时间点。

### 8.2 回退计划

- 回退预计耗时 2 小时（参考 jinke 方案）；
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

按 §6 验证清单检查（节点版本应为回退目标版本）。

---

## 9. 风险与注意事项

> 基于参考资料与已知事实整理，随升级问题清单补充完善。

1. **kubekey 版本兼容**：QKE 3.4.0 使用 kubekey 3.1.7，手动升级到 1.33.4 需使用 kubekey 3.1.12；需确认 3.1.12 能识别现有集群的 kubeadm-config，升级前先在真实环境的测试集群验证（见 §2.1）。
2. **etcd 版本**：kubekey 3.1.12 默认 etcd v3.5.24；当前 etcd v3.5.13 满足 1.33.4 最低要求（3.5.11-0），本次不升级。
3. **feature-gates 残留**：k8s 1.33 移除了 `TTLAfterFinished`、`ExpandCSIVolumes` 等 feature-gate；真实环境升级前必须检查并清理，否则新组件启动失败。qke-cli 的 `removeDeprecatedArgs` 只处理 v1.25/v1.27，**不会自动清理 1.33 的 gate**，必须手动做。
4. **镜像可用性**：控制面镜像需补齐 v1.29.15 / v1.32.8 / v1.33.4 全部版本（逐 minor 升级会拉取中间版本），**必须先把对应镜像推入私有仓库**，否则升级失败。**注意：kk 升级时不会自动升级 coredns**（实测确认）。
5. **数据库一致性**：QKE 平台升级与 k8s 升级后，需保证平台数据库（qke_db_cluster.kubernetes_version、qke_db_component.k8s_versions、qke_db_version.compatibility_k8s）中的版本字段与实际一致，控制台才显示正确版本；只改版本字段，不动状态机字段。
6. **维护窗口**：部分操作（如 pause 升级需重启 containerd）会中断节点业务，需安排在维护窗口执行。
7. **备份先行**：每个升级阶段前都需备份（etcd 快照、kubeadm/kubelet 配置、组件镜像、数据库）。
8. **离线环境**：所有二进制与镜像需在互联网环境准备后导入真实环境，注意架构（amd64/arm64）一致；镜像保存/推送由青云专业人员处理（见 §2.1），勿用 docker save/load（会丢失多架构）。
9. **installer 包备份**：更新 `qke-installer-v3.4.0.tar.gz` 前必须备份原始包（`.20260923` 备份**必须保持不变**），避免覆盖。
10. **KSE 4.1.2 场景（测试环境 B，后续补充）**：若集群安装 KSE 4.1.2，升级 k8s 后需核对 KSE 组件 `k8s_versions` 兼容性并验证 KSE 功能；KSE 安装/升级可能被 k8s 版本兼容性拦截（`CurrentKSENoUpgradeable`）。当前方案先处理无 KSE 情况。

---

## 10. 附录

### 10.1 参考资料

- 本项目内：`refs/upgrade-plan-of-jinke.md`、`refs/k8s-1.33-coredns-compatibility-report.md`、`refs/README.md`；
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

> ⚠️ **重要**：下表为 **kubekey/kubeadm 的默认配套值**。**coredns 版本由 kubekey 版本决定**（**非 QKE**）；**实测确认 kk 升级时不会自动升级 coredns**（QKE 3.3.0 实例为 **1.9.2**，升级后仍为 1.9.x）。**是否需升级 coredns 由 k8s 版本要求决定**——本项目已决策为**建议项**（见 §5.1⑧）。

| k8s 版本 | pause | coredns（kubekey 默认，QKE 不采用） | nodelocaldns |
|----------|-------|---------|--------------|
| 1.27 | 3.9 | v1.10.1 | 1.22.20 |
| 1.28 | 3.9 | v1.10.1 | 1.22.23 |
| 1.29 | 3.9 | v1.11.1 | 1.22.23 |
| 1.30 | 3.9 | v1.11.3 | 1.22.28 |
| 1.31 | 3.10 | v1.11.3 | 1.23.1 |
| 1.32 | 3.10 | v1.11.3 | 1.23.1 |
| 1.33 | 3.10 | v1.12.0 | 1.25.0 |

### 10.3 常用命令速查

> 参考（jinke 方案 + qke 手动升级方案草稿），具体命令以实际环境为准。

**备份类**（每步升级前执行）：

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

# 非 Running Pod 记录（升级前基线）
kubectl get pod -A -owide | grep -v Running
```

> 数据库（qkeplus 库）备份：涉及平台版本/组件记录调整时执行（专业人员操作）。

**检查类**（每步升级前执行）：

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system
kubectl get pod -A -owide | grep -v Running
kubectl top nodes
kubectl -n kube-system get deploy coredns -o yaml | grep -E "image:"
```

**验证类**（升级后验证通用命令，各阶段升级后均可用）：

```bash
# 平台版本（qke-cli 二进制）
qke-cli version

# installer 内 kk 版本
/opt/qingcloud/qke/qke-installer/kk version

# k8s 版本
kubectl version

# 节点状态与系统组件
kubectl get nodes -o wide
kubectl get pods -n kube-system
kubectl get pod -A -owide | grep -v Running
```

**kk 升级类**（第三阶段，来源：qke-cli worklist + qke-artifacts）：

```bash
# 在 master1、installer 目录内执行（./kk 为 kubekey 3.1.12）
cd /opt/qingcloud/qke/qke-installer
./kk upgrade -f config-sample.yaml --skip-pull-images --skip-dependency-check
```

**etcd 升级类**（本次未做，如需可参考）：

```bash
# 每节点：停 etcd → 换二进制 → 起 etcd → 查健康
systemctl stop etcd
tar -zxf /opt/qingcloud/qke/qke-installer/kubekey/etcd/v3.5.24/amd64/etcd-v3.5.24-linux-amd64.tar.gz -C /tmp
cp -f /tmp/etcd-v3.5.24-linux-amd64/etcd* /usr/local/bin/ && chmod +x /usr/local/bin/etcd*
systemctl start etcd
etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/node-master1.pem --key=/etc/ssl/etcd/ssl/node-master1-key.pem endpoint health
```

**镜像保存/推送与 installer 文件包更新**：由**青云专业人员**处理，本方案不展开；见 §2.1 与 `/workspace/qke-artifacts/` 专门文档（更新qke-installer-v3.4.0文件包操作文档、镜像离线保存与推送操作文档）。

---

> **待确认项汇总**：见 [status/project-status.md](../status/project-status.md)「待用户提供的信息」。
