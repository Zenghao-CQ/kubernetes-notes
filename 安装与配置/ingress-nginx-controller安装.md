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
```
失败，**带digest的镜像不能改**，镜像地址是带 digest 引用的，直接用 Docker tag 其实是解决不了问题的，如下会报refusing to create a tag with a digest reference的错误

修改yaml文件中的image信息为
```
...
    image:docker.io/willdockerhub/ingress-nginx-controller:v0.42.0
...
```