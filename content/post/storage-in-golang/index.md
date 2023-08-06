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


## 文件存储

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

ORM 思想是将数据结构和表结构相对应。Gorm是go里最棒的ORM。

