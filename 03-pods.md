## **pod容器**

1. #### 简介：
    一个pod的保证一组容器运行于**同一个节点**，通常一个容器运行一个进程（便于运行和日志的管理），故 pod 类似一台**独立机器**
    * 同一pod中的部分隔离：一个pod中**共享**Linux命名空间，故而有相同的，主机名、IPC空间、网络接口等，但是容器间**文件系统相互隔离**
    * 共享端口空间和IP，可以通过localhost互相访问
    * 平坦网络空间：所有 pod 都在同一个共享网络地址空间中，**类似一个LAN**：pod对应主机结点，容器对应进程
    * pod开销小，故一个pod托管一个应用即可；**多层应用**如前后端、数据库，应分散在不同pod，**保证结点可以充分利用**；而且便于对某一部分**扩缩容**
    * 一个pod可以放多个容器，一个Main containner，数个sidecar
    * 总结：**1个pod：1个主机，1个应用层级，有多个容器，容器进程一对一**

2. #### 用YAML和JSON创建pod
    * 简单方法：
        ```shell
        sudo kubectl run kubia --image=luksa/kubia --port=8080
        ```
    * 获得完整pod信息
        ```
        sudo kubectl describe po kubia
        ```
    * 获得完整的YAML描述
        ```
        sudo kubectl get kubis -o YAML
        ```
    * **pod定义的主要部分**：
        * metadata 元数据，名称、命名空间、标签和注解等）
        * spec 规格/内容( pod 的容器列表、volume、卷、其他数据等）
        * status（***运行数据，创建时不需要**），pod及其内部容器的详细状态，pod的当前信息，例如pod所处的条件、每个容器的描述和状态，以及内部IP等
    * 手动创建YAML，书P61
        ```shell
        #./kubia-manual.yaml
        apiVersion: v1
        kind: Pod
        metadata:
            name: kubia-manual
        spec: 
            containers:
            -   image: luksa/kubia
                name: kubia 
                ports:
                -   containerPort: 8080 #监听端口，指定端口不影响访问，纯粹是展示性的，只是方便查看
                    protocol: TCP
        ```
        
    * 可以通过以下命令查看对象具体意义
        ```
        sudo kubectl explain pods
        sudo kubectl explain pods.spec
        ```
    * **创建**
        ```
        sudo kubectl create -f kubia-manual.yaml #-f means file
        ```
    
    * 查看日志：
        ```
        sudo kubectl logs kubia
        sudo kubectl logs kubia -c kubia # 包含多个容器时候
        sudo kubectl logs kubia --previous # 查看之前崩溃的原因
        #使用ssh命令登录到pod正在运行的节点?
        sudo docker logs kubia
        ```

    * **访问Pod**
        * 创建service服务
            ```
            sudo kubectl expose pod kubia --type=NodePort #暴露结点
            sudo kubectl expose rc kubia --type=LoadBalancer --name kubia-http #暴露rc
            # 在自己搭建的k8s集群 （公有云环境除外）没有LB能力
            ```
        * 绕开service，通过端口转发直接访问
            * 将机器的本地端口 8888转发到kubia-manual pod的端口8080。可能会报错，需要安装socat，此外port-forward会占用shell，需要ctrl C手动退出
                ```
                sudo kubectl port-forward kubia-manual 8888:8000 #--address 0.0.0.0 指定ip
                ```
            
3. #### 用标签组织Pod
    * 组织pod和所有其他资源，可以附加到任何资源的键值对，一个资源可以对应多个标签，两个维度：
        * app，属于哪个应用、 组件或微服务
        * rel，程序版本，stable、beta、canary
    * 标签指定：
        * 查看标签
            ```
            sudo kubectl get po --show-labels #直接列出
            sudo kubectl get po -L creation_method,env#附加的列出
            ```
        * 在创建时添加：
            ```
            metadata:
              name: kubia-manual-v2
              labels:
                creation_method: manual #两个键值对标签
                env: prod            
            ```
        * 修改标签：
            ```
            sudo kubecl label po kub ia-manual creation method=manual
            #在更改现有标签时， 需要使用--overwrite选项
            sudo kubecl label po kub ia-manual creation method=forceManual --overwrite
            ```
        * 利用标签与**选择器**进行选择，（删除等操作也可以）：
            ```
            sudo kubectl get po -l creation_method=manual
            sudo kubectl get po -l creation_method!=manual
            sudo kubectl get po -l env
            sudo kubectl get po -l '!env' #''防止bash解释，不为
            sudo kubectl get po -l env in (prod,devel)
            sudo kubectl get po -l env not in (prod,devel)
            sudo kubectl get po -l app=pc, rel=beta #使用多个标签选择
            sudo kubectl get po -o custom-column=POD:metadata.name,NODE:spec.nodeName --sort-by spec.nodeName -n kube-system
            ```
    * 使用标签和选择器进行pod调度约束
        例如，某些 pod 的应用需要固态、GPU，用标签描述其调度需求
        * 标记结点（主机结点）
            ```
            sudo kubectl label node <node-name> gpu=true #标准主机
            kubectl get nodes -l gpu=true #列出标签
            ```
        * 标记pod，调度得到特定结点（一类）
            ```
            spec
              nodeSelector:
                gpu: "true" #字符串
            ```
        * 将pod调度到某一个特定结点，通过唯一标签，kubernetes.io/hostname即为主机名
    * 注解
        * 获取注解：获取pod的yaml或者describe pod，在metadata.annotation中
        * 添加注解
            ```
            sudo kubectl annotate pod kubia-manual mycompany.com/someannotation="foo bar"
            ```
4. #### 命名空间
    * 获取全部命名空间
        ```
        sudo kubectl get ns
        ```
    * 按照命名空间查看pod
        ```
        sudo kubectl create namespace custom-system #用-n亦可
        ```
    * **创建命名空间**
        ```
        #直接创建
        sudo kubectl create -f custom-namespace.yaml
        ```
        ``` 
        #用yaml创建
        #./custom-namespace.yaml
        apiVersion: v1
        kind: Namespace
        metadata: 
          name: custom-namespace
        
        $ sudo kubectl create -f custom-namespace.yaml 
        ```
        * 在创建pod等资源时指定ns。允许两个同名pod位于**不同ns**中
            ```
            sudo kubectl create -f kubia-manual.yaml -n custom-namespace
            ```
        * 切换**默认**ns
            ```
            sudo kubectl config set-context $ (kubectl config current -context) -- namespace <ns-name>
            ```
5. #### 删除和停止pod
    * 删除pod，**注意**是先通过发送**SIGTERM**关闭，失败则则继续发送**SIGKILL**，进程需要正确处理SIGTERM信号
        ```shell
        sudo kubectl delete po [pod1] [pod2]
        sudo kubectl delete po --all
        ```
    * 通过标签删除（全部）
        ```shell
        sudo kubectl delete po -l creation_method=manual
        ```
    * 通过命名空间删除（包括全部pod）
        ```shell
        sudo kubectl delete ns custom-namespace
        ```
    * 当存在replication controler时，要先删除rc
        ```shell
        sudo kubectl delete rc [rc-name]
        ```
    * 删除几乎全部，不包括secret等
        ```
        sudo kubectl delete all -all #all指代全部资源
        ```
        