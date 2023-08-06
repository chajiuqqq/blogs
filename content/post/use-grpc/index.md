---
title: "介绍一下grpc的使用"
date: 2023-02-28T11:28:33+08:00
draft: false
categories:
    - golang
    - 微服务
tags:
    - go-micro-book
---

gRPC和protocol buffers通常被一起使用，protoBuf是gRPC的接口定义语言，也是其底层消息交换结构。通过gRPC，一个客户端可以像调用本地方法一样调用服务端应用的方法。客户端和服务端可以在多种不同环境下互动，比如java作为服务端语言，而go、python等作为客户端语言。

gRPC使用protoBuf作为结构化数据的序列算法，可以在`.proto`文件中定义数据结构，并用`protoc`生成目标语言的客户端、服务端代码。

下面是一个例子，数据被定义为一个message，并包含一系列的域；用rpc和方法名，参数，返回值构成服务。

    // The greeter service definition.
    service Greeter {
        // Sends a greeting
        rpc SayHello (HelloRequest) returns (HelloReply) {}
    }

    // The request message containing the user's name.
    message HelloRequest {
        string name = 1;
    }

    // The response message containing the greetings
    message HelloReply {
        string message = 1;
    }

在生成客户端、服务端代码后，还需要进一步开发使用。

# string-service例子

下面以一个string-service作为例子介绍，首先定义proto文件：

    syntax = "proto3";

    package pb;
    option go_package = "grpc-string-test/pb";

    service StringService{
        rpc Concat(StringRequest) returns (StringResponse){}
        rpc Diff(StringRequest) returns (StringResponse){}
    }

    message StringRequest{
        string A = 1;
        string B = 2;
    }


    message StringResponse{
        string Ret = 1;
        string err = 2;
    }

proto3需要加上`option go_package = "grpc-string-test/pb";`前面是model名，后面是包名。这样才能生成成功。

编译代码是：

    protoc --go_out=. --go_opt=paths=source_relative \
        --go-grpc_out=. --go-grpc_opt=paths=source_relative \
        string.proto

这个命令需要到proto文件的目录执行。最后string.proto是要转换的proto文件名。最后会在当前目录下生成两个文件`string.pb.go`和`string_grpc.pb.go`，客户端和服务端都需要用到这两个文件。

## server端

为了实现具体的Concat和Diff方法，需要建立一个结构，并实现StringServiceServer接口

你首先可以这么做：

    type StringService struct {
        pb.UnimplementedStringServiceServer
    }

嵌入UnimplementedStringServiceServer，这是自动生成的一个结构，这样就默认实现了StringServiceServer接口，然后再重写Concat和Diff方法：

    func (s *StringService) Concat(ctx context.Context, req *StringRequest) (*StringResponse, error) {
        log.Println(req)
        if len(req.A)+len(req.B) > StrMaxSize {
            response := StringResponse{Ret: ""}
            return &response, nil
        }
        response := StringResponse{Ret: req.A + req.B}
        return &response, nil
    }

    func (s *StringService) Diff(ctx context.Context, req *StringRequest) (*StringResponse, error) {
        log.Println(req)
        if len(req.A) < 1 || len(req.B) < 1 {
            response := StringResponse{Ret: ""}
            return &response, nil
        }
        res := ""
        if len(req.A) >= len(req.B) {
            for _, char := range req.B {
                if strings.Contains(req.A, string(char)) {
                    res = res + string(char)
                }
            }
        } else {
            for _, char := range req.A {
                if strings.Contains(req.B, string(char)) {
                    res = res + string(char)
                }
            }
        }
        response := StringResponse{Ret: res}
        return &response, nil
    }

注意到传入参数是StringRequest，返回StringResponse，也就是我们之前定义的消息结构。

写好了业务到底如何实现，下面要开启server对端口的监听，接受rpc请求，流程如下：

- 获取某端口的Listener
- 新建grpc.Server和上面我们定义的StringService
- 调用RegisterStringServiceServer向grpc.Server注册StringService
- 开启rpc监听

```
    func main() {
        lis, err := net.Listen("tcp", "127.0.0.1:1234")

        if err != nil {
            log.Fatalf("failed to listen: %v", err)
        }
        log.Println("server at 127.0.0.1:1234")
        grpcServer := grpc.NewServer()
        stringService := new(service.StringService)
        pb.RegisterStringServiceServer(grpcServer, stringService)
        grpcServer.Serve(lis)
    }
```

## client端

client只要建立rpc连接，然后调用服务即可。流程如下：

- grpc.Dial拨号建立rpc连接，获得conn
- 从conn获取stringClient，这个变量实现的接口包含所有提供的服务
- 由stringClient调用各种服务

注意传入参数是我们之前定义的StringRequest，返回StringResponse，非常完美

```
    func main() {
        serviceAddress := "127.0.0.1:1234"
        conn, err := grpc.Dial(serviceAddress, grpc.WithInsecure())
        if err != nil {
            panic("connect error")
        }
        defer conn.Close()

        stringClient := pb.NewStringServiceClient(conn)
        stringReq := &pb.StringRequest{A: "A", B: "B"}
        reply, _ := stringClient.Concat(context.Background(), stringReq)
        fmt.Printf("StringService Concat : %s concat %s = %s\n", stringReq.A, stringReq.B, reply.Ret)

    }
```

至此，server和client都写完了，开启server，然后开启client，就可以进行rpc调用。

# 使用流的string-service例子

流指的是请求消息流或响应消息流，每个单位都是对应的请求结构或响应结构，并持续发送，好处是可以连续发送，连续处理。而不是发一个返回一个，发一个返回一个。

grpc支持单向流或双向流。在proto文件中给参数添加`stream`关键字就可以指明这是以流的形式发送的参数（或结果）：

    service StringService{
        rpc Concat(StringRequest) returns (StringResponse){}
        rpc Diff(StringRequest) returns (StringResponse){}
        rpc ConcatServerStream(StringRequest) returns (stream StringResponse){}
        rpc ConcatClientStream(stream StringRequest) returns (StringResponse){}
        rpc ConcatDoubleStream(stream StringRequest) returns (stream StringResponse){}
    }

## server发流

这指的是，客户端发一个req，服务端流返回res，客户端读取res流。我们要关注的就是client调用服务，接受流请求的过程；和服务端发送流的写法。


    //client
    func streamServerConn(stringClient pb.StringServiceClient) {

        stringReq := &pb.StringRequest{A: "A", B: "B"}
        //接受流
        stream, _ := stringClient.ConcatServerStream(context.Background(), stringReq)
        //循环读取流
        for {
            item, err := stream.Recv()
            if err == io.EOF {
                break
            }
            if err != nil {
                log.Println("fail to recv:", err)
            }
            fmt.Printf("StringService Concat : %s concat %s = %s\n", stringReq.A, stringReq.B, item.Ret)
        }

    }


    //server
    func (s *StringService) ConcatServerStream(req *StringRequest, qs StringService_ConcatServerStreamServer) error {
        response := StringResponse{Ret: req.A + req.B}
        for i := 0; i < 10; i++ {
            qs.Send(&response)
        }
        return nil

    }

## client发流

    //client
    func streamClientConn(stringClient pb.StringServiceClient) {

        stream, err := stringClient.ConcatClientStream(context.Background())
        //流式发送requset
        for i := 0; i < 10; i++ {
            if err != nil {
                log.Println("fail to call:", err)
                break
            }
            stream.Send(&pb.StringRequest{A: strconv.Itoa(i), B: strconv.Itoa(i + 1)})
        }

        //发完后读取一个response
        recv, err := stream.CloseAndRecv()
        if err != nil {
            log.Println("fail to recv:", err)
        }

        fmt.Printf("Ret is %s\n", recv.Ret)
    }

    //server
    func (s *StringService) ConcatClientStream(qs StringService_ConcatClientStreamServer) error {
        var params []string
        for {
            item, err := qs.Recv()
            //客户端发完了，服务器才处理并返回一个response
            if err == io.EOF {
                qs.SendAndClose(&StringResponse{Ret: strings.Join(params, "")})
                return nil
            }
            if err != nil {
                log.Println("fail to recv:", err)
                return err
            }
            params = append(params, item.A, item.B)
        }
    }

## 双向流


    //client
    func doubleStreamConn(stringClient pb.StringServiceClient) {

        stream, _ := stringClient.ConcatDoubleStream(context.Background())
        var i int
        for {
            err := stream.Send(&pb.StringRequest{A: strconv.Itoa(i), B: strconv.Itoa(i + 1)})
            if err != nil {
                log.Println("fail to send:", err)
                break
            }
            recv, err := stream.Recv()
            if err != nil {
                log.Println("fail to recv:", err)
                break
            }
            fmt.Printf("Ret is %s\n", recv.Ret)
            i++
            time.Sleep(time.Second)
        }

    }

    //server
    func (s *StringService) ConcatDoubleStream(qs StringService_ConcatDoubleStreamServer) error {
        for {
            in, err := qs.Recv()
            if err == io.EOF {
                return nil
            }
            if err != nil {
                log.Println("fail to recv:", err)
                return err
            }
            qs.Send(&StringResponse{Ret: in.A + in.B})
        }
    }

## 总结

流传输：

- 发送一个message：stream.send()
- 接受一个message：stream.recv()
- 发送一个msg，并关闭：stream.SendAndClose()
- 接受一个msg，并关闭：stream.CloseAndRecv()
- 判断流是否结束：err == io.EOF
