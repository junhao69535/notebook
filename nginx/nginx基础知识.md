# nginx基础知识

以下内容只介绍开源nginx，需要收费nginx的内容可以参考官方文档。

## 由例子开始
### 配置文件结构
nginx的模块由配置文件中的指令控制。指令分为两种：
* 简单指令：由参数名和参数值组合成的指令，参数名和参数值之间用空格隔开，且以分号（；）结束该指令。
* 块指令：由一对大括号（{和}）指定的指令。内部可以再嵌入指令，这称为上下文。例如，http指令在main上下文，server指令在http上下文，location在server上下文等。


### 提供静态内容
不同的server块通过监听的端口和server_names区别。如果不提供端口，默认为80。

使用root指令提供静态内容。如果`root`指令在server中，那么成为所有不提供`root`指令的`location`块的默认`root`指令。

如下例子：

```
server {
    location / {
        root /data/www;
        }

    location /images/ {
        root /data;
        }
}
```

`location`块指定uri，采用最长前缀匹配原则，如"/images/"不会匹配"/"。`root`指令使得uri会加到`root`指令指定的路径，如上面的"/"uri，会被加到/data/www，而"/images/"uri，会被加到/data/images。默认会寻找名为"index.html"的文件作为默认的提供内容提供。


### 配置一个简单的代理服务器
配置一个基本的代理服务器，除了对images的请求之外，其他所有请求都转发给代理服务器。

因为每个server块都一个nginx实例，所以可以利用一个server块作为被代理的服务器，如下：

```
server {
    listen 8080;
    root /data/up1;

    location / {
    }
}
```

另开一个server块，并使用`proxy_pass`指令指定被代理服务器：

```
server {
    location / {
        proxy_pass http://localhost:8080;
    }

    location /images/ {
        root /data;
    }
}
```

这样，所有非"/images/"的请求都会转发给被代理服务器。

`location`指令指定的匹配路径能使用正则表达式，需要以"~"开头，如下：

```
server {
    location / {
        proxy_pass http://localhost:8080;
    }

    location ~ \.(gif|jpg|png)$ {
        root /data/images;
    }
}
```

这样，当匹配到.gif、.jpg或.png结尾的都会在加到"/data/images"。如请求"http://localhost/abc.png"实际上是到/data/images目录找abc.png。

**注意`location`指令指定的路径都会被加到`root`指令后面，因此，假设是上面的配置，如果请求是"http://localhost/static/abc.png"，实际上是到/data/images/static目录找abc.png，并不是在/data/images目录找abc.png，因此会找不到该文件，返回404。**


### 配置FastCGI代理
和`proxy_pass`指令代理差不多，但需要使用的`fastcgi_pass`指令。

```
server {
    location / {
        fastcgi_pass http://localhost:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param QUERY_STRING $query_string;
    }

    location ~ \.(gif|jpg|png)$ {
        root /data/images;
    }
}
```


## 控制nginx进程
### 主进程和工作进程
nginx由一个主进程和多个工作进程。

主进程的工作是读取配置文件和操纵工作进程。

工作进程是实际处理请求的进程，工作进程的个数由配置文件`worker_processes`指令指定。

### 控制nginx
可通过以下命令控制nginx：

```
nginx -s <SIGNAL>
```

其中`<SIGNAL>`是以下几种：
* quit：优雅关闭，等待处理完已经接收的请求再关闭
* reload：重加载配置文件
* reopen：重开日志文件
* stop：立刻关闭


## 创建配置文件
默认配置文件会在/etc/nginx或/usr/local/nginx目录下，如果编译时指定了，则可通过`nginx -V`查看：

```
[root@pg_server1 ~]# nginx -V
nginx version: nginx/1.17.1
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-11) (GCC)
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/usr/share/nginx --sbin-path=/usr/sbin/nginx --conf-path=/etc/nginx/nginx.conf --lock-path=/var/lock/nginx.lock --user=nginx --group=nginx --with-http_ssl_module --with-http_stub_status_module --with-http_gzip_static_module --with-pcre --with-http_v2_module
```

可以看出，配置文件被指定在/etc/nginx目录下。

### 指令
指令分为简单指令和其他指令：
* 简单指令：由指令和它的参数以及表示结束的分号组成，指令和参数之间用空格分开。
* 其他指令：这种指令像一个容器，用一对大括号（称为块）包含了其他指令。

### 特定功能配置文件
可以通过`include1`指令包含其他配置文件：

```
include conf.d/http;
include conf.d/stream;
include conf.d/exchange-enhanced;
```

### 上下文
以下是一些顶层指令，他们都是在main上下文：
* events：通用连接处理
* http：http传输
* mail：邮件传输
* stream：TCP和UDP传输

指令放在上面顶层指令之外的都属于在main上下文。

#### 虚拟服务器
在每一个传输处理上下文，可以包含一个或多个`server`块来定义虚拟服务器以达到控制处理请求的目的。

例如，在`http`上下文，每一个`server`指令控制特定域或ip地址的资源请求处理。一个或多个在`server`上下文中`location`上下文定义怎么处理特定的URIs。

#### 多上下文的配置文件示例

```
user nobody; # a directive in the 'main' context

events {
    # configuration of connection processing
}

http {
    # Configuration specific to HTTP and affecting all virtual servers

    server {
        # configuration of HTTP virtual server 1
        location /one {
            # configuration for processing URIs starting with '/one'
        }
        location /two {
            # configuration for processing URIs starting with '/two'
        }
    }

    server {
        # configuration of HTTP virtual server 2
    }
}

stream {
    # Configuration specific to TCP/UDP and affecting all virtual servers
    server {
        # configuration of TCP virtual server 1
    }
}
```

#### 继承
子上下文会自动继承父上下文的指令，同时子上下文可以重写父上下文的指令。


## 负载均衡
### 代理HTTP流量到一组servers
首先需要在`http`上下文通过`upstream`指令定义一组servers。

在`upstream`指令里配置servers需要使用`server`指令（这个指令和定义virtual server的server块不同）。例如，下面配置定义了一个名为backend的组，其中包含三个server配置：

```
http {
    upstream backend {
        server backend1.example.com weight=5;
        server backend2.example.com;
        server 192.0.0.1 backup;
    }
}
```

为了传递请求到一个server组，那么可以通过`proxy_pass`指令指定（或者`fastcgi_pass`，`memcached_pass`，`scgi_pass`或`uwsgi_pass`指令，根据对应的协议）。例如，在下面，一个virtual server传递所有请求到backend upstream组：

```
server {
    location / {
        proxy_pass http://backend;
    }
}
```

综上，结合起来配置如下：

```
http {
    upstream backend {
        server backend1.example.com;
        server backend2.example.com;
        server 192.0.0.1 backup;
    }

    server {
        location / {
            proxy_pass http://backend;
        }
    }
}
```

其中`server 192.0.0.1 backup`被指定为后备服务器，只要backend upstream组里所有非backup服务器都不可用时，backup服务器才会可用。


### 选择负载均衡算法
开源的nginx只支持四种负载均衡算法：

#### Round Robin
轮询算法（默认算法），根据权重（weights）进行轮询

```
upstream backend {
   # 不指定默认为轮询算法
   server backend1.example.com;
   server backend2.example.com;
}
```

#### Least Connetions
最小连接算法，根据权重（weights）请求会被传递到活动连接最小的服务器

```
upstream backend {
    least_conn;
    server backend1.example.com;
    server backend2.example.com;
}
```

#### IP HASH
根据client ip进行hash计算将请求传递给对应的服务器，即会计算ipv也会计算ipv6。这个算法保证相同的client ip会被分配到相同的服务器，除非该服务器不可用。


```
upstream backend {
    ip_hash;
    server backend1.example.com;
    server backend2.example.com;
}
```

如果某一个服务器需要临时移除，则可以标记`down`参数，这样可以保留当前client ip的hash值，请求会自动地传递到组中的下一个server

```
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com down;
}
```

#### Generic Hash
hash值根据指定的key计算，key可以是文本字符串、变量或者两者结合。

```
upstream backend {
    hash $request_uri consistent;
    server backend1.example.com;
    server backend2.example.com;
}
```

可选`consistent`参数启动ketama一致性哈希负载均衡。利用一致性哈希算法，如果upstream server组里添加或者移除服务器，只有少量的key会被重新计算hash值。在缓存负载服务器组中，这能减少缓存命中丢失情况。


### Server Weights
传递请求会根据权重，默认权重为1。

```
upstream backend {
    server backend1.example.com weight=5;
    server backend2.example.com;  # 不指定默认权重为1
    server 192.0.0.1 backup;
}
```


### 在多工作进程间共享数据
如果在`upstream`块里没有包含`zone`指令，那么每个工作进程都维护自己的服务器配置和相关计数，计数包含组中每个服务器当前连接数和转发请求失败的次数。这导致服务器组配置不能动态修改。

当`zone`指令包含在`upstream`块中，则`upstream`组的配置会存放在所有工作进程共享的内存区域。因为工作进程获取相同的组配置和利用相同的计数，因为组的配置会自动调整。

`zone`指令对于`active health checks`（主动运行状况检查）和`dynamic reconfiguration `（上游组的动态重新配置）是必需的。但是，上游组的其他功能也可以从此指令的使用中受益。

特别要注意`Least Connections`算法，当没有使用`zone`指令，那么这个算法可能会出现和期待不同的结果，因此每个工作进程维护自己的最小连接计数，因此无法进行全局的最小连接分配。

#### 设置Zone大小
需要根据机器的情况设置Zone的大小：

```
upstream backend {
        zone backend 32k;
        least_conn;
        # ...
        server backend1.example.com resolve;
        server backend2.example.com resolve;
}
```


### 利用DNS配置HTTP负载均衡
通过利用DNS，server组的配置能在运行时修改。

通过`resolver`指令指定DNS服务器，当在`upstream`指令块里的`server`使用域名指定服务器时，这些server指定的域名将会被`resolver`指令指定的DNS服务器解析为相应的ip。这样即使域名ip被修改里也没有问题。

```
http {
    resolver 10.0.0.1 valid=300s ipv6=off;
    resolver_timeout 10s;
    server {
        location / {
            proxy_pass http://backend;
        }
    }
    upstream backend {
        zone backend 32k;
        least_conn;
        # ...
        server backend1.example.com resolve;
        server backend2.example.com resolve;
    }
}
```

由于DNS解析即会解析为ipv4也会解析为ipv6，如果其中一个失败了，则解析失败，为了避免这种情况，一般关闭ipv6解析。

**注，在upstream里不能解析域名，除非是商业版nginx使用`resolve`指令。**

#### resolver指令
只有通过设置变量才能使`http`、`server`或`location`上下文的`resolver`指令生效。如下面例子：

##### 例子1

```
http {
    resolver 10.0.0.1 valid=300s ipv6=off;
    resolver_timeout 10s;
    server {
        location / {
            proxy_pass http://www.baidu.com;
        }
    }
}
```

没有通过设置变量代替域名，`resolver`指令无效，如果系统没有设置默认域名解析，那么重加载nginx会报错。

##### 例子2

```
http {
    resolver 10.0.0.1 valid=300s ipv6=off;
    resolver_timeout 10s;
    server {
        location / {
            proxy_pass http://backend;
        }
    }
    upstream backend {
        zone backend 32k;
        least_conn;
        # ...
        server backend1.example.com;
        server backend2.example.com;
    }
}
```

在开源版nginx中，`upstream`指令块里不支持域名解析，`resolver`指令无效。如果系统没有设置默认域名解析，那么重加载nginx会报错。

##### 例子3

```
http {
    resolver 10.0.0.1 valid=300s ipv6=off;
    resolver_timeout 10s;
    server {
        location / {
            set $var "www.baidu.com"'
            proxy_pass http://$var;
        }
    }
}
```

通过设置变量代替域名，`resolver`指令有效，域名解析将由`resolver`指令指定的域名服务器进行域名解析。

参考：http://www.luwenpeng.cn/2018/11/25/nginx%E5%8F%8D%E5%90%91%E4%BB%A3%E7%90%86%E5%8A%A8%E6%80%81%E8%A7%A3%E6%9E%90DNS/