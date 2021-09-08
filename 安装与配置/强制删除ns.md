[参考1](https://blog.csdn.net/qq_31152023/article/details/107056559)
[参考2](https://zhuanlan.zhihu.com/p/364431430?utm_source=wechat_session&utm_medium=social&utm_oi=633239230969810944&utm_campaign=shareopn)
[参考3](https://success.docker.com/article/kubernetes-namespace-stuck-in-terminating)
```shell
# 尝试删除
kubectl delete ns <ns-name>
kubectl  delete ns <ns-name> --force --grace-period=0
```
删除失败处于terminating状态
```shell
kubectl get all -n <ns-name> #为空
```
使用apiserver强行删除
1. 导出json文件
```shell
kubectl get namespace <ns-name> -o json > ymp.json
```
在```tmp.json```中删除字段```"finalizers": [...]```包含的内容为```"finalizers": []```。
2. 使用kubectl代理
```
kubectl proxy --port=8080
```
向apiserver直接删除
```
curl -k -H "Content-Type: application/json" -X PUT --data-binary @tmp.json http://127.0.0.1:8080/api/v1/namespaces/cattle-system/finalize
```