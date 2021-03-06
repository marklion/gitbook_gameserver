# 1.8 总结

## 1.8.1 项目开发流程

![](/assets/开发流程.png)

## 1.8.2 新技术

1. 开源框架的学习方法：看文档---》跑用例---》看源码
2. protobuf实现消息序列化
3. 网格法实现AOI
4. 随机数引擎std::default_random_engine

## 1.8.3 技术难点

1. TCP封包：缓存和滑窗
2. 构造测试用例和函数打桩（服务器架构测试）
3. protobuf嵌套消息的赋值：mutable子消息创建和挂接
4. 容器的使用（AOI网格实现，随机姓名池）
5. 判断容器包含关系（玩家跨网格视野处理）
6. 进程，文件描述符操作（守护进程，进程监控，日志记录）

## 1.8.4 设计思想

1. 业务分层（每条业务处理的流程都被分成四个部分，在四个类中分别实现）
2. 面向框架编程：重写虚函数而不是创建新函数
3. 抽象（用抽象类google::protobuf::Message代表多种具体的消息类）
4. 代理模式（GameMessage对象的编解码方法由其成员对象google::protobuf::Message决定）
5. 简单工厂（实际的google::protobuf::Message对象类型由构造参数_id决定）
6. 注册回调（注册IdProcTalkMsg和IdProcMoveMsg对象实现消息分类处理）