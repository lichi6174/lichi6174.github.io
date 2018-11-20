---
layout: post
title: nginx缓存和缓存清理
date:  2018-11-20 10:11:00 +0900  
description: nginx代理缓存以及缓存清理配置
img: post-11.jpg # Add image post (optional)
tags: [Blog,web,nginx]
author: Lichi # Add name author (optional)
web: true
---

## nginx缓存配置示例

- nginx主配置文件中设置nginx缓存配置信息：

> nginx.conf

```bash
http {
    ...
    proxy_temp_path /dev/shm/cache/proxy_temp_dir;
    proxy_cache_path /dev/shm/cache/proxy_cache_dir levels=1:2 keys_zone=cache_one:512m inactive=30m max_size=5g;
    ....
    
    
}
```

- 虚拟主机配置文件简单示例：
> *针对静态资源部分进行缓存*{: style="color: red"}

> /usr/local/nginx/conf/vhosts/abc.xyz.com.conf

```bash
server {
        listen 80;
        server_name abc.xyz.com;
        access_log  /usr/local/nginx/logs/abc.xyz.access.log  main;
        location / {
                proxy_set_header        X-Real-IP $remote_addr;
                proxy_set_header        X-Forwarded-Host $host;
                proxy_set_header        x-agent $http_user_agent;
                proxy_set_header        X-Forwarded-Server $host;
                proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header        Host $http_host;
                proxy_set_header        X-For $http_x_forwarded_for;
                proxy_pass http://warehouse;
        }

        location ~ ^/$ {
                root /home/server/nginx/test;
                index index.html;
        }


        location ~*\.(js|html|gif|jpg|jpeg|png|css|ico) {
                proxy_pass http://127.0.0.1:90;
                proxy_cache  cache_one;
                proxy_cache_key "$host:$server_port$request_uri";
                proxy_set_header Host $host;
                proxy_set_header X-Forwarded-For $remote_addr;
                proxy_cache_valid  200 20m;
                add_header X-Cache $upstream_cache_status;
        }
}

server {
        listen 127.0.0.1:90;
        server_name abc.xyz.com;
        access_log  /usr/local/nginx/logs/abc.xyz.access.log  main;

        location ~*\.(js|html|gif|jpg|jpeg|png|css|ico) {
                root /home/server/nginx/test;
                index index.html;
        }
}
```

## 以上配置文件部分参数说明

> 1. `proxy_temp_path` 缓存临时目录，后端的响应并不直接返回客户端，而是先写到一个临时文件中，然后被rename一下当做缓存放在 proxy_cache_path。
> 2. `proxy_cache_path` 设置缓存目录，目录里的文件名是`cache_key`的MD5值。
> 3. `levels=1:2` 默认所有缓存文件都放在同一个/path/to/cache下，但是会影响缓存的性能，因此通常会在/path/to/cache下面建立子目录用来分别存放不同的文件。假设levels=1:2，Nginx为将要缓存的资源生成的key为f4cd0fbc769e94925ec5540b6a4136d0，那么key的最后一位0，以及倒数第2-3位6d作为两级的子目录，也就是该资源最终会被缓存到/path/to/cache/0/6d目录中。
> 4. `key_zone` 在共享内存中设置一块存储区域来存放缓存的key和metadata（类似使用次数），这样nginx可以快速判断一个request是否命中或者未命中缓存，1m可以存储8000个key，10m可以存储80000个key。
> 5. `max_size` 最大cache空间，如果不指定，会使用掉所有disk space，当达到配额后，会删除最少使用的cache文件。
> 6. `inactive` 未被访问文件在缓存中保留时间，本配置中如果60分钟未被访问则不论状态是否为expired，缓存控制程序会删掉文件。inactive默认是10分钟。需要注意的是，inactive和expired配置项的含义是不同的，expired只是缓存过期，但不会被删除，inactive是删除指定时间内未被访问的缓存文件。
> 7. `use_temp_path` 如果为off，则nginx会将缓存文件直接写入指定的cache文件中，而不是使用temp_path存储，official建议为off，避免文件在不同文件系统中不必要的拷贝。
> 8. `proxy_cache` 启用proxy cache，并指定key_zone。另外，如果proxy_cache off表示关闭掉缓存。
> 9. `proxy_cache_key` 定义cache_key,nginx对缓存的资源会设置一个key，NGINX生成的键的默认格式是类似于下面的NGINX变量的MD5哈希值:$scheme$proxy_host$request_uri，实际的算法有些复杂。 为了改变变量（或其他项）作为基础键，可以使用proxy_cache_key命令。
> 10. `proxy_cache_valid` 为不同的HTTP返回状态码的资源设置不同的缓存时长。
> 11. `proxy_cache_purge` 缓存清理指令，后面会用得上。

## 缓存清理两种方式
### 方式一
#### nginx缓存批量清理脚本（网上找的，挺好用的，多谢作者！）

- nginx_cache_clean.sh

```bash
#!/bin/bash  
#Auto Clean Nginx Proxy_Cache Shell Scripts  

echo -e "\n\n"  
echo -n -e "\e[35;1m请输入Nginx Proxy_cache缓存的具体路径(友情提示:可以使用Tab补全功能哦!)\e[0m\e[34;5m:\e[0m"  
read -e path  
CACHE_DIR=$path  
echo -e "\e[32;1m----------------------------------------------------------------\e[0m"  
echo -e "\e[32;1m----------------------------------------------------------------\e[0m"  
echo -n -e "\e[32;1m请输入你要删除的动作\n1.按文件类型删除\t2.按具体文件名删除\t3.按文件目录删除\n:"  
read action  
     case $action in  
1)  
echo -e "\e[32;1m----------------------------------------------------------------\e[0m"  
echo -e "\e[32;1m----------------------------------------------------------------\e[0m"  
echo -n -e "\e[34;1m 请输入你要删除的缓存文件类型(可以输入多个参数空格隔开)\e[0m\e[34;5m:\e[0m"  
read -a FILE  
for i in `echo ${FILE[*]}|sed 's/ /\n/g'`  
do  
grep -r -a  \.$i ${CACHE_DIR}| awk 'BEGIN {FS=":"} {print $1}'  > /tmp/cache_list.txt  
 for j in `cat /tmp/cache_list.txt`  
do  
   rm  -rf  $j  
   echo "$i  $j 删除成功!"  
 done  
done  
;;  
2)  
echo -e "\e[32;1m----------------------------------------------------------------\e[0m"  
echo -e "\e[32;1m----------------------------------------------------------------\e[0m"  
echo -n -e "\e[33;1m 请输入你要删除的缓存文件具体名称(可以输入多个参数空格隔开)\e[0m\e[34;5m:\e[0m"  
read -a FILE  
for i in `echo ${FILE[*]}|sed 's/ /\n/g'`  
do  
grep -r -a  $i ${CACHE_DIR}| awk 'BEGIN {FS=":"} {print $1}'  > /tmp/cache_list.txt  
 for j in `cat /tmp/cache_list.txt`  
do  
   rm  -rf  $j  
   echo "$i  $j 删除成功!"  
 done  
done  
;;  
3)  
echo -e "\e[32;1m----------------------------------------------------------------\e[0m"  
echo -e "\e[32;1m----------------------------------------------------------------\e[0m"  
echo -n -e "\e[33;1m支持的模式有:\n1.清除网站store目录下的所有缓存:test.dd.com/data/upload/shop/store\n2.清除网站shop下的所有缓存:test.dd.com/data/upload/shop\e[0m\n"  
echo -n -e "\e[34;1m 请输入你要删除的缓存文件具体目录\e[0m\e[34;5m:\e[0m"  
read -a FILE  
for i in `echo ${FILE[*]}|sed 's/ /\n/g'`  
do  
grep -r -a  "$i" ${CACHE_DIR}| awk 'BEGIN {FS=":"} {print $1}'  > /tmp/cache_list.txt  
 for j in `cat /tmp/cache_list.txt`  
do  
   rm  -rf  $j  
   echo "$i  $j 删除成功!"  
 done  
done  
;;  
*)  
echo "输入错误,请重新输入"  
;;  
esac
```
- 脚本操作演示

![nginx-cache]({{site.baseurl}}/assets/img/nginx-cache.jpg)


> 上面脚本执行后，会提示输入cache的缓存目录，然后选择删除缓存文件的条件（这里我选择"按文件类型删除"），选择了删除html 、htm、js、css、jpg 、gif、 png 、jpeg 、bmp 、flv、 swf 、ico这12中文件格式的缓存文件。
> 或者直接使用find命令查找缓存目录下的文件，直接将文件全部删除:
> # find /path/to/cache -type f|xargs rm -f 


### 方式二
#### nginx添加ngx_cache_purge-2.3缓存清理模块

- 获取nginx缓存清理模块并解压

```bash
#cd /root
#wget http://labs.frickle.com/files/ngx_cache_purge-2.3.tar.gz
#tar xf ngx_cache_purge-2.3.tar.gz

```

- 查看原nginx编译安装时的命令，安装了哪些模块

> #/usr/local/nginx/sbin/nginx -V

- 加入需要安装的模块，--add-module=/root/ngx_cache_purge-2.3

> *进入到nginx源码包目录下输入以下命令*{: style="color: red"}

> *输入make进行编译，千万不要make install 因为会覆盖原来已经安装好的内容，另外，编译必须没错误才行*{: style="color: red"}

```bash
#cd /root/nginx-1.12.2
#./configure --prefix=/usr/local/nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie' --add-module=/root/ngx_cache_purge-2.3
#make
```

- 验证nginx新加模块是否安装成功

> #/root/nginx-1.12.2/objs/nginx -V，查看ngx_cache_purge是否安装成功。

> ![nginx-config]({{site.baseurl}}/assets/img/nginx-config.jpg)

- 停止原nginx服务，替换nginx二进制文件，重启nginx服务

> #/usr/local/nginx/sbin/nginx -s stop
> #cp /usr/local/nginx/sbin/nginx /usr/local/nginx/sbin/nginx.bak
> #cp /root/nginx-1.12.2/objs/nginx /usr/local/nginx/sbin/nginx
> #/usr/local/nginx/sbin/nginx

- 利用此模块删除缓存

> 方法：

> 访问图片资源访，并查看缓存命中情况：
> http://abc.xyz.com/static/img/select.png
> ![nginx-hit]({{site.baseurl}}/assets/img/nginx-hit.png)

> 通过purge模块删除此图片nginx代理缓存：
> http://abc.xyz.com/purge/static/img/select.png
> ![nginx-purge]({{site.baseurl}}/assets/img/nginx-purge.png)

- *purge删除缓存报404错误可能原因*{: style="color: red"}

> 1. location purge的顺序问题导致
> 2. proxy_cache_key设置问题：

![nginx-error]({{site.baseurl}}/assets/img/nginx-error.png)

- 配置示例

> abc.xyz.com.conf

```bash
server {
        listen 80;
        server_name abc.xyz.com;
        access_log  /usr/local/nginx/logs/abc.xyz.access.log  main;
  
        location / {
                proxy_set_header        X-Real-IP $remote_addr;
                proxy_set_header        X-Forwarded-Host $host;
                proxy_set_header        x-agent $http_user_agent;
                proxy_set_header        X-Forwarded-Server $host;
                proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header        Host $http_host;
                proxy_set_header        X-For $http_x_forwarded_for;
                proxy_pass http://warehouse;
        }

  location ~ /purge(/.*) {
          allow       127.0.0.1;
          allow       192.168.1.0/24;
          deny        all;
          proxy_cache_purge cache_one $host$1$is_args$args;
      }
  
  location ~ ^/$ {
          root /home/server/nginx/test;
          index index.html;
      } 


  location ~*\.(js|html|gif|jpg|jpeg|png|css|ico)$ {
          proxy_pass http://127.0.0.1:90;
          proxy_cache  cache_one;
          proxy_cache_key $host$uri$is_args$args;
          proxy_set_header Host $host;
          proxy_set_header X-Forwarded-For $remote_addr;
          proxy_cache_valid  200 20m;
          add_header X-Cache $upstream_cache_status;
        }
  
}

server {
        listen 127.0.0.1:90;
        server_name abc.xyz.com;
        access_log  /usr/local/nginx/logs/abc.xyz.access.log  main;


  location ~*\.(js|html|gif|jpg|jpeg|png|css|ico)$ {
            root /home/server/nginx/test;
            index index.html;
        }
}

```

## Nginx禁用html、js、css缓存

```bash
在本地开发的时候，经常会碰到缓存引起的莫名其妙的问题，最暴力的方式就是清掉浏览器的缓存，或者使用Ctrl + F5，Shift + F5强制刷新页面。
有时候按了好几下，缓存还是清不掉，只能暂时禁用浏览器静态资源缓存了，配置如下：
location ~.*\.(js|css|html|png|jpg)$
{
    add_header Cache-Control no-cache;
}

或者
location /js
{
    add_header Cache-Control no-cache;
}

现在，按F5就行了！
```