# 一种Docker化应用程序的简单方法 #

“Docker化”在本质上就是一个将应用程序运行在Docker容器中的过程。虽然大部分应用程序的Docker化并不复杂，但总有些重复性的问题需要解决。比如下面的两个问题：

1. 当应用程序依赖于配置文件时，能使配置文件使用系统的环境变量
2. 当应用程序默认将日志写入文件时，能使日志内容输出至标准输出（`STDOUT/STDERR`）

这篇文章会给各位介绍一位新朋友：`Dockerize`，它可以简化上述两个在Docker化过程经常遇到的问题。

## 问题 ##

### 配置文件 ###

许多应用程序的运行都依赖于配置文件，而很多配置文件的配置值都依赖于系统运行时的环境变量。比如：开发环境和生产环境的数据库连接配置是不同的。类似的，API keys和其他敏感配置在不同的运行环境下也是不同的。

我们原本有几种方法来处理docker容器间的环境差异：

1. 将包含不同环境变量的配置文件都嵌入到docker image里，然后通过一个控制变量决定运行时使用哪个配置文件（例如：`APP_CONFIG=/etc/dev.config`）
2. 运行时实时挂载存放有不同配置文件的数据卷
3. 使用封装好的脚本，利用`sed`等工具来实时修改配置文件中的变量

第一种方式并不理想，因为每次运行环境的变化都需要重建image。而且，将API keys、用户凭证等敏感信息存储在image中也并不安全。妥协于开发环境的配置必然会泄露生产环境的细节，在任何image中都应该避免这种情况。

第二种动态加载数据卷的方式能将配置信息与image隔离开，但会使部署复杂度大幅提高，因为你不能只通过部署image来解决问题，还必须同时配合配置文件的修改。

第三种实时修改配置文件的方式也并不那么简单，有时你需要精心设计一行`sed`命令或写一个自定义的脚本才能做到这一点。虽然这会产生一个可以在docker生态系统中工作良好的image，但这是个重复性工作。

### 日志 ###

Docker容器可以将标准输出（`STDOUT/STDERR`）记录下来，便于调试故障、监控并整合至[集中式日志系统](http://jasonwilder.com/blog/2012/01/03/centralized-logging/)中去，并可以通过`docker logs`命令或者docker log API调用来查看日志内容。如果应用程序将日志记录在标准输出（`STDOUT/STDERR`），我们可以使用很多工具来自动拉取docker日志并保存下来。

不幸的是，很多应用程序默认都将日志记录在文件里。虽然有一些[变通方法](http://jasonwilder.com/blog/2014/03/17/docker-log-management-using-fluentd/)，但想找出各个应用程序间日志配置的差异仍然是件沉闷乏味的事。

## 使用Dockerize ##

[Dockerize](https://github.com/jwilder/dockerize)是一个用Go语言开发的程序，它能够使docker化更简单：

1. 在启动时通过模板文件和容器内环境变量来生成配置文件
2. 可以将任意的日志文件输出至标准输出（`STDOUT/STDERR`）
3. 启动一个运行在容器内的进程

### 样例 ###

下面我们通过docker化一个普通的nginx容器来说明`dockerize`是如何工作的。首先安装nginx：

	FROM ubuntu:14.04

	# Install Nginx.
	RUN echo "deb http://ppa.launchpad.net/nginx/stable/ubuntu trusty main" > /etc/apt/sources.list.d/nginx-stable-trusty.list
	RUN echo "deb-src http://ppa.launchpad.net/nginx/stable/ubuntu trusty main" >> /etc/apt/sources.list.d/nginx-stable-trusty.list
	RUN apt-key adv --keyserver keyserver.ubuntu.com --recv-keys C300EE8C
	RUN apt-get update
	RUN apt-get install -y nginx

	RUN echo "daemon off;" >> /etc/nginx/nginx.conf

	EXPOSE 80

	CMD nginx


然后我们来安装`dockerize`，并通过`dockerize`启动nginx：

	FROM ubuntu:14.04

	# Install Nginx.
	RUN echo "deb http://ppa.launchpad.net/nginx/stable/ubuntu trusty main" > /etc/apt/sources.list.d/nginx-stable-trusty.list
	RUN echo "deb-src http://ppa.launchpad.net/nginx/stable/ubuntu trusty main" >> /etc/apt/sources.list.d/nginx-stable-trusty.list
	RUN apt-key adv --keyserver keyserver.ubuntu.com --recv-keys C300EE8C
	RUN apt-get update
	RUN apt-get install -y wget nginx

	RUN echo "daemon off;" >> /etc/nginx/nginx.conf

	RUN wget https://github.com/jwilder/dockerize/releases/download/v0.0.1/dockerize-linux-amd64-v0.0.1.tar.gz
	RUN tar -C /usr/local/bin -xvzf dockerize-linux-amd64-v0.0.1.tar.gz

	ADD dockerize /usr/local/bin/dockerize

	EXPOSE 80

	CMD dockerize nginx

Nginx默认会把日志存放在`/var/log/nginx`下。但如果能把nginx的访问日志和错误日志输出至控制台，在使用交互模式运行nginx容器或者使用`docker logs nginx`命令查看日志时就会非常方便。

我们可以在执行命令时传入`-stdout <file>` 和 `-stderr <file>`参数来实现上述功能。想输出多个日志文件内容时可以多次传入。

	CMD dockerize -stdout /var/log/nginx/access.log -stderr /var/log/nginx/error.log nginx


现在当你运行容器时，就可以通过`docker logs nginx`来查看nginx的日志了。


下面我们来讲解模板文件的用法。比如我们要使用环境变量来配置一个通用的代理服务器地址，定义一个URL作为环境变量`PROXY_URL`的值如下：

	PROXY_URL="http://jasonwilder.com"


当包含这个环境变量的容器启动时，`dockerize`会使用这个变量自动生成nginx服务器的location路径。

模板文件如下：

	server {
    	listen 80 default_server;
    	listen [::]:80 default_server ipv6only=on;

    	root /usr/share/nginx/html;
    	index index.html index.htm;

    	# Make site accessible from http://localhost/
    	server_name localhost;

    	location / {
      		access_log off;
      		proxy_pass {{ .Env.PROXY_URL }};
      		proxy_set_header X-Real-IP $remote_addr;
      		proxy_set_header Host $host;
      		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    	}
	}


最终的Dockerfile如下：

	FROM ubuntu:14.04

	# Install Nginx.
	RUN echo "deb http://ppa.launchpad.net/nginx/stable/ubuntu trusty main" > /etc/apt/sources.list.d/nginx-stable-trusty.list
	RUN echo "deb-src http://ppa.launchpad.net/nginx/stable/ubuntu trusty main" >> /etc/apt/sources.list.d/nginx-stable-trusty.list
	RUN apt-key adv --keyserver keyserver.ubuntu.com --recv-keys C300EE8C
	RUN apt-get update
	RUN apt-get install -y wget nginx

	RUN echo "daemon off;" >> /etc/nginx/nginx.conf

	RUN wget https://github.com/jwilder/dockerize/releases/download/v0.0.1/dockerize-linux-amd64-v0.0.1.tar.gz
	RUN tar -C /usr/local/bin -xvzf dockerize-linux-amd64-v0.0.1.tar.gz

	ADD default.tmpl /etc/nginx/sites-available/default.tmpl

	EXPOSE 80

	CMD dockerize -template /etc/nginx/sites-available/default.tmpl:/etc/nginx/sites-available/default -stdout /var/log/nginx/access.log -stderr /var/log/nginx/error.log nginx



`-template <src>:<dest>` 参数表示：模板文件`/etc/nginx/sites-available/default.tmpl` 应该被解析修改并生成至`/etc/nginx/sites-available/default`。`Dockerize`支持同时使用多个模板文件。

使用下面的命令启动容器：

	$ docker run -p 80:80 -e PROXY_URL="http://jasonwilder.com" --name nginx -d nginx


然后尝试访问`http://localhost` 来验证代理配置是否生效。

以上是个最简单的样例，当然还可以使用`Dockerize`内嵌的`split`函数和`range`语句来处理多个proxy值或其他参数来扩展这个样例。[这里](https://github.com/jwilder/dockerize#using-templates)还有一些模板函数供参考。

## 总结 ##

虽然上述样例有些简单，但许多应用程序若想在docker体系下运行良好还需要一些小补充。`Dockerize`就是这样一个会对你有所帮助的小工具。

你可以在[jwilder/dockerize](https://github.com/jwilder/dockerize)下载源代码。



原文链接： [A Simple Way to Dockerize Applications](http://jasonwilder.com/blog/2014/10/13/a-simple-way-to-dockerize-applications/)


