---
title: "一个使用Go Rpc库实现的RPC例子"
date: 2023-02-25T13:14:03+08:00
draft: false

categories:
    - golang
    - 微服务
tags:
    - go-micro-book
---

本文使用go rpc库实现服务的创建、客户端的请求。

server提供字符串拼接和字符串区别的服务，运行在127.0.0.1:1234。client可以调用rpc发送请求，调用server上的服务，并获取到结果。

**注意点：**

client和server都要维护请求参数结构和响应参数结构，如拼接字符串的请求参数：

    type StringRequest struct {
        A string
        B string
    }

响应结构这里用string即可，所以没有专门定义结构。

client调用Call或Go方法，传入远程方法名（结构+方法，如`StringService.Concat`）、请求结构、响应结构的指针，等待结果完成即可。

**结果：**

server运行：

    go build && ./rpc-server 
    2023/02/25 13:11:20 server at 127.0.0.1:1234

运行client：

    go build && ./rpc-client
    StringService Concat : A concat B = AB
    StringService Diff : ABC diff BCD = BC

## server

StringService是实现了Service接口的结构。

### server.go

    package service

    import (
        "errors"
        "strings"
    )

    type StringRequest struct{
        A string
        B string
    }

    type Service interface {
        Concat(req StringRequest, ret *string) error
        Diff(req StringRequest, ret *string) error
    }

    type StringService struct {
    }

    const (
        StrMaxSize = 1024
    )

    var (
        ErrMaxSize = errors.New("over max size of 1024")

        ErrStrValue = errors.New("maximum size of 1024 bytes exceeded")
    )

    func (s StringService) Concat(req StringRequest, ret *string) error {
        if len(req.A)+len(req.B) > StrMaxSize {
            *ret = ""
            return ErrMaxSize
        }
        *ret = req.A + req.B
        return nil
    }
    func (s StringService) Diff(req StringRequest, ret *string) error {
        if len(req.A) < 1 || len(req.B) < 1 {
            *ret = ""
            return nil
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
        *ret = res
        return nil
    }

### main.go

    package main

    import (
        "log"
        "net"
        "net/http"
        "net/rpc"

        "rpc-server/service"
    )

    func main() {
        stringService := new(service.StringService)
        rpc.Register(stringService)
        rpc.HandleHTTP()
        l, e := net.Listen("tcp", "127.0.0.1:1234")
        if e != nil {
            log.Fatal("listen error,", e)
        }
        log.Println("server at 127.0.0.1:1234")
        http.Serve(l, nil)
    }


## Client

### server.go

    package service

    type StringRequest struct {
        A string
        B string
    }


### main.go

    package main

    import (
        "fmt"
        "log"
        "net/rpc"

        "rpc-client/service"
    )

    func main() {
        client, err := rpc.DialHTTP("tcp", "127.0.0.1:1234")
        if err != nil {
            log.Fatal("dialing:", err)
        }
        stringReq := &service.StringRequest{"A", "B"}
        var reply string
        err = client.Call("StringService.Concat", stringReq, &reply)
        if err != nil {
            log.Fatal("Concat err:", err)
        }

        fmt.Printf("StringService Concat : %s concat %s = %s\n", stringReq.A, stringReq.B, reply)

        stringReq = &service.StringRequest{"ABC", "BCD"}
        call := client.Go("StringService.Diff", stringReq, &reply, nil)
        _ = <-call.Done

        fmt.Printf("StringService Diff : %s diff %s = %s\n", stringReq.A, stringReq.B, reply)
    }
