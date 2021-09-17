#### 开启dockers远程访问
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
在vs:code中点开设置，搜索```docker:host```，填入```ip:port```，如```sudo systemctl status docker.service```

#### 给普通用户权限
1.新增docker group（如果没有）：
```
sudo groupadd docker
```
2.将用户加入该 group 内。然后退出并重新登录使之生效。
```
sudo gpasswd -a ${USER} docker 
```
3.重启docker
```
sudo systemctl restart docker
```
5.切换当前会话到新 group 或者重启 X 会话(可能需要reboot)
```
newgrp  docker
pkill X #或者
```