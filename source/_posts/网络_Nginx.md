---
title: Nginx
tags: 
  - HTTP服务器
  - 反向代理
  - 负载均衡
categories:
  - [计算机网络]
date: 2021/01/01 20:46:25
updated: 2021/01/01 20:46:25
---



官方中文文档：https://www.nginx.cn/doc



功能：免费开源的 **高性能的HTTP服务器**、**反向代理**、**IMAP/POP3代理服务器**。

特点：稳定，轻量级，配置简单，资源消耗低，功能丰富！



相同功能还有Apache的Httpd，架构比较笨重，性能不如nginx，主要不会玩。



# 配置服务器名称（域名）

> nginx通过请求的请求头中的Host获取要访问的server,

可用`精确名称`、`通配符`、`正则表达式`

按照顺序选择第一个匹配：

> 精确名称，以'*'开头的最长的通配符，以'*'结尾的最长的通配符，第一个匹配的正则表达式

```nginx
server {
    listen       80;
    # 精确名称
    server_name  example.org  www.example.org;
    # 或 *开头的通配符（以'example.org'结尾的都算）
    server_name  *.example.org;#如果有'*'，则'*'必须在'.'的旁边
    # 或 *结尾的通配符（以'mail.'开头的都算）
    server_name  mail.*;
    # 或 正则
    server_name  ~^(?<user>.+)\.example\.net$;# 使用正则表达式时，服务器名称必须使用波浪线'~’开头
    
    ...
}
```

> 精确名称、以星号开头的通配符、以星号结尾的通配符是分别被存储在绑定到监听端口上的三张哈希表中。
>
> 存放精确名称的哈希表最先被搜索；其次以星号开头的通配符哈希表；最后搜索以星号结尾的哈希表。

一种特殊形式的通配符`.example.org`可以用来匹配精确名称`example.org`和通配符`*.example.org`  ,这种特殊的通配符形式`.example.org`是存储在通配符名称的哈希表中。

所以能写

```nginx
server {
    listen 80;
    server_name example.org  www.example.org  *.example.org;#更高效
    server_name .example.org;
    ...
}
```





##### 指定**正则**表达式**捕获片段**之后可以**当做一个变量**来使用：

> ```nginx
> server {
>     server_name   ~^(www\.)?(?<domain>.+)$;
>     location / {
>         root   /sites/$domain;
>     }
> }
> ```
>
> PCRE库支持以下语法使用捕获片段：
>
> > `?<name>`： Perl 5.10兼容语法，从PCRE-7.0开始支持
> > `?'name'`： Perl 5.10兼容语法，从PCRE-7.0开始支持
> > `?P<name>`： Python兼容语法，从PCRE-4.0开始支持
>
> 如果 NGINX 启动失败，并且显示错误信息：
> `pcre_compile() failed: unrecognized character after (?< in ...`
> 说明PCRE库太老了，这种语法“?P<name>”应该被替换。
>
> 捕获片段也可以使用数字形式引用：
>
> > 这种使用方式尽量限制在简单案例中使用，因为数字引用形式很容易被覆盖。
> >
> > ```nginx
> > server {
> >     server_name   ~^(www\.)?(.+)$;
> >     location / {
> >         root   /sites/$2;
> >     }
> > }
> > ```

##### 如果请求头中没有Host，则会用**默认服务器**来处理请求，

> ```nginx
> http {
>     # 如果没有显式声明 default server 则第一个 server 会被隐式的设为 default server,建议显示声明一下default server
>     server {
>         listen 80;
>         server_name www.aaa.com;
>         ...
>     }
>     server {
>         listen 80;
>         server_name www.bbb.com;
>         ...
>     }
>     # 显示的定义一个 default server
>     server {
>         listen 80 default_server;
>         server_name _;
>         return 403; # 403 forbidden
>     }
>     #---让default server返回403，可以禁止ip访问，也可以禁止未知域名访问
> }
> ```

如果请求直接输入ip，nginx会把请求随机分配到某个域名上，

##### 如果想让直接用ip的请求打到指定的某个域名上

则需要配置该server为

```nginx
server {
    listen x.x.x.x:80;
    #或者
    listen x.x.x.x;#（默认80嘛）
    server_name _;
    ...
}
```



# 配置HTTPS

在server块中监听指令后 `启用ssl` 并指定证书和私钥的位置

```nginx
    server {
        listen 443 ssl;#启用ssl
        #-----或-------
        listen 443;
        ssl on;
        #--------------
        server_name www.melopoz.top;
        ssl_certificate /usr/local/crt/melopoz.top.pem;#证书位置
        ssl_certificate_key /usr/local/crt/melopoz.top.key;#私钥位置

    	ssl_protocols TLSv1 TLSv1.1 TLSv1.2;#默认的不用写 不过每个版本不太一样
        ssl_ciphers HIGH:!aNULL:!MD5;#默认的不用写 不过每个版本不太一样
        ...
    }
    server {
        listen 80;
        server_name melopoz.top;
        #将请求转成https
        rewrite ^(.*)$ https://$host$1 permanent;
    }
```

私钥应该限制访问权限，如果证书和私钥都在一个文件里的话，也只有证书会发送到客户端。

#### 虚拟主机，多个域名都监听443端口，但证书不同

配置如下：

```nginx
server {
    listen          443 ssl;
    server_name     www.aaa.com;
    ssl_certificate www.aaa.com.crt;
    ...
}
server {
    listen          443 ssl;
    server_name     www.bbb.org;
    ssl_certificate www.bbb.org.crt;
    ...
}
```

因为https是先建立SSL连接，再建立HTTP连接，在进行SSL连接的时候还不知道请求的服务器的名称（域名）是什么(不懈怠)，所以nginx只能把默认的服务器证书(aaa的crt)给到客户端，如果请求的是bbb，按说应该是行不通的。。。。。

后来，2006年，TLS协议加入了一个[Server Name Indication扩展](http://tools.ietf.org/html/rfc4366)，请求可以携带客户端要访问的域名。

**所以如上配置现在是没问题的，ssl握手也可以正确拿到要请求的域名对应的证书**



#### 多个域名的SSL证书

还有其他方法允许在几个HTTPS虚拟主机服务器之间共享单个IP地址。然而，他们都有自己的缺点。其中一种方法是在证书的 SubjectAltName 字段中使用多个名称，例如 `www.example.com` 和 `www.example.org` 。 但是， SubjectAltName 字段长度有限。

另一种方法是使用带有通配符名称的证书，例如 `*.example.org` 。 通配符证书能保护指定域的所有子域，但只限一个级别。此证书与 `www.example.org` 匹配，但不匹配 `example.org` 和 `www.sub.example.org` 。这两种方法也可以结合。证书可以在 SubjectAltName 字段中包含完全匹配和通配符名称，例如 `example.org` 和 `*.example.org` 。

最好在配置文件的 `http`区块中放置具有多个名称的证书文件及其私钥文件，以在所有其下的虚拟主机服务器中继承其单个内存副本：

```nginx
ssl_certificate     common.crt;
ssl_certificate_key common.key;#通用
server {
    listen          443 ssl;
    server_name     www.example.com;
    ...
}
server {
    listen          443 ssl;
    server_name     www.example.org;
    ...
}
```

从`github 的 nginx-tutorial`nginx-教程 看到



# 负载均衡

可选策略：

- 轮询（默认）
- 最少连接（下次请求分配给 连接数最少的机器（分给最不忙的机器））
- IP哈希（基于客户端的IP地址进行hash，一个客户端的请求每次都是被同一个机器处理（机器没有下线））

```nginx
http {
    upstream myapp1 {
        # ---这里配置使用的负载均衡策略
        # 默认轮询
        # least_conn
        # ip_hash
                
        server srv1.example.com weight=3;#可以根据机器的性能分配一个合适的权重
        server srv2.example.com;
        server srv3.example.com;
    }
    server {
        listen 80;
        location / {
            proxy_pass http://myapp1;# 如果是https 则直接用https即可
        }
    }
}
```



# 压缩/解压

由于压缩在运行时发生，所以会增加处理开销，这可能会对性能产生负面影响。 

在向客户端发送响应之前，NGINX 会执行压缩，但不会“重复压缩”已经压缩过的响应。

#### 启用压缩

要启用压缩，在 gzip 指令上请使用`on`参数:

```
gzip on;
```

默认情况下，NGINX 仅压缩使用MIME 类型 为 `text/html`的响应。要压缩其他 MIME 类型的响应，请包含`gzip_types`指令并列出相应的类型。

```
gzip_types text/plain application/xml;
```

要指定要压缩的响应的最小长度，请使用`gzip_min_length`指令。默认值为20字节，下面示例调整为1000：

```
gzip_min_length 1000;
```

默认情况下，NGINX 不会压缩对代理请求的响应（来自代理服务器的请求）。请求是否来自代理服务器是由请求中 `Via` 头字段的是否存来确定的。

使用`gzip_proxied` 配置这些响应的压缩，该指令具有多个参数来指定 NGINX 应压缩哪种代理请求。

例如，仅对不会在代理服务器上缓存的请求进行压缩响应，为此，`gzip_proxied`指令具有指示 NGINX 在响应中检查 `Cache-Control` 头字段的参数，如果值是 no-cache、no-store 或 private，则压缩响应。

另外，您必须包括 expired 的参数来检查 `Expires` 头字段的值。这些参数在以下示例中与 auth 参数一起设置，该参数检查`Authorization`头字段的存在（授权响应特定于最终用户，并且通常不被缓存）：

```
gzip_proxied no-cache no-store private expired auth;
```

与大多数其他指令一样，配置压缩的指令可以包含在`http`上下文中，也可以包含在 `server` 或 `location` 块中。

gzip 压缩的整体配置可能如下所示。

```nginx
server {
    gzip on;
    gzip_types      text/plain application/xml;
    gzip_proxied    no-cache no-store private expired auth;
    gzip_min_length 1000;
    ...
}
```

#### 启用解压

```
location /storage/ {
    gunzip on;
    ...
}
```

`gunzip`指令也可以在与`gzip`指令相同的上下文中指定：

```
server {
    gzip on;
    gzip_min_length 1000;
    gunzip on;
    ...
}
```


请注意，此指令在单独的模块中定义（见`ngx_http_gunzip_module`<http://nginx.org/en/docs/http/ngx_http_gunzip_module.html>），默认情况下可能不包含在开源 NGINX 构建中。

#### 发送压缩文件

要将文件的压缩版本发送到客户端而不是常规文件，请在适当的上下文中将`gzip_static`指令设置为 on。


```
location / {
    gzip_static on;
}
```


在这种情况下，为了服务`/path/to/file`的请求，NGINX 尝试查找并发送文件`/path/to/file.gz`。如果文件不存在，或客户端不支持 gzip，则 NGINX 将发送未压缩版本的文件。

请注意，`gzip_static`指令不启用即时压缩。它只是使用压缩工具预先压缩的文件。要在运行时压缩内容（而不仅仅是静态内容），请使用`gzip`指令。



# 安装

Ubuntu：

sudo apt-get update

sudo apt-get install nginx

> 若安装过程报错：
>
> W: GPG error: http://nginx.org/packages/ubuntu xenial Release: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY $key
>
> 则先执行`sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys $key`
>
> 再执行安装

启动

sudo service nginx start

访问localhost即可(默认80端口)



# 改了配置 重新加载

`nginx -s reload`





# 绑核

处理相同线程都用同一个CPU可以保证L1L2缓存的使用，减缓因切换使用的CPU带来的开销，所以如果指定进程使用的CPU可以提高性能。

因为是要用cpu的L1、L2缓存，所以这里绑定的是物理核心！

如果是四核四线程的CPU，则直接配置即可，如果是四核八线程(泛指这种虚拟核心的情况)，要先查看指定的CPU core id

#### 可以通过 `cat /proc/cpuinfo |grep "core id"` 命令查看CPU核心

四核四线程：

> core id         : 0
> core id         : 1
> core id         : 2
> core id         : 3

四核八线程：

> 可能是
>
> core id         : 0
> core id         : 1
> core id         : 2
> core id         : 3
> core id         : 0
> core id         : 1
> core id         : 2
> core id         : 3
>
> 可能是
>
> core id         : 0
> core id         : 0
> core id         : 1
> core id         : 1
> core id         : 2
> core id         : 2
> core id         : 3
> core id         : 3

#### 配置

如果是第一种四核八线程需要配置 affinity 为：

```nginx
worker_processes auto;# cpu 核心数
worker_cpu_affinity 0001 0010 0100 1000;#15 26 37 48分为四组，保证一个物理核心的两个逻辑核心处理的进程都用这个物理cpu
worker_cpu_affinity auto;# cpu亲和性 就是绑核 1.9.几来着 可以设置 auto 了
```

