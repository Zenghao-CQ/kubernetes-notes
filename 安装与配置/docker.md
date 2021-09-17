开启dockers远程访问
```
#修改service
sudo vi /lib/systemd/system/docker.service
#添加启动参数
#原来
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
#改后
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H fd:// --containerd=/run/containerd/containerd.sock

#重启守护进程
sudo systemctl daemon-reload
#重启docker
sudo service docker restart

#查看是否开放
sudo systemctl status docker.service
```
在vs:code中点开设置，```搜索docker:host```，填入```ip:port```，如```sudo systemctl status docker.service```