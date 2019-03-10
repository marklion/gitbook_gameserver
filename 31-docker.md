# 3.2 docker

docker是一个实现容器技术的软件，用到了linux内核的命名空间原理。

### 3.2.1. 初识docker

##### 安装

```bash
# 执行脚本简易安装
$ sudo apt-get install curl
$ curl -sSL https://get.daocloud.io/docker | sh
# 修改添加当前用户到docker用户组,修改socket权限
$ sudo usermod -aG docker dev
$ sudo chmod 777 /var/run/docker.sock
# 退出终端后再次进入测试命令
$ docker --hep
```

##### 第一次运行

```bash
$ docker run hello-world
```

**运行详解：**

1. The Docker client contacted the Docker daemon.
2. The Docker daemon pulled the "hello-world" image from the Docker Hub (amd64)
3. The Docker daemon created a new container from that image which runs the executable that produces the output you are currently reading.
4. The Docker daemon streamed that output to the Docker client, which sent it to your terminal.
 
1. 命令行连接到守护进程
2. 守护进程发现当前没有hello-world镜像，于是去dockerhub下载了一个镜像
3. 守护进程基于hello-world镜像创建了一个容器，容器内有一个可执行程序，现在的内容都是由该程序输出的。
4. 守护进程将容器的输出发送给命令行，也就是当前终端。


### 3.2.2. docker详解

##### 程序架构

docker是CS架构的软件，命令行敲的命令会发送到一个守护进程docker Daemon执行。一般地，命令行和守护进程在同一个计算机运行。容器，镜像的管理由docker Daemon执行，命令行无需关心。

##### 核心概念

docker有三个核心概念，镜像，容器和仓库。

###### 1. 仓库

类似github，docker官方设定了一个docker镜像的仓库：dockerhub（https://hub.docker.com/）

+ 本地计算机可以拉去dockerhub上的镜像

```bash
# 完整的docker镜像名称是 作者/镜像名:标签
$ docker pull ubuntu/ubuntu:latest
```

+ 本地计算机的镜像可以推送到dockerhub的账户内

```bash
# 登陆，按照提示输入github的用户名密码
$ docker login
# 将本地镜像重命名成规范名称
$ docker tag ubuntu marklion/ubuntu:myfirsttag
# 推送自己的镜像
$ docker push marklion/ubuntu:myfirsttag
```

+ 镜像的修改，提交等操作很类似git和github的操作。

###### 2. 镜像

+ **概念：** 一组环境的静态集合，类似操作系统镜像。
+ **特点：** docker镜像有分层依赖的关系。创建镜像的过程就好像写代码，从简单到复杂的过程。

![](/assets/镜像分层.png)

+ **运行：** 镜像运行后会产生容器。基于一个镜像可以运行多个容器。

```bash
# 查看当前所有的镜像
$ docker images
# 运行ubuntu镜像：在ubuntu容器中执行一条ls的命令，不写命令则运行bash
$ docker run --rm -ti ubuntu ls
# --rm -ti参数：运行结束后删除容器，提供虚拟终端和交互式界面
```

+ **创建：** 类似基于原始系统搭环境

  - 手动创建

    1. 下载并运行基础镜像
    2. 进入基础镜像的容器内安装所需环境
    3. 将容器提交为镜像

    ```bash
    # 直接执行ifconfig，报错，因为基础镜像没有安装ifconfig包
    $ docker run --rm ubuntu ifconfig
docker: Error response from daemon: OCI runtime create failed: container_linux.go:344: starting container process caused "exec: \"ifconfig\": executable file not found in $PATH": unknown.
ERRO[0000] error waiting for container: context canceled 
    # 进入基础镜像，安装工具包后退出
    $ docker run -ti ubuntu
    # apt-get update
    # apt-get install -y net-tools
    # exit
    # 找到刚才的容器，基于其创建镜像
    $ docker ps -a
    CONTAINER ID        IMAGE               COMMAND             CREATED          STATUS                      PORTS               NAMES
    034abada670c        ubuntu              "/bin/bash"         31 minutes ago      Exited (0) 20 seconds ago                       zealous_swirles
    # commit命令用于容器---》镜像
    # 容器ID可以用简写
    $ docker commit 034a my_unbuntu:add_net
    $ docker images
    REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
    my_unbuntu          add_net             6ca02b1d0483        5 seconds ago       114MB
    # 用新镜像运行ifconfig
    $ docker run --rm my_unbuntu:add_net ifconfig
    eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ac:11:00:02  txqueuelen 0  (Ethernet)
        RX packets 2  bytes 200 (200.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

    lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
    ```

  - 脚本创建
  
    1. 编写Dockerfile
    2. 运行docker编译命令
    
    ```bash
    # Dockerfile中定义基础镜像，和要运行的安装命令
    $ cat Dockerfile
    FROM ubuntu
    RUN apt-get update
    RUN apt-get install -y net-tools
    # 编译镜像，指定镜像名是df_unbutu:add_net，指定Dockerfile所在目录是当前目录
    $ docker build -t df_unbutu:add_net .
    ```
    
  **小结：** 手动创建方便操作，脚本创建方便分享。
  
###### 3. 容器

+ **概念：** 运行中的一组环境。基于某个镜像创建。
+ **特点：** 容器中要运行程序，最好只运行一个程序。容器的运行不会影响镜像内容。
+ **运行：** 

  - 支持以后台模式运行进程
  - 支持将容器内的端口映射到宿主机
  - 支持以挂载的形式和宿主机共享文件系统
  - 容器退出后可以再次打开

这是一个tcp回传服务器。

  ```c
  #include <stdio.h>
  #include <stdlib.h>
  #include <unistd.h>
  #include <sys/types.h>
  #include <sys/socket.h>
  #include <string.h>
  #include <errno.h>
  #include <netinet/in.h>
  #include <arpa/inet.h>

    int main()
    {
        int listen_sock = socket(AF_INET, SOCK_STREAM, 0);
        if (0 <= listen_sock)
        {
            struct sockaddr_in stServerAddr;
            stServerAddr.sin_family = AF_INET;
            stServerAddr.sin_addr.s_addr = htonl(INADDR_ANY);
            stServerAddr.sin_port = htons(55555);
            if ((0 == bind(listen_sock, (struct sockaddr *)&stServerAddr, sizeof(stServerAddr))) &&
                (0 == listen(listen_sock, 10)))
            {
                struct sockaddr_in stClientAddr;
                socklen_t AddrLen = sizeof(stClientAddr);
                int data_sock = -1;
                while (0 <= (data_sock = accept(listen_sock, (struct sockaddr *)&stClientAddr, &AddrLen)))
                {
                    char szBuff[256];
                    int recv_len = 0;
                    while (0 < (recv_len = recv(data_sock, szBuff, sizeof(szBuff), 0)))
                    {
                        send(data_sock, szBuff, recv_len, 0);
                    }
                    close(data_sock);
                }
            }
            else
            {
                perror("bind:");
            }
        }
        else
        {
            perror("create listen socket:");
        }
        return -1;
    }
  ```

使用ubuntu镜像运行该程序：

```bash
$ docker
````