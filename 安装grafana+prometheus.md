[参考](https://www.freesion.com/article/6216564558/)
下载项目:
```shell
git clone https://github.com/coreos/kube-prometheus.git
```
修改yaml文件，增加nodePort
```shell
cd ./mainifests
vi grafana-service.yaml 
vi prometheus-service.yaml 
vi alertmanager-service.yaml 
#增加type为NodePort，并设置nodePort分别为30100，30200，30300
```
apply安装
```shell
kubectl apply -f manifests/setup #先部署metrix
kubectl apply -f manifests/ #安装有竞争关系，如果资源不在可以多次执行
```
对缺少的镜像手动拉取改tag
```
docker pull willdockerhub/prometheus-adapter:v0.9.0
docker tag willdockerhub/prometheus-adapter:v0.9.0 k8s.gcr.io/prometheus-adapter/prometheus-adapter:v0.9.0
docker pull bitnami/kube-state-metrics:2.2.0
docker tag bitnami/kube-state-metrics:2.2.0 k8s.gcr.io/kube-state-metrics/kube-state-metrics:v2.2.0
docker rmi bitnami/kube-state-metrics:2.2.0 
```
**有时候会找不到CRD**因为文件太多，带来竞争问题，可以多次安装