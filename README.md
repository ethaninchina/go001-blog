# 博客项目部署文档
#### 安装golang
```
wget https://dl.google.com/go/go1.14.2.linux-amd64.tar.gz
tar -C /usr/local -xf go1.14.2.linux-amd64.tar.gz

echo 'export PATH=$PATH:/usr/local/go/bin' >> /etc/profile
source /etc/profile
go version
--------------------------------
go version go1.14.2 linux/amd64
--------------------------------


查看golang源码
ll /usr/local/go/
------------------------------------------------------------
drwxr-xr-x  2 root root  4096 4月   9 03:15 api
-rw-r--r--  1 root root 55383 4月   9 03:15 AUTHORS
drwxr-xr-x  2 root root  4096 4月   9 03:17 bin
-rw-r--r--  1 root root  1339 4月   9 03:15 CONTRIBUTING.md
-rw-r--r--  1 root root 90098 4月   9 03:15 CONTRIBUTORS
drwxr-xr-x  7 root root  4096 4月   9 03:15 doc
-rw-r--r--  1 root root  5686 4月   9 03:15 favicon.ico
drwxr-xr-x  3 root root  4096 4月   9 03:15 lib
-rw-r--r--  1 root root  1479 4月   9 03:15 LICENSE
drwxr-xr-x 12 root root  4096 4月   9 03:15 misc
-rw-r--r--  1 root root  1303 4月   9 03:15 PATENTS
drwxr-xr-x  6 root root  4096 4月   9 03:17 pkg
-rw-r--r--  1 root root  1607 4月   9 03:15 README.md
-rw-r--r--  1 root root    26 4月   9 03:15 robots.txt
-rw-r--r--  1 root root   397 4月   9 03:15 SECURITY.md
drwxr-xr-x 47 root root  4096 4月   9 03:15 src
drwxr-xr-x 23 root root 12288 4月   9 03:15 test
-rw-r--r--  1 root root     8 4月   9 03:15 VERSION
------------------------------------------------------------
```


#### 安装gin框架
```
因为GFW拦截导致go get -u github.com/gin-gonic/gin超时


```
