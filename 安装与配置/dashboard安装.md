1.在github dashboard主页找到与k8s对应的版本
```
https://github.com/kubernetes/dashboard/releases
```
2.下载对应版本yaml文件
```
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.5/aio/deploy/recommended.yaml
```
3.修改service为NodePort，方便访问
```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort #新加
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 32000 #新加
  selector:
    k8s-app: kubernetes-dashboard
```
如果无法获取镜像，可以手动下载镜像和用```docker load```修改deployment的pullpolicy

安装
```
kubectl apply -f recommand.yaml
```
创建用户
```
kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard 
kubectl create clusterrolebinding dashboard-admin-rb --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin
#获取token
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```
