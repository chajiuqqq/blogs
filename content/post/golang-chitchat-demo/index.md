---
title: "Golang Chitchat Demo用到的一些技术"
date: 2023-02-21T12:45:03+08:00
draft: true
categories:
    - golang
tags:
    - go-web
    - demo
---
Chitchat是一个简单的网上论坛Web应用，用户可以登陆并发布新帖子，或回复其他用户发的帖子。

## 请求的接受和处理

### 1. 多路复用器

多路复用器指的是go中一个http请求处理器，负责接受并转发请求到各自正确的处理器上，如`/`转发到index处理器上。

通过`http.NewServeMux()`创建一个多路复用器。

通过`mux.HandleFunc("/",index)`将`/`转发到index处理器函数。

处理器和处理器函数：

- 实现了ServeHTTP接口的结构被称为处理器。也就是要实现ServeHTTP(http.ResponseWriter, *http.Request)方法。
- 如果一个方法接受(http.ResponseWriter, *http.Request)作为参数，并没有返回值，那么可以当作处理器函数。

本质上还是将处理器函数转为处理器来实现请求的处理。处理器函数是处理器的便捷是实现方式。也可以在`mux`中直接指定处理器：

    mux.Handle("/",indexProcessor)

### 2. 处理静态文件

新建一个处理器，专门处理静态文件，并把对应url前缀的请求导向它。在查找时，会把请求URL的/static/去除，再在`./public`下寻找文件。

如：`/static/js/index.js`-->`./public/js/index.js`

	files := http.FileServer(http.Dir("public"))
	mux.Handle("/static/", http.StripPrefix("/static/", files))

这里`http.Dir("public")`会获取当前目录下的public目录，如果写为`http.Dir("/public")`则会去系统根目录下找public。

### 3. 使用cookie的权限认证

登录时接受用户名和密码，和数据库匹配通过后，创建一个Session，并把sessionId作为cookie发到客户端。同时访问主页时检测有没有这个名称的cookie，如果有检查该sessionId的Session是否存在，存在并通过则展示个人页面，否则展示公开页面。

```

//登陆处理器
func authenticate(w http.ResponseWriter, r *http.Request) {
	r.ParseForm()
	user, _ := data.UserByEmail(r.Form["email"])
	if user.Password == data.Encrypt(r.Form["password"]) {
		session := user.NewSession()
		cookie := http.Cookie{
			Name:     "_cookie",
			Value:    session.Uuid,
			HttpOnly: true,
		}
		http.SetCookie(w, &cookie)
		http.Redirect(w, r, "/", 302)
	} else {
		http.Redirect(w, r, "/login", 302)
	}
}


//主页处理器
func index(w http.ResponseWriter, r *http.Request) {
	threads, err := data.Threads()
	if err != nil {
		log.Fatalln("can't get threads:", err)
	} else {
		_, err = checkSession(w, r)
		if err == nil {
			generateHTML(w, threads, "layout", "public.navbar", "index")
		} else {
			generateHTML(w, threads, "layout", "private.navbar", "index")
		}
	}

}

//访问主页时检测是否成功登陆过
func checkSession(w http.ResponseWriter, r *http.Request) (sess *data.Session, err error) {
	cookie, err := r.Cookie("_cookie")
	if err == nil {
		sess = &data.Session{Uuid: cookie.Value}
		if ok, _ := sess.Check(); !ok {
			err = errors.New("Invalid session")
		}
	}
	return
}

```

## 用模版生成HTML响应

模版中的动作默认使用{{和}}包围，也可以自行修改界定符。`{{.}}`指的是用传入的值替换这个动作本身。

go web的模版引擎使用方法分为以下两个步骤：

- 读取文本模版文件（或一个模版字符串），进行语法分析，创建一个模版结构。
- 将数据传入，执行模版，生成HTML。

第一个步骤：
    
    //从文件中生成模版
    t,_ := template.ParseFiles("index.html")

    //读取多个文件，但是只生成第一个文件的模板（因为第一个模版可能调用其他模版）
    t,_ := template.ParseFiles("index.html","content.html","navbar.html")

    //从文件中生成模版
    tmpl := `...`
    t := template.New("tmpl.html)
    t,_ = t.Parse(tmpl)

第二个步骤：

    // t.Execute第一个参数是io.Writer，第二个参数是传入的数据。这个方法只会执行第一个模版，也就是index.html
    t,_ := template.ParseFiles("index.html","content.html","navbar.html")
    t.Execute(w,"hello world")

    //如果想要执行其他模板，用t.ExecuteTemplate,并在第二个参数指定模版名称
    t,_ := template.ParseFiles("index.html","content.html","navbar.html")
    t.ExecuteTemplate(w,"content.html","hello world")

如果你只是在html中加入一些{{.}}的指令，模板的名称和文件名称一样，当然也可以重新定义模版的名字，或在一个模板文件中定义多个模板。

**讲了这么多还不知道模版文件怎么写呢。下面讲讲动作语法。**

如果要访问结构里的field，用：`{{ .Name }}`，或 `{{ .User.Name }}`

### 1. 条件动作

接受true或false，从而决定执行（显示）哪条语句。下面如果传入true，则会显示content1语句。

    {{ if . }}
        content1
    {{ else }}
        content2
    {{ end }}

### 2. 迭代动作

可对传入的数组、切片、映射、或通道进行迭代，迭代里，`.`会被设置为当前迭代的元素.`else`用于当迭代的结构为空或nil时显示备选结果。

    {{ range . }}
        <li>The Elem: {{ . }} </li>
    {{ end }}

    {{ range . }}
        <li>The Elem: {{ . }} </li>
    {{ else }}
        <li> No elems. </li>
    {{ end }}

### 3. 设置动作

类似if，`{{if pipeline}} T1 {{end}}`,如果pipeline为空（false，0，nil，空的array, slice, map,string），则执行T0，否则执行T1.同时with里面{{.}}会被设置为pipeline的值。这是和if唯一的区别。

    {{with pipeline}} T1 {{end}}
    {{with pipeline}} T1 {{else}} T0 {{end}}

可以用来判断当某个集合为空时，显示xxx话。当不为空时，在with里用dot显示这个集合里的东西。

### 4. 包含动作

可以在一个模板里包含另一个模板，用到`{{ template "name" }}`,或`{{template "name" pipeline}}`

第一个参数是模板的名字（如果没定义，那就是模板文件名称），第二个参数是要传给这下一个模板的参数。

通常这么用：`{{template "name" . }}`


还有些参数、变量、管道和函数这里先略过。


## 连接数据库

## 启动服务器