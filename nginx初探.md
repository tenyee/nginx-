# 第一章 新手指引

​	本指引对nginx进行了基本的介绍并描述了利用nginx完成的一些小任务。当前假设读者的设备上已经安装了nginx。如果没有安装，则查看[安装nginx](http://nginx.org/en/docs/install.html)。本指南将介绍：

1. 启动和关闭nginx；
2. 重载配置；
3. 解释配置文件的结构；
4. 建立nginx提供静态内容服务；
5. 配置nginx为代理服务器；
6. 利用FastCGI应用连接nginx；

​    nginx由一个master进程以及多个工作进程组成。master进程的主要作用是读取以及评估配置，和维护worker进程。worker进程处理实际的请求。nginx采用基于事件的模型以及依赖系统去高效地分配请求到各个worker进程中。worker进程的数量可定义中配置文件中，可以是固定的一个值，也可以是自适应于CPU的个数（更多信息参考：[worker process](http://nginx.org/en/docs/ngx_core_module.html#worker_processes)）

​    nginx以及其modules的工作都是由配置文件决定的。默认地，配置文件名为：`nginx.conf`,并安装在目录：`/usr/local/nginx/conf`,`/etc/nginx`,`/usr/local/etc/nginx`



## 1.1 启动、关闭以及重载配置

​	启动nginx，直接运行可行执行文件。一旦nginx启动后，可以通过调用可执行文件，并指定`-s`参数来控制nginx，其语法为：

```shell
nginx -s singal

#signal的类型为：
stop -- 快速关闭
quit -- 优雅关闭
reload -- 重新加载配置文件
reopen -- 重新打开日志文件
```

例如，在关闭nginx进程时，如果要求工作进程先完成当前的请求任务，则可以执行以下命令：

```shell
nginx -s quit
#注意，这个命令的执行用户需要和启动nginx用户一致
```

当配置文件发生改变时，要么重载配置，要么重启nginx，重载配置可执行以下命令：

```shell
nginx -s reload
```

当master进程收到重载配置的信号时，master进程会新的配置文件的语法是否合法，然后尝试应用配置。如果前面的步骤成功了，那么master进程会启动一个新的worker进程并发送关闭信号给老的worker进程；否则，master进程就会回滚配置，并继续用老的配置进行工作。当老的worker进程收到一个关闭的命令时，worker会停止接受连接并继续服务当前的所有请求直接到完成，然后退出。

发送给nginx的信号也可以通过unix的工具kill来执行。在这种情况下，信号是通过给定的pid来发送的。nginx master进程的pid默认写在了nginx.pid中，其所在目录为`/usr/local/nginx/logs` 或者 `/var/run`。例如，master进程的pid为1628，可以通过kill发送QUIT信号来优雅关闭nginx:

```shell
kill -s QUIT 1628
```

通过以下ps命令，可以获取nginx的所有进程pid：

```shell
ps -ax | grep nginx
```

更多关于nginx发送信号，参考：[Controlling nginx](http://nginx.org/en/docs/control.html)



## 1.2 配置文件结构

nginx由配置文件中的配置指令控制的模块组成。而指令又分成**简单指令**以及**块指令**。一个简单指令由名字以及参数组成，其中间由空格分隔，并以`;`结尾。一个块指令和简单指令有相同的结构，但它不是分号，而是以一组由大括号（{and}）包围的附加指令结束。如果一个块指令拥有其它指令，那么这就叫做一个context(examples: [events](http://nginx.org/en/docs/ngx_core_module.html#events), [http](http://nginx.org/en/docs/http/ngx_http_core_module.html#http), [server](http://nginx.org/en/docs/http/ngx_http_core_module.html#server), and [location](http://nginx.org/en/docs/http/ngx_http_core_module.html#location)).

给个例子吧：

```shell
#user  nobody;
worker_processes  1;
#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;
#pid        logs/nginx.pid;

events {
    worker_connections  1024;
}

http {
        upstream demo {
                 server 172.22.179.139:8081;
                 server 172.22.179.139:8082;
        }
        server {
                listen 80;
                location / {
                        proxy_pass http://demo;
                }
        }
}
```

如果指令放置在所有context的外面，那么这个指令就是在main context中。如上面配置文件所示，events和http都在main context中，server在http中，location在server中。

**以`#`开头的都是注释。**

## 1.3 提供静态内容

web服务最重要的一个任务是对外提供文件（如图片，静态HTML页面）。你将会在本小节实现一个demo，依赖不同的请求，文件将会从不同的目录中获取出来：/data/www(包含html页面)和/data/images（包含图片）。这个实现需要编辑配置文件并且在http指令块中配置一个server指令块，server指令块中包含两个location指令块。

首先，创建一个/data/www目录，并创建一个index.html文件，再创建/data/images目录并放置一些图片文件；然后，打开配置文件，默认的配置文件包含了几个server指令块。

通常，配置文件会包含多个server块，这些server块通过不同的监听端口以及服务名称去区分。一旦nginx决定哪个服务器处理一个请求，它就会根据服务器块中定义的位置指令的参数测试请求头中指定的URI。如下：

```nginx
http {
    server {
    	location / {
    		root /data/www;
		}
    }
}
```

location指定了`/`前缀与请求中的URI进行匹配，如果匹配了，这个URI将会被添加到root指定的路径后，变成了`/data/www`，形成了本地系统的目录。如果有多个location是匹配的，则取最长匹配。

```nginx
worker_processes  1;
events {
    worker_connections  1024;
}


http {
        server {
                listen 80;
                location / {
                        root /data/www;
                }
                location /images {
                        root /data;
                }
        }
}
```

为了响应/images的请求，服务器会从本地目录：/data/images中找到对应的文件进行发送。例如：

请求：http://localhost/images/example.png

对于这个请求，nginx会发送/data/images/example.png文件。如果这个文件不存在，则返回404。如果请求的URI并不是以/images开头的，则会映射到/data/www目录。例如，http://localhost/some/example.html ，则会发送文件data/www/some/example.html

为了应用新的配置文件，执行nginx -s reload。如果服务没有正常运行，则查看日志文件：/usr/local/nginx/或/var/log/nginx中的access.log和error.log



## 1.4 配置简单的代理服务器

​	nginx常用作代理服务器，代理服务器把接收到的请求转发到被代理的服务器，并取回响应后发送给客户端。

​	我们将配置一个基本的代理服务器，它使用本地目录中的文件处理图像请求，并将所有其他请求发送到代理服务器中。在本例子中，两个服务器都定义在一个单独的nginx实例中。

​	首先，添加一个新的server块来定义被代理的服务器，如下：

```nginx
server {
    listen 8080;
    root /data/up1;
    location / {
        
    }
}
```

如果声明了一个简单的服务器，监听8080端口（如果listen没有声明指定端口，则默认为80），并且把所有请求映射到本地目录/data/up1。创建这个目录并把index.html放进去。注意到，上面的root指令是在server上下文的。当为服务请求而选择的位置块不包括其自己的根指令时，使用这样的根指令。

接着，利用前面写好的server，修改一下，改成代理服务。在第一个location块中，添加代理服务器地址，如下:

```nginx
server {
    location / {
        proxy_pass http://localhost:8080;
    }

    location /images/ {
        root /data;
    }
}
```

同时，修改第二个location块，当前如果请求是/images则映射到本地目录/data/images目录中，我们把它修改成指后缀的图片路径：

```nginx
location ~ \.(gif|jpg|png)$ {
    root /data/images;
}
```

上面是一个正则表达式，其目录映射固定为`~`，后面的参数匹配URI后缀为gif,jpg,png的文件。

当nginx选择一个location块去服务一个请求时，首先会检查location指令所指定的前缀，并且最长匹配优先，然后，再检查正则表达式。

最终，完整的http指令块如下：

```nginx
http {
        server {
                listen 80;
                location / {
                        proxy_pass http://127.0.0.1:8080;
                }
                location ~ \.(gif|jpg|png)$ {
                        root /data/images;
                }
        }

        server {
                listen 8080;
                root /data/up1;
                location / {

                }
        }
}
```

以下定义的服务器会把请求URI结尾为.gif,.jpg或者.png的请求，映射到/data/images目录中（通过添加URI到root指令的目录参数中形成完成的路径），而把其它请求都转发到代理服务器中。

关于代理服务器的配置指令，可参考：http://nginx.org/en/docs/http/ngx_http_proxy_module.html



## 1.5 配置FastCGI 代理服务器

nginx可以用于路由请求到FastCGI服务器中，这种服务器是一个应用程序，可能其它框架和语言实现，例如PHP。

以上服务器都是返回一个静态文件，下面这种是返回的是被处理的信息，并不是一个静态文件。

```nginx
server {
    location / {
        fastcgi_pass  localhost:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param QUERY_STRING    $query_string;
    }

    location ~ \.(gif|jpg|png)$ {
        root /data/images;
    }
}
```

大多数基本的fastCGI服务器中的，nginx配置都使用fastcgi_pass指令，而不是用proxy_pass指令。fastcgi_param指令调置传给fastCGI服务器的参数。假如FastCGI服务器通过localhost:9000访问。取上一个ngxin配置为基础，把proxy_pass替换成fastcgi_pass指令，并把参数改为localhost:9000。在PHP中，`SCRIPT_FILENAME`参数用于指定脚本名称，`QUERY_STRING`参数指定传递的请求参数。

