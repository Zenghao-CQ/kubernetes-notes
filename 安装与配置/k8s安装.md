注意 ubuntu 18.04之后的静态ip设置不同

1.集群所有主机关闭swap
```
sudo swapoff -a

sudo sed -i 's/.*swap.*/#&/' /etc/fstab
可用 free -h查看
```
如果重启后swap还是自动挂载执行
```

```shell
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

2.集群中所有服务器安装docker
```
sudo apt-get update

sudo apt-get -y install \
apt-transport-https \
ca-certificates \
curl \
gnupg-agent \
software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
"deb [arch=amd64] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) \
stable"

sudo apt-get update

sudo apt-get install -y docker-ce docker-ce-cli containerd.io
#docker换源
sudo vi /etc/docker/daemon.json
**注意systemd cgroupf区别**
#设置systemd并添加源
{ 
    "exec-opts":["native.cgroupdriver=systemd"],
    "registry-mirrors": ["https://registry.docker-cn.com"]
}

```
 sudo systemctl daemon-reload
 sudo systemctl restart docker
3.集群所有节点上安装 kubectl kubelet kubeadm
``` 
添加阿里云k8s源

curl -fsSL https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -

sudo vim /etc/apt/sources.list

deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main

sudo apt-get update

sudo apt-get -y install kubectl=1.22.3-00 kubelet=1.22.3-00 kubeadm=1.22.3-00

sudo apt-mark hold kubelet kubeadm kubectl
```

4.初始化master节点
```
kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.22.3 --pod-network-cidr=10.244.0.0/16 --token-ttl=0 ##注意版本对应关系
//对于阿里云，华为云，要指定内网ip
kubeadm init --image-repository registry.aliyuncs.com/google_containers  --apiserver-advertise-address=192.168.0.97  --kubernetes-version v1.19.3 --pod-network-cidr=10.244.0.0/16 --token-ttl=0 ##注意版本对应关系

# 输出
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.106.11:6443 --token jy2sc9.g3zyl861vmco7lid \
    --discovery-token-ca-cert-hash sha256:bead73f863b9774086962a95991750dc015187fffb3b0a7f04051720e4c4f027 

#复制配置
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

设置自动补全
echo "source <(kubectl completion bash)" >> ~/.bashrc
source ~/.bashrc
```

6.加入master
```
kubeadm join 192.168.0.130:6443 --token 2tun7p.y9ftxhjvwt69djyx \
--discovery-token-ca-cert-hash sha256:63159dd2b07b995894790ec0e7293977b698eccfb9c6b21f1b3a8cb007682e5a
```

默认情况下，token的有效期是24小时，如果我们的token已经过期的话，可以使用以下命令重新生成：
```
kubeadm token create
```
如果我们也没有--discovery-token-ca-cert-hash的值，可以使用以下命令生成：
```
# openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```
 kubeadm reset 可以重置'kubeadm init' or 'kubeadm join'的操作。  同时需要删除家目录下的.kube 目录。才能恢复到初始化之前的状态

7. 安装flannel
```
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
wget https://hub.fastgit.org/flannel-io/flannel/blob/release/v0.14.1/Documentation/kube-flannel.yml

#无法拉取则手动安装镜像（在https://github.com/flannel-io/flannel/releases/下载对应版本）
docker load < flanneld-v0.14.0-amd64.docker
docker load -i flanneld-v0.14.0-amd64.docker #效果一样

# 使用配置文件启动fannel
kubectl apply -f kube-flannel.yml

# 再次查看集群节点的状态为ready即可
kubectl get nodes
```

删除
```
rm -rf /etc/kubernetes
rm -rf ~/.kube
```
8. 单节点
报错：pod 处于pending状态，查看events
```
69s         Warning   FailedScheduling    pod/ingress-nginx-admission-create-77rlp         0/1 nodes are available: 1 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate.
```
```
允许master节点部署pod
kubectl taint nodes --all node-role.kubernetes.io/master-
如果不允许调度
kubectl taint nodes master1 node-role.kubernetes.io/master=:NoSchedule
污点可选参数
	  NoSchedule: 一定不能被调度
      PreferNoSchedule: 尽量不要调度
      NoExecute: 不仅不会调度, 还会驱逐Node上已有的Pod
```

DEGUB
```
systemctl list-units --type=service
netstat -an 查看全部端口
```
kubeadm init时kubeadm 把 kubelet 视为一个系统服务来管理
配置可以查看 /etc/systemd/system/kubelet.service.d/10-kubeadm.conf，[参见](https://blog.csdn.net/wiborgite/article/details/52863913)
此外 /usr/lib/systemd/system/kubelet.service也有service配置
在as服务器时因为多了--cgroup-driver=cgroupfs而启动失败与docker的/etc/docker/daemon.json冲突
此外查看kubelet服务，似乎两个都用到了

```
systemctl status kubelet -l
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/kubelet.service.d #可以看到是第一项建立的
             └─10-kubeadm.conf
     Active: active (running) since Tue 2021-09-07 08:10:00 UTC; 20min ago
```

目录```/lib/systemd/system```以及```/usr/lib/systemd/system```其实指向的是同一目录，在```/```目录下```ll```可知：能看到符号链接；而```/etc/systemd/system/```是系统管理员安装的单元, [优先级更高](https://www.jianshu.com/p/32c7100b1b0c)

9. 删除集群
```
root@node1:~/mycodes/codes# kubeadm reset
[reset] Reading configuration from the cluster...
[reset] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[reset] WARNING: Changes made to this host by 'kubeadm init' or 'kubeadm join' will be reverted.
[reset] Are you sure you want to proceed? [y/N]: y
[preflight] Running pre-flight checks
[reset] Removing info for node "node1" from the ConfigMap "kubeadm-config" in the "kube-system" Namespace
[reset] Stopping the kubelet service
[reset] Unmounting mounted directories in "/var/lib/kubelet"
[reset] Deleting contents of config directories: [/etc/kubernetes/manifests /etc/kubernetes/pki]
[reset] Deleting files: [/etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf /etc/kubernetes/bootstrap-kubelet.conf /etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf]
[reset] Deleting contents of stateful directories: [/var/lib/etcd /var/lib/kubelet /var/lib/dockershim /var/run/kubernetes /var/lib/cni]

The reset process does not clean CNI configuration. To do so, you must remove /etc/cni/net.d

The reset process does not reset or clean up iptables rules or IPVS tables.
If you wish to reset iptables, you must do so manually by using the "iptables" command.

If your cluster was setup to utilize IPVS, run ipvsadm --clear (or similar)
to reset your system's IPVS tables.

The reset process does not clean your kubeconfig files and you must remove them manually.
Please, check the contents of the $HOME/.kube/config file.
```

清空CNI插件
```
rm -rf /var/lib/cni/
rm -rf /etc/cni/net.d/
```