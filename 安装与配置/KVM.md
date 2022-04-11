桥接模式的网络，用``brctl addif virbr0 vnet0``将网桥连在一起
https://blog.csdn.net/wx912820/article/details/108844424
https://www.cnblogs.com/leozhanggg/p/11772251.html
https://blog.csdn.net/weixin_36820871/article/details/80595855


```
#静态ip模板
#/etc/sysconfig/network-scripts/ifcfg-ens3
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens3
UUID=1924541f-a5a2-4bf5-8f14-1ac9b0e48f32
DEVICE=ens3
ONBOOT=yes
IPADDR=192.168.122.61
GATEWAY=192.168.122.1
NETMASK=255.255.255.0
DNS1=8.8.8.8
DNS2=192.168.122.2
```
重启网卡
```
service network start

service network-manager restart

如果是 Kali Linux（Debian），则需要用以下命令：
service networking restart

如果是Centos 8，则需要用以下命令：
nmcli c reload
```

```
virt-clone   -o kvm-4 -n kvm-5 -f /home/vm-volume/kvm-5.qcow2
```

```
ssh -L  12345:192.168.122.61:32677 -L 12346:192.168.122.61:31891 -L 12347:192.168.122.61:32118 root@124.16.138.66 -p 40322
```