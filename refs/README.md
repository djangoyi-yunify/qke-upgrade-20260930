# 参考资料索引（refs/）

> 本项目参考资料统一存放于 `refs/` 目录，并在本文档登记索引。新增资料后必须更新本文档。

## 本项目内参考资料

| 文件 | 类型 | 说明 |
|------|------|------|
| [upgrade-plan-of-jinke.md](upgrade-plan-of-jinke.md) | 文档 | 其他项目 k8s 升级方案参考（v1.23.17 → v1.33.7）：版本兼容表、升级步骤、回退方案。**注意：该方案场景与本项目不同（非 QKE 平台升级路径），仅作方法论参考** |

## 项目外参考资料（只读引用，不在本项目目录内）

| 路径 | 类型 | 说明 |
|------|------|------|
| `/workspace/qke-artifacts/README.md` | 文档 | ⭐ **QKE 3.4.0 离线升级 k8s 到 v1.33.4 资源包**：升级思路、kk 升级命令、installer 文件包布局、镜像清单、下载地址与 SHA256 |
| `/workspace/qke-artifacts/更新qke-installer-v3.4.0文件包操作文档.md` | 文档 | qke-installer 文件包更新操作（kk 3.1.7→3.1.12 + 依赖 + helm v3.18.5） |
| `/workspace/qke-artifacts/镜像离线保存与推送操作文档.md` | 文档 | 镜像离线保存/推送（crane，multi-arch），12 个控制面镜像 |
| `/workspace/qke-artifacts/青云公有云环境调整记录.md` | 文档 | 青云公有云环境现状（对象存储/OCI 注册服务已做调整） |
| `/workspace/qke-artifacts/CONTEXT.md` | 文档 | 术语表（私有对象存储/私有 OCI 注册服务/project/crane 等） |
| `/workspace/qke-cli/manual-upgrade-1.33.4-worklist.md` | 文档 | ⭐ **QKE v3.4.0（k8s 1.31.2）手动升级到 1.33.4 工作清单**：备份、installer 同步、etcd 升级、feature-gates 清理、kk 升级命令、数据库字段调整、回退 |
| `/workspace/qke-docs/qke-k8s-manual-upgrade.md` | 文档 | QKE v3.3.0 手动升级 Kubernetes 方案草稿（另一条升级思路：不升级 QKE 平台，直接手动升级 k8s + 改造 installer 包 + 调整数据库字段） |
| `/workspace/qke-docs/refs/k8s-upgrade-plan-1.23-to-1.33.md` | 文档 | 其他项目 k8s 升级方案（v1.23.17 → v1.33.7），与 `upgrade-plan-of-jinke.md` 内容一致 |
| `/workspace/qke-docs/installer-build-guide.md` | 文档 | qke-installer 制作方法（离线包构建指南）：包内结构、KubeKey manifest/artifact 机制 |
| `/workspace/qke-docs/installer-offline.md` | 文档 | qke-installer 离线部署原理与镜像清单 |
| `/workspace/qke-docs/installer-download.md` | 文档 | qke-cli 对 installer 的下载与处理 |
| `/workspace/qke-docs/qke-cli.md` | 文档 | qke-cli 功能分析（action 清单、config-sample.yaml 渲染、kk 调用方式） |
| `/workspace/qke-docs/qke-apiserver-api.md` | 文档 | qke-apiserver 对外 API（升级链路 UpgradeCluster 等） |
| `/workspace/qke-docs/qke-apiserver-getkseversions.md` | 文档 | GetKSEVersions API 处理链路（KSE 版本目录、k8s 兼容过滤） |
| `/workspace/qke-docs/qke-kse-version-support.md` | 文档 | KSE 版本支持与数据库表结构（qke_db_component/qke_db_version/qke_db_cluster） |
| `/workspace/qke-docs/refs/manifest-demo.yaml` | 配置 | QKE 真实 manifest 样例（离线镜像清单、组件版本） |

## 待补充资料

- [ ] 测试环境升级过程中遇到的问题清单（用户整理中）
- [ ] 测试环境/生产环境集群信息
- [ ] QKE 3.3.0 → 3.4.0 平台升级的具体方法（用户确认中）
