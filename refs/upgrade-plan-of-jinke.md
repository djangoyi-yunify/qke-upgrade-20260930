|  |  |  |  |  |
| --- | --- | --- | --- | --- |
| **版本** | **版本控制信息** | **更新日期** | **更新人** | **审批人** |
| 1.0 | 创建 | 2024.9 |  |  |
|  |  |  |  |  |
|  |  |  |  |  |

目录

[1 上线方案概述 1](#_Toc177043463)

[2 上线时间与人员 1](#_Toc177043464)

[2.1 上线时间 1](#_Toc177043465)

[2.2 参与人员 1](#_Toc177043466)

[3 升级内容 2](#_Toc177043467)

[4 影响范围 3](#_Toc177043468)

[5 前期准备 3](#_Toc177043469)

[6 操作步骤 3](#_Toc177043470)

[6.1 准备工作 3](#_Toc177043471)

[6.2 升级步骤 4](#_Toc177043472)

[6.2.1 备份 4](#_Toc177043473)

[6.2.2 修改configmap 4](#_Toc177043474)

[6.2.3 修改config-sample.yaml 5](#_Toc177043475)

[6.2.4 修改kube-scheduler和kube-controller-manager参数 5](#_Toc177043476)

[6.2.5 升级k8s 6](#_Toc177043477)

[6.3 集群检查 6](#_Toc177043478)

[6.4 容器云平台更新kubeconfig 7](#_Toc177043479)

[6.5 上线验证 7](#_Toc177043480)

[6.5.1 验证平台基础功能 7](#_Toc177043481)

[6.5.2 验证业务空间网关 7](#_Toc177043482)

[6.5.3 验证各组件 7](#_Toc177043483)

[6.5.4 通过巡检功能对当前集群巡检 8](#_Toc177043484)

[7 回退方案 8](#_Toc177043485)

[7.1 回退条件 8](#_Toc177043486)

[7.2 回退计划 8](#_Toc177043487)

[7.3 回退步骤 8](#_Toc177043488)

[8 变更结果通知 9](#_Toc177043489)

[8.1 邮件通知 9](#_Toc177043490)

[8.2 微信通知 9](#_Toc177043491)

# 上线方案概述

按照规划，预计对k8s进行升级，预计从v.1.23.17升级到v1.33.7版本

kubernetes v.1.23.17升级到v1.33.7 Changlog

https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.33.md

# 上线时间与人员 - 待补充

## 上线时间

略

## 参与人员

上线完成后请安排具体业务人员验证和测试

|  |  |  |  |
| --- | --- | --- | --- |
| **实施内容** | **执行人** | **公司** | **电话** |
| 上线 |  | 青云 |  |
| 上线 |  | 青云 |  |
| 上线 |  | 青云 |  |
| 远程支持 |  | 青云 |  |
|  |  |  |  |

# 升级内容 - 待补充

环境信息

|  |  |  |  |
| --- | --- | --- | --- |
| **集群类型** | **集群名称** | **集群别名** | **harbor仓库域名** |
| 管理集群 |  |  |  |
| 管理集群 |  |  |  |
| Member集群 |  |  |  |
|  |  |  |
|  |  |  |
|  |  |  |
|  |  |  |
|  |  |  |
|  |  |  |
|  |  |  |
|  |  |  |
|  |  |  |
|  |  |  |

# 影响范围

## kubernetes 各个组件支持表参考

etcd

当前etcd版本为 3.4.13 需要升级

|  |  |  |
| --- | --- | --- |
| kubernetes 版本 | etcd 最小要求版本 | etcd推荐版本 |
| 1.23.0~1.23.17 | 3.2.18 | 3.5.6 |
| 1.24.0~1.24.17 | 3.2.18 | 3.5.6 |
| 1.25.0~1.25.16 | 3.2.18 | 3.5.9 |
| 1.26.0~1.26.15 | 3.2.18 | 3.5.10 |
| 1.27.0~1.27.16 | 3.2.18 | 3.5.12 |
| 1.28.0~1.28.15 | 3.4.13-4 | 3.5.15 |
| 1.29.0~1.29.15 | 3.4.13-4 | 3.5.15 |
| 1.30.0~1.30.14 | 3.4.13-4 | 3.5.15 |
| 1.31.0~1.31.13 | 3.5.11-0 | 3.5.24 |
| 1.31.14 | 3.5.24-0 | 3.5.24 |
| 1.32.0~1.32.9 | 3.5.11-0 | 3.5.24 |
| 1.32.10~1.32.11 | 3.5.24-0 | 3.5.24 |
| v1.33.0~v1.33.5 | 3.5.11-0 | 3.5.24 |
| v1.33.6~v1.33.7 | 3.5.24-0 | 3.5.24 |

容器运行时 - containerd

当前containerd版本为1.7.22 无需升级

|  |  |
| --- | --- |
| kubernetes 版本 | 推荐containerd版本 |
| 1.23 | v1.6.20 |
| 1.24 | v1.6.20 |
| 1.25 | v1.6.20 |
| 1.26 | v1.6.21 |
| 1.27 | v1.7.13 |
| 1.28 | v1.7.13 |
| 1.29 | v1.7.13 |
| 1.30 | v1.7.13 |
| 1.31 | v1.7.13 |
| 1.32 | v1.7.13 |
| 1.33 | v1.7.13 |

容器网络插件 - 使用的是云衫网络插件，无需升级

相关容器镜像版本推荐

coredns必须升级，不然会出现配置不兼容的问题

|  |  |  |  |
| --- | --- | --- | --- |
| kubernetes | pause | coredns | nodelocaldns |
| 1.23 | 3.6 | v1.8.6 | 1.21.1 |
| 1.24 | 3.7 | v1.8.6 | 1.21.1 |
| 1.25 | 3.8 | v1.9.3 | 1.22.8 |
| 1.26 | 3.9 | v1.9.3 | 1.22.13 |
| 1.27 | 3.9 | v1.10.1 | 1.22.20 |
| 1.28 | 3.9 | v1.10.1 | 1.22.23 |
| 1.29 | 3.9 | v1.11.1 | 1.22.23 |
| 1.30 | 3.9 | v1.11.3 | 1.22.28 |
| 1.31 | 3.10 | v1.11.3 | 1.23.1 |
| 1.32 | 3.10 | v1.11.3 | 1.23.1 |
| 1.33 | 3.10 | v1.12.0 | 1.25.0 |

Kubernetes 升级采用滚动升级，如果升级过程中遇到kubelet启动异常，可能会对业务造成驱逐操作，请业务检查是否符合驱逐到其他节点的条件，如业务连续性有要求，建议扩展副本到3副本以上。

# 前期准备

1. 提前发送变更通知
2. 上传服务上线包并相关镜像

# 操作步骤

## 准备工作

1、准备上线文件并上传

[ ! -d /kube/app-package/upgrade-k8s ]&& echo "/kube/app-package/upgrade-k8s does not exist and will create"&&mkdir -p /kube/app-package/upgrade-k8s/update-backup

tar xf k8s-v1.20-v1.23-amd64-20240618.tar.gz -C /kube/app-package/upgrade-k8s/

cd /kube/app-package/upgrade-k8s/update-backup/k8s-v1.20-v1.23-amd64-20240618

./kk artifact images push -f config-sample.yaml -a kubekey-artifact.tar.gz

## 升级步骤

### 备份

cd /kube/app-package/upgrade-k8s/upgrade-backup

kubectl get cm -n kube-system kubelet-config-1.23 -oyaml >>kubelet-config-1.23.yaml

kubectl get cm -n kube-system kubeadm-config -oyaml>> kubeadm-config.yaml

kubectl get cm -n kube-system coredns -oyaml >>coredns-cm.yaml

kubectl get deploy -n kube-system coredns -oyaml >>coredns-deploy.yaml

kubectl get clusterrole system:coredns -oyaml>> coredns-clusterrole.yaml

kubectl get clusterrolebinding system:coredns -oyaml >>coredns-clusterrolebinding.yaml

cp /usr/local/bin/kube\* .

cp /etc/kubernetes/manifests/\* .

kubectl get pod -A -owide | grep -v Running >>notRuningPod.txt

#备份etcd

bash /usr/local/bin/kube-scripts/etcd-backup.sh

cp -r /var/backups/kube\_etcd/$(ls -rt /var/backups/kube\_etcd/ | sort -rn | head -n 1) ./

### 同步镜像

kk artifact images push -a kubekey-artifact.tar.gz

### 删除 feature-gate 参数

参考：Feature Gates | Kubernetes

RotateKubeletServerCertificate：从1.12开始，默认为true

TTLAfterFinished：1.33中，没有这个feature-gate

ExpandCSIVolumes: 1.33中，没有这个feature-gate

1. 修改 configmap - kubeadm-config

kubectl edit cm -n kube-system kubeadm-config

删除 data.ClusterConfiguration 中的字段：

apiServer.extraArgs.feature-gates

controllerManager.extraArgs.feature-gates

scheduler.extraArgs.feature-gates

1. 修改 configmap - kubelet-config-1.23

kubectl edit cm -n kube-system kubelet-config-1.23

·删除 data.kubelet 中的字段：featureGates

·添加部分配置

containerLogMaxSize: "100Mi"

containerLogMaxFiles: 5

imageGCHighThresholdPercent: 70

imageGCLowThresholdPercent: 50

tlsCipherSuites: [TLS\_ECDHE\_RSA\_WITH\_AES\_128\_GCM\_SHA256,TLS\_ECDHE\_RSA\_WITH\_AES\_256\_GCM\_SHA384,TLS\_ECDHE\_RSA\_WITH\_CHACHA20\_POLY1305\_SHA256,TLS\_ECDHE\_RSA\_WITH\_AES\_128\_CBC\_SHA,TLS\_ECDHE\_RSA\_WITH\_AES\_256\_CBC\_SHA]

![descript](data:image/png;base64...)

1. 删除第一个master节点上的/etc/kubernetes/kubeadm-config.yaml 中的字段：

ClusterConfiguration:

apiServer.extraArgs.feature-gates

controllerManager.extraArgs.feature-gates

scheduler.extraArgs.feature-gates

KubeLetConfiguration:

featureGates:

1. 修改节点上的 /etc/kubernetes/manifests/kube-apiserver.yaml

删除spec.containers[0].command[--feature-gates=xxx]

1. 修改节点上的 /etc/kubernetes/manifests/kube-controller-manager.yaml

删除spec.containers[0].command[--feature-gates=xxx]

1. 修改节点上的 /etc/kubernetes/manifests/kube-scheduler.yaml

删除spec.containers[0].command[--feature-gates=xxx]

1. 删除节点 /var/lib/kubelet/config.yaml 中的 featureGates

### 升级etcd

滚动升级到3.5.24。一个个节点依次升级

1. 备份（6.2.1已做）
2. 停止 etcd 服务：systemctl stop etcd
3. 复制新的etcd二进制文件到/usr/local/bin 目录
4. 启动 etcd 服务：systemctl start etcd
5. 等待所有etcd节点变成healthy: etcdctl endpoint health

### 升级k8s

1. 修改config-sample.yaml 中的字段

设置 spec.network.plugin: none, spec.kubernetes.nodelocaldns: false

1. 升级到v1.33.4版本

每次隔一个版本升级。等所有节点都ready之后再进行下一次升级。

./kk upgrade -f config-sample.yaml --with-kubernetes 1.25.16

./kk upgrade -f config-sample.yaml --with-kubernetes 1.27.16

./kk upgrade -f config-sample.yaml --with-kubernetes 1.29.15

./kk upgrade -f config-sample.yaml --with-kubernetes 1.31.12

./kk upgrade -f config-sample.yaml --with-kubernetes 1.33.4

1. 升级pause到3.10 - 可选

替换 containerd中的配置：/etc/containerd/config.toml 中的

sandbox\_image = "dockerhub.kubekey.local/kse/kubesphere/pause:3.6" 为

sandbox\_image = "dockerhub.kubekey.local/kse/kubesphere/pause:3.10"

重启 containerd 服务 （会中断业务）

1. 升级crictl 到 1.33.0 - 可选

替换 /usr/bin/crictl 为 1.33.0 版本

## 集群检查

1、检查节点

#检查节点版本

kubectl get node | grep -v 1.33.4

#检查节点状态

kubectl get node | grep -v Ready

2、检查pod状态

kubectl get pod -A -owide | grep -v Running

kubectl get pod -n kube-system -owide | egrep "coredns|kube-\*"

## 容器云平台更新kubeconfig

把当前集群的kubeconfig在界面上同步更新到容器平台，以保证纳管所用的kubeconfig和集群的kubeconfig证书有效期一致。

## 上线验证

### 验证平台基础功能

查看pod是否运行正常

kubectl get pod -n kubesphere-system |grep -v Running

kubectl get pod -n kubesphere-logging-system |grep -v Running

kubectl get pod -n kubesphere-monitoring-system |grep -v Running

使用平台管理员账号、普通账号分别登录容器云平台，测试创建工作负载、svc等基础功能。

### 验证业务空间网关

使用平台管理员和业务空间管理员登陆平台，检查配额以及业务空间网关启用/禁用等功能。

### 验证各组件

# 1、混沌工程

kubectl get pod -n kubesphere-chaosmesh-system

# 2、多集群巡检（host集群）

kubectl get pod -n kubeeye-system

# 3、增强型工作负载

kubectl get pod -n kruise-system

# 4、对象存储加速

kubectl get pod -n fluid-system

### 通过巡检功能对当前集群巡检

平台管理--集群巡检--巡检任务--创建及时巡检---按照步骤创建---选择相应的升级集群--开始巡检

巡检完毕之后查看巡检报告，确认巡检报告无异常

# 回退方案

## 回退条件

如果在K8s升级过程中遇到严重问题到计划时间（凌晨2点）仍未解决，则回退。

## 回退计划

回退预计耗时2小时

## 回退步骤

Kubernetes版本回退涉及如下组件:

kube-apiserver

kube-controller-manager

kube-scheduler

kube-proxy

kubelet

CoreDNS

kubeadm

Kubernetes升级调用kubeadm执行，当kubeadm升级失败，会自动执行回滚，升级完成后，可通过替换组件镜像及二进制方式将kubernetes版本回退到低版本，注意回退版本不超过**3**个大版本。

# 回滚 master 组件版本 （所有master节点执行）

# 进入 /etc/kubernetes/manifests 目录中，将如下三个组件的镜像替换为待回滚版本

# kube-apiserver.yaml

# kube-controler-manager.yaml

# kube-scheduler.yaml

# 回滚 kubelet 版本 （所有 worker 节点执行）

# 停止 kubelet

systemctl stop kubelet

# 将 /usr/local/bin/kubelet 替换为待回滚版本, 并重新启动

# 回滚 kube-proxy 版本，kube-proxy以DaemonSet方式运行，可使用kubectl修改kube-proxy daemonset的镜像版本为待会滚版本。

# kubectl edit ds -n kube-system kube-proxy

回滚coredns

# kubectl -n kube-system set image deployment coredns coredns=repos.cloud.cmft/coredns/coredns:1.6.9

# 变更结果通知

## 邮件通知

上线完成并且验证完成后需要邮件告知所有运维人员上线的情况，并告知相关人员在第二天运行过程中需要注意事项。对于特别注意事项，向第二天保障人员特别交接。

## 微信通知

上线完成并且验证完成后需要微信告知所有运维人员上线的情况，并告知相关人员在第二天运行过程中需要注意事项。
