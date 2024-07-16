# 工具安装配置
## GO安装配置
### GO安装
直接在以下链接下载对应的安装包，下载运行安装即可。目前go安装程序会自动写入PATH和GOPATH环境变量[下载链接](https://golang.google.cn/dl/)
### GO 配置
```
go env -w GOPROXY=https://goproxy.io,direct （配置国内代理）
go get github.com/golang/protobuf/proto
go get google.golang.org/grpc
```
#### 配置protoc-gen-go、protoc-gen-go-grpc
##### protoc-gen-go
```
git clone https://github.com/protocolbuffers/protobuf-go
在 .\cmd\protoc-gen-go\目录，执行go install .
之后就可以在 $GOPATH\目录下看到protoc-gen-go.exe
```
##### protoc-gen-go-grpc
```
git clone -b v1.30.0 https://github.com/grpc/grpc-go  #克隆项目
cd cmd/protoc-gen-go-grpc   #用GoLand打开后，进入到指定目录
go install .
```
## cmake
下载安装即可[下载链接](https://cmake.org/download/)
## 
# grpc下载编译
## 代码clone
需要github良好的访问网络，不然可能clone到一半会失败
"--recurse-submodules"会将third_party中的子模块也下载下来，但是1.15.1版本用同样的方法就只下载到third_party下的一层，再下一层文件夹就下载不到了，得自己手动下载。更新子模块也可用“git submodule update --init”。
```
git clone --recurse-submodules -b v1.28.0 --depth 1 --shallow-submodules https://github.com/grpc/grpc
```
## 代码编译
```
mkdir .build
cd .build
cmake .. -G "Visual Studio 16 2019" -A x64   //64位版本，32位版本 -A win32
cmake --build . --config Release //编译Release版本,如果不加Release则默认为Debug版本
```
grpc 编译完成后会在grpc\.build\third_party\protobuf\Release 目录生成 protoc.exe后续生成不同语言代码的时候需要用到该文件

会在grpc\.build\Release目录生成grpc_cpp_plugin.exe 后续编译C++代码时需要使用
# grpc使用
## proto
hello.proto
```
syntax = "proto3"; // 指定proto版本
package hello_grpc;     // 指定默认包名
 
// 指定golang包名
option go_package = "/hello_grpc";
 
//定义rpc服务
service HelloService {
  // 定义函数
  rpc SayHello (HelloRequest) returns (HelloResponse) {}
}
 
// HelloRequest 请求内容
message HelloRequest {
  string name = 1;
  string message = 2;
}
 
// HelloResponse 响应内容
message HelloResponse{
  string name = 1;
  string message = 2;
}
```
## 生成GO版本代码
```
protoc.exe -I . --go_out=. --go-grpc_out=. ./hello.proto
执行以上代码会生成两个文件分别为hello_grpc.pb.go hello.pb.go，包名为hello_grpc
```
## 生成C++版本代码
```
protoc.exe -I=. --grpc_out=. --plugin=protoc-gen-grpc=grpc_cpp_plugin.exe hello.proto
执行以上代码会生成四个文件分别为，hello.pb.h、hello.pb.cc、hello.grpc.pb.h、hello.grpc.pb.cc
```
# demo
## GO server
go mod init GrpcTest
```
package main

import (
	"GrpcTest/grpc_proto/hello_grpc" //grpc_proto/hello_grpc为proto生成的.go文件路径，GrpcTest为mod init时的名称
	"context"
	"fmt"
	"net"

	"google.golang.org/grpc"
	"google.golang.org/grpc/grpclog"
)

// HelloServer 得有一个结构体，需要实现这个服务的全部方法,叫什么名字不重要
type HelloServer struct {
    //实测必须继承proto生成的hello_grpc.UnimplementedHelloServiceServer，才能作为RegisterHelloServiceServer的传参
	hello_grpc.UnimplementedHelloServiceServer
}

func (*HelloServer) SayHello(ctx context.Context, request *hello_grpc.HelloRequest) (*hello_grpc.HelloResponse, error) {
	fmt.Println("入参：", request.Name, request.Message)
	return &hello_grpc.HelloResponse{
		Name:    "server",
		Message: "hello " + request.Name,
	}, nil
}

func (*HelloServer) mustEmbedUnimplementedHelloServiceServer() {}

func main() {
	// 监听端口
	listen, err := net.Listen("tcp", "0.0.0.0:8080")
	if err != nil {
		grpclog.Fatalf("Failed to listen: %v", err)
	}

	// 创建一个gRPC服务器实例。
	s := grpc.NewServer()
	server := HelloServer{}
	// 将server结构体注册为gRPC服务。
	hello_grpc.RegisterHelloServiceServer(s, &server)
	fmt.Println("grpc server running :8080")
	// 开始处理客户端请求。
	err = s.Serve(listen)
}

```
## GO client
```
package main

import (
	"GrpcTest/grpc_proto/hello_grpc"
	"context"
	"fmt"
	"log"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

func main() {
	addr := ":8080"
	// 使用 grpc.Dial 创建一个到指定地址的 gRPC 连接。
	// 此处使用不安全的证书来实现 SSL/TLS 连接
	conn, err := grpc.Dial(addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf(fmt.Sprintf("grpc connect addr [%s] 连接失败 %s", addr, err))
	}
	defer conn.Close()
	// 初始化客户端
	client := hello_grpc.NewHelloServiceClient(conn)
	result, err := client.SayHello(context.Background(), &hello_grpc.HelloRequest{
		Name:    "client",
		Message: "hello",
	})
	fmt.Println(result, err)
}
```
## C++ client
main.cpp
```
#include "pch.h"
// 设置grpc的超时时间
#define GRPC_SET_TIMEOUT(context)\
        context.set_deadline(std::chrono::system_clock::now() + std::chrono::seconds(10));\
int main()
{
	std::string target_str = "127.0.0.1:8080";
	grpc::ClientContext context;
	GRPC_SET_TIMEOUT(context)
	auto stub = hello_grpc::HelloService::NewStub(grpc::CreateChannel(target_str, grpc::InsecureChannelCredentials()));
	hello_grpc::HelloRequest r;
	r.set_message("hello");
	r.set_name("Cpp client");
	hello_grpc::HelloResponse q;
	stub->SayHello(&context, r, &q);
	//grpc::create
}
```
pch.h
```
#pragma once
#include <iostream>
#include <string>
#include <chrono>

//需要在项目配置中新增包含路径
//grpc\third_party\protobuf\src
//C:\Users\Administrator\Desktop\Develop\grpc\include
//建议将以上文件夹下的文件放置在一个目录下，只新增一个包含路径
#include <grpc++/grpc++.h>
#include "grpc_lib.h"
#include "grpc/hello.grpc.pb.h"
#include "grpc/hello.pb.h"
```
grpc_lib.h
```
//该文件中的lib均为grpc编译时生成的产物，直接在.build文件夹内搜索，然后统一放置在一个文件夹内方便调用
#pragma once
#pragma once
#pragma comment(lib, "libprotobuf.lib")
#pragma comment(lib, "libprotoc.lib")
#pragma comment(lib, "zlibstatic.lib")

#pragma comment(lib, "address_sorting.lib")
#pragma comment(lib, "cares.lib")
#pragma comment(lib, "crypto.lib")
#pragma comment(lib, "gpr.lib")

#pragma comment(lib, "grpc.lib")
#pragma comment(lib, "grpc++.lib")

#pragma comment(lib, "ssl.lib")
#pragma comment(lib, "upb.lib")

#pragma comment(lib,"absl_cord.lib")
#pragma comment(lib,"absl_str_format_internal.lib")
#pragma comment(lib,"absl_strings.lib")
#pragma comment(lib,"absl_strings_internal.lib")
#pragma comment(lib,"absl_base.lib")
#pragma comment(lib,"absl_throw_delegate.lib")
#pragma comment(lib,"absl_log_severity.lib")
#pragma comment(lib,"absl_spinlock_wait.lib")
#pragma comment(lib,"absl_scoped_set_env.lib")
#pragma comment(lib,"absl_raw_logging_internal.lib")
#pragma comment(lib,"absl_periodic_sampler.lib")
#pragma comment(lib, "absl_malloc_internal.lib")
#pragma comment(lib, "absl_exponential_biased.lib")
#pragma comment(lib, "absl_dynamic_annotations.lib")
#pragma comment(lib, "absl_status.lib")
#pragma comment(lib,"absl_time_zone.lib")
#pragma comment(lib,"absl_time.lib")
#pragma comment(lib,"absl_civil_time.lib")
#pragma comment(lib,"absl_bad_variant_access.lib")
#pragma comment(lib,"absl_bad_optional_access.lib")
#pragma comment(lib,"absl_bad_any_cast_impl.lib")
#pragma comment(lib,"absl_int128.lib")
```

# 参考
[Windows设备go环境安装配置](https://blog.csdn.net/qq_36791745/article/details/136436521)

[windows安装protoc、protoc-gen-go、protoc-gen-go-grpc](https://blog.csdn.net/YouMing_Li/article/details/134895264)

[grpc-教程（golang版）](https://blog.csdn.net/qq_22321199/article/details/137551376)

[Windows系统下GRPC在C++中的使用](https://blog.csdn.net/yuanxino/article/details/135291648)

[新版本的protoc使用grpc容易遇到的两个坑，gen gRPC，mustEmbedUnimplementedHelloServer](https://blog.csdn.net/neve_give_up_dan/article/details/126920398)

