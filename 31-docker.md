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
2. The Docker daemon pulled the "hello-world" image from the Docker Hub \(amd64\)
3. The Docker daemon created a new container from that image which runs the executable that produces the output you are currently reading.
4. The Docker daemon streamed that output to the Docker client, which sent it to your terminal.

5. 命令行连接到守护进程

6. 守护进程发现当前没有hello-world镜像，于是去dockerhub下载了一个镜像
7. 守护进程基于hello-world镜像创建了一个容器，容器内有一个可执行程序，现在的内容都是由该程序输出的。
8. 守护进程将容器的输出发送给命令行，也就是当前终端。

### 3.2.2. docker详解

##### 程序架构

docker是CS架构的软件，命令行敲的命令会发送到一个守护进程docker Daemon执行。一般地，命令行和守护进程在同一个计算机运行。容器，镜像的管理由docker Daemon执行，命令行无需关心。

##### 核心概念

docker有三个核心概念，镜像，容器和仓库。

###### 1. 仓库

类似github，docker官方设定了一个docker镜像的仓库：dockerhub（[https://hub.docker.com/）](https://hub.docker.com/）)

* 本地计算机可以拉去dockerhub上的镜像

```bash
# 完整的docker镜像名称是 作者/镜像名:标签
$ docker pull ubuntu/ubuntu:latest
```

* 本地计算机的镜像可以推送到dockerhub的账户内

```bash
# 登陆，按照提示输入github的用户名密码
$ docker login
# 将本地镜像重命名成规范名称
$ docker tag ubuntu marklion/ubuntu:myfirsttag
# 推送自己的镜像
$ docker push marklion/ubuntu:myfirsttag
```

* 镜像的修改，提交等操作很类似git和github的操作。

###### 2. 镜像

* **概念：** 一组环境的静态集合，类似操作系统镜像。
* **特点：** docker镜像有分层依赖的关系。创建镜像的过程就好像写代码，从简单到复杂的过程。

![](/assets/镜像分层.png)

* **运行：** 镜像运行后会产生容器。基于一个镜像可以运行多个容器。

```bash
# 查看当前所有的镜像
$ docker images
# 运行ubuntu镜像：在ubuntu容器中执行一条ls的命令，不写命令则运行bash
$ docker run --rm -ti ubuntu ls
# --rm -ti参数：运行结束后删除容器，提供虚拟终端和交互式界面
```

* **创建：** 类似基于原始系统搭环境

  * 手动创建

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

  * 脚本创建

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

* **概念：** 运行中的一组环境。基于某个镜像创建。
* **特点：** 容器中要运行程序，最好只运行一个程序。容器的运行不会影响镜像内容。
* **运行：**

  * 支持以后台模式运行进程
  * 支持将容器内的端口映射到宿主机
  * 支持以挂载的形式和宿主机共享文件系统
  * 容器运行的程序退出后，容器随之退出；容器退出后可以再次打开

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
# 启动容器，
# -d 以守护进程启动容器
# -v /home/dev/:/root/host/ 将本机的/home/dev/目录挂载到容器内的/root/host目录下
# -p 22334:55555 将本机的22334端口映射到容器内的55555端口
# /root/host/tcp_echo，可执行程序的路径
$ docker run -d -v /home/dev/:/root/host/ -p 22334:55555 ubuntu /root/host/tcp_echo
```

##### 使用约束

* docker只能安装在64位linux系统上（或专业版win10）
* docker官方推荐，在一个容器中最好只运行一个进程
* 镜像和容器会占用磁盘空间。好习惯：删除不用的容器和镜像

### 2.3.3 docker和虚拟机的对比

相似处：

* 都通过镜像包装
* 都可以起到隔离进程运行环境的作用

**区别：**

|  | docker | 虚拟机 |
| --- | --- | --- |
| 速度 | 快（基于当前系统创建不同的运行上下文） | 慢（启动操作系统） |
| 体量 | 小（镜像文件可以自由定制裁剪） | 大（依赖厂商决定） |
| 分发 | 容易（dockerhub，Dockerfile） | 困难（一般由厂商分发） |
| 复杂度 | 简单（对于操作系统而言只是一个程序） | 复杂（需要考虑资源分配，与宿主机的通信等问题） |
| 独立性 | 较好（容器只能基于端口独立） | 非常好（跟真实主机几乎没有区别） |

### 2.3.4 常用docker命令

| 类型 | 命令 | 描述 |
| --- | --- | --- |
| 镜像操作 | docker images | 显示存在的当前镜像 |
|  | docker image prune | 删除无用的镜像（被更新的旧镜像） |
|  | docker rmi _镜像ID_ | 删除指定的镜像 |
|  | docker build -t _镜像名称：tag dockerfile所在路径_ | 编译镜像 |
|  | docker login | 登陆dockerhub |
|  | docker pull _镜像作者/镜像名称：tag_ | 从dockerhub拉取镜像 |
|  | docker tag _镜像名称：tag_ _镜像作者/新名称：tag_ | 规范重命名镜像 |
|  | docker push _镜像作者/镜像名称：tag_ | 推送镜像到dockerhub |
| 容器操作 | docker ps -a | 显示当前所有容器 |
|  | docker rm _容器ID_ | 删除指定容器，运行中容器不能删 |
|  | docker start -ai _容器ID_ | 启动之前退出的容器 |
|  | docker stop _容器ID_ | 停止指定容器 |
|  | docker cp _宿主机文件绝对路径_ _容器ID：容器内绝对路径_ | 从宿主机拷贝文件到容器内 |
|  | docker cp _容器ID：容器内绝对路径_ _宿主机文件绝对路径_ | 从容器内拷贝文件到宿主机 |
| run命令 | docker run _参数_ _镜像名_ _执行程序名_ | 创建并运行容器 |
|  | -d | 守护模式运行，适用服务，与ti参数互斥 |
|  | -ti | 打开终端交换模式，适用应用程序，与-d互斥 |
|  | -v _主机绝对路径_:_容器内绝对路径_ | 将宿主机路径挂载到容器内 |
|  | -p _主机端口_:_容器内端口_ | 将容器内端口映射到宿主机端口 |
|  | -e _环境变量名=环境变量值_ | 向容器内定义环境变量 |
|  | --rm | 容器退出后自动删除，适用纯应用程序 |

更多：`man docker,man docker run .....`

### 2.3.5 Dockerfile编写

##### 编写思路

1. 确定基础镜像
2. 安装所需环境
3. 定义执行点

```
FROM ubuntu
RUN apt-get update
RUN apt-get install -y gcc
ENTRYPOINT ["gcc"]
```



