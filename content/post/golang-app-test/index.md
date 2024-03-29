---
title: "Golang中的应用测试"
date: 2023-02-15T11:31:41+08:00
draft: false
categories:
    - golang
tags:
    - go-web
---

本文记录了一下内容：

- golang的单元测试、基准测试、http测试
- 测试替身和依赖注入
- 第三方Go测试库，gocheck，ginkgo

目录：

- [go应用测试概述](#go应用测试概述)
  - [单元测试](#单元测试)
    - [跳过测试用例](#跳过测试用例)
    - [设置并行运行的单元测试数量](#设置并行运行的单元测试数量)
  - [基准测试](#基准测试)
  - [如何测试http](#如何测试http)
    - [生命周期函数（我自己取的名字）](#生命周期函数我自己取的名字)
- [测试替身和依赖注入](#测试替身和依赖注入)
- [第三方go检测库](#第三方go检测库)
  - [gocheck](#gocheck)
  - [ginkgo](#ginkgo)


## go应用测试概述
go的testing包用于测试，net/http/httptest用于测试web程序。对于源码文件server.go，可以在同目录下创建server_test.go文件，定义func TestXxx(*testing.T)函数。函数内部使用Error、Fail等方法表示测试失败，如果测试时没有出现任何失败，则表示测试通过。

编写完测试文件后，在当前目录下用`go test`测试所有测试文件。

### 单元测试

如果一个部分能独立进行测试，那被称为“单元”。向单元输入数据，并检查输出是否符合预期就是单元测试。

testing.T中的有用的方法：

- Log，Logf：把文本记录到错误日志中，不终止测试
- Fail：标记当前测试单元“失败”
- FailNow：标记当前测试函数“失败”，并终止当前测试单元

Error,Errorf,Fatal,Fatalf是上述函数的复合，见下表。

 |         | Log   | Logf   |
 | :------ | :---- | :----- |
 | Fail    | Error | Errorf |
 | FailNow | Fatal | Fatalf |

这些方法只对当前单元有效。

单元测试中的命令：

- `-v`获取详细测试信息
- `-cover`输出代码覆盖率

如`go test -v -cover`

#### 跳过测试用例

如果在单元测试中调用`t.Skip()`函数，则会在执行到这一行时跳过该测试单元的其余部分。

命令中也提供这么个flag，用于逻辑判断：`-short`，当设置了`-short`时（如`go test -v -cover -short`），代码中调用testing.Short()返回true.

将`-short`和`t.Skip()`结合使用，可实现命令行控制是否跳过某些测试函数，如：

    func TestXxxx(t *testing.T){
        if testing.Short(){
            t.Skip("skip for short flag.")
        }
        ...
    }

#### 设置并行运行的单元测试数量

利用`-parallel n`设置并行运行的单元测试数量。如`go test -v -parallel 3`表示最多并行运行3个单元测试。

### 基准测试

利用`-bench [函数名的正则表达式]`flag，执行*_test.go中定义的基准函数的测试，用于评估函数的性能。基准函数格式：

    func BenchmarkXxx(*testing.B){...}

通常在其中添加循环，执行`b.N`次程序，以此观察程序性能：

    func BenchmarkXxx(b *testing.B){
        for i:=0;i<b.N;i++{
            ...
        }
    }

执行所有单元测试和基准函数：`go test -v -bench .`

如要忽略单元测试，使用`-run [函数名的正则表达式]`来指定要运行的单元测试。设置为`-run none`则会忽略所有单元测试。结合一下，`go test -v -run none -bench .`就只会执行基准测试了。

### 如何测试http

测试http就是在测试处理器函数，这种函数接受http.ResponseWriter和*http.Request.问题在于如何提供这两个参数。我们可以在测试函数中伪造http server、http请求，并把response记录下来，实现伪造http整个流程，从而实现测试。

下面代码实现了对处理器函数的测试：

1.   伪造一个http server
2.   指定要测试的处理器函数和路径
3.   伪造GET请求
4.   把response记录在httptest.ResponseRecorder中
5.   读取response，看程序是否符合预期

    func TestHandleGet(t *testing.T) {
        mux := http.NewServeMux()
        mux.HandleFunc("/post/", handleRequest)

        writer := httptest.NewRecorder()
        req, _ := http.NewRequest("GET", "/post/1", nil)
        mux.ServeHTTP(writer, req)

        if writer.Code != 200 {
            t.Errorf("response code is %d", writer.Code)
        }
        var post data.Post
        json.Unmarshal(writer.Body.Bytes(), &post)
        if post.Id != 1 {
            t.Error("can't retrieve JSON post")
        }
    }

#### 生命周期函数（我自己取的名字）

为了在测试前、测试后统一执行一些共有的代码，可以利用生命周期函数实现。首先定义函数TestMain

    func TestMain(m *testing.M) {
        setUp()
        code := m.Run()
        tearDown()
        os.Exit(code)
    }

`setUp()`和`tearDown()`都是为所有测试用例定义的函数，m.Run()会调用测试案例，所以`setUp()`在测试前执行，`tearDown()`在测试后执行。**并且在整个测试中只会执行一次。**

上述TestHandleGet方法我们也可以优化为：

    var mux *http.ServeMux
    var writer *httptest.ResponseRecorder

    func TestMain(m *testing.M) {
        setUp()
        code := m.Run()
        os.Exit(code)
    }

    func setUp() {
        mux = http.NewServeMux()
        mux.HandleFunc("/post/", handleRequest)
        writer = httptest.NewRecorder()
    }


    func TestHandleGet(t *testing.T) {
        req, _ := http.NewRequest("GET", "/post/1", nil)
        mux.ServeHTTP(writer, req)

        if writer.Code != 200 {
            t.Errorf("response code is %d", writer.Code)
        }
        var post data.Post
        json.Unmarshal(writer.Body.Bytes(), &post)
        if post.Id != 1 {
            t.Error("can't retrieve JSON post")
        }
    }

这样就把mux和writer作为全局变量。测试开始前进行初始化设置。

## 测试替身和依赖注入

为了不在测试中执行真实的操作，如测试邮件发送不希望真的发送邮件；测试数据库不希望真的修改数据库（某些场景下），我们需要用接口来实现依赖注入（替换实际对象），由依赖关系代替实际的操作，实现层的解耦。

## 第三方go检测库

### gocheck

这是一个基于testing构建的测试框架。安装：`go get gopkg.in/check.v1`

有几个特点：

- 以suite为单位分组测试（测试某个结构里的所有测试方法）
- suite或单个测试用例粒度的生命周期函数（测试夹具）
- 。。。

使用例子，只有注册过的Suite才会被测试：

```
import(
    . "gopkg.in/check.v1"
)
//定义suite
type XxxSuite struct{}

//注册suite,只有注册过的Suite才会被测试
func init(){
    Suite(&XxxSuite{})
}

//定义suite里的测试方法
func (x *XxxSuite) TestHandleGet(c *C){
    ...
    c.Check(code,Equals,200)
    ...
}
```

`go test -check.vv` 会显示更详细的日志

测试夹具（预定义的生命周期函数）：

- suite粒度（当前套件执行前后调用）
  - SetUpSuite
  - TearDownSuite
- 测试用例粒度（当前套件的每个测试用例执行前后调用）
  - SetUpTest
  - TearDownTest

注意这几个函数需要定义在suite内，如：`func (x *XxxSuite) SetUpSuite(c *C){}` `func (x *XxxSuite) SetUpTest(c *C){}`

### ginkgo

一个行为驱动开发（BDD）风格的Go测试框架。主要用于实现BDD，但是这里只用作测试框架使用。

BDD，软件由目标行为定义。这些行为也就是业务需求，如：

![Alt text](bdd.png)

**1、用ginkgo转换已存在的测试用例为BDD风格**

在包含测试文件的目录下执行`ginkgo convert .`，会生成`xxx_suite_test.go`（相当于原来testing的入口），并对原`xxx_test.go`进行修改，因此注意备份。

![Alt text](ginkgo1.png)

![Alt text](ginkgo2.png)

![Alt text](ginkgo3.png)

**2、自己编写ginkgo用例**

用到2个命令：

- `ginkgo bootstrap`:创建引导文件（我取的名字），类似`xxx_suite_test.go`
- `ginkgo generate`:创建测试用例文件的骨架：
  
![Alt text](ginkgo4.png)

这里首先再导入一个断言包Gomega，为啥要用这个呢，可能功能更强大吧。

`go get github.com/onsi/gomega`

接下来就在这个Describe函数里描述用户故事、情景。也就是用他给定的格式写测试代码。

![Alt text](ginkgo5.png)

当然ginkgo的测试夹具（预定义的生命周期函数）也不可少，`BeforeEach()`会在每个`情景`前执行(也就是每个context函数前执行)：

![Alt text](ginkgo6.png)

