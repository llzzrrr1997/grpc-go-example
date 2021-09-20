# Hello World gRpc

## 环境安装

本地环境:Windows

参考:https://grpc.io/docs/languages/go/quickstart/

1. 安装Go

2. 安装Protocol buffer compiler

3. 安装Go plugins

   1. 安装后会在gopath生成对应的插件

      ```shell
      go get google.golang.org/protobuf/cmd/protoc-gen-go@v1.26
      go get google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.1
      ```

   2. 更新PATH环境变量，确保protoc编译器能找到这两个插件

   

## 跑个Hello World Demo

#### 环境

- 终端输入 protoc –version 能打印出版本信息
- $GOPATH/bin 目录下有 `protoc-gen-go`、`protoc-gen-go-grpc` 这两个可执行文件



#### Demo结构

```
helloworld/
├── client
│   └── main.go
├── helloworld
│   ├── hello_world_grpc.pb.go
│   ├── hello_world.pb.go
│   └── hello_world.proto
└── server
    └── main.go
```

####  编写.proto文件

hello_world.proto 文件代码如下:

```protobuf
//声明proto的版本 只有 proto3 才支持 gRPC
syntax = "proto3";
// 将编译后文件输出在 github.com/llzzrrr1997/grpc-go-example/helloworld/helloworld 目录
option go_package = "github.com/llzzrrr1997/grpc-go-example/helloworld/helloworld";
// 指定当前proto文件属于helloworld包
package helloworld;

// 定义一个名叫 greeting 的服务
service Greeter {
  // 该服务包含一个 SayHello 方法 HelloRequest、HelloReply分别为该方法的输入与输出
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}
// 具体的参数定义
message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}

```



#### 编译.proto文件生成Go代码

使用 protoc 编译生成对应源文件，在 ./github.com/llzzrrr1997/grpc-go-example/helloworld 目录下执行以下命令:

```shell
 protoc --go_out=. --go_opt=paths=source_relative \
     --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    ./helloworld/hello_world.proto
```

最终会生成两个代码文件 hello_world.pb.go 和 hello_world_grpc.pb.go



#### Server端代码

```go
package main

import (
	"context"
	pb "github.com/llzzrrr1997/grpc-go-example/helloworld/helloworld"
	"google.golang.org/grpc"
	"log"
	"net"
)

const (
	port = ":50051"
)

// greeterServer 定义一个结构体用于实现 .proto文件中定义的方法
// 新版本 gRPC 要求必须嵌入 pb.UnimplementedGreeterServer 结构体
type greeterServer struct {
	pb.UnimplementedGreeterServer
}

// SayHello 简单实现一下.proto文件中定义的 SayHello 方法
func (g *greeterServer) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	log.Printf("Received: %v", in.GetName())
	return &pb.HelloReply{Message: "Hello " + in.GetName()}, nil
}

func main() {
	listen, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()
	// 将服务描述(server)及其具体实现(greeterServer)注册到 gRPC 中去.
	// 内部使用的是一个 map 结构存储，类似 HTTP server。
	pb.RegisterGreeterServer(s, &greeterServer{})
	log.Println("Serving gRPC on 0.0.0.0" + port)
	if err := s.Serve(listen); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}

```

1. 定义一个结构体，必须包含`pb.UnimplementedGreeterServer` 对象；
2. 实现 .proto文件中定义的API；
3. 将服务描述及其具体实现注册到 gRPC 中；
4. 启动服务；



#### Client端代码

```go
package main

import (
	"context"
	pb "github.com/llzzrrr1997/grpc-go-example/helloworld/helloworld"
	"google.golang.org/grpc"
	"log"
	"os"
	"time"
)

const (
	address     = "localhost:50051"
	defaultName = "world"
)

func main() {
	conn, err := grpc.Dial(address, grpc.WithInsecure(), grpc.WithBlock())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewGreeterClient(conn)

	// 通过命令行参数指定 name
	name := defaultName
	if len(os.Args) > 1 {
		name = os.Args[1]
	}
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	r, err := c.SayHello(ctx, &pb.HelloRequest{Name: name})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}
	log.Printf("Greeting: %s", r.GetMessage())
}

```

1. 首先使用 `grpc.Dial()` 与 gRPC 服务器建立连接；
2. 使用` pb.NewGreeterClient(conn)`获取客户端；
3. 通过客户端调用ServiceAPI方法`client.SayHello`；

#### 运行

1. 先运行Server端

   ```
   $ go run main.go
   2021/09/20 18:20:51 Serving gRPC on 0.0.0.0:50051
   2021/09/20 18:21:03 Received: world
   2021/09/20 18:21:15 Received: llzzrrr
   ```

   

2. 再运行Client端

   ```
   $ go run main.go
   2021/09/20 18:21:03 Greeting: Hello world
   
   $ go run main.go llzzrrr
   2021/09/20 18:21:15 Greeting: Hello llzzrrr
   ```

## 总结

1. 需要使用 protobuf 定义接口，即编写 .proto 文件;
2. 然后使用 protoc 工具配合编译插件编译生成特定语言或模块的执行代码；
3. 分别编写 server 端和 client 端代码，写入自己的业务逻辑；



