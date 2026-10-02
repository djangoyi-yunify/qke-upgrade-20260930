# QKE 集群 Kubernetes 升级项目（qke-upgrade-20260930）

> 本项目为 **QKE v3.3.0 集群升级 Kubernetes** 提供完整升级方案（升级方法、错误处理、回退方法），并沉淀项目管理文档，使 AI agent 能据此掌握项目情况与进展、持续工作。
>
> **项目起始日期**：2026-09-30

## 1. 项目背景与目标

已有 QKE 集群存在升级 Kubernetes 版本的需求。集群运行环境**无法访问互联网**（离线环境）。

**集群现状**：

| 项 | 当前值 |
|----|--------|
| QKE 平台版本 | v3.3.0 |
| Kubernetes | v1.27.10 |
| containerd | v1.7.13 |
| etcd | v3.5.13 |
| OS | Ubuntu 20.04.5 |

**QKE 对 k8s 升级的支持情况**：

- QKE v3.3.0 最高支持 k8s **v1.27.10**；
- QKE v3.4.0 最高支持 k8s **v1.31.2**；
- QKE 底层使用 KubeKey（`kk`）升级 k8s；QKE 3.3.0 与 3.4.0 均使用 kubekey **3.1.x**，其中 QKE 3.4.0 使用 **kubekey 3.1.7**；
- kubekey **3.1.12**（3.1.x 线最高 tag）支持到 k8s **v1.33.4**。

**拟采用的升级思路（三段式）**：

```
① QKE 平台升级：v3.3.0 → v3.4.0（保持 k8s 版本 v1.27.10 不变）
   ↓
② 通过 QKE web 控制台升级 k8s（QKE 3.4.0 控制台支持 1.27.10 / 1.29.10 / 1.31.2）：
   v1.27.10 → v1.29.10 → v1.31.2
   ↓
③ 手动执行 kk 命令升级 k8s 到 kubekey 3.1.12 支持的最高版本：v1.31.2 → v1.33.4
```

**最终目标**：k8s **v1.33.4**。

## 2. 已做工作

1. 已在**测试环境**完成升级测试，初步验证了整个升级过程；
2. 收集了升级过程中遇到的问题（**整理中**，见 [status/issues-log.md](status/issues-log.md)）；
3. 获取了其他项目 k8s 升级文档作为参考：[refs/upgrade-plan-of-jinke.md](refs/upgrade-plan-of-jinke.md)；
4. 初步撰写了 QKE v3.3.0 手动升级 Kubernetes 方案草稿（外部参考，位于 `/workspace/qke-docs/qke-k8s-manual-upgrade.md`，**不在本项目目录内，仅作参考**）；
5. 调研了 QKE 平台相关内容并撰写文档（外部参考，位于 `/workspace/qke-docs`，**不在本项目目录内，仅作参考**）。

## 3. 目录结构

```
qke-upgrade-20260930/
├── README.md                 # 项目总览（本文档，AI agent 入口）
├── AGENTS.md                 # AI agent 工作指南（约束、工作流、如何持续工作）
├── docs/                     # 产出文档
│   ├── upgrade-plan.md       # ⭐ 升级方案（主文档：背景+可操作步骤+问题处理+回退）
│   ├── test-environment.md   # 测试环境情况（实测数据、验证结果、观察项）
│   └── decisions-rationale.md# 决策思路与依据（为什么这样做）
├── refs/                     # 参考资料
│   ├── README.md             # 参考资料索引
│   ├── upgrade-plan-of-jinke.md  # 其他项目 k8s 升级方案参考
│   └── k8s-1.33-coredns-compatibility-report.md  # coredns 兼容性调研报告
└── status/                   # 项目管理
    ├── project-status.md     # 项目状态与进展（当前阶段、已完成、进行中、待办）
    ├── issues-log.md         # 升级问题记录（测试中遇到的问题，持续整理）
    └── decisions.md          # 决策记录（关键决策与理由）
```

## 4. 关键文档导航

| 文档 | 用途 | 何时阅读 |
|------|------|----------|
| [AGENTS.md](AGENTS.md) | AI agent 工作指南 | **每次开始工作前必读** |
| [status/project-status.md](status/project-status.md) | 项目当前状态与进展 | 每次开始工作前必读 |
| [docs/upgrade-plan.md](docs/upgrade-plan.md) | 升级方案主文档（背景+步骤+问题处理+回退） | 撰写/评审升级方案时 |
| [docs/test-environment.md](docs/test-environment.md) | 测试环境情况（实测数据、验证结果） | 需要测试环境实测信息时 |
| [docs/decisions-rationale.md](docs/decisions-rationale.md) | 决策思路与依据（为什么这样做） | 需要了解决策背景时 |
| [status/issues-log.md](status/issues-log.md) | 升级问题记录 | 整理问题、写错误处理章节时 |
| [status/decisions.md](status/decisions.md) | 决策记录 | 需要了解关键决策背景时 |
| [refs/README.md](refs/README.md) | 参考资料索引 | 查找参考资料时 |

## 5. 外部参考（不在本项目目录内，只读引用）

| 路径 | 内容 |
|------|------|
| `/workspace/qke-docs/qke-k8s-manual-upgrade.md` | QKE v3.3.0 手动升级 Kubernetes 方案草稿（另一条升级思路的参考） |
| `/workspace/qke-docs/` | QKE 平台调研文档（installer 包、qke-cli、qke-apiserver、KSE 版本支持等） |
| `/workspace/qke-docs/refs/k8s-upgrade-plan-1.23-to-1.33.md` | 其他项目 k8s 升级方案（v1.23.17 → v1.33.7） |

> **约束**：本项目所有产出文件均存放于 `/workspace/qke-upgrade-20260930`；**不得修改**该目录以外的任何文件或目录。对用户提供的跳板机与测试 QKE 集群只做**只读**操作，如需修改必须事先征得用户同意。
