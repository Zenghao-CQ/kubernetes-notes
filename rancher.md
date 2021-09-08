部署，需要开发80和433端口，可以```netstat -an | grep```查看
```
sudo docker run -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher:stable #用稳定版本实验
```
启动失败，查看日志
```
docker logs --tails=1000 <container-name>
[ERROR] Rancher must be ran with the --privileged flag when running outside of Kubernetes
```
Rancher 2.5.x 及之后的版本docker无法启动，加上 --privileged参数，见[链接](http://www.mayanpeng.cn/archives/94.html)
```
sudo docker run --name rancher -d --restart=unless-stopped --privileged -p 80:80 -p 443:443 rancher/rancher:stable #顺便always会重启容器，unless-stopped在daemon停止时不会重启
```
访问https:/<IP>:80，获取密码
```
docker logs  container-id  2>&1 | grep "Bootstrap Password:
.....4b4ddkk64mbvlhpl9twfzr674nvft5g8vx4qk8dwbt7f6f5g5q6h6l
```

根据将VMware中的3节点k8s 1.19.3加入rancher，选择insecure

加入后scheduler和control manager不健康
```
kubectl get cs #也可以看到探针有问题
NAME                 STATUS      MESSAGE                                                                                       ERROR
controller-manager   Unhealthy   Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection refused   
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused   
etcd-0               Healthy     {"health":"true"} 
```
解决：
找到主节点如下两个文件：
```
/etc/kubernetes/manifests/kube-controller-manager.yaml #kubeadm init的时候也是从这里部署
/etc/kubernetes/manifests/kube-scheduler.yaml
```
找到中 --port=0 去掉即可；
重启kubelet：
```
systemctl daemon-reload #kubeadm 把 kubelet 视为一个系统服务来管理
systemctl restart kubelet.service
```
另外，这是国外的一个网站的帖子部分内容：意思是从1.13版本开始，kube-controller-manager 和 kube-scheduler 使用使用10257、10259作为暴露的secure-ports（安全端口），而--insecure-ports（非安全端口）常用的10251、10252已经被废弃。

Since 1.13, kube-controller-manager and kube-scheduler exposing 10259, 10257 as a secure ports，Insecure ports 10251, 10252 has been deprecated. - #1327
You should use the secure ports as the default the livenessProbes going forward.
  --secure-port=10257

**注意**rancher的k3s运行在docker容器中所以对应nodeport，要sshdao容器内部curl localhost:port查看