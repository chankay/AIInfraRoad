## 引言
在高性能计算（HPC）和 AI 训练任务中，**RoCE v2（RDMA over Converged Ethernet v2）**因其低延迟、高吞吐的特性，成为 RDMA 解决方案的主流选择。在 Kubernetes（K8s）集群中，为了支持 RoCE v2，我们需要让 Pod 直接访问 RDMA 网卡，同时确保多个 Pod 能够自动获取 IP 地址并进行通信。

本文介绍一种 **基于 Multus 实现多网络支持，使用 host-device 直通 RDMA 网卡，并通过 Whereabouts 进行 IP 分配** 的完整解决方案。

## 概述
该方案的目标是在 K8s 中支持 **RoCE v2 通信**，具体包括以下关键点：

1. **多网络支持**<font style="color:black;">：使用 </font>**Multus CNI**<font style="color:black;"> 允许 Pod 同时连接多个网络（例如业务网和 RDMA 专用网）。</font>
2. **RDMA 设备直通**<font style="color:black;">：使用 </font>**host-device**<font style="color:black;"> 方式，让 Pod 直接访问宿主机 RDMA 网卡，确保 RDMA 功能正常工作。</font>
3. **自动 IP 分配**<font style="color:black;">：采用 </font>**Whereabouts CNI**<font style="color:black;"> 自动管理 RDMA 网络 IP，避免手动分配的复杂性。</font>

## 架构图


![](https://cdn.nlark.com/yuque/0/2025/png/725898/1743069960909-2126be35-e999-4bee-8900-bd3c02bee894.png)



## 关键技术解析
### Multus：支持 K8s 多网络
**默认 K8s 仅支持一个网络**，而 RDMA 通信通常需要一个专用子网。Multus 允许 Pod **拥有多个网络接口（Multi-homing）**，从而在一个 Pod 内同时支持业务流量和 RDMA 通信。

### host-device：RDMA 网卡直通
为了让 Pod 访问宿主机的 **RDMA 网卡**，我们使用 `host-device` CNI 插件，它允许 Pod 直接绑定到特定物理设备。

### Whereabouts：自动 IP 分配
**Whereabouts** 是一个 IPAM（IP Address Management）插件，支持 **无中心化的动态 IP 分配**，特别适用于 RoCE 网络，避免了传统静态 IP 配置的复杂性。

## 部署过程
### memlock配置优化
RoCE 使用 RDMA（远程直接内存访问），其核心特性是**零拷贝、低延迟、用户态直接访问网卡（NIC）**。RDMA 依赖**注册内存（Pinned Memory）**，这部分内存必须始终留在物理内存中，不能被操作系统 swap 到磁盘，否则：

+ RDMA 设备（HCA/NIC）无法正确访问数据，导致传输失败。
+ 访问 swap 会引入**高延迟**，影响 RDMA 的实时性。
+ RDMA 操作依赖的 **Memory Region (MR)** 可能失效，导致应用崩溃。

因此，Linux 需要手动配置 `memlock`，允许进程**锁定足够的物理内存**。

### `memlock` 配置对 RoCE 的影响
| `memlock`配置 | 影响 |
| --- | --- |
| 低（默认 64 KB） | RDMA 进程无法锁定足够的内存，可能导致 RDMA 操作失败或延迟增加 |
| 适中（几百 MB） | 可以支持小规模 RDMA 操作，但高吞吐时可能不足 |
| 高（unlimited 或几 GB） | 允许 RDMA 进程锁定足够的内存，确保 RoCE 低延迟和高带宽 |


对于高吞吐 RDMA 应用（如 RoCEv2 传输大数据时），建议：

+ `memlock` 设置为 **unlimited** 或 **8GB 以上**。
+ 确保 `**ulimit -l**`** 显示 unlimited**。

#### 宿主机配置
<font style="color:black;">主机的 </font><font style="color:black;">ulimit -l</font><font style="color:black;"> 需要足够高，否则容器设置会被主机限制覆盖。检查主机：</font>

```bash
ulimit -l
```

<font style="color:black;">如果过低，修改 </font><font style="color:black;">/etc/security/limits.conf</font><font style="color:black;">：</font>

```plain
* soft memlock unlimited
* hard memlock unlimited
```

#### docker配置
<font style="color:black;">RoCE 需要锁定内存，而 Docker 默认的 </font><font style="color:black;">memlock</font><font style="color:black;"> 限制太低，添加 </font><font style="color:black;">"memlock": {"soft": -1, "hard": -1}</font><font style="color:black;"> 解除了限制。</font>修改 `/etc/docker/daemon.json` 文件，添加以下内容：

```plain
{
  "default-ulimits": {
    "memlock": {
      "name": "memlock",
      "hard": -1,
      "soft": -1
    }
  }
}
```

然后重新启动 Docker 服务：

```plain
sudo systemctl restart docker
```

### 部署 RDMA 设备插件
安装 RDMA 设备插件：

```plain
kubectl apply -f https://raw.githubusercontent.com/Mellanox/k8s-rdma-shared-dev-plugin/main/deployments/rdma_shared_device_plugin.yaml
```

修改configmap

```plain
kubectl edit cm rdma-devices -n kube-system
```



修改主要内容如下：(ifNames部分配置8张RoCE网卡名称)

```plain


apiVersion: v1
data:
  config.json: |
    {
            "periodicUpdateInterval": 300,
            "configList": [
            {
             "resourceName": "hca_gpu",
             "rdmaHcaMax": 8,
             "selectors": {
               "ifNames": ["ens12np0", "ens11np0", "ens13np0","ens14np0","ens16np0","ens15np0","ens17np0","ens18np0"]
              }
            }
           ]
    }
kind: ConfigMap

```



检查 RDMA 设备：

```plain
kubectl get no -o json | jq -r '[.items[] | {name:.metadata.name, allocable:.status.allocatable}]'
```

确保 **RDMA 设备数量为配置数量8**：

```plain
rdma/hca_gpu: 8
```



### 配置 Multus CNI 绑定 RoCE 网卡
### **(1) 安装 Multus CNI**
```plain
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml

ps: 网络情况不容许的环境下，先下载multus-daemonset.yml，修改其中image部分
```



### (2) 安装whereabout(用于自动分配IP)
```plain
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/whereabouts/master/doc/crds/whereabouts.cni.cncf.io_ippools.yaml

kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/whereabouts/master/doc/crds/whereabouts.cni.cncf.io_overlappingrangeipreservations.yaml

kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/whereabouts/master/doc/crds/daemonset-install.yaml
```

确定安装成功：

```plain
root@aitest1:~/rdma# kubectl get crds | grep whereabouts
ippools.whereabouts.cni.cncf.io                          2025-03-27T09:04:21Z
overlappingrangeipreservations.whereabouts.cni.cncf.io   2025-03-27T09:08:30Z

root@aitest1:~/rdma# kubectl get pods -A | grep whereabouts
kube-system   whereabouts-cbbgn                          1/1     Running   0          41m
kube-system   whereabouts-mwhvk                          1/1     Running   0          41m
kube-system   whereabouts-wp4f5                          1/1     Running   0          41m
```



### **(3) NAD配置：定义多个 RoCE 网络**
编写 `roce-network.json`：

这里type使用host-device，表示RDMA物理网卡直通，该方式在pod启动后，物理网卡会被pod独占，宿主机中将看不到该网卡。

```plain
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: roce-net-1
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "roce-net1",
    "type": "host-device",
    "device": "ens12np0",
    "ipam": {
      "type": "whereabouts",
      "range": "172.31.1.0/24",
      "gateway": "172.31.1.1"
    }
  }'
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: roce-net-2
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "roce-net2",
    "type": "host-device",
    "device": "ens11np0",
    "ipam": {
      "type": "whereabouts",
      "range": "172.31.2.0/24",
      "gateway": "172.31.2.1"
    }
  }'
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: roce-net-3
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "roce-net3",
    "type": "host-device",
    "device": "ens13np0",
    "ipam": {
      "type": "whereabouts",
      "range": "172.31.3.0/24",
      "gateway": "172.31.3.1"
    }
  }'
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: roce-net-4
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "roce-net4",
    "type": "host-device",
    "device": "ens14np0",
    "ipam": {
      "type": "whereabouts",
      "range": "172.31.4.0/24",
      "gateway": "172.31.4.1"
    }
  }'
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: roce-net-5
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "roce-net5",
    "type": "host-device",
    "device": "ens16np0",
    "ipam": {
      "type": "whereabouts",
      "range": "172.31.5.0/24",
      "gateway": "172.31.5.1"
    }
  }'
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: roce-net-6
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "roce-net6",
    "type": "host-device",
    "device": "ens15np0",
    "ipam": {
      "type": "whereabouts",
      "range": "172.31.6.0/24",
      "gateway": "172.31.6.1"
    }
  }'
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: roce-net-7
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "roce-net7",
    "type": "host-device",
    "device": "ens17np0",
    "ipam": {
      "type": "whereabouts",
      "range": "172.31.7.0/24",
      "gateway": "172.31.7.1"
    }
  }'
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: roce-net-8
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "roce-net8",
    "type": "host-device",
    "device": "ens18np0",
    "ipam": {
      "type": "whereabouts",
      "range": "172.31.8.0/24",
      "gateway": "172.31.8.1"
    }
  }'



```

这里 `device: "ens18np0"` 代表 **RoCE 网卡**，你需要为 **每张网卡** 创建不同的 `network-attachment-definition`，比如 ens11np0、ens12np0 等。

应用该配置：

```plain
kubectl apply -f roce-network.json
```

查看：

```plain
root@aitest1:~/rdma# kubectl get network-attachment-definitions
NAME         AGE
roce-net-1   43m
roce-net-2   43m
roce-net-3   43m
roce-net-4   43m
roce-net-5   43m
roce-net-6   43m
roce-net-7   43m
roce-net-8   43m
```



## 启动pod测试
### pod1配置test.yml
```plain
apiVersion: v1
kind: Pod
metadata:
  name: gpu-rdma-pod-1
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [
        { "name": "roce-net-1" },
        { "name": "roce-net-2" },
        { "name": "roce-net-3" },
        { "name": "roce-net-4" },
        { "name": "roce-net-5" },
        { "name": "roce-net-6" },
        { "name": "roce-net-7" },
        { "name": "roce-net-8" }
      ]
spec:
  containers:
  - name: rdma-gpu-container
    image: ghcr.io/spidernet-io/rdma-tools:v12.5.7
    command: ["/bin/bash", "-c", "sleep 3600"]
    resources:
      limits:
        nvidia.com/gpu: 8
        rdma/hca_gpu: 8
        memory: "1024Gi"
        cpu: "128"
    securityContext:
      capabilities:
        add: ["IPC_LOCK"]


```

### pod2配置test2.yml
```plain
apiVersion: v1
kind: Pod
metadata:
  name: gpu-rdma-pod-2
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [
        { "name": "roce-net-1" },
        { "name": "roce-net-2" },
        { "name": "roce-net-3" },
        { "name": "roce-net-4" },
        { "name": "roce-net-5" },
        { "name": "roce-net-6" },
        { "name": "roce-net-7" },
        { "name": "roce-net-8" }
      ]
spec:
  containers:
  - name: rdma-gpu-container
    image: ghcr.io/spidernet-io/rdma-tools:v12.5.7
    command: ["/bin/bash", "-c", "sleep 3600"]
    resources:
      limits:
        nvidia.com/gpu: 8
        rdma/hca_gpu: 8
        memory: "1024Gi"
        cpu: "128"
    securityContext:
      capabilities:
        add: ["IPC_LOCK"]


```

启动pod

```plain
root@aitest1:~/rdma# kubectl apply -f test.yml 
pod/gpu-rdma-pod-1 created
root@aitest1:~/rdma# kubectl apply -f test2.yml 
pod/gpu-rdma-pod-2 created
```

进入pod1，查看

```plain
root@gpu-rdma-pod-1:/# ibdev2netdev 
mlx5_0 port 1 ==> net1 (Up)
mlx5_1 port 1 ==> net2 (Up)
mlx5_2 port 1 ==> net3 (Up)
mlx5_3 port 1 ==> net4 (Up)
mlx5_4 port 1 ==> net5 (Up)
mlx5_5 port 1 ==> net6 (Up)
mlx5_6 port 1 ==> net7 (Up)
mlx5_7 port 1 ==> net8 (Up)

root@gpu-rdma-pod-1:/# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
3: eth0@if70: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1480 qdisc noqueue state UP group default qlen 1000
    link/ether 86:07:cf:63:37:86 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.233.92.20/32 scope global eth0
       valid_lft forever preferred_lft forever
6: net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
    link/ether 94:6d:ae:c6:93:9a brd ff:ff:ff:ff:ff:ff
    altname enp4s0np0
    inet 172.31.1.1/24 brd 172.31.1.255 scope global net1
       valid_lft forever preferred_lft forever
7: net2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
    link/ether 94:6d:ae:c6:78:9e brd ff:ff:ff:ff:ff:ff
    altname enp35s0np0
    inet 172.31.2.1/24 brd 172.31.2.255 scope global net2
       valid_lft forever preferred_lft forever
8: net3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
    link/ether 94:6d:ae:c7:0d:3e brd ff:ff:ff:ff:ff:ff
    altname enp67s0np0
    inet 172.31.3.1/24 brd 172.31.3.255 scope global net3
       valid_lft forever preferred_lft forever
9: net4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
    link/ether 94:6d:ae:c6:84:ba brd ff:ff:ff:ff:ff:ff
    altname enp100s0np0
    inet 172.31.4.1/24 brd 172.31.4.255 scope global net4
       valid_lft forever preferred_lft forever
10: net5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
    link/ether 94:6d:ae:c6:43:9a brd ff:ff:ff:ff:ff:ff
    altname enp132s0np0
    inet 172.31.5.1/24 brd 172.31.5.255 scope global net5
       valid_lft forever preferred_lft forever
11: net6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
    link/ether 94:6d:ae:c6:37:56 brd ff:ff:ff:ff:ff:ff
    altname enp164s0np0
    inet 172.31.6.1/24 brd 172.31.6.255 scope global net6
       valid_lft forever preferred_lft forever
12: net7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
    link/ether 94:6d:ae:c6:93:92 brd ff:ff:ff:ff:ff:ff
    altname enp196s0np0
    inet 172.31.7.1/24 brd 172.31.7.255 scope global net7
       valid_lft forever preferred_lft forever
15: net8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
    link/ether 94:6d:ae:c7:0d:5e brd ff:ff:ff:ff:ff:ff
    altname enp228s0np0
    inet 172.31.8.1/24 brd 172.31.8.255 scope global net8
       valid_lft forever preferred_lft forever
```

进入pod2,查看

```plain
root@gpu-rdma-pod-2:/# ibdev2netdev 
mlx5_0 port 1 ==> net1 (Up)
mlx5_1 port 1 ==> net2 (Up)
mlx5_2 port 1 ==> net3 (Up)
mlx5_3 port 1 ==> net4 (Up)
mlx5_4 port 1 ==> net5 (Up)
mlx5_5 port 1 ==> net6 (Up)
mlx5_6 port 1 ==> net7 (Up)
mlx5_7 port 1 ==> net8 (Up)

root@gpu-rdma-pod-2:/# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
3: eth0@if59: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1480 qdisc noqueue state UP group default qlen 1000
    link/ether 16:69:04:c7:f0:d7 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.233.96.28/32 scope global eth0
       valid_lft forever preferred_lft forever
6: net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
    link/ether 94:6d:ae:c6:78:96 brd ff:ff:ff:ff:ff:ff
    altname enp4s0np0
    inet 172.31.1.2/24 brd 172.31.1.255 scope global net1
       valid_lft forever preferred_lft forever
7: net2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
    link/ether 94:6d:ae:c6:a8:02 brd ff:ff:ff:ff:ff:ff
    altname enp35s0np0
    inet 172.31.2.2/24 brd 172.31.2.255 scope global net2
       valid_lft forever preferred_lft forever
8: net3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
    link/ether 94:6d:ae:c6:54:aa brd ff:ff:ff:ff:ff:ff
    altname enp67s0np0
    inet 172.31.3.2/24 brd 172.31.3.255 scope global net3
       valid_lft forever preferred_lft forever
9: net4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
    link/ether 94:6d:ae:c6:36:92 brd ff:ff:ff:ff:ff:ff
    altname enp100s0np0
    inet 172.31.4.2/24 brd 172.31.4.255 scope global net4
       valid_lft forever preferred_lft forever
10: net5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
    link/ether 94:6d:ae:c6:36:96 brd ff:ff:ff:ff:ff:ff
    altname enp132s0np0
    inet 172.31.5.2/24 brd 172.31.5.255 scope global net5
       valid_lft forever preferred_lft forever
11: net6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
    link/ether 94:6d:ae:c6:77:1a brd ff:ff:ff:ff:ff:ff
    altname enp164s0np0
    inet 172.31.6.2/24 brd 172.31.6.255 scope global net6
       valid_lft forever preferred_lft forever
12: net7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
    link/ether 94:6d:ae:c6:42:86 brd ff:ff:ff:ff:ff:ff
    altname enp196s0np0
    inet 172.31.7.2/24 brd 172.31.7.255 scope global net7
       valid_lft forever preferred_lft forever
15: net8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP group default qlen 1000
    link/ether 94:6d:ae:c6:38:5a brd ff:ff:ff:ff:ff:ff
    altname enp228s0np0
    inet 172.31.8.2/24 brd 172.31.8.255 scope global net8
       valid_lft forever preferred_lft forever
```

### IB时延测试
pod1 执行

```plain
root@gpu-rdma-pod-1:/# ib_read_lat 

************************************
* Waiting for client to connect... *
************************************



```



pod2 执行

```plain
root@gpu-rdma-pod-2:/# ib_read_lat 172.31.6.1
---------------------------------------------------------------------------------------
                    RDMA_Read Latency Test
 Dual-port       : OFF          Device         : mlx5_0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
 ibv_wr* API     : ON
 TX depth        : 1
 Mtu             : 4096[B]
 Link type       : Ethernet
 GID index       : 3
 Outstand reads  : 16
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x0236 PSN 0x8815dd OUT 0x10 RKey 0x203d00 VAddr 0x00557d25d29000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:172:31:01:02
 remote address: LID 0000 QPN 0x0234 PSN 0xa23615 OUT 0x10 RKey 0x203d00 VAddr 0x0055b6cc722000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:172:31:01:01
---------------------------------------------------------------------------------------
 #bytes #iterations    t_min[usec]    t_max[usec]  t_typical[usec]    t_avg[usec]    t_stdev[usec]   99% percentile[usec]   99.9% percentile[usec] 
Conflicting CPU frequency values detected: 1500.000000 != 1663.676000. CPU Frequency is not max.
Conflicting CPU frequency values detected: 1500.000000 != 3100.000000. CPU Frequency is not max.
 2       1000          4.70           13.20        4.83                4.82             0.00            4.91                    13.20  
---------------------------------------------------------------------------------------

```



pod1 有结果

```plain
---------------------------------------------------------------------------------------
                    RDMA_Read Latency Test
 Dual-port       : OFF          Device         : mlx5_0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
 ibv_wr* API     : ON
 Mtu             : 4096[B]
 Link type       : Ethernet
 GID index       : 3
 Outstand reads  : 16
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x0234 PSN 0xa23615 OUT 0x10 RKey 0x203d00 VAddr 0x0055b6cc722000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:172:31:01:01
 remote address: LID 0000 QPN 0x0236 PSN 0x8815dd OUT 0x10 RKey 0x203d00 VAddr 0x00557d25d29000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:172:31:01:02
---------------------------------------------------------------------------------------

```

### IB带宽测试
```plain
root@gpu-rdma-pod-1:/# ib_write_bw -a -d mlx5_0

************************************
* Waiting for client to connect... *
************************************



---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF          Device         : mlx5_0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
 ibv_wr* API     : ON
 CQ Moderation   : 100
 Mtu             : 4096[B]
 Link type       : Ethernet
 GID index       : 3
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x023b PSN 0xcf95c6 RKey 0x203d00 VAddr 0x007fbd7d593000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:172:31:01:01
 remote address: LID 0000 QPN 0x0239 PSN 0x1137b2 RKey 0x203d00 VAddr 0x007f5f29b83000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:172:31:01:02
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 8388608    5000             196.07             196.07             0.002922
---------------------------------------------------------------------------------------

```



```plain
root@gpu-rdma-pod-2:/# ib_write_bw  -a -F 172.31.1.1 -d mlx5_0 --report_gbits
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF          Device         : mlx5_0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
 ibv_wr* API     : ON
 TX depth        : 128
 CQ Moderation   : 100
 Mtu             : 4096[B]
 Link type       : Ethernet
 GID index       : 3
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x0239 PSN 0x1137b2 RKey 0x203d00 VAddr 0x007f5f29b83000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:172:31:01:02
 remote address: LID 0000 QPN 0x023b PSN 0xcf95c6 RKey 0x203d00 VAddr 0x007fbd7d593000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:172:31:01:01
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
 2          5000           0.090062            0.088598            5.537347
 4          5000             0.18               0.18               5.557853
 8          5000             0.36               0.36               5.589335
 16         5000             0.72               0.71               5.513707
 32         5000             1.56               1.49               5.815027
 64         5000             2.88               2.87               5.608749
 128        5000             5.73               5.71               5.577967
 256        5000             11.40              11.33              5.530008
 512        5000             22.64              22.36              5.458244
 1024       5000             44.49              44.23              5.398909
 2048       5000             85.83              84.74              5.172380
 4096       5000             148.10             147.39             4.498039
 8192       5000             194.71             194.57             2.968878
 16384      5000             195.46             195.44             1.491082
 32768      5000             195.80             195.78             0.746830
 65536      5000             195.91             195.91             0.373667
 131072     5000             196.01             196.00             0.186922
 262144     5000             196.04             196.03             0.093476
 524288     5000             196.06             196.06             0.046744
 1048576    5000             196.07             196.07             0.023373
 2097152    5000             196.06             196.06             0.011686
 4194304    5000             196.07             196.07             0.005843
 8388608    5000             196.07             196.07             0.002922
---------------------------------------------------------------------------------------

```

### 其他 IB 测试工具
+ `ib_read_lat`：测试 **RDMA Read** 时延。
+ `ib_write_lat`：测试 **RDMA Write** 时延。
+ `ib_read_bw`：测试 **RDMA Read** 带宽。
+ `ib_write_bw`：测试 **RDMA Write** 带宽。

