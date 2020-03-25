---
title: Centos部署Python3+Django+uWSGI+Nginx
date: 2018-08-27 18:04:04
tags: Python
---


Centos部署Django+uWSGI+Nginx

>背景：最近打算自己写一个小网站，接口不多，就打算用Django来写了，正好提升一下自己的python的代码水平。这篇博客主要是用来记录从本地运行到部署服务器这过程遇到的一些小问题。

<!-- more -->

提示：
**如果是阿里云的服务器，我们要去阿里云的后台管理业去配置安全规则，开放我们即将要使用的那些端口的权限，他很多的端口都是默认不开放的。**

#### Djano简单介绍

我使用的IDE是PyCharm，操作简单，可直接创建Django项目，然后python版本选择3.x。

简单的说两句Django。这个框架个人感觉上手容易，毕竟框架做了比较好的分层，不同文件的只能划分也是很明确的，当然里面涉及一些配置参数什么的，这个大家可以去网上找Django的教程，先看看入门。

在目录结构中我们可以看到一个venv的文件，这个算是给我们创建的一个虚拟环境，为应用提供了隔离的Python运行环境，解决了不同应用间多版本的冲突问题。这个大家可以自己去查一下加深理解。

在本地敲代码的时候，使用到某些库的时候我们会使用 pip install xxxxx 来安装。使用了venv这种虚拟环境，所有的包默认安装在本项目下，新的项目使用的话需要在安装。

前期的本地工作就没什么了，从网上找到教程一点一点的学习就可以了。当然了，我们要准备部署到服务器的时候，首先得保证本地能成功的运行，本地都跑不了，那配置到服务器上肯定也不会，别的不说，代码首先别有问题。

#### 必要依赖包的安装

之前呢，搞过几次java，当时记得是把项目打成war包，然后上传，有点笨。把Django项目压缩一下再上传应该也是可以的。但是何必呢，我们直接把代码传到github（或者自己存代码的地方）上面，然后在登上服务器，直接git clone多好~ 有的时候太紧张，或者把事情想的太难了。

我们在本地pip install的那些包，到了服务器那边还是要再重新install的。

在服务器端，clone下项目来之后，cd到当前项目，我们可以把之前安装的包再一个一个的重新pip install，但是太多了我们也不一定记得清楚有哪些，所以我们现在本地生成一个记录引用的包的文件。

```objc
依赖文件生成 （本地操作）
pip freeze > requirements.txt
依赖文件安装 （服务器端操作）
pip install -r requirement.txt
```



我们创建项目时候用的python的版本要跟服务器上面的版本相同，我的服务器系统是centos7，记得好像是自带的是python2.x，然后我项目使用的python3.x, 所以我要在服务器上装python3.x, 具体的安装操作，大家自己搜索一下。

安装完高版本的python之后，运行pip，可能会出现一些问题，我忘记当时的报错了，好像是pip的版本问题，因为Django的版本，所以需要安装pip3, 这地方安装有些啰嗦，大家不要着急，慢慢的装，遇到报什么错就去响应的搜索一下。

有时候会提醒你升级pip，然你使用这个`pip install --upgrade pip`命令，个人建议不要乱试，我就是使用这个升级之后，pip和pip3都找不到了。

如果pip install显示不存在pip的时候，可以尝试一下下面这个命令  `python -m pip install xxx` 。

如果没什么别的报错了的话我们继续执行`pip install -r requirement.txt`来安装我们需要的包。

**下面我遇到了让我纠结了一天的一个错误。**
![](https://raw.githubusercontent.com/Sunxb/blog_img/master/07/1.png)

其实主要的问题就是**在安装mysqlclient的过程中`mysql_config not found`**

我这还特地上Stack Overflow上问了问，也没人回到我。我在网上搜索了关于这个问题的错误，尝试了所有能尝试的方法都没能解决。

我搜索了一下，我的文件里压根就没有mysql_config这个文件，太气了。

其中有几个比较靠谱的方法，是让安装`yum install mysql-devel`这个东西，我也跟着做运行了这个命令，不过安装出问题了，从终端上的提示来看，是有冲突，是与数据库有冲突，很抱歉我忘记截图了，但是我从网上的一片文章中也找到了一样的问题，大家参看一下。

```objc
--> Running transaction check
---> Package mysql-devel.x86_64 0:5.1.69-1.el6_4 will be installed
--> Processing Dependency: mysql = 5.1.69-1.el6_4 for package: mysql-devel-5.1.69-1.el6_4.x86_64
--> Running transaction check
---> Package mysql.x86_64 0:5.1.69-1.el6_4 will be installed
--> Processing Conflict: MySQL-client-5.5.30-1.el6.x86_64 conflicts mysql
--> Processing Conflict: MySQL-server-5.5.30-1.el6.x86_64 conflicts mysql
--> Processing Conflict: mysql-5.1.69-1.el6_4.x86_64 conflicts MySQL
--> Finished Dependency Resolution
Error: mysql conflicts with MySQL-devel-5.5.30-1.el6.x86_64
 You could try using --skip-broken to work around the problem
 You could try running: rpm -Va --nofiles --nodigest
```

可以看得出来有几个Conflict, 最可气的是底下他还教你通过`--skip-broken `来避开这些冲突，我照做了，结果依然不管用，问题就出在这个冲突上。

好吧，你不是说跟MySQL冲突吗，我就把MySQL卸载了再说（我很早之前学习MySQL的时候就安装了）。

具体的MySQL移除过程我就不多说了，只提下面两个点:
**使用 `rpm -qa|grep mysql`查看mysql的安装情况**
**`rpm -e 文件名 --nodeps `来移除**
如果有不管用的地方大家看一下下面这两篇文章，说的还不错，借鉴一下。
http://834945712.iteye.com/blog/1979042
https://blog.csdn.net/lsa000/article/details/77374351

然后就是重新安装MySQL,步骤可借鉴下面这篇文章
https://www.jianshu.com/p/d679db1ab27f

反正我的项目在重新安装了MySQL之后就好了，mysql_config文件也有了，再一次执行`pip install -r requirement.txt`，然后所有的包都顺利的安装完成。

最后执行python manager.py runserver，就可以看到我们的项目跑起来了 ~ 

但是呢，这样运行使用Django自带的WSGI Server运行的，是有性能缺陷的，并发承受能力不行 ~ 要不然为什么我们还要使用uWSGI+Nginx呢 

具体差距看一下这篇对比文章
[django自带wsgi server vs 部署uwsgi+nginx后的性能对比](https://www.cnblogs.com/alexkn/p/4416008.html)

#### uWSGI

首先在使用之前，我们得先大致了解他们到底是什么东西，为什么要使用。

##### 了解uWSGI

uWSGI旨在为部署分布式集群的网络应用开发一套完整的解决方案。主要面向web及其标准服务。由于其可扩展性，能够被无限制的扩展用来支持更多平台和语言。uWSGI是一个web服务器，实现了WSGI协议，uwsgi协议，http协议等。

uWSGI又很多优点，具体对uWSGI的讲解看这个
https://www.jianshu.com/p/679dee0a4193

具体uWSGI的安装我就不详细的说了，网上教程很多，我想说的是配置文件的问题，因为我搜了很多关于uWSGI的文章，可能是由于我们每个人的项目的结构不同，或者别的愿意，所以总感觉这个配置文件很难写。

网上大多数的教程在教我们使用uWSGI的时候会让我们新建一个test.py的文件，然后`uwsgi --http :8000 --wsgi-file test.py`, 检测是否能正常运行。

运行成功的话就说明uWSGI这个桥梁是起作用的~ 

下面这个说明我很喜欢
>the web client <-> uWSGI <-> Python

但是我们使用uWSGI是为了服务于Django项目的，所以说我压根就没尝试上面这一步，而是直接尝试通过uwsgi命令来运行django项目。

这里会有一点小分歧，因为我从网上搜到的很多教程是说运行了一个.wsgi的文件，但是我整个项目中就没有发现这么一个文件，只有一个wsgi.py的文件，所以我就是用了下面这个命令来运行整个项目
`uwsgi --http :8000 --wsgi-file wsgi.py` 
注意后面wsgi.py的路径问题.

结果还不错，项目跑得起来了。

##### 配置uwsgi

我其实并不是真的明白为什么要配置这个.ini文件，听别人说是这样的：

如果你喜欢用命令行的方式（如shell）敲命令，那可以省去任何配置。
但是，绝大多数人，还是不愿意记那么长的命令，反复敲的。所以uwsgi里，就给大家提供了多种配置，省去你启动时候，需要敲一长串命令的过程。

配置的方式很多，我选择使用ini的方式，也是因为网上教程多。

我们在根目录下面建一个.ini的文件，由于最初跟着网上的教程学，所以起了个test.ini的名字。

下面是这个文件里面的内容，首先我得成人有很多我是不知道什么意思的，而且网上教程中的配置会有各种各样的参数，这个以后我再慢慢研究。

```objc
http = :8000
chdir = /root/myblog/
module = blog.wsgi:application
processes = 4
threads = 2
master = true
enable-threads = true
daemonize = /root/searchmarket/marketSearch/uwsgi.log
buffer-size = 21573
```

稍微解释一下，不一定多，可作参考。
chdir是当前项目的路径
module算是wsgi.py这个文件的路径吗 ？好像是的，但是为什么要写成这样子呢 ？
（在我的项目中，myblog目录下有一个blog文件夹，wsgi.py就在这个blog文件夹中~是在搞不懂为啥要写成blog.wsgi而且后面还加上个 :application）
daemonize是放log日志的地方，自己创建好文件，路径贴上。

差不多了...

然后运行`uwsgi --ini test.ini `这个命令就行了 ~ 

运行完之后，从终端里我看不出来倒是成功了没有，我用postman请求了一下试了试，失败的话我就从那个.log文件中看日志，然后处理问题。

成功的话，那就是uWSGI部分成功了。

注：也许会碰到端口占用的问题 
我的端口使用的8000 然后执行这个命令`lsof -i:8000`, 然后你会看到PID， 然后`kill xxx`，就行了。
我发现了一个快速的方法
`killall -9 uwsgi`

#### Nginx

##### 启动Nginx

我使用的是yum安装，简单易操作。
`yum -y install nginx` 就OK了

然后nginx启动页非常简单，直接`nginx`就好了。

在启动nginx时候碰到一个小问题，我干了什么导致的我也不太清楚，反正就是报了这么个错：

```objc
Job for nginx.service failed because the control process exited with error code
```

就是没启动的了，我忽然想起来之前看的一篇博客，在启动之前他检测了80端口的状态，我就隐隐感觉是80端口已经被占用的缘故。

然后我运行了这个命令`netstat -ntpl`来查看当前活跃的链接

netstat -ntpl 查看一下80端口是否被占用

```objc
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:3306            0.0.0.0:*               LISTEN      14267/mysqld        
tcp        0      0 127.0.0.1:6379          0.0.0.0:*               LISTEN      709/redis-server 12 
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      4225/nginx: master  
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1184/sshd           
tcp        0      0 0.0.0.0:8000            0.0.0.0:*               LISTEN      4212/uwsgi    
```

发现nginx已经被我启动过一次了，我也记不清楚了，反正现在启动nginx没有什么提示。（网上一些文章显示有一个OK的提示）。

然后可以`kill 4225`， 在`nginx`，就好了 ~

用浏览器打开 xxx.xxx.xxx.xx:80 （xx代表我们服务器的ip），就可以看到nginx启动成功的标识了。

##### 配置Nginx

yum在线安装会将nginx的安装文件放在系统的不同位置，可以通过命令 `rpm -ql nginx `来查看安装路径

```
[root@sunxb ~]# rpm -ql nginx
/etc/logrotate.d/nginx
/etc/nginx
/etc/nginx/conf.d
/etc/nginx/conf.d/default.conf
/etc/nginx/fastcgi_params
/etc/nginx/koi-utf
/etc/nginx/koi-win
/etc/nginx/mime.types
/etc/nginx/modules
/etc/nginx/nginx.conf
/etc/nginx/scgi_params
/etc/nginx/uwsgi_params
/etc/nginx/win-utf
/etc/sysconfig/nginx
/etc/sysconfig/nginx-debug
/usr/lib/systemd/system/nginx-debug.service
/usr/lib/systemd/system/nginx.service
/usr/lib64/nginx
/usr/lib64/nginx/modules
/usr/libexec/initscripts/legacy-actions/nginx
/usr/libexec/initscripts/legacy-actions/nginx/check-reload
/usr/libexec/initscripts/legacy-actions/nginx/upgrade
/usr/sbin/nginx
/usr/sbin/nginx-debug
/usr/share/doc/nginx-1.14.0
/usr/share/doc/nginx-1.14.0/COPYRIGHT
/usr/share/man/man8/nginx.8.gz
/usr/share/nginx
/usr/share/nginx/html
/usr/share/nginx/html/50x.html
/usr/share/nginx/html/index.html
/var/cache/nginx
/var/log/nginx
```

配置config文件
vim /etc/nginx/conf.d/default.conf

然后把location / {} 里面改为如下

```
 location / {
       include  uwsgi_params;

       uwsgi_pass  127.0.0.1:8000;
       uwsgi_param UWSGI_SCRIPT blog.wsgi;
       uwsgi_param UWSGI_CHDIR /root/myblog;
       index  index.html index.htm;
       client_max_body_size 35m;
    }

```
uwsgi_pass 必须和uwsgi中的设置一致
UWSGI_CHDIR和UWSGI_SCRIPT 其实也跟uwsgi中差不多，一个项目路径，一个wsgi文件的路径。

保存之后，Nginx的配置就完成了。最后我们得稍微修改一下uwsgi中的一条配置。

之前在test.ini文件中是 `http = :8000` 我们要改为`socket = :8000`。

重新运行uwsgi，重启nginx。

结束。

-----



