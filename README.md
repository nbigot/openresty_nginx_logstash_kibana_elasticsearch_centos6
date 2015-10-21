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

