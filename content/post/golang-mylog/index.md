---
title: "golang自定义日志记录器"
date: 2023-02-07T11:03:38+08:00
draft: false
categories:
    - golang
tags:
    - go-in-action
    - demo 
---

标准库的log包无法实现日志分级处理，这个demo可以实现日志的多级处理。原理是定义多个log对象，并指定不同的输出，实现不同等级日志不同输出。

mlog包代码如下：

    //  mlog.go
    package mlog

    import (
        "io"
        "log"
        "os"
    )

    var (
        Trace   *log.Logger
        Info    *log.Logger
        Warning *log.Logger
        Error   *log.Logger
    )

    func init() {
        file, err := os.OpenFile("error.txt", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0666)
        if err != nil {
            log.Fatalln("Fail to open error file:", err)
        }
        Trace = log.New(io.Discard, "TRACE: ", log.Ldate|log.Ltime|log.Lshortfile)
        Info = log.New(os.Stdout, "INFO: ", log.Ldate|log.Ltime|log.Lshortfile)
        Warning = log.New(os.Stdout, "WARNING: ", log.Ldate|log.Ltime|log.Lshortfile)
        Error = log.New(file, "ERROR: ", log.Ldate|log.Ltime|log.Lshortfile)
    }

调用测试：

    package main

    import "github.com/chajiu/go-example/mlog"

    func main() {
        mlog.Trace.Println("trace...")
        mlog.Info.Println("Info...")
        mlog.Warning.Println("warning...")
        mlog.Error.Println("error...")
    }

error.txt会在main目录下生成。

    ERROR: 2023/02/07 10:56:13 main.go:9: error...
    ERROR: 2023/02/07 10:56:23 main.go:9: error...

Info和Warning则是直接输出到stdout：

    INFO: 2023/02/07 10:56:23 main.go:7: Info...
    WARNING: 2023/02/07 10:56:23 main.go:8: warning...



