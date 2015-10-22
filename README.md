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

##### Edit the main nginx configuration file:

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


##### Now add a config for kibana service

Some doc at https://www.digitalocean.com/community/tutorials/how-to-install-elasticsearch-logstash-and-kibana-4-on-ubuntu-14-04

```bash
vim /etc/nginx/conf.d/kibana.conf
```

```
# configuration file for kibana

server {
    #listen 8181 default_server;
    listen 9292;
    #access_log logs/server-access_log;
    access_log off;
    server_name _; # This is just an invalid value which will never trigger on a real hostname.

    # access for kibana
    auth_basic "Administrator Login";
    auth_basic_user_file /var/www/.htpasswd;

    location / {
            proxy_pass http://localhost:5601;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
    }
}
```


##### Now add a config for elasticsearch service

```bash
vim /etc/nginx/conf.d/elasticsearch.conf
```

```
# config file for elasticsearch

server {
    #listen 8181 default_server;
    listen 9393;
	
    #access_log logs/server-access_log;
    access_log off;
    #server_name _; # This is just an invalid value which will never trigger on a real hostname.
    #server_name localhost;

    # access elasticsearch
    location / {
        auth_basic "Administrator Login";
        auth_basic_user_file /var/www/.htpasswd;

	proxy_pass http://elasticsearch;
	proxy_http_version 1.1;
	proxy_set_header Connection "Keep-Alive";
	proxy_set_header Proxy-Connection "Keep-Alive";
    }
}
```



##### Now add a config for munin service

```bash
vim /etc/nginx/conf.d/munin.conf
```

```
# http://munin.readthedocs.org/en/latest/example/webserver/nginx.html

server {
    #listen 8181 default_server;
    listen 8181;
    #access_log logs/server-access_log;
    access_log off;
    server_name _; # This is just an invalid value which will never trigger on a real hostname.
    #server_name localhost;

    server_name_in_redirect off;
    root  /var/www/html/munin/;

    # access munin
    location / {
        auth_basic "Administrator Login";
        auth_basic_user_file /var/www/.htpasswd;
    }
# disabled because stub_status module is not supported in nginx openrety
#    location /nginx_status {
#        stub_status on;
#        access_log   off;
#        allow 127.0.0.1;
#        deny all;
#    }

    #location ^~ /munin-cgi/munin-cgi-graph/ {
    #    fastcgi_split_path_info ^(/munin-cgi/munin-cgi-graph)(.*);
    #    fastcgi_param PATH_INFO $fastcgi_path_info;
    #    fastcgi_pass unix:/var/run/munin/fastcgi-munin-graph.sock;
    #    include fastcgi_params;
    #}
}
```

##### Now reload nginx config without stopping the nginx server

```bash
service nginx reload
```


## Install elasticsearch

```bash
# configure package
vim /etc/yum.repos.d/elasticsearch.repo
```

```
[elasticsearch-1.7]
name=Elasticsearch repository for 1.7.x packages
baseurl=http://packages.elastic.co/elasticsearch/1.7/centos
gpgcheck=1
gpgkey=http://packages.elastic.co/GPG-KEY-elasticsearch
enabled=1
```

```bash
# install package
yum install elasticsearch

# check config
more /etc/init.d/elasticsearch

# add some plugins (as you wish)
/usr/share/elasticsearch/bin/plugin -install mobz/elasticsearch-head
/usr/share/elasticsearch/bin/plugin -install lukas-vlcek/bigdesk
/usr/share/elasticsearch/bin/plugin -install royrusso/elasticsearch-HQ
/usr/share/elasticsearch/bin/plugin -install elasticsearch/marvel/latest
```

```bash
# edit existing config file (change the original file)
vim /etc/elasticsearch/elasticsearch.yml
```

```
# somewhere in the file...

#################################### Paths ####################################

# Path to directory containing configuration (this file and logging.yml):
#
#path.conf: /path/to/conf
path.conf: /etc/elasticsearch

# Path to directory where to store index data allocated for this node.
#
#path.data: /path/to/data
path.data: /var/lib/elasticsearch

#
# Can optionally include more than one location, causing data to be striped across
# the locations (a la RAID 0) on a file level, favouring locations with most free
# space on creation. For example:
#
#path.data: /path/to/data1,/path/to/data2

# Path to temporary files:
#
#path.work: /path/to/work
path.work: /tmp/elasticsearch

# Path to log files:
#
#path.logs: /path/to/logs
path.logs: /var/log/elasticsearch

# Path to where plugins are installed:
#
#path.plugins: /path/to/plugins
path.plugins: /usr/share/elasticsearch/plugins

#
# somewhere else...
#

# http://slacklabs.be/2012/04/02/force-Elastic-search-on-ipv4-debian/
# https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-network.html
#network.host: 192.168.0.1
#network.host: localhost
network.host: 127.0.0.1
#network.host: _non_loopback:ipv4
```

```bash
# restart the elasticsearch service
service elasticsearch restart

# check elastic search server is ok
curl http://localhost:9200/_status?pretty=true

# start service at boot time
chkconfig elasticsearch on

# display tcp listen ports
netstat -putan | grep LISTEN
```

Some uri:

```bash
# uri:
# http://mypublichostname:9200/ is not accessible from outside but is now mapped to public tcp port 9393 (see nginx config)
# http://mypublichostname:9393/
# http://mypublichostname:9393/_status?pretty=true
# http://mypublichostname:9393/_plugin/HQ/
# http://mypublichostname:9393/_plugin/marvel
# http://mypublichostname:9393/_plugin/bigdesk
# http://mypublichostname:9393/_plugin/head/
```


## Install logstash

```bash
# install logstash
cd ~/tmp
mkdir logstash
cd logstash/

# download elastic search package
wget https://download.elastic.co/logstash/logstash/packages/centos/logstash-1.5.4-1.noarch.rpm

# install package
yum install logstash-1.5.4-1.noarch.rpm -y

# configure logstash
vim /etc/init.d/logstash
```

Should looks like something like this :

```
LS_USER=logstash
LS_GROUP=logstash
LS_HOME=/var/lib/logstash
LS_HEAP_SIZE="500m"
LS_LOG_DIR=/var/log/logstash
LS_LOG_FILE="${LS_LOG_DIR}/$name.log"
LS_CONF_DIR=/etc/logstash/conf.d
LS_OPEN_FILES=16384
LS_NICE=19
LS_OPTS=""
```

##### Configure file for nginx logger 

```bash
vim /etc/logstash/conf.d/nginx.conf
```

```
input {
	file {
		path => "/var/log/nginx/*access*"
	}
}

filter {
	mutate { replace => { "type" => "nginx_access" } }
	grok {
		patterns_dir => "/etc/logstash/patterns/"
		match => { "message" => "%{NGINXACCESS}" }
	}
	date {
		match => [ "timestamp" , "dd/MMM/YYYY:HH:mm:ss Z" ]
	}
	geoip {
		source => "clientip"
	}
}

output {
	elasticsearch {
		host => "localhost"
		protocol => "http"
	}
	stdout { codec => rubydebug }
}
```

```bash
# create pattern file
mkdir /etc/logstash/patterns/
vim /etc/logstash/patterns/nginx
```

```
NGUSERNAME [a-zA-Z\.\@\-\+_%]+
NGUSER %{NGUSERNAME}
NGINXACCESS %{IPORHOST:http_host} %{IPORHOST:clientip} \[%{HTTPDATE:timestamp}\] \"(?:%{WORD:verb} %{NOTSPACE:request}(?: HTTP/%{NUMBER:httpversion})?|%{DATA:rawrequest})\" %{NUMBER:response} (?:%{NUMBER:bytes}|-) %{QS:referrer} %{QS:agent} %{NUMBER:request_time:float} %{NUMBER:upstream_time:float}
NGINXACCESS %{IPORHOST:http_host} %{IPORHOST:clientip} \[%{HTTPDATE:timestamp}\] \"(?:%{WORD:verb} %{NOTSPACE:request}(?: HTTP/%{NUMBER:httpversion})?|%{DATA:rawrequest})\" %{NUMBER:response} (?:%{NUMBER:bytes}|-) %{QS:referrer} %{QS:agent} %{NUMBER:request_time:float}
```

```bash
# check it logs correctly
# start logstash manualy
# it should print log on the stdout
# ways about 10 seconds after having typing this line
# then after try to use a webbrowser or curl to call a web page to generate log
/opt/logstash/bin/logstash -f /etc/logstash/conf.d/nginx.conf

# if we see lines printing then it's ok
# then we can start the service using the right way
service logstash start

# ensure start service at boot time
chkconfig logstash on
```

Use this web site for debug:
https://grokdebug.herokuapp.com/


## Install kibana

```bash
# download and install
cd ~/tmp
mkdir kibana
cd kibana/
wget https://download.elastic.co/kibana/kibana/kibana-4.1.2-linux-x64.tar.gz
tar xvf kibana-4.1.2-linux-x64.tar.gz
sudo mkdir -p /usr/share/nginx/kibana4
sudo cp -R kibana-4.1.2-linux-x64/ /usr/share/nginx/kibana4/

# see the nginx section up above for this config file
vim /etc/nginx/conf.d/kibana.conf

# configure kibana for security
# https://www.digitalocean.com/community/tutorials/how-to-install-elasticsearch-logstash-and-kibana-4-on-ubuntu-14-04
vim /usr/share/nginx/kibana4/kibana-4.1.2-linux-x64/config/kibana.yml
# in the file replace with host: "localhost"
>>>
# The host to bind the server to.
#host: "0.0.0.0"
host: "localhost"
<<<

# executable file is
# /usr/share/nginx/kibana4/kibana-4.1.2-linux-x64/bin/kibana

# we can perform a simple test by running manualy the bianry file
/usr/share/nginx/kibana4/kibana-4.1.2-linux-x64/bin/kibana

# then user a web browser and go to
# http://mypublichostname:5601
# if a web page displays then it's ok
# stop the running process in unix shell by pressing ctrl+c

# run kibana in background
# warning!!! must be restarted again after server boot
nohup /usr/share/nginx/kibana4/kibana-4.1.2-linux-x64/bin/kibana &

# check it's working on localhost (private network)
curl http://127.0.0.1:5601/

# check it's NOT working using public address
curl http://mypublichostname:5601/

# check it's working using port 9292 with login and password
curl http://login:password@mypublichostname:9292/

# verify structure (change date to day date)
curl localhost:9200/logstash-2015.10.21/_mapping?pretty

# search for some doc entries
curl -XGET localhost:9200/logstash-2015.10.21/_search?*:*
```
