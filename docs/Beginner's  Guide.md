##入门指引  
<br>
[启动，停止和重载配置](#1)<br>
[配置文件的结构](#2)<br>
[服务静态内容](#3)<br>
[建立简单的代理服务器](#4)<br>
[建立FastCGI代理](#5)<br>

这个指引对nginx做了一些简单的介绍并且描述了一些它可以完成的简单任务。我们假定读者已经安装了nginx，如果没有的话，请看[安装nginx](http://nginx.org/en/docs/install.html)页面。这个指引描述了如何启动和停止nginx，以及重载它的配置、解释了配置文件的结构以及如何让nginx服务静态内容、如何配置nginx使之成为代理服务器、和如何用将它和一个FastCGI的应用连接起来。

nginx有一个主进程和若干个工作进程。主进程的主要目的是读取和分析配置文件以及管理工作进程。工作进程处理实际的请求。nginx使用了基于事件的模型和操作系统依赖型的机制来有效地向工作进程分发请求。工作进程的数量由配置文件定义，也可以通过修改一个给定的配置文件中来改变，或者自动地根据cpu的核数进行调整（详见[工作进程](http://nginx.org/en/docs/ngx_core_module.html#worker_processes)）

nginx和它的模块们的工作方式由配置文件决定，默认情况下，配置文件名叫nginx.conf，它被放置在/usr/local/nginx/conf，/etc/nginx 或 usr/local/etc/nginx 目录下。

<h3 id='1'>启动，停止和重载配置</h3>
运行可执行文件来启动nginx。一旦nginx被启动，它可以通过调用带 -s 参数的可执行命令被控制。使用如下语法：  

nginx -s *signal*

其中，*signal* 可以是以下几种之一：

* stop —— 快速关闭
* quit —— 平稳关闭
* reload —— 重载配置文件
* reopen —— 重新打开日志文件

例如，当需要等待工作进程处理完当前请求时才关闭nginx，可以使用以下命令：

nginx -s quit  

`这个命令应该由启动nginx的用户执行`

对配置文件的更改不会立即生效直接到reload命令被执行或者nginx被重启。执行reload命令：

nginx -s reload

一旦主进程收到重载配置文件的信号，它会检查新配置文件的语法并尝试应用其中的配置。如果应用成功了，它会启动新的工作进程并且发送关闭的请求给旧的工作进程；否则，主进程回滚修改并且继续使用旧的配置工作。旧的工作进程收到关闭的命令后，停止接收新的连接并继续服务当前的请求直到所有这样的请求都被服务。然后，旧的工作进程就会关闭。

一个信号也可以通过unix工具被发送给nginx的进程，比如kill。在这种情况下，一个信号通过一个进程id直接被发送给进程。默认情况下，nginx的主进程id被记录在 /usr/local/nginx/logs 或 /var/run 目录下的nginx.pid文件里。例如，如果主进程id是1628，为了发送quit信号使nginx平稳关闭，可以执行：
kill -s QUIT 1628

为了获得所有运行中的nginx进程的列表，可以使用ps命令，例如：
ps -ax | grep nginx  
更多的关于发送信号的信息，参加[控制nginx](http://nginx.org/en/docs/control.html)

<h3 id='2'>配置文件的结构</h3>
nginx包含一些被配置文件中的指令控制的模块。指令被分为简单指令和块级指令两种。一个简单指令包含了由空格分开的名称和参数，并且以分号结尾。一个块级指令和简单指令有一样的结构，但它不使用分号结尾而使用一些被{(和)}包围的额外结构来结尾。如果一个块级指令中包含了其它指令，那么它被称为一个上下文。比如events，http，server和location  

\#号后的内容是注释

<h3 id='3'>服务静态内容</h3>
web服务器的一个重要任务是对外输出文件，比如图片和静态网页。你会实现一个这样的例子：根据不同的请求，文件会从不同的本地目录，如： /data/www (html) 和 /data/images 被输出。这需要修改配置文件并且在http块指令中建立带有两个location块的server块。

首先,创建/data/www的目录并且放置一个index.html的文件，然后创建/data/images目录并放置一些图片  

接下来，打开配置文件。默认的配置文件中已经包含了几个server块指令的例子，其中大多数被注释掉了。现在，注释掉所有这样的块指令并且建立一个新的server块指令。  
    
    http {  
    	server {  
        }  
    }
  
通常一个配置文件会包含若干个server块，并通过他们监听的端口和他们的服务名称来区分。一旦nginx决定了那个由哪个server来处理一个请求，它就会检验请求头部的URI并用location指令的参数和其对比，location块被定义在server块中。  

    location / {
    	root /data/www;
    }  
    
这里location块声明了一个"/"前缀来和请求中的URI进行对比。对于成功匹配的请求，URI会被添加到root指令声明的路径后面，形成一个在本地文件系统中对于所需文件的请求。如果有多个匹配的location块，则nginx选择最长前缀的那个。以上的location块的前缀只有一个字符，是最短的，因此只有当其它location都匹配失败时，这个location才会被选择。  

现在，添加第二个location块：

    location /images/ {
    	root /data;
    }
    
它会匹配以/images/开始的请求(location / 也会匹配这个请求，但它的前缀更短)  
现在，server块指令看起来就像这样

    server {
        location / {
            root /data/www;
        }

        location /images/ {
            root /data;
        }
    }
这个配置已经可以工作了，它监听在标准的80端口上，并且可以在本机上通过http://localhost/访问。为了响应以/images/开头的URI,服务器会从/data/images目录中发送文件。比如：为了响应http://localhost/images/example.png，nginx会发送/data/images/example.png这个文件，如果不存在这样的文件，nginx就会发送404错误。而不以/images/开头的请求则被映射到/data/www目录，比如：http://localhost/some/example.html被映射到/data/www/some/example.html文件。

<h3 id='4'>建立简单的代理服务器</h3>
nginx一个最常见的用途就是用作代理服务器，也就是把收到的请求传递给被代理的服务器，并从被代理服务器中取回响应，再将其发送给客户端。

我们会配置一个基本的代理服务器，对于图片文件的请求，从本地目录中发送文件，而对于其它的请求，则把请求转发给另一个被代理服务器。在这个例子里，两个服务器都会在一个单一的nginx实例中被定义。  

首先，通过添加一个块指令定义一个被代理服务器：

    server {
        listen 8080;
        root /data/up1;

        location / {
        }
    }
这是一个监听在8080端口的简单服务器(之前我们定义的server块不声明listen指令是因为使用了标准的80端口)并且会把所有请求映射到本地的 /data/upl 文件夹。创建这个文件夹并且放入一个index.html文件。注意，这里的root指令被放在了server上下文中。当有一个location被选择了而它的内部却没有root指令时，它就会使用server中的这个root指令。  

接下来，修改在前一节中的server配置使它变为一个代理服务器的配置。在第一个location块中，添加proxy_pass指令，它的参数是被代理服务器的协议，名称和端口。(本例中，参数是 http://localhost:8080):

    server {
        location / {
            proxy_pass http://localhost:8080;
        }

        location /images/ {
            root /data;
        }
    }
我们现在修改第二个location块，使它由原先的匹配/images/前缀变为匹配典型的图片文件扩展名。修改后的location如下：

    location ~ \.(gif|jpg|png)$ {
        root /data/images;
    }
这个参数是一个匹配所有以.gif,.jpg或.png结尾的URI的正则表达式。~ 应该被写在正则表达式前面。  

当nginx选择一个location时，它先检查前缀，并且记录匹配的location(最长前缀),然后nginx再检查正则表达式，如果有一个正则表达式匹配，它就选择这个location，否则，选择之前记录的location。  

最终的代理服务器配置：  

    server {
        location / {
            proxy_pass http://localhost:8080/;
        }

        location ~ \.(gif|jpg|png)$ {
            root /data/images;
        }
    }
现在，这个服务器可以将以.gif,.jpg或.png结尾的请求映射到本机目录，将其它所有请求发送到被代理服务器。  

为了使配置生效，要发送reload信号。

<h3 id='5'>建立FastCGI代理</h3>

nginx可以用来把请求路由给FastCGI服务器。FastCGI服务器可以运行由多种框架和程序语言(如PHP)构建的应用。  

最基本的配置方法是使用fastcgi_pass指令取代proxy_pass指令，并且用fastcgi_param指令设置参数。假设FastCGI服务器可访问路径是localhost:9000，用前一节的代理服务器的配置作为基础进行修改。在PHP下，SCRIPT_FILENAME 参数是用来指定脚本名称, QUERY_STRING 是用来传递请求参数。改变结果如下：

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
到此就建立了一个FastCGI代理服务器，可以将静态图片以外的请求通过FastCGI协议传递给被代理服务器 localhost:9000