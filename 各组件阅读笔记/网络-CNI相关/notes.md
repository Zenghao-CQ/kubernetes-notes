kubelet 的默认--root-dir为/var/lib/kubelet，其中pods目录由pods的相关信息


### flannel:
* UDP模式，工作在三层，虚拟TUN设备
* VXLAN，工作在二层，虚拟VTEP设备，相当于手动设置vlan

CNI插件，容器间的网络实现（pod内部共享network namespace）配置在``/etc/cni/net.d``以及二进制在``/opt/cni/bin``，IPAM plugin分配IP，确定容器网络接口的 IP、子网以及网关和路由； Kubelet 将为 CNI 插件指定 Kubernetes 集群内部 DNS 服务器 IP 地址，确保正确设置容器的``resolv.conf``文件

fdb:维护转发的端口
```
[root@KVM01 .kube]# bridge fdb show
01:00:5e:00:00:01 dev ens3 self permanent
33:33:00:00:00:01 dev ens3 self permanent
33:33:ff:87:fa:0a dev ens3 self permanent
33:33:00:00:00:01 dev docker0 self permanent
01:00:5e:00:00:01 dev docker0 self permanent
02:42:72:d1:75:d4 dev docker0 vlan 1 master docker0 permanent
02:42:72:d1:75:d4 dev docker0 master docker0 permanent
a2:82:f9:b7:b4:af dev flannel.1 dst 192.168.122.64 self permanent
2a:4b:0f:ff:c5:7b dev flannel.1 dst 192.168.122.62 self permanent
9a:02:6a:5f:62:3d dev flannel.1 dst 192.168.122.63 self permanent
ba:83:d8:fd:9c:29 dev flannel.1 dst 192.168.122.65 self permanent
33:33:00:00:00:01 dev cni0 self permanent
01:00:5e:00:00:01 dev cni0 self permanent
33:33:ff:e8:47:a6 dev cni0 self permanent
5a:9c:3b:e8:47:a6 dev cni0 vlan 1 master cni0 permanent
```

查看arp的记录
```
[root@KVM01 .kube]# ip neigh show
10.244.0.31 dev cni0 lladdr 4a:f0:22:d4:bc:17 REACHABLE
10.244.2.0 dev flannel.1 lladdr ba:83:d8:fd:9c:29 PERMANENT
10.244.0.27 dev cni0 lladdr fe:83:6c:33:e8:9f REACHABLE
10.244.3.0 dev flannel.1 lladdr a2:82:f9:b7:b4:af PERMANENT
10.244.0.34 dev cni0 lladdr 7a:3a:15:49:83:90 REACHABLE
192.168.122.1 dev ens3 lladdr 52:54:00:9b:31:ff REACHABLE
10.244.0.18 dev cni0 lladdr e2:1d:57:ce:26:bb REACHABLE
10.244.1.0 dev flannel.1 lladdr 2a:4b:0f:ff:c5:7b PERMANENT
10.244.0.3 dev cni0 lladdr 66:57:a5:3a:29:40 REACHABLE
10.244.0.30 dev cni0 lladdr 52:fe:d1:4f:5f:2a REACHABLE
10.244.0.33 dev cni0 lladdr b2:37:3c:7b:2b:16 REACHABLE
192.168.122.2 dev ens3  INCOMPLETE
10.244.0.26 dev cni0 lladdr be:93:25:4b:9b:0b REACHABLE
10.244.0.17 dev cni0 lladdr 12:0e:17:0c:1a:71 REACHABLE
192.168.122.62 dev ens3 lladdr 52:54:00:87:98:c2 REACHABLE
10.244.0.29 dev cni0 lladdr 9e:37:3a:ae:6d:49 REACHABLE
10.244.0.2 dev cni0 lladdr ae:bb:47:6d:54:c5 REACHABLE
10.244.0.32 dev cni0 lladdr fe:e9:ea:1b:a1:a8 REACHABLE
192.168.122.64 dev ens3 lladdr 52:54:00:b4:0d:86 REACHABLE
192.168.122.63 dev ens3 lladdr 52:54:00:5f:99:0a REACHABLE
10.244.0.28 dev cni0 lladdr 72:fa:b5:a1:39:f9 REACHABLE
10.244.0.35 dev cni0 lladdr 16:bc:40:c5:72:78 REACHABLE
192.168.122.65 dev ens3 lladdr 52:54:00:ef:ee:86 REACHABLE
10.244.4.0 dev flannel.1 lladdr 9a:02:6a:5f:62:3d PERMANENT
```


### CNI插件流程——flannel为例：
* 二进制文件在```/opt/cni/bin```
* 启动flanneld，flanneld安装配制文件，在：```/etc/cni/net.d/```，只加载一个，但是可以有多个plugin字段
* dockershim加载配置文件，并设置flannel为默认插件。用flannel配置容器网络栈，用portmap进行端口映射
* CNI 插件的工作原理：
    1. dockershim 就会先调用 Docker API 创建并启动 Infra 容器
    2. 执行 SetUpPod 方法，其作用为 CNI 插件准备参数，然后调用 CNI 插件(```/opt/cni/bin/flannel```)为 Infra 容器配置网络
    3. 参数有：
        1. 为CNI准备的**环境变量**，主要包括```CNI_COMMAND```，其值为```ADD```或者```DEL```。```ADD```参数包括：
            * 容器里网卡的名字 eth0（CNI_IFNAME）
            * Pod 的 Network Namespace 文件的路径（CNI_NETNS）。Pod（Infra 容器）的 Network Namespace 文件的路径在宿主机```/proc/<容器进程的 PID>/ns/net```
            * 容器的 ID（CNI_CONTAINERID）等。
            * ```CNI_ARGS```，以 Key-Value 的格式，传递自定义信息给网络插件
        2. dockershim 把 Network Configuration(配置文件) 以 JSON 数据的格式，通过stdin传给Flannel CNI插件。这里配置是dockershim从```/etc/cni/net.d/10-flannel.conflist```加载的CNI配置。（dockershim传给flannel cni的）
        3. 注意，Flannel 的 CNI 配置文件有```delegate```字段，表明该CNI插件(flannel)，调用 Delegate 指定的某种 CNI 内置插件(CNI bridge)来完成。dockershim 对 Flannel CNI 插件的调用，只是走了个过场。Flannel CNI 插件唯一需要做的，就是对 dockershim 传来的 Network Configuration 进行补充。（flannel cni传给bridge cni的）
    4. 根据上述的两部分参数，flannel插件调用CNI bridge（二进制文件），参数即前面提到环境变量的1，以及flannel补充后的(补充了flannel设置的子网等信息)配置3。此时cni bridge delegate(代表)flannel将容器加入cni网络。
    5. cni bridge在容器中创建veth peer，将另一端移动到host上，并将其连接到cni0网桥。同时还会设置该端口为发卡(hairpin)模式，允许数据包从一个端口进来后，再从这个端口发出去，hairping允许从容器内访问自己映射到宿主机的端口，保证Pod 可以访问到自己。
    6. 之后，由CNI ipam根据```ipam.subnet```的网段为容器分配ip，CNI bridge 插件，在**容器内部**将该IP地址添加在容器eth0网卡上，同时为容器设置默认路由。
* flannel直接使用的主机网络， hostNetwork: true

### 一些概念

CNI插件：k8sv1.7后CNI插件可以通过Api server访问Etcd，无需额外部署。CNI网络插件的作为是，为infra容器设置网络栈，把host上的特殊设备连通。

网络方案包括：
1. 方案本身的逻辑：如flannel的Vxlan，创建网桥、设置路由、配置ARP和FDB，即为flanneld
2. 实现网络插件：如Vxlan是设置infra的网络栈，连接到cni0



### 三层网络方案

例子：flannel的host-gw，calico

#### host-gw(host gate，性能最高)：不是overlay方案
原理：设置其他主机Node2 eth0的ip为本机路由表中，Node2上容器子网ip对应下一跳的地址。进入链路层时，会使用下一跳地址对应的MAC地址(即Node2的Mac)。将每个 Flannel 子网（Flannel Subnet，比如：10.244.1.0/24）的“下一跳”，设置成了该子网对应的宿主机的 IP 地址。即**主机host充当网关gateway**，要求**主机之间二层连通**，避免了额外的封包和解包带来的性能损耗。

### calico
原理： BGP，Border Gateway Protocol，边界网关协议。连接自洽系统的路由器称作边界网关，用TCP传输路由表信息。BGP是在大规模网络中实现节点路由信息共享的一种协议。即为UNDERLAY，overlay是基于隧道技术。
组件：
    1. CNI插件：该部分与k8s对接
    2. Felix，一个DaemonSet，在宿主机插入路由规则(FIB，保存下一跳，维护最佳路由，指导报文转发。路由表只存储目标，掩码，下一跳，转发表存储更详细的信息如输出端口信息，标记信息等。转发表描述了主机方面的信息，在主机内部将一个数据包从一个端口导向另一端口，而路由表描述网络信息，将数据包从一个机器导向另一机器)
    3. Bird，BGP客户端，分发路由信息
区别：**calico不会创建网桥设备**，其CNI插件为每个容器创建veth peer，并且为每个容器都配置一条路由(flannel之间只维护)；相比之间cni0和docker0是虚拟网桥，其通过二层转发route中的网关为0.0.0.0，并且虚拟网桥代理了ARP。出现在veth peer的数据包通过FIB路由的下一跳转发，对于本机的容器，之间二层网桥，别的节点上则通过host的```eth0网卡```直接转发过去，因为节点通过二层连接，在eth0发出去后不用路由，直接通过网桥/交换机ARP然后发送即可，也就是ARP的含义

Felix：维护下一跳的路由规则，在主机插入路由，Calico 将所有节点当作是边界路由器处理，一起组成全连通的网络，互相通过 BGP 协议交换路由规则，称为 BGP Peer。

BIRD：BGP Client，默认采用```Node-to-Node Mesh```模式，client互相通信交互路由信息，node数目超过100采用```Route Reflector```。同样的calico要求主机**二层连通**。否则，下一跳应该为主机2的IP，但是如果不二层联通，不在一个子网，无法直接MAC寻址；三层则需要路由器，这与iptables的下一跳(还是说FIB？)冲突，因为数据包的```dst是容器的ip，路由器不知道向哪里转发```

IPIP模式：对于上述的三层连通的主机，通过```tnul0```转发，本质是一个IP隧道（**跟flannel的tunnel TUN0不一样，这里是IP隧道，前者是内核流向用户程序的隧道**，TUN0根据etcd的子网数据找到宿主机IP，通过UDP发送），封装一个外层的目的主机的ip包并发送，如此路由器可以正确路由。原本src是容器1的ip，dst是容器2的ip，BGP是设置node2的ip为下一跳；封装后原来的包成了payload，dst为node2的ip，可以路由了。性能也Vxlan相当。

替代：将不同子网的主机变成BGP peer：
1. 全部主机与路由器建立peer关系，需要Dynamic Neighbors 的 BGP 配置方式，从特定网段
直接建立peer
2. 用独立组件收集路由信息并且发送给各个网关，网关与宿主机通过Route Reflector通信即可。此时，网关的peer数目固定，可以直接把这些独立组件配置成路由器的 BGP Peer，而无需 Dynamic Neighbors 的支持


### OverLay：通过软件构建一个覆盖在已有宿主机网络之上的、可以把所有容器连通在一起的虚拟网络
注意是在物理网络上再架构一个虚拟网络，用一种传输协议封装另一种传输协议，将容器虚拟ip包作为数据，用宿主机ip进行再次封装，在底层网络传输flannel的UDP和VXLAN都是

**比较：**
* Overlay：覆盖网络，虚拟网络，用隧道技术连接，通过虚拟或者逻辑链路进行通信，其实现基于ip技术的基础网络为主
* underlay：底层网络，现实的物理基础层网络设备，负责互联互通
* Calico主要有两种互联模式：
    * 基于IPIP封装的overlay模式
    * 基于BGP路由的underlay模式
* Flannel主要三种：
    * UDP，三层overlay
    * VXlan，二层的overlay（因为改动的是二层的fdb表）
    * hoat-gateway，underlay，没有经过任何封装，纯路由实现，数据只经过协议栈一次，因此性能比更高