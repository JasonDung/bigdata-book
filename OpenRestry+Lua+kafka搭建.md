# OpenRestry+Lua+kafka搭建
## 1. 搭建背景简介
目前市面上现有的绝大多数的大数据系统数据采集流程都是用如下方案处理：

* 第一层：日志服务器层
``` nginx、haproxy```

* 第二层：数据采集组件层
``` flume、fliebeat、logstash```

* 第三层：中间件层 
``` kafka 、redis、rabbitmq```

本文的目的是为了减少数据采集过程中一些不必要的层面，提高数据采集效率，避免运维成本。

## 2.openrestry安装
### 2.1 下载openrestry依赖
```  yum install readline-devel pcre-devel openssl-devel gcc  ```
### 2.2 编译安装openrestry
*  创建目录

```
mkdir /opt/software
mkdir /opt/module
cd /opt/software
```
* 下载包并编译

```
wget https://openresty.org/download/openresty-1.9.7.4.tar.gz 
tar -xzf openresty-1.9.7.4.tar.gz -C /opt/module/
cd /opt/module/openresty-1.9.7.4
./configure --prefix=/opt/openresty --with-luajit  --without-http_redis2_module --with-http_iconv_module
make && make install 
```
### 2.3 安装lua-resty-kafka模块
* 下载并解压模块

```
cd /opt/software/ 
wget https://github.com/doujiang24/lua-resty-kafka/archive/master.zip 
unzip master.zip -d /opt/module/ 
```
* 拷贝kafka相关依赖到openresty中

```
cp -rf /opt/module/lua-resty-kafka-master/lib/resty/kafka/ /opt/openresty/lualib/resty/
```

### 2.4 完成安装并验证

```
[root@bigdata-2 /]# cd /opt/openresty/
[root@bigdata-2 openresty]# ll
总用量 0
drwxr-xr-x.  2 root root  19 6月   4 06:57 bin
drwxr-xr-x.  6 root root  56 6月   4 06:57 luajit
drwxr-xr-x.  6 root root  70 6月   4 06:57 lualib
drwxr-xr-x. 12 root root 162 6月   4 07:13 nginx
```

## 3.对openrestry中的nginx文件进行配置
* 备份nginx.conf文件

``` cp nginx.conf nginx.conf.bak ```

* 编辑/opt/openresty/nginx/conf/nginx.conf文件

```
user  root;  #Linux的用户
worker_processes  auto;
worker_rlimit_nofile 100000;
 
#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;
 
#pid        logs/nginx.pid;
 
events {
    worker_connections  102400;
    multi_accept on;
    use epoll;
}
  
http {
    include       mime.types;
    default_type  application/octet-stream;
 
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
 
    access_log  /var/log/nginx/access.log  main;
 
    sendfile on;
 
    keepalive_timeout  65;
 
    underscores_in_headers on;
 
    gzip  on;
 
    include /opt/openresty/nginx/mycfg/nginx_kafka.conf; 
}
```
* 创建/opt/openresty/nginx/mycfg/nginx_kafka.conf文件

```
lua_package_path "/opt/openresty/lualib/resty/kafka/?.lua;;";
lua_package_cpath "/opt/openresty/lualib/?.so;;";
 
lua_shared_dict ngx_cache 128m;  # cache
lua_shared_dict cache_lock 100k; # lock for cache
 
server {
    listen       80; #监听端口
    server_name  196.168.35.11; #埋点域名，多个用空格隔开
    root         html; 
    lua_need_request_body on; #开启接受消息体
 
    access_log /var/log/nginx/message.access.log  main;
    error_log  /var/log/nginx/message.error.log  notice;
 
    location = /nginx/kafka {
        lua_code_cache on;
        empty_gif;
        client_max_body_size 50m;
        client_body_buffer_size 10m;
        charset utf-8;
        #default_type 'application/json';
        content_by_lua_file "/opt/openresty/nginx/lua/nginx_kafka.lua";#引用的lua脚本
    }
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root html;
    }
}
```

* 创建/opt/openresty/nginx/lua/nginx_kafka.lua文件

```
local producer = require("resty.kafka.producer")
local client = require("resty.kafka.client")
local cjson = require("cjson.safe")
 
-- kafka的集群信息，单机也是可以的
local broker_list = {
    {host = "192.168.35.11", port = 9092},
    {host = "192.168.35.10", port = 9092},
    {host = "192.168.35.12", port = 9092}
}

-- 请求体为空时，直接返回不处理
local body_data = ngx.req.get_body_data()
if not body_data then
ngx.log(ngx.ERR,"no request body found")
end

-- 客户端连接测试，连接不上kafka服务端直接返回错误 
local cli = client:new(broker_list)
local brokers, partitions = cli:fetch_metadata("test")
if not brokers then
	ngx.say("fetch_metadata failed, err:", partitions)
	return
end
ngx.say("brokers: ", cjson.encode(brokers), "; partitions: ", cjson.encode(partitions))

-- 构建消息体
local log_json = {}
log_json["remote_addr"] = ngx.var.remote_addr
log_json["time_stamp"] = ngx.now() #只到秒，到毫秒需x1000
log_json["req_data"] = ngx.req.get_body_data()
local sendMsg = cjson.encode(log_json)

-- 开始往kafka服务端异步发送消息
-- 同步方式：local bp = producer:new(broker_list)
local now_date = os.date("%Y%m%d%H",unixtime)
-- 按天生成topic，默认第一次请求没有会报错，第二次则自动创建topic
local topic = "testMessage_"..now_date

local producer_server = producer:new(broker_list)
local ok, err = producer_server:send(topic,nil, sendMsg)

if not ok then
   ngx.log(ngx.ERR, 'kafka send err:', err)
   ngx.say("400")   
elseif ok then
   ngx.say("200")
else
   ngx.say("未知错误")
end
```
## 4.kafka消费者验证
```
[root@bigdata-3 kafka_2.11-0.10.2.1]# bin/kafka-console-consumer.sh --zookeeper bigdata-1:2181 --from-beginning --topic testMessage_20200610
```


  

