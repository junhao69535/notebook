# hosts和resolv

在linux下，存在/etc/hosts和/etc/resolv.conf。

## /etc/hosts
这个文件存储的是主机和ip的映射，例如存在下面映射
```
192.168.4.25    myos
```

那么我们请求主机名为"myos"的主机时，会自动解析到192.168.4.25。如，ping myos，traceroute myos。


## /etc/resolv.conf
这个文件存储的是DNS域名服务器，例如我们ping baidu.com，会将域名解析为ip。格式一般如下：
```
search localdomain
nameserver xx.xx.xx.xx
```

search用于域名补全，如：
```
search abc.com abc.net
nameserver xx.xx.xx.xx
```

### 主机名没有点
如果host -a test，一般来说主机名没有点，test先被认为是主机名，然后进行先进行补全，test.abc.com，如果解析失败，就跳过第二项补全为test.abc.net，如果还是解析失败，那么就会把主机名test当成域名进行解析。

### 主机名有点（不是末尾有点）
如果host -a test.com，因为主机名中有点，就会被认为是完全合格域名，因此会查询test.com，其次test.com.abc.com，最后test.com.abc.net。

### 主机名有点（末尾有点）
如果host -a test.，因为主机名末尾有点，就会被认为是完全合格域名，不会补全，直接把test当成域名使用。




## 解析顺序
先解析dns缓存（如果存在），然后再解析/ect/hosts，最后通过/etc/resolv.conf的域名服务器解析。
1. 解析dns缓存（如果存在）
2. 解析/etc/hosts
3. /etc/resolv.conf的域名服务器解析

第3点如果失败，会search进行补全，然后重新执行第1和2点。
