# calico-node "already using IPv4 Address" 故障调研报告

> **调研对象**：QKE v3.4.0 集群（k8s 已升级到 v1.33.4，离线环境）中 `calico-node-xxxx` pod 长期 **CrashLoopBackOff**，日志提示 `Kubernetes Calico node 'XXXXXXXXXXX' already using IPv4 Address XXXXXXXXX`。
> **调研内容**：① 报错原因分析；② 用户当前处理方法 `pkill runsvdir` 评估；③ 推荐处理方法（含命令）；④ 离线 QKE 环境注意事项；⑤ 参考资料。
> **用途**：供 [docs/upgrade-plan.md](../docs/upgrade-plan.md) §7.1 问题处理登记（核对/定位方法、处理方法），本文件为详细参考。
> **调研日期**：2026-10-03（基于公开资料与社区案例；2026-10-03 在测试环境 master1 172.22.8.7 实测核对，并更正 calicoctl 获取方式）。

> **实测核对（2026-10-03，测试环境 master1 172.22.8.7）**：
> - calico 版本 **v3.27.4**（`registry.qingcloud.com/calico/node:v3.27.4`），Kubernetes datastore（CRD，cluster type `k8s,bgp,kubeadm,kdd`）；
> - `IP_AUTODETECTION_METHOD=can-reach=$(NODEIP)`（非默认 first-found）；
> - **calico-node 容器内无 calicoctl**（`/usr/local/bin` 仅有 runit 系列），`kubectl exec ... calicoctl` 报 `executable file not found in $PATH`；
> - 主机 `/usr/local/bin/calicoctl`（v3.27.4）可用：`calicoctl node status` BGP 正常、`calicoctl get node` 6 节点；
> - 当前该集群 calico-node 全部 Running（无 CrashLoopBackOff）。

---

## 1. 问题现象与原因分析

### 1.1 现象

- `calico-node-xxxx` pod 状态长期为 **CrashLoopBackOff**；
- 容器日志（calico-node 启动阶段 startup 日志）典型输出如下（社区案例中的真实日志格式）：

```
[INFO][8] startup/startup.go 664: Using autodetected IPv4 address on interface cni0: 10.217.0.1/16
[INFO][8] startup/startup.go 495: Node IPv4 changed, will check for conflicts
[WARNING][8] startup/startup.go 1010: Calico node 'k8s-master-0.k8s.lab' is already using the IPv4 address 10.217.0.1.
[INFO][8] startup/startup.go 263: Clearing out-of-date IPv4 address from this node IP="10.217.0.1/16"
[WARNING][8] startup/startup.go 1214: Terminating
```

即：calico-node 启动时自动检测到本机 IPv4 地址，随后在 Calico 数据存储（datastore）中做冲突检查，发现该 IP **已被记录为某个节点正在使用**，于是主动 `Terminating` 退出，容器反复重启形成 CrashLoopBackOff。

### 1.2 报错机制（calico-node 启动逻辑）

1. calico-node 启动时按 **IP 自动检测**（默认 `first-found`，即"第一个有效网卡上的第一个有效 IP"，排除 docker bridge 等本地接口）确定本节点 IPv4 地址；
2. 将检测到的 IP 与 datastore 中已存在的 **Node 资源**（`projectcalico.org/v3 Node`，含 `spec.bgp.ipv4Address`）比对；
3. 若该 IP 已被**同名节点**（但记录的是另一个 IP）或**其他节点**占用，则判定冲突，打印 `Calico node '<name>' is already using the IPv4 address <IP>.` 并退出。

> 注意：报错中的 "already using IPv4 Address" 是 **calico-node 启动阶段的节点 IP 冲突检查**，与 CNI 为 Pod 分配 IP 的 IPAM 分配是两回事，但两者都可能源于 datastore 中的**残留/重复记录**。

### 1.3 常见根因（按出现频率）

| # | 根因 | 说明 |
|---|------|------|
| 1 | **IPAM/节点记录残留**：节点被删除/下线后，calico 的 Node 资源与 IPAM 分配未清理 | 若未启用 node controller（`calico-kube-controllers` 的 `ENABLED_CONTROLLERS=...,node`）或未按规范 decommission 节点，datastore 中会残留旧节点记录；新节点（同名或同 IP）加入时即冲突。这是社区最常见原因 |
| 2 | **节点记录重复 / 同名不同 IP**：同一节点名在 datastore 中登记了多个 IP | 节点 IP 变更（如 DHCP 重新分配、网卡变化）后，旧 IP 记录未更新/未清理，新 IP 与旧记录冲突 |
| 3 | **上一个 calico-node 实例未完全清理**：旧 pod/旧实例仍占用该 IP | 节点重启、pod 被强制删除但旧实例未释放 IP 时，新实例启动即冲突 |
| 4 | **IP 自动检测选错网卡**：`first-found` 选中了 cni0 / docker 网桥（br-xxx）/ harbor 网桥等 | 检测到的是**网桥 IP**（如 `172.19.0.1`、`10.217.0.1`），该 IP 恰与 datastore 中某节点记录冲突。社区案例：cri-o 的 cni0、harbor 遗留网桥均触发过此问题 |
| 5 | **节点重命名 / 重新加入集群** | 节点名变化但 IP 未变，或反之，导致新旧记录冲突 |

### 1.4 与本次升级场景的关联（推断，需实测确认）

- 本次为 **k8s 升级到 v1.33.4** 后出现，可能诱因包括：
  - 升级过程中节点被重启/替换，节点 IP 或主机名变化，旧 calico 记录残留；
  - 升级期间 calico-node 被反复重建，旧实例未及时释放 IP；
  - 节点上存在多个网卡/网桥（如 docker、harbor、cni0），`first-found` 选错网卡。
- **待确认**：真实集群中冲突的具体 IP、涉及节点、calico 数据存储类型（Kubernetes datastore / etcd datastore）、calico 版本。

---

## 2. `pkill runsvdir` 方法评估

### 2.1 runsvdir 在 calico-node 容器中的角色

- calico-node 镜像内部使用 **runit** 作为进程监督器：`runsvdir` 是 runit 的**监督目录守护进程**，负责拉起并监督容器内的子进程（**felix、bird/bird6、confd** 等）；
- 在 calico-node 容器中，`runsvdir` 通常是 **PID 1（容器主进程）**；
- 因此 **`pkill runsvdir` 会杀掉容器主进程 → 容器退出 → kubelet 按重启策略重建容器 → calico-node pod 重启**。

### 2.2 评估结论

| 维度 | 结论 |
|------|------|
| 是否合理 | **不合理/不推荐**。属于"粗暴重启"，绕过了 k8s 的正常管理手段（`kubectl delete pod`），且**不解决根因** |
| 是否有效 | **仅可能临时有效**。若冲突是**瞬态**（如旧实例尚未释放 IP、检测到的是临时网桥 IP），重启后重新执行 startup 逻辑可能碰巧通过；但**若根因（datastore 残留/重复记录）仍在，重启后必然再次 CrashLoopBackOff** |
| 风险 | ① 掩盖问题：看似"重启后好了"，实际根因未除，后续节点重启/扩容会复发；② 非标准操作：绕过 k8s 生命周期管理，可能留下不一致状态；③ 若冲突 IP 是**网桥/错误网卡**的 IP，重启多少次都没用，反而延误定位；④ 在容器内 `pkill` 属于对运行中容器的侵入操作，不符合"只读排查、规范处理"原则 |

### 2.3 结论一句话

> **`pkill runsvdir` 只能"重启容器"，不能"解决 IP 冲突"；可作为临时应急手段，但必须按 §3 定位并清理根因，否则问题会反复出现。**

---

## 3. 推荐处理方法（含命令）

> 以下命令基于 calico 官方文档与社区案例整理。**执行前先只读排查，确认根因后再做修改类操作**（QKE 环境修改需征得用户同意，见 §4）。

### 3.1 第一步：查看完整日志，定位冲突 IP 与节点

```bash
# 查看当前 pod 日志（含启动阶段 startup 日志）
kubectl logs -n kube-system calico-node-xxxx

# 查看上一次崩溃的日志（CrashLoopBackOff 时关键信息常在 --previous）
kubectl logs -n kube-system calico-node-xxxx --previous

# 查看 pod 事件与重启原因
kubectl describe pod -n kube-system calico-node-xxxx
```

**重点确认**：日志中 `Using autodetected IPv4 address on interface <iface>: <IP>/<mask>` 的**网卡名与 IP**，以及 `already using the IPv4 address <IP>` 的**冲突 IP**。

### 3.2 第二步：检查 calico 节点状态与 IPAM

> ⚠️ **重要更正（2026-10-03 实测）**：**calicoctl 不在 calico-node 镜像内**。自 Calico **v3.20** 起，calicoctl 已从 `calico/node` 镜像中移除，不再随镜像内置；`kubectl exec -n kube-system calico-node-xxxx -- calicoctl ...` 会报 `exec: "calicoctl": executable file not found in $PATH`。calicoctl 需**单独安装**（独立二进制，或使用合并镜像 `calico/calico` 以容器方式运行），且版本须与集群 calico 版本一致。
> **测试环境实测（master1 172.22.8.7，calico v3.27.4）**：calico-node 容器内无 calicoctl（`/usr/local/bin` 仅有 runit 系列）；主机 `/usr/local/bin/calicoctl`（v3.27.4）可用，`calicoctl node status` / `calicoctl get node` 正常。

```bash
# 方式 A：calicoctl（独立二进制，在节点主机上执行；不在 calico-node 容器内）
#   - 前提：主机已安装与集群版本一致的 calicoctl（如 /usr/local/bin/calicoctl）
#   - 前提：主机可访问集群（kubeconfig，或 DATASTORE_TYPE=kubernetes + KUBECONFIG）
calicoctl node status
calicoctl get node
calicoctl ipam show
calicoctl ipam check

# 方式 A'：若主机无 calicoctl 二进制，可用合并镜像 calico/calico 以容器方式运行（需离线环境已有该镜像）
# docker run --rm -v /root/.kube:/root/.kube calico/calico:v3.27.4 calicoctl node status

# 方式 B：kubectl 直接查看 calico CRD（Kubernetes datastore 场景，无需 calicoctl）
kubectl get nodes -o wide
kubectl get node -o yaml <nodeName>          # 查看 projectcalico.org/IPv4Address 等注解
kubectl get ippool -o yaml                    # 查看 IP 池
kubectl get ipamhandle -A -o yaml             # 查看 IPAM 分配句柄（若 CRD 存在）
```

**重点确认**：
- 冲突 IP 属于哪个节点、哪个网卡；
- 是否存在**同名节点记录**（`calicoctl get node` 中同一名字出现多次/IP 不一致）；
- 是否存在**已下线节点**的残留记录。

### 3.3 第三步：确认冲突 IP 归属，判断是否残留

- 若冲突 IP 属于**已下线/已删除的节点** → 残留记录，需清理；
- 若冲突 IP 属于**本节点旧记录**（节点 IP 变更后未更新）→ 需更新/清理旧记录；
- 若冲突 IP 是**网桥/错误网卡**的 IP（如 `172.19.0.1`、`10.217.0.1`、`br-xxx`）→ 根因是 **IP 自动检测选错网卡**，走 §3.5。

### 3.4 第四步：清理残留记录（确认是残留后再执行）

> calicoctl 在**节点主机**上执行（不在 calico-node 容器内，见 §3.2 更正）。

```bash
# 删除已下线节点的 calico Node 资源（会连带清理其 WorkloadEndpoint/HostEndpoint/IP 等子资源）
# ⚠️ 仅当该节点确实已下线/不再服务时执行，否则会影响在网节点
calicoctl delete node <已下线节点名>

# 释放 IPAM 中残留的 IP 分配（仅用于"未干净删除的端点"遗留地址）
# ⚠️ 不会从仍在使用的端点移除 IP，务必先确认该 IP 无实际使用者
calicoctl ipam release --ip=<冲突IP>
```

> 参考：官方 [Decommission a node](https://docs.tigera.io/calico/latest/operations/decommissioning-a-node) 明确将 "receiving an error about an IP address already in use" 列为需要手动 decommission 节点的场景之一。

### 3.5 第五步：若为"选错网卡"，修正 IP 自动检测

在 calico-node DaemonSet 中设置 `IP_AUTODETECTION_METHOD`（或 operator 场景的 `nodeAddressAutodetectionV4`）：

```bash
# 查看当前配置
kubectl get ds -n kube-system calico-node -o yaml | grep -A2 IP_AUTODETECTION_METHOD

# 修改为指定网卡（推荐，如 eth0）
#   IP_AUTODETECTION_METHOD=interface=eth0
# 或修改为"能到达某地址的源 IP"（如集群内某 Ready 节点 IP）
#   IP_AUTODETECTION_METHOD=can-reach=<某Ready节点IP>
kubectl edit ds -n kube-system calico-node
```

> 参考：官方 [Configure IP autodetection](https://docs.tigera.io/calico/latest/networking/ipam/ip-autodetection) 支持 `first-found`（默认）、`kubernetes-internal-ip`、`can-reach`、`interface`、`skip-interface`、`cidrs` 等方法。

### 3.6 第六步：正确重启 calico-node

```bash
# 推荐：删除 pod，由 DaemonSet 自动重建（不要用 pkill runsvdir）
kubectl delete pod -n kube-system calico-node-xxxx

# 若节点上 calico 服务以 systemd 方式运行（非 DaemonSet 场景），则重启服务
# systemctl restart calico-node   # 视实际部署方式而定
```

重启后观察：

```bash
kubectl get pod -n kube-system -o wide | grep calico-node
kubectl logs -n kube-system calico-node-<新pod> -f
```

**验证通过标准**：pod 进入 `Running` 且 `1/1 Ready`；日志不再出现 `already using the IPv4 address`；主机上 `calicoctl node status` 显示 BGP 正常建立。

---

## 4. 注意事项（离线 QKE 环境）

1. **只读优先、修改需确认**：按项目约定，对真实集群只做只读操作；§3.4/§3.5 的修改类操作（删除 node 资源、释放 IP、改 DaemonSet env）**必须事先逐次征得用户同意**。
2. **calicoctl 获取方式（重要更正）**：**calico-node 镜像不内置 calicoctl**（自 v3.20 起已移除），不能用 `kubectl exec` 进容器执行。离线环境需**预先将 calicoctl 二进制放到节点主机**（版本与集群 calico 一致，如测试环境 `/usr/local/bin/calicoctl` v3.27.4），或使用离线已有的合并镜像 `calico/calico` 以容器方式运行；也可直接用 `kubectl` 查看 calico CRD（Kubernetes datastore 场景，无需 calicoctl）。
3. **数据存储类型**：先确认 calico 使用 **Kubernetes datastore（CRD）** 还是 **etcd datastore**。QKE 通常为 Kubernetes datastore，此时 Node 资源即 CRD，`kubectl get node`（calico CRD）与 `kubectl get nodes`（k8s 节点）是两套对象，注意区分。
4. **平台托管组件**：QKE 平台可能对 calico DaemonSet/配置有**回写/纳管**逻辑，直接 `kubectl edit ds` 修改 env 可能被平台覆盖；修改前确认平台是否纳管，必要时通过 QKE 控制台/平台配置修改。
5. **离线限制**：无法访问互联网拉取新镜像/文档，**不要尝试更换 calico 镜像或升级 calico 版本来"修复"**；所有操作基于现有镜像与二进制。
6. **升级场景联动**：本次问题出现在 k8s 升级到 v1.33.4 之后，处理前先确认是否与升级动作（节点重启、pod 重建、主机名/IP 变化）相关，避免只修 calico 而遗漏升级侧根因。
7. **登记到主文档**：核对/定位方法与处理方法要点登记到 [docs/upgrade-plan.md](../docs/upgrade-plan.md) §7.1；本文件为详细参考，登记到 [refs/README.md](README.md) 索引。

---

## 5. 参考资料

### 官方文档
- [Calico: Install calicoctl](https://docs.tigera.io/calico/latest/operations/calicoctl/install) —— calicoctl 为**独立工具**，需单独安装（二进制或 `calico/calico` 合并镜像容器方式），版本须与集群一致；**v3.20 起不再随 calico-node 镜像内置**
- [Calico: Configure IP autodetection](https://docs.tigera.io/calico/latest/networking/ipam/ip-autodetection) —— IP 自动检测方法（first-found / can-reach / interface 等）
- [Calico: Decommission a node](https://docs.tigera.io/calico/latest/operations/decommissioning-a-node) —— 官方明确将 "IP address already in use" 列为需手动 decommission 节点的场景；`calicoctl delete node`
- [Calico: calicoctl ipam release](https://docs.tigera.io/calico-enterprise/latest/reference/clis/calicoctl/ipam/release) —— 释放 IPAM 残留 IP 的命令与注意事项
- [Calico: calicoctl ipam show](https://docs.tigera.io/calico-enterprise/3.21/reference/clis/calicoctl/ipam/show) —— 查看 IPAM 分配
- [Calico: calicoctl node status](https://docs.tigera.io/calico-enterprise/latest/reference/clis/calicoctl/node/status) —— 查看节点 BGP 状态
- [Calico: Anatomy of a calico-node container](https://raw.githubusercontent.com/projectcalico/calico/refs/heads/release-legacy/v3.0/reference/architecture/components.md) —— calico-node 容器内组件（felix/bird/confd）与 **runit（runsv/svlogd）** 角色说明

### GitHub Issue / 社区案例
- [projectcalico/calico#3616: Worker nodes calico-node CrashloopBackOff due to already allocated IP](https://github.com/projectcalico/calico/issues/3616) —— 真实日志（`already using the IPv4 address` + `Clearing out-of-date IPv4 address` + `Terminating`）；根因：`first-found` 选中 cni0 网桥 IP 冲突；解决：显式配置 IP 自动检测
- [projectcalico/calico#4611: IPv4Address annotation pointing to wrong node's IP CIDR](https://github.com/projectcalico/calico/issues/4611) —— 节点 IPv4 注解指向错误 IP 的相关问题
- [Kops で calico-node が CrashLoopBackOff になる事象（varu3/techBlog）](https://varu3.hatenablog.com/entry/2018/06/07/190000) —— 根因：节点删除后 etcd 中 calico host 记录残留，新节点同 IP 冲突；解决：启用 calico-kube-controllers 的 node controller（`ENABLED_CONTROLLERS=...,node`）+ `CALICO_K8S_NODE_REF`，并手动清理既有残留
- [CSDN: calico 报错 Calico node 'node1' is already using the IPv4 address](https://blog.csdn.net/vah101/article/details/104968657) —— 根因：harbor 遗留网桥（br-xxx）被 `first-found` 选中，删除网桥后重启 pod 解决
- [超图 FAQ: calico-node 容器报错 already using the IPv4 address](https://ask.supermap.com/122759) —— 处理：修改 `IP_AUTODETECTION_METHOD` 为 `can-reach`/`interface` 策略选择正确 IP

### 其他
- [Stack Overflow: Kubernetes Calico node already using IPv4 Address, CrashLoopBackOff](https://stackoverflow.com/questions/52390481/kubernetes-calico-node-xxxxxxxxxxx-already-using-ipv4-address-xxxxxxxxx-crash) —— 同报错问题讨论（页面需登录/反爬，正文未完整获取，仅作线索）
