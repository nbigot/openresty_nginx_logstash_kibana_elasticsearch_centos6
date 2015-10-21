# openresty_nginx_logstash_kibana_elasticsearch_centos6

Howto install (openresty + nginx + logstash + kibana + elasticsearch) on centos6

## Some links for help and inspiration:

http://www.bravo-kernel.com/2014/12/setting-up-logstash-1-4-2-to-forward-nginx-logs-to-elasticsearch/
https://www.youtube.com/watch?v=ge8uHdmtb1M
https://www.elastic.co/downloads/kibana
https://www.elastic.co/guide/en/elasticsearch/reference/1.4/setup-dir-layout.html
https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-plugins.html
https://www.elastic.co/blog/playing-http-tricks-nginx


## Install openresty

Let's suppose nginx is already installed on the machine.
We want an improved version of nginx with more modules.
We want the redis module, so we install openresty.

```bash
# display current nginx version
which nginx
nginx -V
sudo passenger-install-nginx-module
service nginx status
service nginx stop
sudo yum install readline-devel pcre-devel openssl-devel gcc -y
wget https://openresty.org/download/ngx_openresty-1.9.3.1.tar.gz
tar -xvzf ngx_openresty-1.9.3.1.tar.gz
cd ngx_openresty-1.9.3.1
./configure
gmake
gmake install

# display openresty/nginx version
/usr/local/openresty/nginx/sbin/nginx -V

more /usr/local/openresty/nginx/conf/nginx.conf

# change nginx binary path
sudo vim /etc/init.d/nginx

>>>
#nginx="/usr/sbin/nginx"
nginx="/usr/local/openresty/nginx/sbin/nginx"
<<<

service nginx restart

```

## Configure nginx

Edit the main nginx configuration file:

(note: this is an example but you realy should customize it for your own needs)

```bash
vim /etc/nginx/nginx.conf
```

```
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user              nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log;
#error_log  /var/log/nginx/error.log  notice;
#error_log  /var/log/nginx/error.log  info;

pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {

	# add an upstream for the redis backend
	upstream redisbackend {
		server 127.0.0.1:6379;
		# a pool with at most 1024 connections
		# and do not distinguish the servers:
		keepalive 1024;
	}

	# add an upstream for elasticsearch
	upstream elasticsearch {
		server 127.0.0.1:9200;
		keepalive 15;
	}

    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    # Custom logstash format
    log_format logstash '$http_host '
            '$remote_addr [$time_local] '
            '"$request" $status $body_bytes_sent '
            '"$http_referer" "$http_user_agent" '
            '$request_time '
            '$upstream_response_time';

    #access_log  /var/log/nginx/access.log  main;
    access_log /var/log/nginx/access.log logstash;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    gzip  on;
    gzip_min_length  1100;
    gzip_buffers  4 32k;
    gzip_types    text/plain application/x-javascript text/xml text/css;
    gzip_vary on;

    # Load config files from the /etc/nginx/conf.d directory
    # The default server is in conf.d/default.conf
    include /etc/nginx/conf.d/*.conf;

}
```


##### Now add a config for simple ping/pong echo service

```bash
vim /etc/nginx/conf.d/my-service-echo.conf
```

```
#
# simple test for service echo
#

server {
	listen       9090;

	location /echo {
		#root   html;
		#index  index.html index.htm;
		default_type 'text/html';
		echo "hello world!";
	}
}
```



##### Now add a config for redis getkey service

Don't use this one if you don't understand what it does!
It may introduce major security issues because it expose your redis data to the world!
This is for demo only.

```bash
vim /etc/nginx/conf.d/my-service-redis.conf
```

```
#
# test for service redis
#

server {
    listen       9091;

    location / {
        #debug
        #default_type 'text/html';
        #echo "debug: check key uri = $uri";
        #echo "debug: args = $args";
        #echo "debug: args param1= $arg_param1";

        #set $redis_key $uri;
        set $redis_key $arg_myrediskey;
        #refers to the redisbackend upstream configured in the file /etc/nginx/nginx.conf
        redis_pass     redisbackend;
        default_type   text/html;
        error_page     404 = /fallback;
    }

    location = /fallback {
        #proxy_pass backend;
        default_type 'text/html';
        echo "debug: key not found $uri";
    }
}
```

##### note: this is a benchmark command for redis and nginx perfs

First use the command redis-cli to set a new key named mykey1

```bash
redis-cli
```

Then type

```
redis 127.0.0.1:6379> set myrediskey "it works"
OK
redis 127.0.0.1:6379> exit
```

Then run the benchmark

```bash
ab -n 1000 -c 2 http://127.0.0.1:9091/?myrediskey=mykey1
```

The result looks like:

```
This is ApacheBench, Version 2.3 <$Revision: 655654 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 127.0.0.1 (be patient)
Completed 100 requests
Completed 200 requests
Completed 300 requests
Completed 400 requests
Completed 500 requests
Completed 600 requests
Completed 700 requests
Completed 800 requests
Completed 900 requests
Completed 1000 requests
Finished 1000 requests


Server Software:        openresty/1.9.3.1
Server Hostname:        127.0.0.1
Server Port:            9091

Document Path:          /?myrediskey=mykey1
Document Length:        11 bytes

Concurrency Level:      2
Time taken for tests:   0.354 seconds
Complete requests:      1000
Failed requests:        0
Write errors:           0
Total transferred:      158158 bytes
HTML transferred:       11011 bytes
Requests per second:    2826.83 [#/sec] (mean)
Time per request:       0.708 [ms] (mean)
Time per request:       0.354 [ms] (mean, across all concurrent requests)
Transfer rate:          436.61 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.5      0       6
Processing:     0    0   0.3      0       4
Waiting:        0    0   0.3      0       4
Total:          0    1   0.6      0       7
WARNING: The median and mean for the total time are not within a normal deviation
        These results are probably not that reliable.

Percentage of the requests served within a certain time (ms)
  50%      0
  66%      1
  75%      1
  80%      1
  90%      1
  95%      2
  98%      3
  99%      4
 100%      7 (longest request)
```
