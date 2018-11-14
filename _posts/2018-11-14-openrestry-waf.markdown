---
layout: post
title: Openrestry（Nginx+Lua）实现自定义WAF
date:  2018-11-14 10:11:00 +0900  
description: web端轻量级安全防御实现
img: post-2.jpg # Add image post (optional)
tags: [Blog,security,web,openrestry]
author: Lichi # Add name author (optional)
security: true
web: true
---

# openrestry 简介:

- OpenResty® 是一个基于 Nginx 与 Lua 的高性能 Web 平台，其内部集成了大量精良的 Lua 库、第三方模块以及大多数的依赖项。用于方便地搭建能够处理超高并发、扩展性极高的动态 Web 应用、Web 服务和动态网关。

- OpenResty® 通过汇聚各种设计精良的 Nginx 模块（主要由 OpenResty 团队自主开发），从而将 Nginx 有效地变成一个强大的通用 Web 应用平台。这样，Web 开发人员和系统工程师可以使用 Lua 脚本语言调动 Nginx 支持的各种 C 以及 Lua 模块，快速构造出足以胜任 10K 乃至 1000K 以上单机并发连接的高性能 Web 应用系统。

- OpenResty® 的目标是让你的Web服务直接跑在 Nginx 服务内部，充分利用 Nginx 的非阻塞 I/O 模型，不仅仅对 HTTP 客户端请求,甚至于对远程后端诸如 MySQL、PostgreSQL、Memcached 以及 Redis 等都进行一致的高性能响应。

## openrestry 安装部署:

- 部署文档参考资料

> https://openresty.org/cn/installation.html
https://openresty.org/cn/linux-packages.html

> 我是在CentOS发行版下通过yum方式安装的，图个简单，毕竟openrestry的部署不是我们的重点。

## openrestry的WAF功能列表:

> 1. 支持IP白名单和黑名单功能，直接将黑名单的IP访问拒绝。
> 2. 支持URL白名单，将不需要过滤的URL进行定义。
> 3. 支持User-Agent的过滤，匹配自定义规则中的条目，然后进行处理（返回403）。
> 4. 支持CC攻击防护，单个URL指定时间的访问次数，超过设定值，直接返回403。
> 5. 支持Cookie过滤，匹配自定义规则中的条目，然后进行处理（返回403）。
> 6. 支持URL过滤，匹配自定义规则中的条目，如果用户请求的URL包含这些，返回403。
> 7. 支持URL参数过滤，原理同上。
> 8. 支持日志记录，将所有拒绝的操作，记录到日志中去。
> 9. 日志记录为JSON格式，便于日志分析，例如使用ELKStack进行攻击日志收集、存储、搜索和展示。

#### WAF实现基本原理:

> WAF实现 WAF一句话描述，就是解析HTTP请求（协议解析模块），规则检测（规则模块），做不同的防御动作（动作模块），并将防御过程（日志模块）记录下来。所以本文中的WAF的实现由五个模块(配置模块、协议解析模块、规则模块、动作模块、错误处理模块）组成。

## openrestry启用WAF配置:

*参考资料，感谢这两个项目人员的贡献*{: style="color: red"}

> https://github.com/loveshell/ngx_lua_waf
https://github.com/unixhot/waf

- 获取WAF实现的lua代码，并放到openrestry配置文件存放路径:

> PS: yum方式安装openrestry后，程序会在部署在:*/usr/local/openresty*{: style="color: red"}路径。

```bash
#git clone https://github.com/unixhot/waf.git
#cp -a ./waf/waf /usr/local/openresty/nginx/conf/
```

- 修改Nginx的配置文件，加入以下配置。在nginx.conf的*http*{: style="color: red"}段添加:

```bash
#WAF
lua_shared_dict limit 50m;
lua_package_path "/usr/local/openresty/nginx/conf/waf/?.lua";
init_by_lua_file "/usr/local/openresty/nginx/conf/waf/init.lua";
access_by_lua_file "/usr/local/openresty/nginx/conf/waf/access.lua";
```

- 根据需要修改WAF配置参数:

> 一般需要修改WAF日志默认存放路径/tmp/,其他配置项参数的修改参考注解即可，被拦截后的提醒页面内容根据需要自行定义：

![openrestry-waf]({{site.baseurl}}/assets/img/openrestry-waf.jpg)

- 验证配置和启动openrestry服务：

```bash
#/usr/local/openresty/nginx/sbin/nginx –t
#/usr/local/openresty/nginx/sbin/nginx
```

## nginx配置参考:

```bash
worker_processes  4;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  logs/access.log  main;
    sendfile        on;
    keepalive_timeout  65;
    lua_shared_dict limit 50m;
    lua_package_path "/usr/local/openresty/nginx/conf/waf/?.lua";
    init_by_lua_file "/usr/local/openresty/nginx/conf/waf/init.lua";
    access_by_lua_file "/usr/local/openresty/nginx/conf/waf/access.lua";
    server {
        listen       80;
        server_name  localhost;
        location / {
            root   html;
            index  index.html index.htm;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
include vhosts/*.conf;
}
```