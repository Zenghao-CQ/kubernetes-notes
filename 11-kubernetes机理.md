## Kubernetes机理

#### 1. Kubernetes架构
* kubernetes的组件
  * Kubernetes Control Plante控制面板：etcd分布式持久化存储；API服务器；etcd分布式持久化存储；scheduler调度器；controller manager控制器管理器
  * (Worker) Nodes工作节点：kubelet；kubelete服务代理（kube-proxy）；container runtime容器
  * ADD-ON components附加组件：kubernetes DNS服务器；dashboard仪表盘；Ingress 控制器；heapster(容器监控插件)；容器网络接口插件；
    ```shell
    $ sudo kubectl get componentstatuses #查看各个组件健康状态
    ```
* 分布式特性
  * 各组件之间的依赖关系
  ![](./pictures/kubernetes-architect.png)
  * 组件间通信：
    * 只能通过API服务器，其他组件不直接与API服务器通信，而是通过API服务器来修改集群状态。
    * 通常与API服务器的连接由组件发起，但是**获取日志**、**kubectl attch(附着到主程序)**、**port-forward**时候由API发起