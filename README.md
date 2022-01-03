# `openresty`实现隧道代理

按照惯例开头先来两句废话

> What?
>
> 本项目在proxy_pool项目基础上使用`openresty`服务达到隧道代理的功能
> 
> proxy_pool项目地址：https://github.com/jhao104/proxy_pool.git

> Q：隧道代理与常规代理的不同之处是什么
>
> A：隧道代理是一种代理IP存在方式，一般是代理IP。与传统的固定代理IP相比，它的特殊之处在于它会在代理服务器上自动更改IP，这样每个请求都会使用不同的IP
>
> Q：什么是`openresty`
>
> A：OpenResty® 是一个基于 [Nginx](https://openresty.org/cn/nginx.html) 与 Lua 的高性能 Web 平台，其内部集成了大量精良的 Lua 库、第三方模块以及大多数的依赖项。用于方便地搭建能够处理超高并发、扩展性极高的动态 Web 应用、Web 服务和动态网关。

## 创建下面三个文件

## `docker-compose.yml`

``` yaml
version: '3.5'
services:
  proxy_pool:
    build: .
    container_name: proxy_pool
    ports:
      - "5010:5010"
    depends_on:
      - proxy_redis
    restart: always
    environment:
      DB_CONN: "redis://@proxy_redis:6379/0"
    networks:
      proxy_network:
        ipv4_address: 192.168.112.2

  proxy_redis:
    image: "redis"
    container_name: proxy_redis
    restart: always
    networks:
      proxy_network: 
        ipv4_address: 192.168.112.3

  proxy_nginx: 
    build:
      context: .
      dockerfile: Nginx
    restart: always
    depends_on: 
      - proxy_redis
    ports:
      - "8888:80"
    container_name: proxy_nginx
    networks:
      proxy_network: 
        ipv4_address: 192.168.112.112


networks:
  proxy_network:
    name: "proxy_network"
    driver: "bridge"
    ipam:
      config:
        - subnet: 192.168.112.0/24
```

## `Nginx`

```
FROM openresty/openresty:1.19.9.1-4-bionic

RUN sed -i "s/archive.ubuntu.com/mirrors.aliyun.com/g" /etc/apt/sources.list

COPY nginx.conf /usr/local/openresty/nginx/conf/nginx.conf

CMD ["/usr/local/openresty/bin/openresty", "-g", "daemon off;"]
```

## `nginx.conf`

```nginx
worker_processes  16;
error_log /usr/local/openresty/nginx/logs/perror.log;
events {
    worker_connections 1024;
}
stream {

    log_format tcp_proxy '$remote_addr [$time_local] '
                         '$protocol $status $bytes_sent $bytes_received '
                         '$session_time "$upstream_addr" '
                         '"$upstream_bytes_sent" "$upstream_bytes_received" "$upstream_connect_time"';

    access_log /usr/local/openresty/nginx/logs/paccess.log tcp_proxy;
    open_log_file_cache off;


    upstream backend{
        server 106.52.172.214:8088;
        balancer_by_lua_block {

            local balancer = require "ngx.balancer"
            local host = ""
            local port = 0
            host = ngx.ctx.proxy_host
            port = ngx.ctx.proxy_port
            -- 设置 balancer
            local ok, err = balancer.set_current_peer(host, port)
            -- local ok=0
            if not ok then
                ngx.log(ngx.ERR, "failed to set the peer: ", err)
            end
        }
    }
    server {
        preread_by_lua_block{

                local redis = require "resty.redis"
                local red = redis:new()
                red:set_timeouts(1000, 1000, 1000)
                local ok, err = red:connect("192.168.112.3", 6379)
                
                if not ok then
                    ngx.log(ngx.ERR,"failed to connect: ", err)
                    return red:close()
                end
                
                local rkey = "use_proxy"
                local res, err = red:hkeys(rkey)
                if not res then
                        ngx.log(ngx.ERR,"res num error : ", err)
                        return red:close()
                end

                local radmnum = math.randomseed(tonumber(tostring(ngx.now()):reverse():sub(1, 6)))
                local proxy = res[math.random(#res)]
                -- ngx.log(ngx.ERR,"res num : ", proxy)
                local colon_index = string.find(proxy, ":")
                local proxy_ip = string.sub(proxy, 1, colon_index - 1)
                local proxy_port = string.sub(proxy, colon_index + 1)
                ngx.log(ngx.ERR,"redis data = ", proxy_ip, ":", proxy_port);
                ngx.ctx.proxy_host = proxy_ip
                ngx.ctx.proxy_port = proxy_port

                local ok, err = red:close()
                if not ok then
                     ngx.log(ngx.ERR,"failed to close: ",tostring(err))
                     return
                end

}

       listen 0.0.0.0:80;
       proxy_connect_timeout 3s;
       proxy_timeout 10s;
       proxy_pass backend;
   }
}
```

1. 其中`rkey`值在`redis-cli`中使用`keys *`

2. 由于`redis`采用默认配置并未设置密码所以`resty.redis`的连接并未采用身份验证，同时切记，一点不要把`redis`端口映射出来，除非你的`redis`已做身份验证

3. 如果你的`redis`有身份验证，只需要在上面的`nginx.conf`中`local res, err = red:hkeys(rkey)`的下方添加下面代码
```nginx
-- pass参数为你的redis连接密码
local pass = ""
local res, err = red:auth(pass)
if not res then
    ngx.log(ngx.ERR,"failed to authenticate: ", err)
    return
end
```

## 下载proxy_pool

```bash
git clone https://github.com/jhao104/proxy_pool.git
```

将上面创建的三个文件放在proxy_pool根目录下

docker-compose up 启动容器

### python 测试代码

```python
import requests
import time

proxies={"http":"http://IP:8888"}
for i in range(20):
    try:
        res = requests.get("http://httpbin.org/ip",headers = {"Connection":"close"},proxies=proxies)
        if res.status_code == 200:
            print(res.status_code,res.text)
    except:
        pass
    time.sleep(5)
```

![image-20220102152115518](https://gitee.com/anyewuxin/img/raw/master/img/image-20220102152115518.png)

恭喜现在你已经在疯狂乱跳了

