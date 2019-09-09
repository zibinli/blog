
### 1 防盗链
相关配置：
> [valid_referers](http://nginx.org/en/docs/http/ngx_http_referer_module.html)

```
location ~* \.(gif|jpg|png)$ {
    # 只允许 192.168.0.1 请求资源
    valid_referers none blocked 192.168.0.1;
    if ($invalid_referer) {
       rewrite ^/ http://$host/logo.png;
    }
}
```

### 2 根据文件类型设置过期时间

```
location ~.*\.css$ {
    expires 1d;
    break;
}
location ~.*\.js$ {
	expires 1d;
    break;
}
location ~* \.(js|css|jpg|jpeg|gif|png)$ {
    expires 1h;
    break;
}

```

### 3 禁止访问某个目录
```
location ~* \.(txt|doc)${
    root /opt/htdocs/site/test;
    deny all;
}
```

### 4 静态资源访问
```
http {
    # 这个将为打开文件指定缓存，默认是没有启用的，max 指定缓存数量，
    # 建议和打开文件数一致，inactive 是指经过多长时间文件没被请求后删除缓存。
    open_file_cache max=204800 inactive=20s;

    # open_file_cache 指令中的inactive 参数时间内文件的最少使用次数，
    # 如果超过这个数字，文件描述符一直是在缓存中打开的，如上例，如果有一个
    # 文件在inactive 时间内一次没被使用，它将被移除。
    open_file_cache_min_uses 1;

    # 这个是指多长时间检查一次缓存的有效信息
    open_file_cache_valid 30s;

    # 默认情况下，Nginx的gzip压缩是关闭的， gzip压缩功能就是可以让你节省不
    # 少带宽，但是会增加服务器CPU的开销哦，Nginx默认只对text/html进行压缩 ，
    # 如果要对html之外的内容进行压缩传输，我们需要手动来设置。
    gzip on;
    gzip_min_length 1k;
    gzip_buffers 4 16k;
    gzip_http_version 1.0;
    gzip_comp_level 2;
    gzip_types text/plain application/x-javascript text/css application/xml;

    server {
        listen       80;
        server_name www.test.com;
        charset utf-8;
        root   /data/www.test.com;
        index  index.html index.htm;
    }
}
```

### 5 日志配置

#### 5.1 日志字段说明
字段 | 说明 |
--- | --- | ---
remote_addr 和 http_x_forwarded_for | 客户端 IP 地址
remote_user | 客户端用户名称
request | 请求的 URI 和 HTTP 协议
status | 请求状态
body_bytes_sent | 返回给客户端的字节数，不包括响应头的大小
bytes_sent | 返回给客户端总字节数
connection | 连接的序列号
connection_requests | 当前同一个 TCP 连接的的请求数量
msec | 日志写入时间。单位为秒，精度是毫秒
pipe | 如果请求是通过HTTP流水线(pipelined)发送，pipe值为“p”，否则为“.”
http_referer | 记录从哪个页面链接访问过来的
http_user_agent | 记录客户端浏览器相关信息
request_length | 请求的长度（包括请求行，请求头和请求正文）
time_iso8601 | ISO8601标准格式下的本地时间
time_local | 记录访问时间与时区

#### 5.1 access_log 访问日志
```
http {
    log_format  access  '$remote_addr - $remote_user [$time_local] $host "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for" "$clientip"';
    access_log  /srv/log/nginx/talk-fun.access.log  access;
}
```

#### 5.2 error_log 日志
```
error_log  /srv/log/nginx/nginx_error.log  error;
# error_log /dev/null; # 真正的关闭错误日志
http {
    # ...
}
```

### 6 反向代理
```
http {
    include mime.types;
    server_tokens off;

    ## 配置反向代理的参数
    server {
        listen    8080;

        ## 1. 用户访问 http://ip:port，则反向代理到 https://github.com
        location / {
            proxy_pass  https://github.com;
            proxy_redirect     off;
            proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }

        ## 2.用户访问 http://ip:port/README.md，则反向代理到
        ##   https://github.com/zibinli/blog/blob/master/README.md
        location /README.md {
            proxy_set_header  X-Real-IP  $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass https://github.com/zibinli/blog/blob/master/README.md;
        }
    }
}
```

### 7 负载均衡
```
http {
    upstream test.net {
        ip_hash;
        server 192.168.10.13:80;
        server 192.168.10.14:80  down;
        server 192.168.10.15:8009  max_fails=3  fail_timeout=20s;
        server 192.168.10.16:8080;
    }
    server {
        location / {
            proxy_pass  http://test.net;
        }
    }
}
```
