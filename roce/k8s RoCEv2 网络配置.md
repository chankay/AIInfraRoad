

整体思路：

--> 部署rdma device plugin 

--> 安装Multus CNI  

--> 安装Whereabouts 

--> 定义8个RoCE网络  

--> 启动pod绑定gpu和网络

### 
![](https://cdn.nlark.com/yuque/0/2025/png/725898/1743069960909-2126be35-e999-4bee-8900-bd3c02bee894.png)





## 1. 部署 RDMA 设备插件
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



## **2. 配置 Multus CNI 绑定 RoCE 网卡**
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



### **(3) 定义多个 RoCE 网络**
编写 `roce-network.json`：

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



## 3.启动pod测试
### (1）pod1配置test.yml
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

```

### (2)   pod2配置test2.yml
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

### (3) ib测试
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



如上，K8S上的RoCE 网络配置成功

