```
#! /bin/bash

VERSION="1"

kube_pod_subnet="10.244.0.0/16" #使用flannel网络.  Calico:192.168.0.0/16 #https://docs.projectcalico.org/getting-started/kubernetes/quickstart
kube_version="1.17.4" #要安装的集群版本（kubectl kubeadm kubelet）
kube_image_server="registry.cn-hangzhou.aliyuncs.com/google_containers"

# flannel针对Kubernetes v1.17+,如果安装k8s v1.6~v1.15 或 v1.16 ,则flannel的配置参考如下：
# https://github.com/coreos/flannel/blob/master/Documentation/kubernetes.md

###########################################################
##
##  install-env
##
##########################################################

function init-env-set-selinux-permissive()
{
  # Set SELinux in permissive mode (effectively disabling it)
  sudo setenforce 0
  sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
  # Disable SELinux
  # sed -ie 's/SELINUX=permissive/SELINUX=disabled/g' /etc/selinux/config
  # sed -ie 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
}

function init-env-network()
{
  systemctl stop firewalld
  systemctl disable firewalld
  
  # For installing kubeadm
  swapoff -a # Swap disabled. You MUST disable swap in order for the kubelet to work properly.
  cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
  sudo sysctl --system
}

function init-env-repository()
{
  #docker repo
  echo -e "[docker-ce-stable]\nname=Docker CE Stable - \$basearch \nbaseurl=https://download.docker.com/linux/centos/7/\$basearch/stable\nenabled=1 \ngpgcheck=1 \ngpgkey=https://download.docker.com/linux/centos/gpg" > /etc/yum.repos.d/docker-ce.repo
  #k8s repo
  echo -e "[kubernetes] \nname=Kubernetes - \$basearch \nbaseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-\$basearch/ \nenabled=1 \ngpgcheck=0 \nrepo_gpgcheck=0 \ngpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg" > /etc/yum.repos.d/kubernetes.repo
  #这个是做什么？
  yum install centos-release-openstack-rocky -y
}

function init-env-install-software()
{
  yum install -y docker-ce kubeadm-${kube_version} kubectl-${kube_version} kubelet-${kube_version} 
  # 这个是做什么？
  yum install -y openvswitch* certbot
  
  ## Create /etc/docker
  mkdir /etc/docker
  # Set up the Docker daemon
  cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn","http://hub-mirror.c.163.com","https://registry.docker-cn.com"],
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

  mkdir -p /etc/systemd/system/docker.service.d
  # Restart Docker
  systemctl daemon-reload
  systemctl restart docker
  systemctl enable docker
  systemctl enable kubelet


  # cat /etc/sysconfig/kubelet
  # KUBELET_EXTRA_ARGS=--fail-swap-on=false
}


function init-env-kubecomp()
{
   echo -e "https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml" > /etc/kubernetes/kubeenv.list
#  echo -e "https://docs.projectcalico.org/archive/v3.14/manifests/calico.yaml" > /etc/kubernetes/kubeenv.list
#  echo -e "https://raw.githubusercontent.com/kubesys/kubernetes-installer/master/prom/exportor.yaml" >> /etc/kubernetes/kubeenv.list
#  echo -e "https://raw.githubusercontent.com/kubesys/kubernetes-installer/master/prom/grafana-dashboards-configmaps.yaml" >> /etc/kubernetes/kubeenv.list
#  echo -e "https://raw.githubusercontent.com/kubesys/kubernetes-installer/master/prom/grafana-datasources-configmaps.yaml" >> /etc/kubernetes/kubeenv.list
#  echo -e "https://raw.githubusercontent.com/kubesys/kubernetes-installer/master/prom/grafana.yaml" >> /etc/kubernetes/kubeenv.list
#  echo -e "https://raw.githubusercontent.com/kubesys/kubernetes-installer/master/prom/loki.yaml" >> /etc/kubernetes/kubeenv.list
#  echo -e "https://raw.githubusercontent.com/kubesys/kubernetes-installer/master/prom/prometheus.yaml" >> /etc/kubernetes/kubeenv.list
}

function install-env()
{
  init-env-set-selinux-permissive
  init-env-network
  init-env-repository
  init-env-install-software
  init-env-kubecomp
}

###########################################################
##
##  create-k8s
##
##########################################################

function create-k8s()
{
  # --pod-network-cidr string指明 pod 网络可以使用的 IP 地址段。如果设置了这个参数，控制平面将会为每一个节点自动分配 CIDRs。
  kubeadm init \
  	--image-repository ${kube_image_server} \
  	--kubernetes-version v${kube_version} \
  	--pod-network-cidr=${kube_pod_subnet} \
  	--token-ttl 0  #令牌被自动删除之前的持续时间,默认值：24h0m0s,如果设置为 '0'，则令牌将永不过期  
  # For more info: https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/

  rm -rf $HOME/.kube
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  iptables -P FORWARD ACCEPT
  
  while read line
  do
    kubectl apply -f $line
  done  < /etc/kubernetes/kubeenv.list
  
  kubectl taint nodes --all node-role.kubernetes.io/master-
}

###########################################################
##
##  help
##
##########################################################

function help()
{
  echo -e "Welcome to k8s-install ($VERSION), install Kubernetes-based systems from scratch.\n"
  echo -e "Commands:"
  echo -e "  install-env       :\t(Init): install pre-requisite environment"
  echo -e "  create-k8s      :\t(Init): deploy Kubernetes as your want by editing /etc/kubernetes/kubeenv.list. Now it includes flannel,calico"
  echo -e "------------"
  echo -e "For kubeadm join:"
  echo -e "https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#join-nodes"
}
```

