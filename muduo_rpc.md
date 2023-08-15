## 1.集群和分布式理解

### 1.单机服务器的缺点

- 受限于硬件资源，聊天服务器所能承受的用户的并发量
  单机聊天服务器
- 任意模块的修改，都会导致整个项目代码重新编译、部署
- 系统中，有些模块是属于CPU密集型的，有些模块是属于I/O密集型的，造成各模块对于硬件资源的需求是不一样的

![image-20230814134609439](C:\Users\86187\AppData\Roaming\Typora\typora-user-images\image-20230814134609439.png)

###2.集群服务器

优点:并发量提升

缺点:

- 部署麻烦
- 功能冗余(有些模块不需要每个服务器都部署)

### 3.分布式

服务器的模块**按需拆分**部署到不同服务器上;分布式和集群共同作用

问题:既然不同模块部署在不同服务器,那如何相互调用呢 - -rpc

![image-20230814135450052](C:\Users\86187\AppData\Roaming\Typora\typora-user-images\image-20230814135450052.png)



## 2.RPC通信原理

![image-20230814140506622](C:\Users\86187\AppData\Roaming\Typora\typora-user-images\image-20230814140506622.png)

## 3.protobuf

优点:

- 二进制存储,xml和json都是文本存储,内存利用率更高
- 不许存储额外信息,xml和json基本都是key value存储
- 对于rpc调用,提供的功能更全面

命令:

```bash
protoc --version    //查看版本
protoc -I=$SRC_DIR --cpp_out=$DST_DIR $SRC_DIR/xxx.proto  #protoc test.proto --cpp_out=./
# $SRC_DIR: .proto 所在的源目录
# --cpp_out: 生成 c++ 代码
# $DST_DIR: 生成代码的目标目录
# xxx.proto: 要针对哪个 proto 文件生成接口代码
option cc_generic_services = true; #表示生成service服务类和rpc方法描述，默认不生成

```

```c++
google::protobuf::Service *service
// 获取了服务对象的描述信息
const google::protobuf::ServiceDescriptor *pserviceDesc = service->GetDescriptor();
// 获取服务的名字
std::string service_name = pserviceDesc->name();(类名)
// 获取服务对象service的方法的数量
int methodCnt = pserviceDesc->method_count();
// 获取了服务对象指定下标的服务方法的描述（抽象描述） UserService   Login
const google::protobuf::MethodDescriptor* pmethodDesc = pserviceDesc->method(i);
std::string method_name = pmethodDesc->name();
// 生成rpc方法调用的请求request和响应response参数  获取方法对应的返回参数类型
google::protobuf::Message *request = service->GetRequestPrototype(method).New();
// 在框架上根据远端rpc请求，调用当前rpc节点上发布的方法
// new UserService().Login(controller, request, response, done)
service->CallMethod(method, nullptr, request, response, done);
```



- message 继承子Message ,会生成对应的同名类以及提供对属性的操作方法

![image-20230814144828950](C:\Users\86187\AppData\Roaming\Typora\typora-user-images\image-20230814144828950.png)

- service 继承于 Service类,会生成对应的同名类和同名_Stub类,其中Stub用于发起调用,同名类用于提供服务,对应于rpc工作原理图的callee端

![image-20230814144952061](C:\Users\86187\AppData\Roaming\Typora\typora-user-images\image-20230814144952061.png![image-20230814145412164](C:\Users\86187\AppData\Roaming\Typora\typora-user-images\image-20230814145412164.png)

## 4.RPC框架

1.RPC服务提供方(用户自己编写代码)使用RPC框架分三步

- 初始化框架(加载配置文件:配置文件内容包括:ip,port等)
- 注册Rpc服务(自己写的供消费者调用的服务)
- 启动服务(等待消费者调用)

## 5.服务提供类provider

功能:

- 注册服务NotifyService(google::protobuf::Service *service);service是服务类的基类,这里用到了多态
- 启动服务(run)
- 接受服务调用(涉及到多线程,网络通信)

### run函数,启动服务

需要做的事情:

- 通过初始化获取配置文件的ip和port
- 启动muduo的TcpServer
- 注册onConnection和onMessage回调函数,设置线程数量
- 调用TcpServer的start

### 注册服务

服务提供者用我们的rpc框架,它需要做的只是初始化,注册服务,启动(然后等待消费者端调用),其他部份需要由框架做,包括

- 获取服务类的描述符,根据描述符获取服务类名和方法数目
- 遍历获取方法描述符,根据方法描述符获取方法名和方法

- 把注册的服务保存起来(使用map表和定义结构体来记录已经注册的可以调用的方法)

### provider中muduo需要做的事情

服务提供者用我们的rpc框架,它需要做的只是初始化,注册服务,启动(然后等待消费者端调用),其他部份需要由框架做,包括

- 主要在onMessage回调函数中实现
- 和客户端协商好通讯协议形式(定义一个proto文件):header_size(4个字节) + header_str + args_str;header_size代表 header_str的长度,header_str包含service_name method_name args,args为后面args_str方法参数的长度,防止粘包

- 网络接受消费者端发来的信息,反序列化成request,并且解析信息,获取想调用的函数,并调用
- 函数执行完毕后,会返回结果,网络需要将结果序列化,并返回

![image-20230814154734826](C:\Users\86187\AppData\Roaming\Typora\typora-user-images\image-20230814154734826.png)

##6.rpc框架客户端逻辑

stub类初始化时需要传入google::protobuf::RpcChannel,会调用里面重写的CallMethod,所以客户端主要重写一个channel类继承

google::protobuf::RpcChannel,重写CallMethod方法,该方法的执行流程:

- 获取方法对应的服务类名和方法名,同时将request序列化(这作为方法参数),并获取它的大小;用于初始化通讯协议
- 将协议protobuf类序列化为字符串
- socket连接服务端,连接成功后发送字符串
- 因为是同步的,所以等待接收返回值,并反序列化返回值response->ParseFromArray(recv_buf, recv_size)
- 关闭socketfd

## 7.rpc使用客户端逻辑

- 初始化(加载配置文件)
- 使用对应服务的stub类调用对应的方法,同时初始化参数和返回response
- stub类调用方法会跳转到channel的CallMethod(这是需要框架写的)
- 等待返回的response(这是同步调用)

##8.rpc使用服务端逻辑

1.注册服务

- 写一个proto,包含message和service
- 写一个服务类,继承proto的service
- 重写service的方法,方法需要包含以下几步:
  - 获取方法参数
  - 调用函数
  - 获取返回结果,并写入response
  - 执行回调操作(把返回结果发送回去),done->run

2.启动服务器

- 初始化(加载配置文件,监听ip和port
- 使用provider类,注册服务
- provider.run(启动服务,等待连接)

## 9.zookeeper

zookeeper功能很多,分布式锁等,本项目用它是用来获取不同的模块所在的服务器地址.因为客户端发起rpc调用时并不知道该模块在哪个服务器上,该连接哪个ip和port。同时rpc服务所在的服务器也会出现变化，所以使用固定的配置文件不方便。而rpc具有心跳机制，由watcher模块，可以自动监控所对应的rpc服务是否还存在，以及对新加的rpc服务也能灵活更新。

- 针对不同服务的不同方法创建节点，每个节点存储服务所在的服务器的ip和port
- provider调用run方法时会获取已经注册的服务和方法，连接zookeeper并创建节点（如果节点不存在的话），并在节点存储ip和port（通过加载配置文件可得，每台服务器的配置文件的ip和port不一样）
- mrchannel类的callmethod在了解服务器时，根据服务名和方法可以找到zookeeper对应的节点，如果节点存在说明服务还在（由心跳机制保证，每10秒发送一次），获取节点存储的ip和port，连接connect，并发送数据

### zookeeper的节点znode

 ![image-20230814204424413](C:\Users\86187\AppData\Roaming\Typora\typora-user-images\image-20230814204424413.png)

![image-20230814204931181](C:\Users\86187\AppData\Roaming\Typora\typora-user-images\image-20230814204931181.png)