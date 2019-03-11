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

