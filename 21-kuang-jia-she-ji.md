# 2.1 框架设计

设计zinx框架用于处理IO并发。

## 2.1.1 用例拆解

从简单场景入手，考虑易变化的部分。

#### 简单场景

键盘输入字符，控制台直接输出该字符

![](/assets/简单场景.png)

#### 功能多样化

若输入小写字母，则输出大写字母，若输入大写字母则原样输出

![](/assets/功能多样化.png)

#### 数据多样化

输入和输出均要字母的ascii码

![](/assets/数据多样化.png)

#### 通道多样化

UDP输入数据，输出到FIFO文件

![](/assets/通道多样化.png)

#### 抽象用例

+ 整体流程固定化（实体类--》非虚函数）
+ 子流程多样化（抽象类--》虚函数）

![](/assets/抽象用例.png)


## 2.1.2 类设计

将抽象用例中的类进行详细的设计，类图：

![](/assets/类图.png)

+ A开头的类均为抽象类，只需定义成员函数原型。根据业务不同，由不同的派生类实现成员函数。
+ server类用于组织和调度四个类的对象们。

## 2.1.3 并发模型

并发处理的常见思路：

+ 多进程/多线程
+ 非阻塞轮询
+ IO多路复用

基于之前设计的类模型，我们可以设计server类来实现这三种并发模型。

+ 多进程/多线程：

  - install每个Channel对象时，为其绑定一个进程或线程
  - uninstall某个Channel对象时，server要回收其线程或进程资源
  - 进程或线程内循环执行阻塞读取数据，读取到数据后顺序执行protocol对象的解析，role对象的处理等操作

+ 非阻塞轮询

  - Channel对象install到server时要修改成员fd的属性为非阻塞
  - server对象维护Channel对象容器，循环调用每个Channel对象的读取函数，若读到数据则执行后续操作

+ IO多路复用

  - server对象维护一个epoll实例和Channel对象容器。
  - install和uninstall本质上就是将Channel对象的fd从epoll实例中添加或摘除。
  - 某个fd有数据后，调用对应的处理流程。

> 三种模型怎么取舍？
