# Kubernetes Pod 长时间 Terminating 故障排查与处理记录

## 一、问题描述

更新 Kubernetes 后，节点 `worker-s002` 上有一个 Pod 长时间处于 `Terminating` 状态，无法被清理。

**Pod 名称**：`kube-proxy-9crxf`（kube-system 命名空间，DaemonSet 组件）

**Pod 日志报错**：

```
unable to retrieve container logs for containerd://a08b9e361bfc748fa7495e509035e355ae52cf8722e23c5fd60d4a6e24617ae0
```

**节点 kubelet 日志报错**（反复出现）：

```
time="2026-09-28T12:15:09+08:00" level=error msg="Failed to remove cgroup" error="rmdir /sys/fs/cgroup/net_cls,net_prio/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-pod76399cc0_aafc_4e1e_8fc5_6d1bde109fa5.slice/cri-containerd-4c897f3bf1432dc8b8ae0de2480daa172d658a6677f6bf39b9b663e4b6314ad1.scope: device or resource busy"
```

涉及 cgroup 控制器：`net_cls,net_prio`、`devices`、`hugetlb`、`unified`、`memory`、`cpu,cpuacct`、`rdma` 等，全部报 `device or resource busy`（EBUSY）。

---

## 二、故障排查与定位

### 2.1 初步判断

kubelet 报 `device or resource busy`（EBUSY）说明：**cgroup 目录内仍有进程（task）存在**。Linux 内核规定，只有当 cgroup 内没有任何 task 时才能 rmdir 删除该目录。因此 Pod 卡 Terminating 的直接原因是**容器 cgroup 内有残留进程**。

### 2.2 排查步骤与关键证据

| 步骤 | 命令 | 结果 | 结论 |
|------|------|------|------|
| 1 | `cat .../cri-containerd-4c897f3b...scope/cgroup.procs` | 输出 `17656` | 容器 cgroup 内残留 1 个进程，即 EBUSY 的直接原因 |
| 2 | `cat .../kubepods-besteffort-pod76399cc0...slice/cgroup.procs` | 空 | 残留进程在容器 scope 内，不在 Pod 级 |
| 3 | `ps -eo pid,ppid,stat,wchan:30,cmd \| grep -E '^ *[0-9]+ D'` | 无结果 | ⚠️ 该命令有 bug（正则未考虑 PPID 列），**漏检了 D 状态** |
| 4 | `crictl ps -a \| grep -E 'a08b9e36\|4c897f3b'` | 无结果 | containerd 已完全丢失这两个容器的元数据 |
| 5 | `ps -ef \| grep 17656` | `root 17656 1 0 10:48 ? 00:00:00 runc init` | 残留进程是 `runc init`（容器 init 进程），父进程为 systemd（PID 1） |
| 6 | `ps -ef \| grep runc` | 所有 `containerd-shim-runc-v2` 的 PPID 均为 1，但**没有任何 shim 的 id 是 `4c897f3b...`** | 该容器的 shim 已消失，`runc init` 是纯孤儿进程 |
| 7 | `crictl ps -a`（全量） | 找不到 pod UID `76399cc0...` 的任何容器；但存在新的 `kube-proxy-kltmv`（Running，attempt 0） | 旧 pod 已被 force delete，DaemonSet 已创建替代 pod；containerd 对旧 pod 一无所知 |
| 8 | `kill -9 17656` | **进程未死** | 进程处于不可中断状态，SIGKILL 无效 |
| 9 | `ps -o pid,ppid,stat,wchan:30,cmd -p 17656` | `17656 1 Dsl refrigerator runc init` | **决定性证据**：STAT=`Dsl`（不可中断睡眠），WCHAN=`refrigerator` → 进程被 **cgroup freezer 冻结** |

### 2.3 根因分析

`refrigerator` 是内核 cgroup freezer（冻结器）的等待点。进程被冻结后处于 TASK_FROZEN 状态，**不处理任何信号**，因此 `kill -9` 无效。

完整事件链：

```
升级 k8s → containerd 重启 → kube-proxy-9crxf 容器元数据丢失、shim 消失
→ kubelet 终止 pod 时已通过 freezer cgroup 冻结容器进程
→ 但"解冻 + 发送 SIGKILL"这一步因 containerd 丢失跟踪而永远未执行
→ 进程 17656 永久冻结在 refrigerator() 里
→ kill -9 排队但不生效 → 进程占着 cgroup → kubelet 删不掉 → EBUSY
→ Pod 卡 Terminating
```

所有现象（取不到日志、crictl 查不到、kill -9 无效、cgroup 删不掉）均由此根因解释。

---

## 三、处理方法

### 3.1 解冻被冻结的进程

```bash
# 1. 确认 freezer 状态（显示 FROZEN 或 FREEZING）
cat /sys/fs/cgroup/freezer/kubepods.slice/kubepods-besteffort.slice/\
kubepods-besteffort-pod76399cc0_aafc_4e1e_8fc5_6d1bde109fa5.slice/\
cri-containerd-4c897f3bf1432dc8b8ae0de2480daa172d658a6677f6bf39b9b663e4b6314ad1.scope/freezer.state

# 2. 解冻（THAWED）
echo THAWED > /sys/fs/cgroup/freezer/kubepods.slice/kubepods-besteffort.slice/\
kubepods-besteffort-pod76399cc0_aafc_4e1e_8fc5_6d1bde109fa5.slice/\
cri-containerd-4c897f3bf1432dc8b8ae0de2480daa172d658a6677f6bf39b9b663e4b6314ad1.scope/freezer.state

# 3. 解冻后，之前排队的 SIGKILL 会立即生效；若未生效再补一刀
kill -9 17656

# 4. 确认进程已消失、cgroup 已清空
ps -p 17656
cat /sys/fs/cgroup/cpu,cpuacct/kubepods.slice/kubepods-besteffort.slice/\
kubepods-besteffort-pod76399cc0_aafc_4e1e_8fc5_6d1bde109fa5.slice/\
cri-containerd-4c897f3bf1432dc8b8ae0de2480daa172d658a6677f6bf39b9b663e4b6314ad1.scope/cgroup.procs
```

### 3.2 收尾清理

```bash
# 让 kubelet 完成剩余清理（可选，cgroup 已删除时非必需）
systemctl restart kubelet

# 确认 API 侧旧 pod 已不存在（若还在 Terminating 则 force delete）
kubectl get pod -n kube-system | grep kube-proxy
kubectl delete pod kube-proxy-9crxf -n kube-system --force --grace-period=0   # 仅当上一步还有输出

# 验证
kubectl get nodes
kubectl get pod -n kube-system | grep kube-proxy   # 应只剩 kube-proxy-kltmv
```

---

## 四、处理结果

- `freezer.state` 由 `FROZEN` 变为 THAWED 后，**cgroup 目录被成功删除**（freezer.state 文件随之消失，`No such file or directory`）；
- 孤儿进程 `runc init`（PID 17656）**已消失**；
- 节点上剩余的 8 个 `containerd-shim-runc-v2` 全部是 containerd 正常跟踪的健康 pod（calico-node、kube-proxy-kltmv、csi-qingcloud、nodelocaldns）；
- 集群功能层面早已恢复（新 pod `kube-proxy-kltmv` 正常运行），本次处理彻底清除了节点残留，**问题解决**。

---

## 五、经验总结与预防措施

### 5.1 快速诊断要点（遇到类似问题可复用）

1. `crictl ps -a` 查不到容器 + `ps -ef` 看到孤儿 `runc init` → containerd 丢失容器跟踪；
2. `kill -9` 无效 → 用 `ps -o pid,ppid,stat,wchan:30,cmd -p <pid>` 查状态，看到 `Dsl` + `refrigerator` → 被 cgroup freezer 冻结；
3. 解法：`echo THAWED > .../freezer.state` 解冻 → 进程立即被杀 → cgroup 自动清理。

### 5.2 预防措施

1. **升级前先 drain 节点**，避免容器在 containerd 重启时被"孤儿化"：
   ```bash
   kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
   # 升级完成后
   kubectl uncordon <node>
   ```
2. 升级 containerd/kubelet 时，避免在容器运行期间直接重启运行时；
3. 升级后检查节点上是否有异常 Terminating 的 pod 和孤儿 `runc init` 进程。

---

## 附：关键日志与命令原文

### Pod 日志

```
unable to retrieve container logs for containerd://a08b9e361bfc748fa7495e509035e355ae52cf8722e23c5fd60d4a6e24617ae0
```

### kubelet 日志（节选）

```
time="2026-09-28T12:15:09+08:00" level=error msg="Failed to remove cgroup" error="rmdir /sys/fs/cgroup/net_cls,net_prio/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-pod76399cc0_aafc_4e1e_8fc5_6d1bde109fa5.slice/cri-containerd-4c897f3bf1432dc8b8ae0de2480daa172d658a6677f6bf39b9b663e4b6314ad1.scope: device or resource busy"
```

### 决定性诊断输出

```
# ps -o pid,ppid,stat,wchan:30,cmd -p 17656
  PID  PPID STAT WCHAN                          CMD
17656     1 Dsl  refrigerator                   runc init
```

### 关键路径

- Pod UID：`76399cc0-aafc-4e1e-8fc5-6d1bde109fa5`
- 容器 ID（cgroup 残留）：`4c897f3bf1432dc8b8ae0de2480daa172d658a6677f6bf39b9b663e4b6314ad1`
- 容器 ID（取不到日志）：`a08b9e361bfc748fa7495e509035e355ae52cf8722e23c5fd60d4a6e24617ae0`
- 节点：`worker-s002`
