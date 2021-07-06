## minikube安装配置

1. #### 下载安装
    国内环境下利用阿里云发布的 minikube 来安装，整个过程会变得十分流畅。至于版本 v1.2.0 这里可以自行查看最新版 
    ```shell
    curl -Lo minikube http://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/releases/v1.2.0/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
    ```

2. #### 启动
    ```shell
    minikube start --vm-driver=none --registry-mirror=https://registry.docker-cn.com
    ```
    minikube实际是跑在虚拟机上的，但由于这里已经是在 VM 中运行的 minikube，所以采用 --vm-driver=none 方式，便不需要格外的创建虚拟机了 
    同时指定镜像下载使用 docker 国内源

3. #### 重复
    失败了可以先minikube delete然后重复2

4. #### kubectl安装
    ```shell
	# 安装依赖
	$ sudo apt-get update && apt-get install -y apt-transport-https
	# 加载key
	$ sudo curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
	# 添加源（su 到root用户）
	$ sudo cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
	deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
	EOF
	# 安装
	$ sudo apt-get update
    $ sudo apt-get install -y kubectl ///kubelet kubeadm
    ```
5. CoreDNS 处于CrashLoopBackOff 状态，导致无法通过域名访问
    见 ```05-服务service.md```


6. #### 踩坑
    * 没试过的国内源
        * https://blog.csdn.net/ccgshigao/article/details/111660067
        * https://zhuanlan.zhihu.com/p/111422247
        * https://www.cnblogs.com/ExMan/p/11613750.html
    * 通过github安装，无法执行```minikube start```：
        ```shell
        $ sudo curl -Lo minikube https://github.com/kubernetes/minikube/releases/download/v1.5.0/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
        ```
    * 通过google(需要梯子)：
        ```shell
        curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
        ```