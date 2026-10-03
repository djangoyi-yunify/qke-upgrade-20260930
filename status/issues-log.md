# 升级问题记录（Issues Log）

> 本文档记录升级过程中遇到的问题（测试环境实测 + 方案推演），是升级方案「错误处理」章节的重要输入。**持续整理中。**

## 记录格式

每条问题包含：编号、阶段、现象、根因、处理/规避方法、状态、来源（测试实测 / 方案推演）。

## 问题清单

| # | 阶段 | 现象 | 根因 | 处理/规避 | 状态 | 来源 |
|---|------|------|------|-----------|------|------|
| 1 | 第一阶段（QKE 3.3.0→3.4.0） | 方案 §4.2 曾记录 cloud-controller-manager、csi-qingcloud-controller、qke-operator 为平台升级"**新增**"组件；用户指出这些组件在 **QKE 3.3.0 就已存在** | **已核实**：新建 QKE 3.3.0 实例（master1 172.22.8.7）中三个组件**全部原生存在**（cloud-controller-manager 1/1、csi-qingcloud-controller 1/1、qke-operator 1/1，均 Running）；此前升级前检查未见，应为检查时点过早（平台组件尚未部署完成） | 方案 §4.2/§2.2.1 已修正为"QKE 3.3.0 原生组件" | ✅ **已解决**（原生存在） | 新建实例核对（2026-09-30） |
| 2 | 第三阶段（1.31.2→1.33.4） | **真实 QKE 实例的 feature-gates 情况目前无法知晓** | 测试环境仅有 `--feature-gates=RotateKubeletServerCertificate=true`（1.33.4 默认开启，清理与否不影响升级），故本次未清理；真实环境若存在 k8s 1.33 已移除的 gate（如 `TTLAfterFinished`、`ExpandCSIVolumes`），残留会导致新 kube-apiserver/kubelet 拒绝启动 | **已决策：暂时按测试环境情况撰写方案**（§6.1④）；真实环境升级前仍建议检查并清理 1.33 已移除的 feature-gate（检查/清理位置见 §6.1④）；qke-cli v3.4.0-patch.1 的 `removeDeprecatedArgs` 只处理 v1.25/v1.27，不会自动清理 | ✅ **已决策**（按测试环境撰写，真实环境待升级前确认） | 用户决策（2026-09-30） |
| 3 | 第三阶段（1.31.2→1.33.4） | 升级后 coredns 镜像仍为 `registry.qingcloud.com/coredns/coredns:1.9.3`，未升级到 v1.12.0 | **已调研**（子 agent 调研 k8s v1.33.4 官方源码/文档）：① coredns 版本由 kubekey 决定（**非 QKE**）；② kk 不自动升级 coredns；③ k8s 1.33 **无 coredns 最低版本要求**，1.9.x 基本可用（监听 v1 EndpointSlice），但 **v1.12.0 是官方配套版本，强烈建议升级**（非"必须"） | **已决策：作为建议项（非必做）**。若升级：准备 v1.12.0 镜像 → `set image` → 核对 pod 与 DNS；若不升级：升级后必须验证 DNS（新建 Service 解析、Headless 变更解析、coredns 日志无 watch 失败告警）（见 §6.1⑦） | ✅ **已决策**（建议项，非必做） | 用户决策（2026-09-30） |
| 4 | 第二阶段（控制台升级 k8s） | 节点 k8s 版本不一致时，QKE 控制台会**阻止升级** | 控制台升级前检查要求所有节点 k8s 版本一致 | **等待 k8s 集群状态稳定、所有节点 k8s 版本一致后再升级**（已并入 §3.1 第 3 步"控制台升级前检查"，QKE 平台升级与 k8s 版本升级通用） | ✅ **已确认**（处理方式） | 用户反馈（2026-09-30） |
| 5 | 第一阶段（QKE 3.3.0→3.4.0） | QKE v3.3.0 实例安装 KSE 4.1.2 时，QKE web 控制台报**兼容性错误**，无法升级到 QKE v3.4.0 | QKE v3.4.0 **实际兼容** KSE 4.1.2，报错是**平台升级检查代码的 bug**（DependencyCheck 误判） | **数据库绕过**（专业人员操作）：① 备份原值 → ② 清空 `kubesphere_version`/`qkcp_version`（让平台误以为未装 KSE）→ ③ 升级完成后恢复原值（SQL 见 [upgrade-plan.md](../docs/upgrade-plan.md) §7.1） | ✅ **已确认**（处理方式） | 用户反馈（2026-09-30） |

## 待办

- [ ] 用户提供测试环境升级过程中遇到的问题清单
- [ ] 按格式登记到上表
- [ ] 将已确认的问题及处理/规避方法同步到 [docs/upgrade-plan.md](docs/upgrade-plan.md) 的「错误处理」章节
- [ ] **问题 #1**：用户新建 QKE 3.3.0 实例后核对 cloud-controller-manager / csi-qingcloud-controller / qke-operator 是否原生存在，据此修正 §4.2
- [x] ~~**问题 #2**：真实环境 feature-gates 情况~~ → ✅ 已决策：暂时按测试环境情况撰写方案（§6.1④）；真实环境升级前仍建议检查清理
- [x] ~~**问题 #1**：核对平台组件是否原生存在~~ → ✅ 已解决：新建 QKE 3.3.0 实例核对确认三个组件原生存在
- [x] ~~**问题 #3**：确认是否需要升级 coredns~~ → ✅ 已决策：**建议项（非必做）**，升级到 v1.12.0 由用户/运维决定
- [x] ~~**问题 #5**：KSE 4.1.2 场景控制台报兼容性错误无法升级 QKE 平台~~ → ✅ 已确认：数据库绕过（清空/恢复 `kubesphere_version`/`qkcp_version`），已同步到 upgrade-plan §3.1/§7.1、test-environment §3.1、decisions D-011
