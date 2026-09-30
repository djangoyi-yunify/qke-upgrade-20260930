# AGENTS.md — AI Agent 工作指南

> 本文件是 AI agent 在本项目中工作的**必读指南**。开始任何工作前，先读 [README.md](README.md) 与 [status/project-status.md](status/project-status.md)，掌握项目背景与当前进展。

## 1. 项目一句话

为 **QKE v3.3.0 集群**（k8s v1.27.10，离线环境）提供升级到 **k8s v1.33.4** 的完整升级方案（升级方法、错误处理、回退方法），并持续维护项目管理文档。

## 2. 硬性约束（必须遵守）

1. **产出目录**：本项目所有产出文件一律存放于 `/workspace/qke-upgrade-20260930`。
2. **禁止越界**：**不得修改** `/workspace/qke-upgrade-20260930` 以外的任何文件或目录（包括 `/workspace/qke-docs`、`/workspace/qke-apiserver` 等外部参考，只读引用）。
3. **跳板机/测试集群只读**：对用户提供的跳板机与测试 QKE 集群，**只做只读操作**（查看状态、日志、配置等），**不得修改任何配置**。如需修改，**必须事先逐次征得用户同意**。
4. **不确定就问**：撰写方案时**只依据已知事实**；遇到无法确定的内容，**立即停下询问用户**，不要臆测填充。
5. **迭代撰写**：升级方案采用**先骨架后内容**的方式，通过多轮迭代不断完善，不必一次完成整篇文档。迭代期间**积极询问用户**——部分操作必须由用户完成，部分信息只有用户能提供。

## 3. 工作流

### 3.1 每次开始工作前

1. 读 [README.md](README.md)（项目背景与导航）；
2. 读 [status/project-status.md](status/project-status.md)（当前进展与待办）；
3. 若涉及升级问题，读 [status/issues-log.md](status/issues-log.md)；
4. 若涉及关键决策，读 [status/decisions.md](status/decisions.md)。

### 3.2 工作过程中

- 更新 [status/project-status.md](status/project-status.md) 反映进展（当前阶段、已完成、进行中、待办）；
- 新问题/新决策及时记入 [status/issues-log.md](status/issues-log.md) 与 [status/decisions.md](status/decisions.md)；
- 升级方案主文档在 [docs/upgrade-plan.md](docs/upgrade-plan.md) 中迭代。

### 3.3 每次工作结束

- 确保 [status/project-status.md](status/project-status.md) 反映最新状态；
- 在最终回复中说明：本次完成了什么、下一步建议做什么、有哪些待用户确认/提供的信息。

## 4. 升级方案撰写约定（docs/upgrade-plan.md）

- **结构**：按「升级方法 → 错误处理 → 回退方法」组织每个升级阶段；整体含：方案概述、环境信息、升级前准备、各阶段升级、升级后验证、回退方案、风险与注意事项、附录。
- **事实来源**：只使用已知事实（用户提供、测试验证、参考资料）。引用外部资料时注明来源。
- **占位与待确认**：无法确定的内容用「待确认：…」标注，并在询问用户后填充；不要臆测。
- **版本兼容性**：涉及版本号（k8s/etcd/containerd/kubekey/QKE）时，以用户提供与参考资料为准，必要时交叉核对。

## 5. 工作方法：暂停目标再询问（用户认可的方法，务必沿用）

> 用户明确认可并希望沿用此方法（2026-09-30）。

**核心原则**：遇到需要用户提供/确认的信息时，**先暂停目标（update_goal action=pause），集中列出问题，等待用户回答；用户回答后再恢复目标（update_goal action=resume）继续工作**。不要在提问后继续执行、填充依赖用户回答的章节。

**操作步骤**：

1. 需要用户输入时，用 `get_goal` 获取当前目标 id 与 revision；
2. 用 `update_goal`（action=pause）暂停目标，防止自动续跑触发继续执行；
3. 用 `ask_user_question` 或直接在回复中**集中列出**待确认问题（按优先级分组：阻塞核心章节的 A 类、影响次要章节的 B 类）；
4. **停止执行，等待用户回答**；
5. 用户回答后，用 `update_goal`（action=resume）恢复目标，据回答更新文档；
6. 若仍有未回答的问题，再次暂停并等待，循环直至信息齐备。

**注意**：暂停期间仍可做不依赖用户输入的整理工作（如登记已确认信息、更新状态文档），但**不得填充依赖用户回答的章节**。

## 6. 参考资料索引

- 本项目内：见 [refs/README.md](refs/README.md)。
- 项目外（只读）：`/workspace/qke-docs/`（QKE 平台调研文档）、`/workspace/qke-docs/qke-k8s-manual-upgrade.md`（手动升级方案草稿）、`/workspace/qke-docs/refs/k8s-upgrade-plan-1.23-to-1.33.md`（其他项目升级方案）。
