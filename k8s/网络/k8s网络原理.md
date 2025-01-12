# Kubernetes 网络模型

## K8S网络设计原则

k8s对于任何网络实现均有以下要求:

- 所有pods可以不通过NAT与所有pods通信
- 所有nodes可以不通过NAT与所有pods通信
- pod看到的自身的ip = 其他pod看到的他的ip

鉴于这些限制，我们需要解决四个不同的网络问题：

1. 容器到容器网络
2. Pod 到 Pod 网络
3. Pod 到服务网络
4. 互联网到服务网络

## 容器到容器网络

### Linux中的网络命名空间

理想视图下，虚拟机中的网络通信可以看作其直接与ethernet网卡通信

![image-20241231180240141](https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image/image-20241231180240141.png)、

在Linux中，默认情况下，每一个运行的进程都在一个network namespace通信，**此namespace提供了一个逻辑网络栈，拥有其自己的路由表，防火墙规则（netfilter）和网络设备**。本质上，network namespace为此命名空间内所有进程提供了一个全新的网络栈。

```bash
# 创建一个网络命名空间
root@mounto:~# ip netns add ns1
```

创建namespace后，会在/var/run/netns目录下创建一个挂载点，以保证此命名空间的持久性, 即使此命名空间下没有进程也会存在。

```bash
# 查看已经创建的命名空间
root@mounto:~# ls /var/run/netns
cni-04bf6091-5ae7-3a78-4d7b-e3280f80e6ef
...
ns1
root@mounto:~# ip netns ls
cni-28415f0f-d286-3331-5f63-139d4996f746 (id: 0)
...
ns1
```

可以看到，各个网络命名空间有自己的name和id

下面的例子验证不同网络命名空间的独立性：

```bash
# 使用下面的命令在对应的命名空间下执行ip命令，验证不同命名空间下网络设备、路由等（网络栈）是独立的
root@mounto:~# ip netns exec ns1 ip a 
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
root@mounto:~# ip netns exec ns1 ip route

# 使用另外一个命名空间
root@mounto:~# ip netns exec cni-9eb8f642-bea4-ca7a-3b89-360425324972 ip a 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0@if793: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default 
    link/ether 0e:29:ab:ae:84:4b brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.42.0.68/24 brd 10.42.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::c29:abff:feae:844b/64 scope link 
       valid_lft forever preferred_lft forever
root@mounto:~# ip netns exec cni-9eb8f642-bea4-ca7a-3b89-360425324972 ip route
default via 10.42.0.1 dev eth0 
10.42.0.0/24 dev eth0 proto kernel scope link src 10.42.0.68 
10.42.0.0/16 via 10.42.0.1 dev eth0
```

默认情况下，Linux会将新创建的进程分配到根命名空间以提供其对外部网络的访问：

![image-20241231180315730](https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image/image-20241231180315730.png)

### Pod 网络模型

在k8s的网络模型下，**Pod为一组共享网络命名空间的container集合**，在同一个Pod中的container通过此Pod的network namespace**共享同一个IP和端口空间**，可以**直接通过回环网卡lo通信**。如下图所示，每个pod有自己的namespace，其中的多个container共享同一个namespace。此时每个namespace都相互独立且隔离。

根据上述的分析，其实每个pod通过网络命名空间进行隔离，因此每个pod都是一套独立的逻辑网络栈，有自己的路由表、iptables和网络设备。

![image-20241231180336049](https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image/image-20241231180336049.png)

## Pod 到 Pod 网络

k8s中，每个Pod中都有自己的真实IP并可以通过此IP与其他Pod通信。Pod的IP地址是由CNI网络插件分配的，例如使用k3s时，默认的网络插件是flannel，可以通过如下命令查看Pod的IP：

```bash
# 查看pod的ip
root@mounto:~# kubectl get pod -o wide -n mars
NAME                                       READY   STATUS    RESTARTS   AGE     IP           NODE     NOMINATED NODE   READINESS GATES
dacs-version-conversion-68848dc7bd-6fzbl   1/1     Running   0          7d21h   10.42.0.20   node1    <none>           <none>
mars-db-64ffbb4f66-pbwjb                   1/1     Running   0          7d21h   10.42.0.17   node1    <none>           <none>
mars-web-server-96bb748f5-z7xnh            1/1     Running   0          7d21h   10.42.0.29   node1    <none>           <none>
mysql-web-556b84575b-2vvrs                 1/1     Running   0          5d23h   10.42.1.74   23-113   <none>           <none>
ntp-client-2cf7k                           1/1     Running   0          7d20h   10.42.2.2    23-114   <none>           <none>
ntp-client-5tqbn                           1/1     Running   0          7d20h   10.42.1.2    23-113   <none>           <none>
ntp-client-h5dtk                           1/1     Running   0          7d21h   10.42.0.19   node1    <none>           <none>
ntp-server-6d8597b478-6pdv8                1/1     Running   0          7d21h   10.42.0.18   node1    <none>           <none>
rsyncd-5d9dc67759-gg2dh                    1/1     Running   0          112m    10.42.0.79   node1    <none>           <none>
```

当前的任务是了解 Kubernetes 如何使用真实 IP 实现 Pod 到 Pod 通信，无论 Pod 部署在同一物理节点上还是集群中的不同节点上。我们从考虑**驻留在同一台机器上**的 Pod 开始讨论，以避免通过内部网络跨节点通信的复杂性。

从 Pod 的角度来看，它存在于自己的以太网命名空间中，需要与同一节点上的其他网络命名空间进行通信。幸运的是，可以使用 Linux[虚拟以太网设备](http://man7.org/linux/man-pages/man4/veth.4.html)或由两个虚拟接口组成的*veth*对连接命名 空间，这些虚拟接口可以分布在多个命名空间中。要连接 Pod 命名空间，我们可以将 veth 对的一侧分配给根网络命名空间，将另一侧分配给 Pod 的网络命名空间。每个 veth 对都像一条跳线一样工作，连接两侧并允许流量在它们之间流动。可以为机器上尽可能多的 Pod 复制此设置。下图显示了将虚拟机上的每个 Pod 连接到根命名空间的 veth 对。

<img src="https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image/image-20250109171344346.png" alt="image-20250109171344346"  />

此时，我们已将 Pod 设置为各自拥有一个网络命名空间，以便它们相信自己拥有自己的以太网设备和 IP 地址，并且它们已连接到节点的根命名空间。现在，我们希望 Pod 通过根命名空间相互通信，为此我们使用*网桥*。

Linux 以太网桥（bridge）是一种虚拟的第 2 层网络设备，用于将两个或多个网络段连接起来，以透明的方式将两个网络连接在一起。桥的工作原理是，通过检查经过它的数据包的目的地并决定是否将数据包传递到连接到桥的其他网络段，从而维护源和目的地之间的转发表。桥接代码通过查看网络中每个以太网设备独有的 MAC 地址来决定是桥接数据还是丢弃数据。

![image-20250110115613456](https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image/image-20250110115613456.png)、

### Pod 到 Pod，同一节点

![pod-to-pod-same-node](https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image/pod-to-pod-same-node.gif)

首先进入到Pod的网络命名空间：

```bash
root@datacloak:~# ip netns list
...
cni-febeb7db-2ab6-d3ac-7aab-eb9fc9761793 (id: 1)
cni-e8ce09ac-7086-7b5d-546a-9771709acf6f (id: 0)
# 进入网络命名空间
root@datacloak:~# nsenter --net=/var/run/netns/cni-febeb7db-2ab6-d3ac-7aab-eb9fc9761793
# 查看路由表
root@datacloak:~# ip route 
default via 10.42.0.1 dev eth0 
10.42.0.0/24 dev eth0 proto kernel scope link src 10.42.0.3 
10.42.0.0/16 via 10.42.0.1 dev eth0
```

可以看到pod网络命名空间的路由表非常简单：

- 第一项是默认路由，via关键字标识了网关的ip地址 `10.42.0.1`，因此如果不符合下面规则的数据包都会发送至网关，例如访问集群外的ip地址，如 8.8.8.8
- 第二项是 10.42.0.0/24 的掩码匹配规则，如果在 10.42.0.0 的 24 位网段内，则匹配到第二项，直接通过eth0网卡发送数据包。这个匹配范围其实是表示目标地址在同一个node节点，使用flannel网络插件时，每个node会分配一个24位的网段地址，即 10.42.x.0，其中node1节点的网段地址为 10.10.42.0
- 第三项是 10.42.0.0/16 的掩码匹配规则，如果在 10.42.0.0 的 16 位网段内，则匹配到第三项，数据包经由网关发送出去。在flannel网络插件环境下，10.42.0.0 16位网段的地址，表示是集群内的PodIP，且没有匹配到第二项，表示目的Pod并不在同一个node上，因此需要先经过cni0网桥

总结一下同一node中Pod的通信过程：

1. Pod发出网络请求包
2. 封装IP数据报时，原地址是PodIP，目的地址是目标PodIP，端口同理
3. 根据Pod所属网络命名空间的路由表进行路由，发现匹配第二项，将直接经过eth0发送网络包
4. 在Pod所属网络命名空间中，IP数据报进一步通过网络协议栈封装为以太帧，需要填写目的mac地址，此时通过查询arp缓存或者通过eth0网络接口发送arp请求获取目的Pod的mac地址
   1. 如果是发送arp请求，则请求会通过eth0到达veth0，然后到达网桥cni0，进而会广播arp请求到与cni0连接的所有veth，即当前node所有的Pod
   2. 目标Pod回应arp请求，通过cni0网桥到达原Pod
   3. 原Pod获取到mac地址，进一步封装以太帧
5. 以太帧封装完成后，通过Pod所属命名空间eth0网络接口发送以太帧
6. 以太帧通过veth0到达宿主机root网络命名空间，进入宿主机的网络协议栈
7. 沿着宿主机网络协议栈进行解析，先到达二层解析，此时发现与内核bridge cni0网桥连接，因此以太帧传递到网桥
8. 网桥通过arp协议获取目标Pod的mac地址，并记录到自己的mac地址接口映射表中（二层交换机）
9. 网桥将以太帧发送到相应的veth接口
10. veth接口连接到目标Pod的eth0接口，目标Pod获取到以太帧，并沿着目标Pod的网络命名空间内的网络协议栈进行网络包的解析，最终传递到目标Pod的应用程序中

另外，之所以有走路由表转发网络包这个特性，是因为配置了网络包转发内核参数：

```bash
# 永久启用（修改 sysctl 配置文件）
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```

这样才有了拆包后发现目的地址不是自己，走自身的路由表进一步封包转发的逻辑

**判断目标 IP 目标是否是自己**：

- 如果目标 IP 地址是本机（**任何本地接口的 IP**），网络包会进入本机的上层协议栈（如 TCP/UDP）。
- 如果目标 IP 地址不是本机，内核会根据路由表决定如何转发数据包。

### Pod 到 Pod，跨节点

![pod-to-pod-different-nodes](https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image/pod-to-pod-different-nodes.gif)

跨节点的情况其实也是类似的，此时我们假设目的PodIP为10.42.1.123，上述同一节点中Pod到Pod传递网络包的过程有几处地方有所不同：

1. 在Pod网络命名空间下的网络协议栈封装网络包，根据目标IP走路由表，此时发现匹配的是第三项，即网络包经由网关转发，此时确定网络接口为eth0，且以太帧的mac地址取网关（即cni0网络接口）的mac地址
2. 以太帧到达宿主机root网络命名空间后，走宿主机的网络协议栈，同理在二层会匹配到cni0，因而会先进入cni0。cni0网桥通过arp协议查询IP地址，发现没有匹配的出口，进而匹配失败
3. 匹配失败后网络包会继续往上走宿主机的网络协议栈，会继续拆包，到达第三层，此时可以获取到网络包的目的IP地址，进而会走宿主机网络协议栈的iptables和路由表
4. K8s flannel网络插件环境下的路由表类似下面的情况，由于目的地址是10.42.1.123，会匹配到路由表的第五项，通过flannel.1网络接口，经由10.42.1.0网关服务器将网络包发送出去

```bash
root@datacloak:~# ip route 
default via 10.10.20.1 dev ens18 proto static metric 100
10.10.20.0/22 dev ens18 proto kernel scope link src 10.10.21.190 metric 100 
10.42.0.0/24 dev cni0 proto kernel scope link src 10.42.0.1 
10.42.1.0/24 via 10.42.1.0 dev flannel.1 onlink 
10.42.2.0/24 via 10.42.2.0 dev flannel.1 onlink
```

5. 二层的以太帧到达flannel.1网卡, 数据包将通过VXLAN协议进行封装, 包含src node ip和 dest node ip等
6. 封装后的数据包通过物理网络传输, 经过Node A的eth0发出, Node B的eth0接受
7. Node B收到数据包后, 其flannel.1接口接受数据包并解封VXLAN协议的封装，恢复为原始的数据包
8. 看到此数据包中的dest pod ip，依照Node B自身的路由表，数据包被路由到其cni0网桥
9. 到达网桥后进一步转发对应的veth，最终到达目的Pod命名空间的eth0，进入Pod命名空间的网络协议栈

## Pod 到 Service 网络

### netfilter 和 iptables

为了在集群内执行负载平衡，Kubernetes 依赖于 Linux 内置的网络框架 — `netfilter`。Netfilter 是 Linux 提供的一个框架，允许以自定义处理程序的形式实现各种与网络相关的操作。Netfilter 提供了用于数据包过滤、网络地址转换和端口转换的各种功能和操作，这些功能和操作提供了引导数据包通过网络所需的功能，以及提供了禁止数据包到达计算机网络内敏感位置的能力。

`iptables`是一个**用户空间程序**，提供基于表的系统，用于定义使用 netfilter 框架操纵和转换数据包的规则。在 Kubernetes 中，**iptables 规则由 kube-proxy 控制器配置**，该控制器监视 Kubernetes API 服务器的更改。当对服务或 Pod 的更改更新服务的虚拟 IP 地址或 Pod 的 IP 地址时，iptables 规则会更新，以将指向服务的流量正确路由到支持 Pod。iptables 规则监视发往服务虚拟 IP 的流量，如果匹配，则从可用 Pod 集合中选择一个随机 Pod IP 地址，iptables 规则将数据包的目标 IP 地址从服务的虚拟 IP 更改为所选 Pod 的 IP。当 Pod 启动或关闭时，iptables 规则集会更新以反映集群的变化状态。换句话说，**iptables 已在机器上完成负载平衡，将指向服务 IP 的流量转移到实际 Pod 的 IP**。

在返回路径上，IP 地址来自目标 Pod。在这种情况下，**iptables 再次重写 IP 标头，将 Pod IP 替换为服务的 IP，以便 Pod 认为它一直只与服务的 IP 进行通信**。

### IPVS

Kubernetes 的最新版本 (1.11) 包含第二个集群内负载平衡选项：IPVS。IPVS（IP 虚拟服务器）**也是基于 netfilter 构建的**，并作为 Linux 内核的一部分实现传输层负载平衡。IPVS 被整合到 **LVS（Linux Virtual Server）**中，它在主机上运行，并充当真实服务器集群前端的负载平衡器。IPVS 可以将对基于 TCP 和 UDP 的服务的请求定向到真实服务器，并使真实服务器的服务在单个 IP 地址上显示为虚拟服务。这使得 IPVS 非常适合 Kubernetes 服务。

声明 Kubernetes 服务时，您可以指定是否要使用 iptables 或 IPVS 进行集群内负载平衡。IPVS 专为负载平衡而设计，使用更高效的数据结构（哈希表），与 iptables 相比，几乎可以无限扩展。创建使用 IPVS 进行负载平衡的服务时，会发生三件事：在节点上创建虚拟 IPVS 接口，将服务的 IP 地址绑定到虚拟 IPVS 接口，并为每个服务 IP 地址创建 IPVS 服务器。

将来，IPVS 有望成为集群内负载平衡的默认方法。此更改仅影响集群内负载平衡，在本指南的其余部分中，您可以安全地将 iptables 替换为 IPVS 以实现集群内负载平衡，而不会影响其余讨论。现在让我们看看数据包通过集群内负载平衡服务的生命周期

### Pod 到 Service

![pod-to-service](https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image/pod-to-service.gif)

从Pod到Service的网络过程和上面的Pod到Pod又会有些不一样的地方，例如我们这里的Service ClusterIP取10.43.199.175为例：

1. 在Pod网络命名空间下的网络协议栈封装网络包，根据目标IP走路由表，此时发现后面的项目都无法匹配，走**默认路由**，即网络包经由网关转发，此时确定网络接口为eth0，且以太帧的mac地址取网关（即cni0网络接口）的mac地址
2. 以太帧到达宿主机root网络命名空间后，走宿主机的网络协议栈，同理在二层会匹配到cni0，因而会先进入cni0。cni0网桥通过arp协议查询IP地址，发现没有匹配的出口，进而匹配失败
3. 匹配失败后网络包会继续往上走宿主机的网络协议栈，会继续拆包，到达第三层，此时可以获取到网络包的目的IP地址，进而会走宿主机网络协议栈的iptables和路由表
4. 经过iptables的PREROUTING链，目标IP svc会转化为对应的PodIP，这里假设为 10.42.1.20
5. 然后进入路由表，发现匹配第四条路由，即通过flannel.1网络接口，经由10.42.1.0网关服务器将网络包发送出去
6. 然后后续过程和跨节点Pod通信的过程一致

### Service 到 Pod

![service-to-pod](https://codereaper-image-bed.oss-cn-shenzhen.aliyuncs.com/image/service-to-pod.gif)

接收此数据包的 Pod 将做出响应，将源 IP 标识为其自己的 IP，将目标 IP 标识为最初发送数据包的 Pod (1)。进入节点后，数据包将流经 iptables，iptables 用于`conntrack`记住其先前做出的选择，并将数据包的源重写为服务的 IP，而不是 Pod 的 IP (2)。从这里，数据包通过网桥流向与 Pod 命名空间配对的虚拟以太网设备 (3)，然后流向我们之前看到的 Pod 的以太网设备 (4)。

## 更多案例

### 浏览器通过NodePort访问服务

1. 浏览器访问 10.10.23.110:30000
2. 浏览器发出http请求，封装为以太帧发送到服务器的eth0接口
3. 服务器收到请求包，走宿主机网络协议栈并拆包
4. 拆包到第三层，发现IP是本机网卡IP，继续向上层网络协议栈提交
5. 拆包到第四层，发现是TCP请求，得到端口号
6. 走iptables L4规则匹配，发现PREROUTING可以匹配到相关的规则，进而进行DNAT，转化为PodIP

```bash
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
```

7. 然后发现目标IP地址和当前机器不一致，走转发网络包逻辑
8. 然后走匹配路由表相关的过程
