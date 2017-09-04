---
layout: post
title: centOs下nginx安装以及配置
excerpt: "记录关于centOs下nginx的安装以及配置"
modified: 2017-09-04
categories:  [服务器配置, nginx]
comments: true
---

# 简介
Nginx 是一个网页服务器，它能反向代理HTTP, HTTPS, SMTP, POP3, IMAP的协议链接，以及一个负载均衡器和一个HTTP缓存。

# 安装
直接通过 yum install nginx 肯定是不行的,因为yum没有nginx，所以首先把 nginx 的源加入 yum 中

```ruby
// 1.将nginx放到yum repro库中
rpm -ivh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
// 2.执行查询操作，这时候库应该有nginx了
yum info nginx
// 3.执行安装操作
yum install nginx
// 4.验证nginx已经安装
nginx -v
// 5.  启动nginx
service nginx start
//这时候通过公网ip直接访问就可以显示出nginx的映射了
```

# 配置
默认的配置文件在 /etc/nginx/nginx.conf中
但是不建议在这个文件中直接修改/etc/nginx/conf.d/目录中，可以新建自己的配置文件***.conf

配置文件的编写
```ruby
server {
    listen 8888;      //监听的端口号
    server_name example.com;    //映射的域名

    //代理映射   主要作用：把指定的端口映射到对应的文件上,这里运行example.com:8888将会自动加载/var/www/example/index.html文件
    location / {
    root  /var/www/example;    //映射的根目录
    index      index.html;        //映射的主页
    }

    //启用反向代理       主要作用：把服务器所监听的端口映射到nginx端口上，这里把服务器设置要代理的 uri比如（127.116.60:5200）映射到example.com:8888/api上
    location /api{
            proxy_pass http://ip:端口号;          #设置要代理的 uri
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Scheme $scheme;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $http_host;

            proxy_connect_timeout 10;            #nginx跟后端服务器连接超时时间(代理连接超时)
            proxy_send_timeout 15;                #后端服务器数据回传时间(代理发送超时)
            proxy_read_timeout 300;              #连接成功后，后端服务器响应时间(代理接收超时)

            client_max_body_size 10m;          #允许客户端请求的最大单文件字节数
            client_body_buffer_size 128k;     #缓冲区代理缓冲用户端请求的最大字节数，
    }
}


//配置完conf文件后可以执行 /usr/local/sbin/nginx -t  来查看配置文件是否正确,检查如果返回ok，则说明修改文件没有出现任何错误
//service nginx reload重新加载nginx配置,访问example.com:8888可以加载指定的deinx主页
```
