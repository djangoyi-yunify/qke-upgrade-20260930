# 决策思路与依据

> 本文档记录 QKE 升级方案的**决策思路与依据**（为什么这样做），与 [upgrade-plan.md](upgrade-plan.md)（可操作步骤）分离。
> 决策状态登记见 [status/decisions.md](../status/decisions.md)。
> 更新日期：2026-09-30

## 1. 为什么采用三段式升级路径

- QKE v3.3.0 最高支持 k8s **v1.27.10**，无法直接升级更高 k8s 版本；
- QKE v3.4.0 控制台支持 k8s 1.27.10 / 1.29.10 / 1.31.2，但最高只到 **1.31.2**；
- kubekey 3.1.12 支持到 k8s **v1.33.4**；
- 因此需要三段式：① QKE 平台升级（3.3.0→3.4.0）→ ② 控制台升级 k8s（1.27.10→1.29.10→1.31.2）→ ③ 手动 kk 升级（1.31.2→1.33.4）。

## 2. 为什么必须更新 qke-installer 包（kubekey 版本）

- QKE v3.3.0 使用的 kubekey 版本为 **3.1.0**；
- QKE v3.4.0 使用的 kubekey 版本为 **3.1.7**；
- 但 **kubekey 3.1.7 不支持升级 k8s 到 v1.33.4**，因此**必须更换 QKE v3.4.0 的 qke-installer 包中的 kubekey，使其版本为 3.1.12**；
- 基于以上考虑，才有制作和更新 QKE v3.4.0 qke-installer 包的动作。

> **测试环境与真实环境的差异**：测试环境中用户**调整了 installer 包内的 kubekey 版本**（QKE 3.3.0：3.1.0→3.1.7；QKE 3.4.0：3.1.7→3.1.12）；**真实离线环境** QKE 3.3.0 为 3.1.0、QKE 3.4.0 为 3.1.7（原始），需更新为 3.1.12。

## 3. 为什么 etcd 不升级

- k8s 1.33.4 可以使用 etcd **3.5.13**（满足最低要求 3.5.11-0），无需升级；
- 实测确认：升级后 etcd 仍为 3.5.13，集群正常；
- kubekey 3.1.12 默认 etcd 为 v3.5.24，installer 包内已备 v3.5.24，如需升级可逐节点滚动替换二进制。

## 4. 为什么 feature-gates 不清理（暂时按测试环境情况撰写）

- 测试环境仅有 `--feature-gates=RotateKubeletServerCertificate=true`，在 k8s 1.33.4 中**默认开启**，清理与否不影响升级；
- 真实 QKE 实例的 feature-gates 情况**目前无法知晓**，本方案**暂时按测试环境情况撰写**；
- 真实环境升级前**仍建议**检查并清理 k8s 1.33 已移除的 feature-gate（如 `TTLAfterFinished`、`ExpandCSIVolumes`），残留会导致新 kube-apiserver/kubelet 拒绝启动；
- qke-cli v3.4.0-patch.1 的 `removeDeprecatedArgs` 只处理 v1.25/v1.27，**不会自动清理 1.33 的 gate**。

## 5. 为什么 coredns 升级作为建议项（非必做）

- **coredns 版本由 kubekey 版本决定**（**非 QKE**）；
- **kk 升级时不会自动升级 coredns**（实测确认）；
- **是否需升级 coredns 由 k8s 版本的要求决定**；
- 调研结论（子 agent 调研 k8s v1.33.4 官方源码/文档，2026-09-30）：k8s 1.33 **无 coredns 最低版本要求**，1.9.x 基本可用（监听 v1 EndpointSlice），但 **v1.12.0 是官方配套版本，强烈建议升级**（非"必须"）；
- **已决策**：coredns 升级作为**建议项（非必做）**，是否执行由用户/运维决定。

> 详细调研报告见 [refs/k8s-1.33-coredns-compatibility-report.md](../refs/k8s-1.33-coredns-compatibility-report.md)。

## 6. 为什么需要平台自洽调整（configmap + 数据库）

- 使用户通过 QKE web 控制台管理 QKE 实例时能够**自洽**——控制台能正确显示实例使用的 k8s 版本；
- 垂直/水平扩容、修改配置参数时，`config-sample.yaml` 中的 k8s 版本**不会被错误渲染成 1.31.2** 而造成未知风险。

## 7. 为什么中间版本是 1.29.10 / 1.31.2

- **控制台升级路径**（1.27.10 → 1.29.10 → 1.31.2）是 QKE 控制台提供的路径，在 kubekey 支持的 k8s 版本内，无问题；
- qke-artifacts 资源包准备的控制面镜像 **v1.29.15 / v1.32.8 / v1.33.4** 是 **kubekey 3.1.12 升级阶段**（1.31.2 → 1.33.4）所需的：kubekey 执行升级时**逐 minor 版本**进行，会拉取各中间版本镜像（这是 kubekey 本身的升级逻辑，与是否手动调用无关）；
- 资源包已**预下载 kubekey 升级过程中自动下载的各 k8s 版本所需文件**（etcd、calicoctl、kube 二进制、helm 等），避免离线环境因文件下载故障而停顿。

## 8. 为什么整体升级步骤是 3 步

1. **导入 QKE v3.4.0**（此工作由**青云专业人员**处理）；
2. **更新离线环境中 QKE v3.4.0 的 qke-installer 文件包**（参考 `/workspace/qke-artifacts/` 中的步骤）；
3. **执行具体的升级步骤**（三段式）。

> 第 2 步的必要性见 §2（kubekey 3.1.7 不支持 1.33.4，必须换 3.1.12）。
