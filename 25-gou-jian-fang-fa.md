# 2.5 构建方法

框架存在就是为了让开发者使用，一般提供给别人使用时的两种思路：

1. 源文件共享
2. 库+头文件共享

更推荐的是库+头文件共享。因为共享源文件有：编译过程复杂，可执行文件大等等缺点。

接下来研究怎样将zinx框架的代码编译成库（动态库）

**编写Makfile**

```makefile
libzinx.so:./channel/*.cpp ./message/*.cpp ./role/*.cpp ./protocol/*.cpp ./*.cpp
    g++ -std=c++11 -fPIC -shared $^ -o $@ -I ./include
```

> -fPIC -shared用于指定生成动态库，-I用于指定头文件引用路径

**执行构建和安装**

```bash
$ make
$ sudo cp libzinx.so /usr/lib/
$ sudo cp include/* /usr/include/zinx/
```