---
title: "Golang中的数据存储"
date: 2023-02-12T08:47:06+08:00
draft: false
categories:
    - golang
tags:
    - go-web
---

Golang中的数据存储有三种方案：

- 利用容器的内存存储（非内存数据库）
- 利用文件读写的存储（文本文件如csv、二进制文件如gob）
- 数据库存储

本文简述后面两种。

- [文件存储](#文件存储)
  - [csv读写](#csv读写)
  - [gob读写](#gob读写)
- [利用database/sql连接数据库并执行CRUD](#利用databasesql连接数据库并执行crud)
- [go orm 关系映射器](#go-orm-关系映射器)
  - [安装](#安装)
  - [连接数据库](#连接数据库)
  - [模型声明](#模型声明)
  - [预加载](#预加载)
  - [关联模式](#关联模式)


# 文件存储

下面展示了golang中读写文件的常用方式。

- ioutil的函数读写文件
- os创建File，再读写文件

code:

    package main

    import (
        "fmt"
        "io/ioutil"
        "os"
    )

    func main() {
        //ioutil读写文件
        data := []byte("hello world!")
        err := ioutil.WriteFile("data1", data, 0644)
        if err != nil {
            panic(err)
        }
        read1, _ := ioutil.ReadFile("data1")
        fmt.Println(string(read1))

        //os读写文件
        file1, _ := os.Create("data2")
        defer file1.Close()

        nbytes, _ := file1.Write(data)
        fmt.Printf("wrote %d bytes to file\n", nbytes)

        file2, _ := os.Open("data2")
        defer file2.Close()

        read2 := make([]byte, len(data))
        nbytes, _ = file2.Read(read2)
        fmt.Printf("Read %d bytes from file\n", nbytes)
        fmt.Println(string(read2))
    }
    //
    hello world!
    wrote 12 bytes to file
    Read 12 bytes from file
    hello world!

### csv读写

利用"encoding/csv"可实现csv读写，下面是小例子。将File传给csv.NewWriter()或csv.NewReader()

    package main

    import (
        "encoding/csv"
        "fmt"
        "os"
        "strconv"
    )

    type Post struct {
        Id      int
        Content string
        Author  string
    }

    func main() {
        csvFile, err := os.Create("posts.csv")
        if err != nil {
            panic(err)
        }
        defer csvFile.Close()

        allPosts := []Post{
            Post{1, "hello world 1", "Jack1"},
            Post{2, "hello world 2", "Jack2"},
            Post{3, "hello world 3", "Jack3"},
            Post{4, "hello world 4", "Jack4"},
            Post{5, "hello world 5", "Jack5"},
        }
        writer := csv.NewWriter(csvFile)
        for _, post := range allPosts {
            line := []string{strconv.Itoa(post.Id), post.Content, post.Author}
            if err := writer.Write(line); err != nil {
                panic(err)
            }
        }
        writer.Flush()

        file, err := os.Open("posts.csv")
        if err != nil {
            panic(err)
        }
        defer file.Close()

        reader := csv.NewReader(file)
        reader.FieldsPerRecord = -1
        records, err := reader.ReadAll()
        if err != nil {
            panic(err)
        }

        var posts []Post
        for _, item := range records {
            id, _ := strconv.Atoi(item[0])
            post := Post{id, item[1], item[2]}
            posts = append(posts, post)
        }

        fmt.Println(posts[0].Id)
        fmt.Println(posts[0].Content)
        fmt.Println(posts[0].Author)
    }

生成posts.csv:

    1,hello world 1,Jack1
    2,hello world 2,Jack2
    3,hello world 3,Jack3
    4,hello world 4,Jack4
    5,hello world 5,Jack5

### gob读写

利用"encoding/gob"包，gob.NewEncoder()接受一个io.Writer; gob.NewDecoder()接受一个io.Reader.

也就是说，可以提供一个buffer，供编码时存储结果；供解码时存储输入。

最后传入`&postRecord`才能修改原变量内容。

    package main

    import (
        "bytes"
        "encoding/gob"
        "fmt"
        "io/ioutil"
    )

    type Post struct {
        Id      int
        Content string
        Author  string
    }

    func store(data interface{}, filename string) {
        buffer := new(bytes.Buffer)
        encoder := gob.NewEncoder(buffer)
        err := encoder.Encode(data)
        if err != nil {
            panic(err)
        }
        err = ioutil.WriteFile(filename, buffer.Bytes(), 0600)

        if err != nil {
            panic(err)
        }
    }

    func load(data interface{}, filename string) {
        raw, err := ioutil.ReadFile(filename)

        if err != nil {
            panic(err)
        }
        buffer := bytes.NewBuffer(raw)
        decoder := gob.NewDecoder(buffer)
        err = decoder.Decode(data)

        if err != nil {
            panic(err)
        }
    }

    func main() {
        post := Post{1, "hello world", "jack"}
        store(post, "post1")
        var postRecord Post
        load(&postRecord, "post1")
        fmt.Println(postRecord)

    }


## 利用database/sql连接数据库并执行CRUD

首先需要注入对应的数据库驱动，如postgres常用github.com/lib/pq驱动。在main包中注入：

    import(
        _ "github.com/lib/pq"
    )

数据库连接，使用sql.Open(数据库名称，数据源字符串)，其中数据源字符串需要符合对应驱动的格式，如pq的话，有这么几种合法的格式：

- "user=pqgotest dbname=pqgotest sslmode=verify-full"
- "postgres://pqgotest:password@localhost/pqgotest?sslmode=verify-full"

见 https://pkg.go.dev/github.com/lib/pq

CRUD总结：

1. 预处理语句：使用sql.Stmt结构
   
        stmt,err := db.Prepare("...")

2. 执行语句并获取一个sql.Row,一般后跟Scan(),用于把这一行读取到某些变量中
   
        db.QueryRow("...", , )
        或
        stmt.QueryRow( , )

3. 执行并获取多个sql.Row:

        rows,err := db.Query("...",  ,  )
        rows.Next()
        rows.Scan()

4. 直接执行语句，只返回最后插入ID(如果insert)、影响的行数（insert、update、delete），不返回查询结果：

        res,err := db.Exec("...", , )

## go orm 关系映射器

ORM 思想是将数据结构和表结构相对应。Gorm是go里最棒的ORM。下面介绍Gorm的大概用法，具体可见文档 https://gorm.io/zh_CN/docs/index.html

## 安装

需要安装gorm和对应的数据库驱动

    go get -u gorm.io/gorm
    go get -u gorm.io/driver/postgres

## 连接数据库

一般在init中执行连接和数据库自动迁移：

    func init() {
        var err error
        dsn := "user=postgres password=mkQ445683 dbname=chitchat sslmode=disable"
        Db, err = gorm.Open(postgres.Open(dsn), &gorm.Config{})
        if err != nil {
            log.Fatal(err)
        }
        Db.AutoMigrate(&Post{}, &Thread{}, &Session{}, &User{})
    }

> 迁移：Db.AutoMigrate()，AutoMigrate 会创建表、缺失的外键、约束、列和索引，出于保护您数据的目的，它**不会**删除未使用的列
>
> db.AutoMigrate(&User{})
>
> db.AutoMigrate(&User{}, &Product{}, &Order{})

## 模型声明

Gorm默认用ID作为主键，使用结构体名的 `蛇形复数` 作为表名，字段名的 `蛇形` 作为列名，并使用 CreatedAt、UpdatedAt 字段追踪创建、更新时间

如：MemberNumber --> member_number; CreatedAt --> created_at

通常将gorm.Model嵌入自己的结构体中，如：

	User struct {
		gorm.Model
		Uuid     string
		Name     string
		Email    string
		Password string
	}

gorm.Model包括字段 ID、CreatedAt、UpdatedAt、DeletedAt，所以你的结构体也会包含这些字段。对于包含的field，也可以用标签embedded嵌入：

    type Blog struct {
        ID      int
        Author  Author `gorm:"embedded"`
        Upvotes int32
    }

## 预加载

首先我们定义User、Order、City. User和Order是一对多关系，User和City是一对一关系。

    type User struct {
        gorm.Model
        Username string
        Orders   []Order
        City     City
    }

    type Order struct {
        gorm.Model
        UserID uint
        Price  float64
    }
    type City struct{
        gorm.Model
        UserID uint
        Name string
    }

这种关系由外键定义，Gorm默认会将类似UserID识别为User的外键。所以上述关系会被Gorm建立。

同时我们可以在`User`里包含`[]Order`和`City`，这很方便我们获取和User相关联的对象。如果仅仅通过

    users := []User{}
    db.Find(&users)

无法加载`[]Order`和`City`。一个加载内在关联的对象的方法是`预加载Preload`：

    users := []User{}
    db.Preload("Orders").Preload("City").Find(&users)

通过在Preload里指定结构字段名，再查询，Gorm会自动加载对应关联的对象。这样我们就可以获得User和与其匹配的Orders和City。

**1、条件预加载**

上述Preload会匹配所有UserID符合的Order，如果想进一步添加约束条件，如已完成的Order，如何实现？可以用条件预加载：

    db.Preload("Orders", "state=?", "completed").Find(&users)
    // SELECT * FROM users;
    // SELECT * FROM orders WHERE user_id IN (1,2,3,4) AND state NOT IN ('cancelled');

    db.Where("state = ?", "active").Preload("Orders", "state NOT IN (?)", "cancelled").Find(&users)
    // SELECT * FROM users WHERE state = 'active';
    // SELECT * FROM orders WHERE user_id IN (1,2) AND state NOT IN ('cancelled');

第一个是在查找Order时添加约束，第二个是在查找User时添加where，然后查找Order时添加约束

**2、嵌套预加载**

假设Order里又包含has one关系，对应一个产品，查User时能把Orders.Product查出来吗？

    type User struct {
        gorm.Model
        Username string
        Orders   []Order
        City     City
    }

    type Order struct {
        gorm.Model
        UserID uint
        Price  float64
        ProductID uint
        Product Product
    }
    
    type Product struct {
        gorm.Model
        Name   string
        Price  float64
    }
    
是可以的，但是要用嵌套预加载，指定加载`Orders.Product`

    db.Preload("Orders.Product").Preload("City").Find(&users)

## 关联模式

由于关系的存在（belongs to，has one，has many，many to many），我们可以通过关系进行CRUD。

比如假设一个人有多张卡，User和CreditCard是一对多关系，想给User添加一张新的卡，一个方法是直接操作CreditCard表，设置卡属性、UserID，再保存。

另一个方法是通过User和CreditCard的一对多关系操作，给User Append一张卡：

    db.Model(&user).Association("CreditCard").Append(&CreditCard{Number: "411111111111"})

上述代码指明了User是谁（&user，源模型，主键不能为空），指明哪个关系（CreditCard，结构字段名），操作是什么（append+卡信息），注意此处卡不需要声明UserID，因为这个关系已经给出了。

这是通过关联模式的添加功能。我们可以通过关系实现完整的CRUD。

// to be continuing