## 副本机制、控制器：pod托管

部署项目健康，当节点上的pod进程崩溃时，kubelet会重启，但如果整个节点崩溃，则pod丢失（无rc，rs时候）。另外时候进程没有崩溃，但应用程序也无法正常运行（如JVM虚拟机异常，程序已经停止，但虚拟机不会崩溃），此时应该重启pod。
1. #### 存活探针
    * 可以为**pod**的每个**容器**指定探针，检查其是否运行
        * HTTP GET探针对IP地址进行http get请求，错误相应或者无响应则重启
            ```shell
            #./kubia-liveness-probe.yaml
            apiVersion: v1
            kind: Pod
            metadata:
              name: kubia-liveness
            spec:
              containers:
              - image: luksa/kubia-unhealthy #作者设置的在第六次返回500的版本
                name: kubia
                livenessProbe: #探针
                  httpGet:
                    path: /# http请求路径
                    port: 8080
                  initialDelaySeconds: 100  #首次探针延迟时间，也就是预计的系统启动时间
                  timeoutSeconds: 5 #每次探针的超时时间
            ```
            ```bash
            $ sudo kubectl logs kubia --previous # 查看之前崩溃的日志 
            $ sudo kubectl describe po kubia-liveness #查看状态
            #输出
            Last State:     Terminated
                Reason:       Error
                Exit Code:    137 #上一次的错误码，137=128+x, x=9是终止进程的信号编号
                                  # SIGKILL，进程被强行终止。145=128+17，17为SIGTERM
                Started:      Tue, 29 Jun 2021 06:36:16 -0700
                Finished:     Tue, 29 Jun 2021 06:38:05 -0700
            Ready:          True
            Restart Count:  1
            Liveness:       http-get http://:8080/ delay=0s timeout=1s period=10s #success=1 #failure=3 #立即开始，1s超时，10s一次，1次成功则成功，3次失败则失败
            ```
        * TCP 套接字（socket），建立TCP连接，失败则重启
        * Exec，在容器内部执行命令，检查退出码，不为0则重启
    * **delay=0s时可能未准备好，导致退出**，要配置合适的开始时间
    * 可以配置特定URL如“/health”，保证其可以直接访问，且轻量化
    * 重复尝试无意义，kubernetes为了确认一次失败本来就会多次尝试
    * 当databsae无法连接时，前端不应该返回失败，因为此时重启前端没有意义；

2. #### ReplicationController（replicas副本）
    * rc根据pod的标签进行操作，**保证pod数量始终与标签选择器匹配**，rc的三个部分：
        * label selector，选择器，确定rc作用域里的pod。更改selector对已有的pod无影响，只是使得pod不再被rc关注，它只影响**创建新的pod**
        * replica count副本个数
        * pod template pod模板，用于创建新的副本，模板的label应该与selector一样，否则会不停的创建副本（rc需要的pod为0）
    * rc的创建
        ```shell
        #./kubia-rc.yaml
        apiVersion: v1
        kind: ReplicationController
        metadata:
          name: kubia
        spec:
          replicas: 3
          selector:  #选择器 指定标签
            app: kubia
          template:  #pod模板
            metadata:
              labels:
                app: kubia
            spec:
              containers:
              - name: kubia
                image: luksa/kubia
                ports:
                - containerPort: 8080
        ```
    * 将pod移出rc：
        pod在rnetadata.ownerReferences字段中引用rc。可以手动更改selector对应的标签，然后rc会新建一个pod
    * 修改模板template
        只会影响新建的pod，不影响已有的
        ```shell
        sudo kubectl edit rc kubia
        $ export KUBE_EDITOR = "/usr/bin/nano" #更改编辑器
        ```
    * 水平放缩rc
        ```shell
        #方法一：
        sudo kubectl scale rc kubia --replicas=10
        #方法2
        sudo kubectl edit rc kubia
        ```
    * 删除rc，pod会同时删除，也可以保留pod
        ```
        sudo kubectl delete rc kubia --cascade=false
        sudo kubectl delete rc kubia --cascade=orphan #false已经弃用
        ```
3. #### ReplicationSet,新一代控制器
    rc只能匹配一个标签，rs可以匹配多个标签甚至是缺省标签，或者特定key的标签
    * 创建rs
      ```shell
      apiVersion: apps/v1beta2
      kind: ReplicationSet
      metadata:
        name: kubia
      spec:
        replicas: 3
        selector: 
          matchLabels: #使用matchLabel选择器
            app: kubia
        template:  #pod模板
          metadata:
            labels:
              app: kubia
          spec:
            containers:
            - name: kubia
              image: luksa/kubia
      ```
    * 更加强的selector:matchExpressions 
        ```shell
        selector:
          matchExpressions:
            - key: app
              operator: In
              values:
                - kubia
        # In 在values列表中时为true
        # NotIn 不在list中
        # Exits 包含一个指定名称的标签（值不重要），此时不使用values
        # DoesNotExit 不存在标签
        ```
        单个matchExpress之间是**and语义**，表达式全为true时生效
        多个matchExpress之间是or语义
        matchLabel和matchExpression都有时，要二者都为true

4. #### DaemonSet
    * 保证每个节点运行一个pod，**避开kubenetes调度机制**，不可调度的节点也可以
    * 用pod模板中的nodeSelector属性指定节点
    * 创建，假定是ssd硬盘驱动
        ```shell
        #./ssd-monitor-daemonset.yaml
        apiVersion: apps/v1beta2
        kind: DaemonSet
        metadata:
            name: ssd-monitor
        spec:
            selector:
                matchLabels:
                    app: ssd-monitor #选择器选定的标签
            template:
                metadata:
                    labels:
                        app: ssd-monitor #模板pod的标签，应该与选择器一致
                spec:
                    nodeSelector:
                        disk: ssd #部署pod时，选择节点的标签
                    containers:
                    -   name: main
                        image: luksa/ssd-monitor
        ```
    * 手动将节点标为disk: ssd使得ds可以发现。若修改标签则pod会被终止。**删除ds时pod会一同删除**
        ```shell
        sudo label node minikube disk=ssd 
        ```
5. #### 执行单个任务job
    * 节点故障或程序出错时会重新部署，直到任务完成，任务完成后不再重启
    * 例子：转换并导出数据
        ```shell
        #./exporter.yaml
        apiVersion: batch/v1
        kind: Job
        metadata:
            name: batch-job
        spec:
            template:
                metadata:
                    labels:
                        app: batch-job
                spec:
                    restartPolicy: OnFailure #pod等默认为Always，还可以Never
                    containers:
                    -   name: main
                        image: luksa/batch-job
        ```
    * 结束后pod不会马上删除，因为可以查看logs
        ```shell
        sudo kubectl logs batch-job-6vhzt
        ```
    * 一个job中运行多个pod，先建两个pod，并行执行直到5个全部结束
        ```shell
        apiVersion: batch/v1
        kind: Job
        metadata:
            name: batch-job
        spec:
            template:
                metadata:
                    labels:
                        app: batch-job
                spec:
                    completions: 5                #顺序的执行5个 Pod，也就是执行 5 次
                    parallelism: 2                #最多可以两个 Pod 并行
                    restartPolicy: OnFailure      # 重启策略，默认为 Always，如无必要，不要使用默认
                    activeDeadlineSeconds: 15 #要求pod必须该时间内完成
                    containers:
                    -   name: main
                        image: luksa/batch-job
        ```
    * job 水平放缩
        ```shell
        sudo kubectl scale job [job-name] --replicas 3
        ```
    * 时间：设置pod的activeDeadlineSeconds属性，限制最长时间，超过则认为pod**失败**
    * 通过指定Job manifest中的spec.backoffLimit字段，配置Job在被标记为失败之前可以重试的**次数***
6. #### 定期执行或未来执行：CronJob
    * 创建。job套壳cronJob，**CronJob 会创建 Job，Job 会创建 Pod**
        ```shell
        apiVersion: batch/v1beta1
        kind: CronJob
        metadata:
          name: cron-job
        spec:
          schedule: "0,15,30,45 * * * *" # 在每h的0、15、30、45分钟运行、
          startingDeadlineSeconds: 15 #认为开始15s未启动则记为failed
          jobTemplate: #job的模板，可以完全按照job配置，这里省略metadata
            spec:
              template: #job的pod的模板
                metadata:
                  labels:
                    app: cron-job
                spec:
                  restartPolicy: OnFailure     
                  containers:
                  - name: main
                    image: luksa/batch-job
        ```
    * 时间表schedule：
        ```shell
        分钟 小时 第几天 第一个月 星期几(周日为0) 空格间隔，条目内部用逗号
        "0, 15, 30, 45 * * * *" #每15min一次
        "0, 30 * * * *" #每个月第一天，每30min一次
        "0 3 * * 0" #每周日0分3时（3:00am）
        ```
    * 设置pod最晚的时间startingDeadlineSeconds，秒数内未开启认为则为failed，保证任务开始不能落后于预定的时间过太多