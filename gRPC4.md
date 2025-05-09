# gRPC的开发例子

## 登录开发例子

### proto文件编辑

IM.Login.proto文件

```protobuf
syntax = "proto3";
 package IM.Login;
 // 定义grpc服务的名称
service ImLogin {
 // 定义服务函数
	rpc Regist (IMRegistReq) returns (IMRegistRes) {}
 	rpc Login (IMLoginReq) returns (IMLoginRes){}  
}
// 后面的请求响应结构，无论名字是大写还是小写，最终调用的时候都是小写
// 注册账号
message IMRegistReq{
 	string user_name = 1; // 用户名
	string password = 2;  // 密码
}
 message IMRegistRes{
 	string user_name = 1;
 	uint32 user_id = 2;
 	uint32 result_code = 3;   // 返回0的时候注册正常
}
 // 登录账号
message IMLoginReq{
 	string user_name = 1;
 	string password = 2;
 }
 message IMLoginRes{ 
	uint32 user_id = 1;
 	uint32 result_code = 2;  // 返回0的时候正确
}
```

### 生成序列化代码

在 .proto 文件中定义了数据结构，这些数据结构是面向开发者和业务程序的，并不面向存储和传输。当需要把这些数据进行存储或传输时，就需要将这些结构数据进行序列化、反序列化以及读写。通过protoc 这个编译器，ProtoBuf 将会为我们提供相应的接口代码。可通过如下命令生成相应的接口代码：

```shell
# $SRC_DIR: 指定在解析导入指令时查找 .proto 文件的目录。如果省略，则使用当前目录。可以通过多次传递 --proto_path 选项来指定多个导入目录，他们将按顺序搜索
# --cpp_out: 生成 c++ 代码
# $DST_DIR: 生成代码的目标目录
# xxx.proto: 要针对哪个 proto 文件生成接口代码
# -I 是 --proto_path 的缩写形式
protoc -I=$SRC_DIR --cpp_out=$DST_DIR  $SRC_DIR/xxx.proto
```

生成gRPC服务框架代码

```shell
protoc -I ./ --grpc_out=. --plugin=protoc-gen-grpc=`which grpc_cpp_plugin` simple.proto
# 执行上面面命令后，将在当前目录下生成 simple.grpc.pb.h 和 simple.grpc.pb.cc 文件
# 上面的 `which grpc_cpp_plugin` 也可以替换为 grpc_cpp_plugin 程序的路径（如果不在系统PATH下）
# 生成的代码需要依赖第一步生成序列化代码，所以在使用的时候必须都要有
```

生成IM.Login.proto对应的c++代码

```shell
protoc --cpp_out=. IM.Login.proto
protoc --cpp_out=. --grpc_out=. --plugin=protoc-gen-grpc=/usr/local/bin/grpc_cpp_plugin IM.Login.proto
# 或者一步
protoc -I . --cpp_out=. --grpc_out=. --plugin=protoc-gen-grpc=`which grpc_cpp_plugin`  *.proto
```

### grpc server端

开放命名空间

```c++
// 命名空间
// grcp
 using grpc::Server;
 using grpc::ServerBuilder;
 using grpc::ServerContext;
 using grpc::Status;
 // 自己proto文件的命名空间
using IM::Login::ImLogin;
 using IM::Login::IMRegistReq;
 using IM::Login::IMRegistRes;
 using IM::Login::IMLoginReq;
 using IM::Login::IMLoginRes;
```

重写服务，定义服务端的类，继承 proto 文件定义的 grpc 服务 ImLogin ，重写 grpc 服务定义的方法

```c++
class IMLoginServiceImpl : public ImLogin::Service {
  // 注册
  virtual Status Regist(ServerContext *context, const IMRegistReq *request,
                        IMRegistRes *response) override {
    std::cout << "Regist user_name: " << request->user_name() << std::endl;
    response->set_user_name(request->user_name());
    response->set_user_id(10);
    response->set_result_code(0);
    return Status::OK;
  }
  // 登录
  virtual Status Login(ServerContext *context, const IMLoginReq *request,
                       IMLoginRes *response) override {
    std::cout << "Login user_name: " << request->user_name() << std::endl;
    response->set_user_id(10);
    response->set_result_code(0);
    return Status::OK;
  }
};
```

启动服务

```c++
std::string server_address("0.0.0.0:50051");
/* 定义重写的服务类 */
ImLoginServiceImpl service;
/* 创建工厂类 */
ServerBuilder builder;
/* 监听端口和地址 */
builder.AddListeningPort(server_address, grpc::InsecureServerCredentials());
builder.AddChannelArgument(GRPC_ARG_KEEPALIVE_TIME_MS, 5000);
builder.AddChannelArgument(GRPC_ARG_KEEPALIVE_TIMEOUT_MS, 10000);
builder.AddChannelArgument(GRPC_ARG_KEEPALIVE_PERMIT_WITHOUT_CALLS, 1);
/* 注册服务 */
builder.RegisterService(&service);
/** 创建和启动一个RPC服务器*/
std::unique_ptr<Server> server(builder.BuildAndStart());
std::cout << "Server listening on " << server_address << std::endl;
/* 进入服务事件循环 */
server->Wait();
```

完整代码

```c++
#include <iostream>
#include <string>
// grpc头文件
#include <grpcpp/ext/proto_server_reflection_plugin.h>
#include <grpcpp/grpcpp.h>
#include <grpcpp/health_check_service_interface.h>
// 包含我们自己proto文件生成的.h
#include "IM.Login.grpc.pb.h"
#include "IM.Login.pb.h"
// 命名空间
// grcp
using grpc::Server;
using grpc::ServerBuilder;
using grpc::ServerContext;
using grpc::Status;
// 自己proto文件的命名空间
using IM::Login::ImLogin;
using IM::Login::IMLoginReq;
using IM::Login::IMLoginRes;
using IM::Login::IMRegistReq;
using IM::Login::IMRegistRes;
class IMLoginServiceImpl : public ImLogin::Service {
  // 注册
  virtual Status Regist(ServerContext *context, const IMRegistReq *request,
                        IMRegistRes *response) override {
    std::cout << "Regist user_name: " << request->user_name() << std::endl;
    response->set_user_name(request->user_name());
    response->set_user_id(10);
    response->set_result_code(0);
    return Status::OK;
  }
  // 登录
  virtual Status Login(ServerContext *context, const IMLoginReq *request,
                       IMLoginRes *response) override {
    std::cout << "Login user_name: " << request->user_name() << std::endl;
    response->set_user_id(10);
    response->set_result_code(0);
    return Status::OK;
  }
};
void RunServer() {
  std::string server_addr("0.0.0.0:50051");
  // 创建一个服务类
  IMLoginServiceImpl service;
  ServerBuilder builder;
  builder.AddListeningPort(server_addr, grpc::InsecureServerCredentials());
  builder.AddChannelArgument(GRPC_ARG_KEEPALIVE_TIME_MS, 5000);
  builder.AddChannelArgument(GRPC_ARG_KEEPALIVE_TIMEOUT_MS, 10000);
  builder.AddChannelArgument(GRPC_ARG_KEEPALIVE_PERMIT_WITHOUT_CALLS, 1);
  builder.RegisterService(&service);
  // 创建/启动
  std::unique_ptr<Server> server(builder.BuildAndStart());
  std::cout << "Server listening on " << server_addr << std::endl;
  // 进入服务循环
  server->Wait();
}
int main(int argc, const char **argv) {
  RunServer();
  return 0;
}
```

### gRPC client端

使用命名空间

```c++
// 命名空间
// grcp
using grpc::Channel;
using grpc::ClientContext;
using grpc::Status;
 // 自己proto文件的命名空间
using IM::Login::ImLogin;
using IM::Login::IMRegistReq;
using IM::Login::IMRegistRes;
using IM::Login::IMLoginReq;
using IM::Login::IMLoginRes;
```

定义客户端的类，实现两个方法用来发送grpc请求以及接收grpc响应

```c++
class ImLoginClient {
public:
  ImLoginClient(std::shared_ptr<Channel> channel)
      : stub_(ImLogin::NewStub(channel)) {}
  std::string Regist(const std::string &user) {
    IMRegistReq request;
    request.set_name(user);
    request.set_test("test");
    IMRegistRes reply;
    ClientContext context;
    Status status = stub_->Regist(&context, request, &reply);
    if (status.ok()) {
      return reply.message();
    } else {
      std::cout << status.error_code() << ": " << status.error_message()
                << std::endl;
      return "RPC failed";
    }
  }
  std::string Test(const std::string &user) {
    ClientContext context;
    IMLoginReq req;
    IMLoginRes res;
    req.set_sessionid(10);
    req.set_elestmname("BStar");
    Status status = stub_->Login(&context, req, &res);
    if (status.ok()) {
      std::cout << "result:" << res.result() << std::endl;
      return res.extension();
    } else {
      std::cout << status.error_code() << ": " << status.error_message()
                << std::endl;
      return "RPC failed";
    }
  }

private:
  std::unique_ptr<ImLogin::Stub> stub_;
};
```

完整代码

```c++
#include <iostream>
#include <memory>
#include <string>
// /usr/local/include/grpcpp/grpcpp.h
#include <grpcpp/grpcpp.h>
// 包含自己proto文件生成的.h
#include "IM.Login.grpc.pb.h"
#include "IM.Login.pb.h"
// 命名空间
// grcp
using grpc::Channel;
using grpc::ClientContext;
using grpc::Status;
// 自己proto文件的命名空间
using IM::Login::ImLogin;
using IM::Login::IMLoginReq;
using IM::Login::IMLoginRes;
using IM::Login::IMRegistReq;
using IM::Login::IMRegistRes;
class ImLoginClient {
public:
  ImLoginClient(std::shared_ptr<Channel> channel)
      : stub_(ImLogin::NewStub(channel)) {}
  void Regist(const std::string &user_name, const std::string &password) {
    IMRegistReq request;
    request.set_user_name(user_name);
    request.set_password(password);

    IMRegistRes response;
    ClientContext context;
    std::cout << "-> Regist req" << std::endl;
    Status status = stub_->Regist(&context, request, &response);
    if (status.ok()) {
      std::cout << "user_name:" << response.user_name()
                << ", user_id:" << response.user_id() << std::endl;
    } else {
      std::cout << "user_name:" << response.user_name()
                << "Regist 
          failed : " << response.result_code()<< std::endl;
    }
  }
  void Login(const std::string &user_name, const std::string &password) {
    IMLoginReq request;
    request.set_user_name(user_name);
    request.set_password(password);

    IMLoginRes response;
    ClientContext context;
    std::cout << "-> Login req" << std::endl;
    Status status = stub_->Login(&context, request, &response);
    if (status.ok()) {
      std::cout << "user_id:" << response.user_id() << " login ok" << std::endl;
    } else {
      std::cout << "user_name:" << request.user_name()
                << "Login failed: 
                   " << response.result_code()<< std::endl;
    }
  }

private:
  std::unique_ptr<ImLogin::Stub> stub_;
};

int main() {
  // 服务器的地址
  std::string server_addr = "localhost:50051";
  ImLoginClient im_login_client(
      grpc::CreateChannel(server_addr, grpc::InsecureChannelCredentials()));
  std::string user_name = "yangshuangxin";
  std::string password = "1234567";
  im_login_client.Regist(user_name, password);
  im_login_client.Login(user_name, password);
  return 0;
}
```







