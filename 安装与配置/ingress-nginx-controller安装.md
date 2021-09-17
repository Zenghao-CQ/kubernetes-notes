[参考](https://www.cnblogs.com/v-fan/p/13252372.html)
#下载v0.42.0的deployment脚本
```
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.42.0/deploy/static/provider/baremetal/deploy.yaml
```
部署
```
kubectl apply -f deploy.yaml
```
部署失败，desctiber查看相关pod发现无法拉取google的镜像```k8s.gcr.io/ingress-nginx/controller:v0.42.0```
```
#方法一，dockerhub上拉取相关镜像
docker pull willdockerhub/ingress-nginx-controller:v0.42.0
修改tag
docker tag willdockerhub/ingress-nginx-controller:v0.42.0 k8s.gcr.io/ingress-nginx/controller:v0.42.0
docker images | grep ingress | awk  '{print $1,$2}' OFS=':' |xargs -I {} docker tag {} k8s.gcr.io/ingress-nginx/controller:v0.42.0@sha256:f7187418c647af4a0039938b0ab36c2322ac3662d16be69f9cc178bfd25f7eee 
#失败不能改digests，不加digests则之后为<node>还是要重新拉镜像

#方法二，外网拉取
#在vps上拉取
docker pull k8s.gcr.io/ingress-nginx #google的服务，在境外
#打包
docker save -o image.tar <image-id>
#在本地
scp root@<IP>:/root/image.tar /root/image.tar
docker load < image.tar 或者 docker load -i image.tar
#可能要修改tag，均失败
docker tag <image-id> k8s.gcr.io/ingress-nginx/controller:v0.42.0
#失败，一样没有digests

```
失败，**带digest的镜像不能改**，镜像地址是带 digest 引用的，直接用 Docker tag 其实是解决不了问题的，如下会报```refusing to create a tag with a digest reference```的错误

修改yaml文件中的image信息为
```
...
image:docker.io/willdockerhub/ingress-nginx-controller:v0.42.0
...
```
部署成功，在dashnoard上查看
```
kubectl get po -n ingress-nginx -o wide #前两个没起来
NAME                                        READY   STATUS      RESTARTS   AGE     IP            NODE    NOMINATED NODE   READINESS GATES
ingress-nginx-admission-create-tccfl        0/1     Completed   0          13m     10.244.1.40   node3   <none>           <none>
ingress-nginx-admission-patch-v48gc         0/1     Completed   0          13m     10.244.1.41   node3   <none>           <none>
ingress-nginx-controller-68c94c7b49-vnhlt   1/1     Running     0          2m51s   10.244.1.44   node3   <none>           <none>
```
删掉重试
```
kubectl delete po -n ingress-nginx ingress-nginx-admission-create-tccfl 
kubectl delete po -n ingress-nginx ingress-nginx-admission-patch-v48gc 
#没有自动重建看来没有RC或者RS
```
再次部署
```
kubecl apply -f ingress.yaml
```
还是不行，在node3查看kubelet日志
```
journalctl -u kubelet -n 1000 #没有明显报错只有一个不相关的问题
Sep 08 23:30:51 node3 kubelet[10143]: E0908 23:30:51.037924   10143 dns.go:135] Nameserver limits were exceeded, some nameservers have been omitted, the applied nameserver line is: 114.114.114.114 8.8.8.8 8.8.4.4
RunPodSandbox from runtime service failed: rpc error: code = Unknown desc = failed to start sandbox container for pod "ingress-nginx-admission-creat

#查看没有run的pod的日志
# 同docker logs 22e0e5d0689d
kubectl logs po -n ingress-nginx  ingress-nginx-admission-patch-ggrk9
W0909 02:39:25.831945       1 client_config.go:608] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
{"err":"secrets \"ingress-nginx-admission\" not found","level":"info","msg":"no secret found","source":"k8s/k8s.go:106","time":"2021-09-09T02:39:25Z"}
{"level":"info","msg":"creating new secret","source":"cmd/create.go:23","time":"2021-09-09T02:39:25Z"}
#直接docker 查看容器同理
kubectl logs  -n ingress-nginx  ingress-nginx-admission-create-b54dm 
docker logs dc95dac3a639
W0909 02:39:25.831945       1 client_config.go:608] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
{"err":"secrets \"ingress-nginx-admission\" not found","level":"info","msg":"no secret found","source":"k8s/k8s.go:106","time":"2021-09-09T02:39:25Z"}
{"level":"info","msg":"creating new secret","source":"cmd/create.go:23","time":"2021-09-09T02:39:25Z"}
```
发现两个跑起来的都是ingress-admission相关组件，报错也是缺少secret，回到[博客](https://www.cnblogs.com/v-fan/p/13252372.html)发现部署完本来就是这样，:)
```
#查看命名空间的内容
kubectl get all -n ingress-nginx 
NAME                                            READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create-b54dm        0/1     Completed   0          57m
pod/ingress-nginx-admission-patch-ggrk9         0/1     Completed   1          57m
pod/ingress-nginx-controller-68c94c7b49-6zxcb   1/1     Running     0          57m

NAME                                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             NodePort    10.104.96.227   <none>        80:30970/TCP,443:30395/TCP   57m
service/ingress-nginx-controller-admission   ClusterIP   10.96.63.0      <none>        443/TCP                      57m

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           57m

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-68c94c7b49   1         1         1       57m

NAME                                       COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   1/1           2s         57m
job.batch/ingress-nginx-admission-patch    1/1           4s         57m
```
可以看到ingress-nginx的svc的端口映射关系为：
80:30361/TCP,443:31087/TCP
后边的所有测试，需访问http则访问30361端口，访问https则访问31087端口
可以通过以下命令查看对象具体意义
```
sudo kubectl explain pods
sudo kubectl explain pods.spec
```
部署一遍kubia(不用nodePort部署svc)
kubectl get ingress
Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
NAME    CLASS    HOSTS           ADDRESS          PORTS   AGE
kubia   <none>   mycluster.com   192.168.106.13   80      6m48s
将ip写入host

