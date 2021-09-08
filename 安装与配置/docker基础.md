## Docker 基础

1. #### 安装docker：
    ```shell
    sudo curl -sSL https://get.daocloud.io/docker | sh
    ```

2. #### busybox 运行
    ```shell
    sudo docker run busybox echo "Hello docker!"
    sudo docker run <image>:<tag> #默认latest
    ```

3. #### 用dockerfile构建image
    
    ```shell
    $ mkdir tmp
    $ cd tmp
    $ vi Dockerfile
    #写入
    FROM node:7 #起始镜像内容
    ADD app.js /app.js #将app.js加入根目录下保持app.js
    ENTRYPOINT ["node", "app.js"] #定义执行的命令

    $ sudo docker build -t kubia
    ```

4. #### 运行image
    ```shell
    docker run --name kubia-container -p 8080:8080 -d kubia
    ```
    指定容器名字与端口，容器与命令行分离(-d标志），这意味着在后台运行。本机上的8080端口会被映射到容器内的8080端口(-p8080:8080选项），所以可以通过http://localhost: 8080访间这个应用

5. #### 打标签并推送
    ```shell
    $ sudo docker tag <oldtag> <newtag>
    $ sudo docker tag kubia zneghaocq/kubia

    $ sudo docker push zenghaocq/kubia
    ```

6. #### 操作image
    ```shell
    #查看
    sudo docker image ls
    #删除,也可删除tag
    sudo docker image rm <imge>:tag
    ```

7. #### 查看容器
    ```shell
    sudo docker inspect <container-name>
    ```
8. ### ssh到容器
    ```shell
    sudo docker exec -it <container-name> bash
    ```
    -i，确保开放stdin，可以在shell输入命令
    -t，分离一个伪终端tty，可以看到输出

9. ### 查看日志
    ```
    docker log --tail=1000 <container-name>
    #从内部
    sudo docker exec -it <container-name> bash
    cat /var/lib/docker/containers/<container-id>/<container-id>-json.log
    ```