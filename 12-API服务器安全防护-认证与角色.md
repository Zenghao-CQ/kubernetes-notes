## API服务器安全防护

#### 1. 认证机制
* API服务器通过**认证插件**的列表提取请求中的用户名、IP、组信息以确定请求的发送者，可用的方法有：证书、HTTP验证、HTTP头中发token验证等
* 用户和组：认证插件返回用户的用户名和组的信息，只用于验证，**不会被存储**
  * 客户端可以分为两种
    * 真实的人（用户），在外部系统如单点登陆系统（SSO）管理，不能通过API服务器进行管理
    * pod（运行在pod中的应用），用service account资源进行管理
  * 组：插件返回的组仅仅是组名字符串，系统内置的组有：
    * system:unauthenticated 所有认证插件都不会通过
    * system:authenticated 自动分配通过认证的用户
    * system:serviceaccounts 全部serviceAccount
    * system:serviceaccounts:<namespace> 特点命名空间中的serviceAccount
* serviceAccount介绍
  * pod通过```/var/run/secrets/kubernetes.io/serviceaccount/token```文件进行身份认证。pod与serviceAccount相关联，通过卷挂载持有认证token。应用程序使用token连接API服务器，API服务器通过token认证serviceAccount进行身份认证，将如下用户名传入**授权插件**：
    ```
    system:serviceaccount:<namespace>:<service account name>
    ```
  * serviceAccount资源，目前只有```default```的sa，注意一个pod可以包含多个sa，**pod和sa必须在命名空间**。
    ```
    sudo kubectl get sa
    ```
    API服务器用管理员配置好的系统级别认证插件，验证发送token的pod所关联的sa是否有执行请求操作的权限
  * 创建serviceAccount
    ```shell
    $ sudo kubectl create serviceaccount foo
    $ sudo kubectl describe serviceaccount foo
    Name:                foo
    Namespace:           default
    Labels:              <none>
    Annotations:         <none>
    Image pull secrets:  <none> #会自动添加到pod
    Mountable secrets:   foo-token-nxrzd #若强制使用可挂载密钥则pod只能用这些
    Tokens:              foo-token-nxrzd
    Events:              <none>
    ```
    sa自动创建了secret```foo-token-nxrzd```（为JWT认证token）,secret有**ca.crt、namespace、token**三个条目    
    * 可挂载密钥列表：默认情况pod可以挂载任何secret，可以在sa中开启强制挂载：在注解annotation中包含```kubernetes.io/enforce-mountable-secrets='true'```，这样挂载该sa的pod只能挂载列表中的secret
    * 从镜像拉取密钥，密钥secret可以从docker私有仓库中拉取，sa包含secret后，secret会被自动添加到引用sa的pod中
      ```shell
      # 第七章手动创建的镜像拉取secret
      sudo kubectl create secret docker-registry my-dockerhub-secret \
      --docker-username=myusername \
      --docker-password=mypassword \
      --docker-email=my.email@provider.com
      ```
      ```yaml
      apiVersaion: v1
      kind: SecretAccount
      metadata:
        name: my-sa
      imagePullSecret:
      - name: my-dockerhub-secret #包含secret
      ```
  * 将sa分配给pod
    只能在pod**创建**时，在```spec.serviceAccountName```字段进行配置。例子，修改第8章ambassador代理的pod，使其包含foo
      ```yaml
      #./curl-custom-sa.yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: curl-custom-sa
      spec:
      serviceAccountName: foo
        containers:
        - name: main
          iamge: curlimages/curl
          command: ["sleep","9999999"]
        - name: ambassador
          image: luksa/kubectl-proxy:1.6.2
      ```
      查看token与foo-token-nxrzd一致
      ```shell
      sudo kubectl exec -it curl-custom-sa -c main cat \
      /var/run/secrets/kubernetes.io/serviceaccount/token
      ```
      尝试与API服务器通信，注意该pod开启了代理
      ```shell
      $ kubectl exec -it curl-custom-sa -c main curl localhost:8001/api/v1/pods
      ```

#### 2. RBAC 基于角色的权限控制
SA的用处在于，限制可挂载密钥，提供镜像拉取密钥，现在引入RBAC授权插件
* 动作：API服务器暴露RESTFUL接口。客户端可以发送GET、POST、PUT、DELETE请求到特定URL上，URL对应pod、service等资源，RBAC**授权插件**判断是否允许在资源上执行动作，额外的动词use用于PodSeurityPolicy

  |HTTP方法|单一资源动作|资源集合动作|
  |---|---|---|
  |GET, HEAD|get（以及watch）用于监听|list（以及watch）|
  |POST|create|n/a|
  |PUT|update|n/a|
  |PATCH|patch|n/a|
  |DELETE|delete|deletecollection|

RBAC规则可以对应一类资源，也可以对应某类实例，甚至非资源url(ru、healthz或者/api本身)。主体(一个人、一个sa、一组用户或sa)可以对应一个或多个**角色**，角色有各种动作的权限。
* RBAC资源四种资源分为两组：
  * Role、ClusterRole，角色和集群角色，指定在资源上可以执行哪些动作（可以做什么）
  * RoleBinding、ClusterRoleBinding，绑定和集群绑定，将角色绑定到特定用户、组或ServiceAccount（谁做）
  * 角色和角色绑定是单个**命名空间**中的资源；集群角色和集群角色绑定的是**集群级别的资源**，角色绑定也可以引用集群角色
  ![](./pictures/rbac-model.png)
* RBAC使用
  * 在GKE中，创建集群时使用```--no-enable-legacy-authorization```禁用老版本授权，在minikube可能需要```--extra-config=apiserver.Authorization.Mode=RBAC```
  * 恢复RBAC功能：在第八章中已经设置为赋予了全部权限，如下
    ```shell
    $ sudo kubectl create clusterrolebinding permissive-binding \
    --clusterrole=cluster-admin \
    --group=system:serviceaccounts
    ```
    这里可以先删除该clusterrolebinding
    ```shell
    $ sudo kubectl clusterrolebinding permissive-binding
    ```
  * 创建命名空间和pod，用之前的ambassador镜像
    ```shell
    $ sudo kubectl create ns foo
    $ sudo kubectl create ns bar
    $ sudo kubectl run test --image=luksa/kuberctl-proxy:1.6.2 -n foo
    $ sudo kubectl run test --image=luksa/kuberctl-proxy:1.6.2 -n bar
    ```
  * 尝试在pod中访问API server以查看全部服务，由于此时删除了clusterrolebinding，自然没有权限
    ```shell
    $ sudo kubectl exec -it test -n foo sh
    / $ curl localhost:8001/api/vl/namespaces/foo/services #此时打开了ambassador容器
    # 返回403
    ...
    "message": "services is forbidden: User \"system:serviceaccount:foo:default\" cannot list resource \"services\" in API group \"\" in the namespace \"foo\
    ...
    ```
    可以看到API服务器用foo空间默认的```system:serviceaccount:foo:default```sa进行认证，该sa没有权限
  * 使用Role和RoleBinding
    Role资源定义了哪些操作（HTTP请求）可以在哪些资源上执行，下面的Role允许用户get并list全部svc，资源名必须使用**复数**
    ```yaml
    ./service-reader.yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      namespace: foo #指定命名空间
      name: service-reader
    rules: 
    - apiGroups: [""] #svc是核心apiGroip资源，没有apiGroup名
      verbs: ["get", "list"] #
      resources: ["services"] #使用复数
    ```