# 升级问题记录（Issues Log）

> 本文档记录升级过程中遇到的问题（测试环境实测 + 方案推演），是升级方案「错误处理」章节的重要输入。**持续整理中。**

## 记录格式

每条问题包含：编号、阶段、现象、根因、处理/规避方法、状态、来源（测试实测 / 方案推演）。

## 问题清单

| # | 阶段 | 现象 | 根因 | 处理/规避 | 状态 | 来源 |
|---|------|------|------|-----------|------|------|
| 1 | 第一阶段（QKE 3.3.0→3.4.0） | 方案 §4.2 曾记录 cloud-controller-manager、csi-qingcloud-controller、qke-operator 为平台升级"**新增**"组件；用户指出这些组件在 **QKE 3.3.0 就已存在**，该"新增"结论**存疑** | 待查：升级前只读检查（master1 172.22.8.17 实例）时仅见 calico-kube-controllers、coredns 两个 Deployment，未见上述组件；可能是该实例为新建、平台组件尚未部署完成，或检查时点/实例不同 | 待验证：用户在当前 QKE 实例完成整个升级后，**新建一个 QKE 3.3.0 实例**核对上述组件是否原生存在 | **待确认** | 用户反馈（2026-09-30） |
| 2 | 第三阶段（1.31.2→1.33.4） | **真实 QKE 实例的 feature-gates 可能与测试环境存在差异**，该信息目前无法获取 | 测试环境仅有 `--feature-gates=RotateKubeletServerCertificate=true`（1.33.4 默认开启，清理与否不影响升级），故本次未清理；真实环境若存在 k8s 1.33 已移除的 gate（如 `TTLAfterFinished`、`ExpandCSIVolumes`），残留会导致新 kube-apiserver/kubelet 拒绝启动 | 真实环境升级前**必须检查并清理** 1.33 已移除的 feature-gate（检查/清理位置见 §6.1④）；qke-cli v3.4.0-patch.1 的 `removeDeprecatedArgs` 只处理 v1.25/v1.27，不会自动清理 | **待确认**（信息无法获取） | 用户反馈（2026-09-30） |
| 3 | 第三阶段（1.31.2→1.33.4） | 升级后 coredns 镜像仍为 `registry.qingcloud.com/coredns/coredns:1.9.3`，**未升级到 v1.12.0**（此前资料提到 k8s 1.33 目标 coredns 为 v1.12.0） | 待查：`config-sample.yaml` 未指定 coredns 版本，由 kubekey 决定；可能 QKE 使用自有 coredns 版本，或 coredns 升级需单独步骤 | 待确认：coredns 1.9.3 在 k8s 1.33.4 下是否满足要求；是否需要单独升级 coredns | **待确认** | 只读检查（2026-09-30） |

## 待办

- [ ] 用户提供测试环境升级过程中遇到的问题清单
- [ ] 按格式登记到上表
- [ ] 将已确认的问题及处理/规避方法同步到 [docs/upgrade-plan.md](docs/upgrade-plan.md) 的「错误处理」章节
- [ ] **问题 #1**：用户新建 QKE 3.3.0 实例后核对 cloud-controller-manager / csi-qingcloud-controller / qke-operator 是否原生存在，据此修正 §4.2
- [ ] **问题 #2**：真实环境升级前检查 feature-gates，确认是否存在 1.33 已移除项
- [ ] **问题 #3**：确认 coredns 1.9.3 是否满足 k8s 1.33.4 要求，是否需要单独升级
