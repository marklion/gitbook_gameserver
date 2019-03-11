# 3.3 Nginx+Fastcgi

接下来我们创建登陆服务器，用于管理游戏容器，向客户端返回游戏端口。

简单的工作流程：

![](/assets/登陆服务器流程图.png)

**特点：**

+ 效率要求低
+ 每个客户端的请求都是独立处理的

> 这其实是典型的BS架构的服务器需求

![](/assets/两种架构结构.png)

## 3.3.1 HTTP简介

##### 为什么要用HTTP？

服务器的开发不容易，尤其是开发高性能、稳定性好服务器，更加不容易，因此人们尝试更好简单的方式来开发软件。

在服务器方面，使用Web服务器，采用HTTP协议来代替底层的socket，是常见的选择。采用HTTP协议更加除了能得到稳定的服务器支持外，更加可以兼容各种客户端（手机、PC、浏览器）等等。这样实现了一个服务器之后，多个客户端可以通用。

![](/assets/HTTP通信过程.png)

##### HTTP是什么？

超文本传输协议(HTTP，HyperText Transfer Protocol)是互联网上应用最为广泛的一种网络协议，它详细规定了浏览器和万维网服务器之间互相通信的规则，通过因特网传送万维网文档的数据传送协议。

HTTP协议通常承载于TCP协议之上，有时也承载于TLS或SSL协议层之上，这个时候，就成了我们常说的HTTPS。如下图所示：

![](/assets/HTTP协议分层.png)

HTTP协议的特点：
+ 支持C/S架构
+ 简单快速：客户向服务器请求服务时，只需传送请求方法和路径，常用方法：GET、POST
+ 灵活：HTTP允许传输任意类型的数据对象
+ 无连接：限制每次连接只处理一个请求
+ 无状态：即如果后续处理需要前面的信息，它必须重传

##### 实例：

1. 打开Chrome浏览器，按下F12，弹出调试界面。
2. 访问`http://www.12371.cn/`（中共中央组织部官网）
3. 查看调试界面

![](/assets/一次http访问的过程.png)

#### 小结

HTTP是基于文本传输的通信协议。

+ 在请求报文中包含：

  - 请求的类型：GET或POST
  - 请求的资源路径：Request URL
  - 一些额外数据

+ 在回复报文中包含

  - 请求状态码：`Status Code: 200 OK`
  - 回复给浏览器的内容：html文件内容（静态网页），JS脚本（网页交互逻辑），媒体数据（图片等）
  - 相关的额外数据

自己写http服务器不是好的选择。

**BS架构下典型的服务器模型：Nginx+Fastcgi**

## 3.3.2 Nginx

#### 什么是Nginx

Nginx是一款轻量级的Web 服务器、反向代理服务器、电子邮件（IMAP/POP3）代理服务器，并在一个BSD-like 协议下发行。

由俄罗斯的程序设计师Igor Sysoev所开发，供俄国大型的入口网站及搜索引擎Rambler（俄文：Рамблер）使用。

其特点是占有内存少，并发能力强，事实上nginx的并发能力确实在同类型的网页服务器中表现较好，中国大陆使用nginx网站用户有：百度、京东、新浪、网易、腾讯、淘宝等。

#### Nginx优势
1)	更快
正常情况下单次请求得到更快的响应，高峰期(数以万计的并发时)Nginx可
以比其它web服务器更快的响应请求。

2)	高扩展性
低耦合设计的模块组成,丰富的第三方模块支持。

3)	高可靠性
经过大批网站检验，每个worker进程相对独立，master进程在一个worker 进程出错时，可以快速开启新的worker进程提供服务。

4)	低内存消耗
一般情况下，10000个非活跃的HTTP Keep-Alive连接在Nginx中仅消耗 2.5M内存，这是Nginx支持高并发的基础。

5)	单机支持10万以上的并发连接
取决于内存，10万远未封顶。

6)	热部署
master和worker的分离设计，可实现7x24小时不间断服务的前提下，升级Nginx可执行文件，当然也支持更新配置项和日志文件。

7)	最自由的BSD许可协议
BSD许可协议允许用户免费使用Nginx、修改Nginx源码，然后再发布。这吸引了无数的开发者继续为 Nginx贡献智慧。

#### Nginx安装

`sudo apt-get install nginx`

安装后的目录布局：

+ 可执行文件：`/usr/sbin/nginx`
+ 配置文件目录：`/etc/nginx/`
+ 网页范例目录：`/usr/share/nginx/`
+ 运行日志目录：`/var/log/nginx/`

一般地，Nginx要访问很多系统资源，所以，需要超级用户权限才能启动

测试：`sudo nginx -t`
启动：`sudo nginx`
停止: `sudo nginx -s stop`
重启：`sudo nginx -s reload`

#### Nginx配置方式

Nginx配置文件结构：

![](/assets/Nginx配置结构.png)

|配置层次|描述|
|-|-|
|main|Nginx在运行时与具体业务功能无关的参数，比如工作进程数、运行身份等|
|http|与提供http服务相关的参数，比如keepalive、gzip等|
|server|http服务上支持若干虚拟机，每个虚拟机一个对应的server配置项，配置项里包含该虚拟机相关的配置|
|location|http服务中，某些特定的URL对应的一系列配置项|
|mail|实现email相关的SMTP/IMAP/POP3代理时，共享的一些配置项|

配置文件的生效阶段：

![](/assets/Nginx工作流程.png)

**重点关注server配置项和location配置项**

通过不同的配置可以将nginx作为不同的角色使用。

##### 静态页面

修改`/usr/share/nginx/html/index.html`为：

```html
<!DOCTYPE html>
<html>
<head>
<title>this is itcast</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>this is itcast</h1>
</body>
</html>
```

修改`/etc/nginx/nginx.conf`为

```
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
	worker_connections 768;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    server {
        listen 80 default_server;
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
}
```

##### 反向代理

在nginx.conf的server配置段添加：

```
location /1/{
    proxy_pass http://www.baidu.com/;
}
location /2/{
    proxy_pass http://www.sina.com/;
}
```

![](/assets/反向代理示意图.png)

##### fastcgi

fastcgi是用来处理动态请求的。当用户请求匹配到配置fastcgi的location时，请求会发给fastcgi程序处理。

添加如下配置段，意味着用户的每条请求都会封装成fastcgi标准的格式转发给127.0.0.1这台主机（本机）的12345端口。

```
location /fcgi/{
    fastcgi_pass 127.0.0.1:12345;
}
```

服务器可以通过编写丰富的fastcgi程序和用户进行多种互动。

接下来我们看一下怎样使用fastcgi。

## 3.3.3 CGI和FAST-CGI

### CGI

**简介**

通用网关接口(Common Gateway Interface、CGI)描述了客户端和服务器程序之间传输数据
的一种标准，可以让一个客户端，从网页浏览器向执行在网络服务器上的程序请求数据。
CGI独立于任何语言的，CGI 程序可以用任何脚本语言或者是完全独立编程语言实现，只要这个语言可以在这个系统上运行。Unix shell script、Python、 Ruby、PHP、 perl、Tcl、 C/C++和 Visual Basic 都可以用来编写 CGI 程序。

最初，CGI 是在 1993 年由美国国家超级电脑应用中心(NCSA)为 NCSA HTTPd Web 服务
器开发的。这个 Web 服务器使用了 UNIX shell 环境变量来保存从 Web 服务器传递出去的参数，然后生成一个运行 CGI 的独立的进程。

**CGI处理流程**
 
1. web服务器收到客户端(浏览器)的请求Http Request，启动CGI程序，并通过环境变量、标准输入传递数据
2. CGI进程启动解析器、加载配置(如业务相关配置)、连接其它服务器(如数据库服务器)、逻辑处理等
3. CGI进程将处理结果通过标准输出、标准错误，传递给web服务器
4. web服务器收到CGI返回的结果，构建Http Response返回给客户端，并杀死CGI进程

![](/assets/cgi运行流程.png)

web服务器与CGI通过环境变量、标准输入、标准输出、标准错误互相传递数据。在遇到用户连接请求：

+ 先要创建CGI子进程，然后CGI子进程处理请求，处理完事退出这个子进程：fork-and-execute
+ CGI方式是客户端有多少个请求，就开辟多少个子进程，每个子进程都需要启动自己的解释器、加载配置，连接其他服务器等初始化工作，这是CGI进程性能低下的主要原因。当用户请求非常多的时候，会占用大量的内存、cpu等资源，造成性能低下。

CGI使外部程序与Web服务器之间交互成为可能。CGI程序运行在独立的进程中，并对每个Web请求建立一个进程，这种方法非常容易实现，但效率很差，难以扩展。面对大量请
求，进程的大量建立和消亡使操作系统性能大大下降。此外，由于地址空间无法共享，也限
制了资源重用。

**环境变量**
GET请求，它将数据打包放置在环境变量QUERY_STRING中，CGI从环境变量
QUERY_STRING中获取数据。

常见的环境变量如下表所示：

环境变量|含义
-|-
AUTH_TYPE|存取认证类型
CONTENT_LENGTH|由标准输入传递给CGI程序的数据长度，以bytes或字元数来计算
CONTENT_TYPE|请求的MIME类型
GATEWAY_INTERFACE|服务器的CGI版本编号
HTTP_ACCEPT|浏览器能直接接收的Content-types, 可以有HTTP Accept header定义
HTTP_USER_AGENT|递交表单的浏览器的名称、版本和其他平台性的附加信息
HTTP_REFERER|递交表单的文本的URL，不是所有的浏览器都发出这个信息，不要依赖它
PATH_INFO|传递给CGI程序的路径信息
QUERY_STRING|传递给CGI程序的请求参数，也就是用"?"隔开，添加在URL后面的字串
REMOTE_ADDR|client端的host名称
REMOTE_HOST|client端的IP位址
REMOTE_USER|client端送出来的使用者名称
REMOTE_METHOD|client端发出请求的方法(如get、post)
SCRIPT_NAME|CGI程序所在的虚拟路径，如/cgi-bin/echo
SERVER_NAME|server的host名称或IP地址
SERVER_PORT|收到request的server端口
SERVER_PROTOCOL|所使用的通讯协定和版本编号
SERVER_SOFTWARE|server程序的名称和版本

**标准输入**
环境变量的大小是有一定的限制的，当需要传送的数据量大时，储存环境变量的空间可能会
不足，造成数据接收不完全，甚至无法执行CGI程序。

因此后来又发展出另外一种方法：POST，也就是利用I/O重新导向的技巧，让CGI程序可以由stdin和stdout直接跟浏览器沟通。

当我们指定用这种方法传递请求的数据时，web服务器收到数据后会先放在一块输入缓冲区
中，并且将数据的大小记录在CONTENT_LENGTH这个环境变量，然后调用CGI程序并将
CGI程序的stdin指向这块缓冲区，于是我们就可以很顺利的通过stdin和环境变数CONTENT_LENGTH得到所有的信息，再没有信息大小的限制了。

**CGI程序结构**

![](/assets/CGI程序流程图.png)

### FastCGI

**什么是FastCGI**

快速通用网关接口(Fast Common Gateway Interface／FastCGI)是通用网关接口(CGI)的改进，描述了客户端和服务器程序之间传输数据的一种标准。

FastCGI致力于减少Web服务器与CGI程式之间互动的开销，从而使服务器可以同时处理更多的Web请求。与为每个请求创建一个新的进程不同，FastCGI使用持续的进程来处理一连串的请求。这些进程由FastCGI进程管理器管理，而不是web服务器。

**FastCGI处理流程**

1. Web 服务器启动时载入初始化FastCGI执行环境。 例如IIS、ISAPI、apache mod_fastcgi、nginx ngx_http_fastcgi_module、lighttpd mod_fastcgi。
2. FastCGI进程管理器自身初始化，启动多个CGI解释器进程并等待来自Web服务器的连接。启动FastCGI进程时，可以配置以ip和UNIX 域socket两种方式启动。
3. 当客户端请求到达Web 服务器时， Web 服务器将请求采用socket方式转发FastCGI主进程，FastCGI主进程选择并连接到一个CGI解释器。Web 服务器将CGI环境变量和标准输入发送到FastCGI子进程。
4. FastCGI子进程完成处理后将标准输出和错误信息从同一socket连接返回Web 服务器。当FastCGI子进程关闭连接时，请求便处理完成。
5. FastCGI子进程接着等待并处理来自Web 服务器的下一个连接。

![](/assets/fastcgi执行流程.png)

由于FastCGI程序并不需要不断的产生新进程，可以大大降低服务器的压力并且产生较高的应用效率。它的速度效率最少要比CGI 技术提高 5 倍以上。它还支持分布式的部署，即
FastCGI 程序可以在web 服务器以外的主机上执行。

CGI 是所谓的短生存期应用程序，FastCGI 是所谓的长生存期应用程序。FastCGI
像是一个常驻(long-live)型的CGI，它可以一直执行着，不会每次都要花费时间去fork一次(这是CGI最为人诟病的fork-and-execute 模式)。

**FastCGI程序结构**

![](/assets/FastCGI程序流程图.png)

**FCGI库和spawn-cgi**

上图可以看出，fast-cgi程序和cgi程序的相似度很大，但又不完全相同。fcgi库的出现统一了两者。

fcgi是开发FastCGI程序常用的一个函数库：https://github.com/FastCGI-Archives/fcgi2.git

+ fcgi库把socket数据收发和编结FastCGI数据封装成函数。方便开发者着眼于业务处理。
+ fcgi库在解析完FastCGI数据后会将模拟CGI的规范，设置环境变量和重定向标准输入。
+ 利用fcgi编写的程序也可以当做cgi程序运行。

这个fastcgi程序完成了一个返回客户端IP地址的功能。

```c
#include <stdlib.h>
#include <fcgi_stdio.h>

int main()
{
    while (FCGI_Accept() >= 0)
    {
        printf("Content-Type:text\r\n\r\n");
        printf("clint ip is %s\r\n", getenv("REMOTE_ADDR"));

    }

    return 0;
}
```

上边的代码中并没有体现守护进程和socket收发数据，所以我们需要借助一个fastcgi程序的管理器帮助。spawn-fcgi是一个通用的选择（apt下载安装）。

命令`spawn-fcgi -a 127.0.0.1 -p 7777 -f test-cgi`的意思是：按照守护模式启动test-cgi程序，并且监听本地地址（127.0.0.1）的7777端口。

## 3.3.4 组合使用

**需求：** 用户访问http://XXXXXXXXX/ip时，显示用户的IP

**步骤：**

1. 使用fcgi库编写FastCGI程序（上边的例子）。编译成可执行文件test-cgi
2. 执行`spawn-fcgi -a 127.0.0.1 -p 7777 -f test-cgi`启动FastCgi程序。
3. 在nginx配置文件中增加如下配置后重启nginx（nginx -s  reload）

```
location /cgi/ {
    include /etc/nginx/fastcgi_params;
    fastcgi_pass 127.0.0.1:7777;
}
```

## 3.3.5 开发技巧

+ 使用fcgi库时的三要素：

  - `while (FCGI_Accept() >= 0)`循环内写业务
  - 用`getenv`和`fread(buf, sizeof(buf), 1, stdin)`获取用户的请求
  - 用`printf`向用户展示数据；数据格式是
  
    * 若干行回复数据头（最简形式`Content-Type:text\r\n`）
    * 一个空行
    * 回复数据体

+ spawn-cgi启动fastcgi程序时要和nginx的fastcgi_pass配置项对应好
+ 良好的设计是：不同目的的请求用不同的FastCGI程序处理。


