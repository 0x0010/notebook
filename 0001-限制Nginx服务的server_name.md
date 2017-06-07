---
title: First Article Testing
date: 2017/6/6 23:11:00
id: 20170001
---

在写个人网站时碰到这么一个问题。
比如，我有两个域名0x0010.com和iamdigger.com。在测试阶段我只想让test.0x0010.com和test.iamdigger.com两个二级域名可以正常访问服务，其他的所有域名都指向无意义的静态目录。

<!-- more -->

nginx服务的server配置中提供了server_name选项，可以用它来指定域名。
````shell
    server {
        listen       80;
        server_name  _;
    
        #charset koi8-r;
    
        access_log  logs/default.80.access.log  main;
    
        location / {
            root html;
            index index.html index.htm;
        }
    }

    server {
        listen       80;
        server_name  test.iamdigger.org;
    
        #charset koi8-r;
    
        access_log  logs/iamdigger.80.access.log  main;
    
        location / {
            root html;
            index index.html index.htm;
        }
    }

    server {
        listen       80;
        server_name  test.0x0010.org;
        
        access_log  logs/0x0010.80.access.log  main;

        location / {
            proxy_pass   http://127.0.0.1:8080;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
````
以上代码是配置了三个Server同时监听同一个端口80。当Nginx接收到请求时会优先匹配完全一致的server_name。如果匹配不到，会转向通配的server。

举个例子：
+ http_host是test.iamdigger.org/in/something， 则会匹配test.iamdigger.org的Server。
+ http_host是test.0x0010.org/in/something，则会匹配test.0x0010.org的Server。
+ 其他的任何情况只能匹配server_name是_的Server。
