---
title: "docker搭建Joplin流程"
date: 2023-03-14T20:37:50+08:00
draft: false
categories:
    - 技术
---

Joplin是一个跨平台的开源笔记应用程序，支持Windows、macOS、Linux、Android和iOS。它具有强大的笔记管理功能，包括笔记的创建、编辑、笔记本的管理、标签的管理、笔记搜索等。

Joplin支持多平台同步，用户可以将笔记同步到所有设备上，包括桌面电脑、手机和平板电脑。官方提供免费Joplin Cloud进行笔记同步，也支持很多自建平台的笔记同步，如
- Dropbox
- OneDrive
- Nextcloud/ownCloud
- WebDAV
- SFTP
- Joplin Server

本文介绍自建Joplin Server。

Joplin默认使用SQLite，但是不支持多用户并发访问，所以我们使用postgres，我们用docker-compose组合创建和管理容器，在～下创建docker-compose.yml:

```docker
version: '3.1'

services:

  db:
    image: "postgres:13"
    restart: always
    environment:
      POSTGRES_USER: joplin
      POSTGRES_PASSWORD: joplinpwd
      POSTGRES_DB: joplin
  joplin:
    image: "joplin/server:latest"
    env_file:
      - .env
    ports:
      - "22300:22300"
```

复制[.env-sample](https://raw.githubusercontent.com/laurent22/joplin/dev/.env-sample)到～下的`.env`，这是joplin的配置文件。

修改
```
APP_BASE_URL=https://example.com/joplin
APP_PORT=22300
```
APP_BASE_URL可以不用https，/joplin是用来跳转的，在nginx中设置访问/joplin时进行反向代理到本地的22300端口。一般直接改域名即可（如果用http，这里一定要改为http）。

修改数据库配置数据，和上面对应好：

```
DB_CLIENT=pg
POSTGRES_PASSWORD=joplinpwd
POSTGRES_DATABASE=joplin
POSTGRES_USER=joplin
POSTGRES_PORT=5432
POSTGRES_HOST=db
```

## 配置nginx，修改/etc/nginx/nginx.conf
使用http，而不是https的话nginx配置文件这么写：

```nginx
server {
    server_name www.chajiuqqq.cn;
        listen 80;
    location /joplin/ {
        proxy_pass http://127.0.0.1:22300/;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_set_header X-Forwarded-Host $host;
        client_max_body_size 400M;
    }
}
```
 

 如果要用ssl，准备.pem和.key文件放到一个目录下， 这里以/root为例：

在Nginx中修改nginx.conf：
```nginx
server {
    server_name www.chajiuqqq.cn;
        listen 443 ssl;
        ssl_certificate /root/xxx.cn.pem;
        ssl_certificate_key /root/xxx.key;
    location /joplin/ {
        proxy_pass http://127.0.0.1:22300/;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_set_header X-Forwarded-Host $host;
        client_max_body_size 400M;
    }
}
```

注意两个地方结尾有/：
- `location /joplin/`
- `proxy_pass http://127.0.0.1:22300/;`
 
 ## 运行容器
 
在docker-compose.yml目录下运行`docker compose up`，访问
`https://example.com/joplin/login`，默认用户名admin@localhost，密码admin。

建议创建普通用户，在app上登陆使用同步功能。app或pc设置里选择同步，然后选最底下的joplin server，地址就是`https://example.com/joplin`


