Title: 八种 Docker 开发模式（译）
date: 2014-10-25
url: eight-docker-development-patterns
tags: docker, pattern, cloud, devops
toc: yes
status: public

曾经我写过两篇文章：一篇是 [利用 OpenVZ 容器构建家庭云服务](http://www.hokstad.com/the-home-cloud)；另一篇则提出 [服务器必须能够自动化重建](http://www.hokstad.com/rebuilding-the-build-server-on-every-build) 的新思路。后来 Docker 出现了，它能够保证每次重建环境的一致性，已然成了我的最爱。

这篇文章我将介绍自己使用 Docker 过程中不断重复的那些模式，我并不认为这些模式有多新奇，只希望它们能对各位有用，同时我也非常期待听到各位关于 Docker 使用的各种模式。

Docker 实验的基础在于它能够将容器状态持久化到存储介质中，并且可在任意时候无损还原回来（除非修改容器状态时忘了更新 Dockerfile 文件，定期重建容器可避免这样的杯具）。

## 1. 可重用的基础容器

Docker 鼓励使用继承特性，善用继承能大大减少构建容器的时间，这是高效使用 Docker 的关键。Docker 在缓存过程容器方面表现过于出色，因此经常错过分享和重用的机会。

在迁移其它容器到 Docker 的过程中，存在太多重复冗余的安装步骤；当项目需要部署到很多地方，或者项目运行周期较长，又或项目需要安装一些特定包时，我都会考虑为该项目创建单独的容器。长此以往，我有了很多的容器，并且数量仍在快速增加；我试图在 Docker 中运行一切（包括某些桌面应用），只为让宿主环境更加纯粹。

以上这些，促使我开始思考如何减少容器的构建时间。我开始从各种项目容器中提取基本安装步骤，构建适用各种场景的可重用容器。下面是 "devbase" 基础容器对应的 Dockerfile 文件：

```docker
FROM debian:wheezy
RUN apt-get update
RUN apt-get -y install ruby ruby-dev build-essential git
RUN apt-get install -y libopenssl-ruby libxslt-dev libxml2-dev

# For debugging
RUN apt-get install -y gdb strace

# Set up my user
RUN useradd vidarh -u 1000 -s /bin/bash --no-create-home

RUN gem install -n /usr/bin bundler
RUN gem install -n /usr/bin rake

WORKDIR /home/vidarh/
ENV HOME /home/vidarh

VOLUME ["/home"]
USER vidarh
EXPOSE 8080
```

这段代码简单易懂：随便选择一个发行版，安装一些经常使用的工具，这些工具因人而异；开放容器的 `8080` 端口；新增一个用户，将其 `userid` 设置为服务器的用户ID，不创建 `/home` 目录；将 `/home` 目录设为共享文件夹，共享文件夹将在下一节介绍。

## 2. 支持共享文件夹的开发容器

为了方便开发，我的所有开发容器都至少有一个共享文件夹 `/home` 。凭借开发模式下的代码重载（code-reloader）特性，容器可以在运行时重载文件系统的改变，这样就不需要为每一次代码提交重启或者重建容器，当然你也可以偶尔重启容器来确认是否有所遗漏。对于测试和生产环境的容器，我不推荐启用共享文件夹功能。取而代之，你应该使用 `ADD` 命令添加修改。

 下面是 "homepage" 开发容器的 Dockerfile 文件，它被我用于开发个人维基。这个容器继承自 "devbase" 容器，这个例子演示了基础容器的重用和共享文件夹的使用。

```docker
FROM vidarh/devbase
WORKDIR /home/vidarh/src/repos/homepage
ENTRYPOINT bin/homepage web
```
*（备注：我本应给 "devbase" 容器加上标签）*

下面是我的个人博客开发容器的 Dockerfile 文件：

```docker
FROM vidarh/devbase

WORKDIR /
USER root

# For Graphivz integration
RUN apt-get update
RUN apt-get -y install graphviz xsltproc imagemagick

USER vidarh
WORKDIR /home/vidarh/src/repos/hokstad-com
ENTRYPOINT bundle exec rackup -p 8080
```

因为这些容器可直接从公共仓库拉取，且基于可重用的基础容器构建，当容器依赖发生变更时，我们可以快速重建容器，这对于避免引入未文档化的依赖（未写入 Dockerfile 中的依赖）也非常重要。

即使我们的基础系统已经非常轻量，却仍有提升的空间：容器中的大部分功能将不会被使用。 Docker 采用写时复制（copy-on-write）层技术，也许不会造成太大资源开销，但这意味着我们没能构建一个最小依赖系统，也无法将被攻击和出错的风险降到最低。

## 3. 开发工具专用容器

本节内容对于通过 `ssh` 等工具进行开发的用户（而非 IDE 用户）比较有价值，最吸引我的是这种模式可以将代码编辑和测试执行（test-executing）从应用中完全分离出来。

一直以来，令我非常困扰的问题是：开发、生产环境和开发工具的依赖经常混在一起。这个问题很难真正解决，在开发过程中我们总会不经意的安装一些未文档化的依赖（比如：通过 `apt-get` 安装 strace / tcpdump / emacs 这些最终不被需要的工具时，不可避免会安装大量的依赖）。过去，我必须花上数周时间 "逆向工程（reverse engineering）" 这些应用的依赖，因为它们已经和我的开发、测试甚至生产环境混在一起。

定期的测试部署（test-deployments）可以帮助我们解决这个问题，但这并不完美。结合以上两种模式，我找到了从根本解决问题的办法。我首先创建一个专用容器，安装 emacs 等必需的开发工具，然后通过笔记本安装的 "autossh" 工具，保持在专用容器的会话里。再借助共享文件夹功能，就可以在专用容器直接编辑开发容器中的代码。考虑到专用容器只会影响调试和开发，有时我会在其中直接试用一些工具（遇到好用的就加入 Dockerfile 中，没用的下次重建容器的时候会自动消失）。下面是开发工具专用容器的 Dockerfile 文件：

```docker
FROM vidarh/devbase
RUN apt-get update
RUN apt-get -y install openssh-server emacs23-nox htop screen

# For debugging
RUN apt-get -y install sudo wget curl telnet tcpdump

# For 32-bit experiments
RUN apt-get -y install gcc-multilib 

# Man pages and "most" viewer:
RUN apt-get install -y man most

RUN mkdir /var/run/sshd
ENTRYPOINT /usr/sbin/sshd -D

VOLUME ["/home"]
EXPOSE 22
EXPOSE 8080
```

利用共享文件夹 "/home" ，通过 `ssh` 可以方便的使用专用容器，我会慢慢加强专用容器的功能，但目前来看，它已经能满足我的需求。

## 4. 针对不同环境的测试容器

我喜欢 Docker 的另一个理由是可以轻松的在不同环境的容器中测试代码。举个栗子，我升级了 [Ruby compiler project](http://www.hokstad.com/compiler) 项目，使其支持 Ruby 1.9 ，但对 Ruby 1.8 的支持情况未知。此时，我仅需一个小小的 Dockerfile ，便可在 Ruby 1.8 下对项目进行测试。

```docker
FROM vidarh/devbase
RUN apt-get update
RUN apt-get -y install ruby1.8 git ruby1.8-dev
```

当然也可以使用 `rbenv` 这类工具，但当需要大量的系统包时这些工具并不省心（和曾经的 Ruby 一样痛苦）。有了 Docker 容器，当我需要不同的运行环境时，运行 `docker run` 命令即可，且不局限于 Ruby 这种自带版本切换工具的环境。完整的虚拟机也能胜任这项任务，但相比之下，Docker 消耗的时间少很多。

## 5. 构建专用容器

虽然大多数时候我使用解释性语言编程，但依然有些时候，我需要执行昂贵的构建（build）操作。举个栗子，用于运行 Ruby 应用的 "bundler" 工具在更新 Rubygems 依赖关系时，如果应用很大，会消耗大量时间。并且，我们经常需要安装一些运行时不需要的依赖，比如：安装 gems 时有时会引入一些依赖的系统包，而这些包通常没有文档化（未写入 Dockerfile 中），非常容易引入整个 build-essential 包及其依赖。同时，因为宿主环境和部署容器的环境可能并不兼容，我也不想在宿主环境中构建（build）应用。

解决方案之一是创建构建（build）专用容器，如果专用容器和应用容器的依赖环境不同，就单独创建 Dockerfile 文件；如果专用容器和应用容器的依赖环境一致，则可复用应用容器的 Dockerfile 文件，使其运行所期望的构建（build）命令。下面是构建（build）专用容器的 Dockerfile 文件。

```docker
FROM myapp
RUN apt-get update
RUN apt-get install -y build-essential [assorted dev packages for libraries]
VOLUME ["/build"]
WORKDIR /build
CMD ["bundler", "install","--path","vendor","--standalone"]
```

运行构建（build）专用容器，并挂载源代码目录到构建（build）专用容器的 "/build" 目录。这样我就可以将构建流程及其依赖封装在两个或多个容器中，最终实现应用构建（build）或者打包（packaging）的解偶。

*（我经常检查整条构建流水线，为了能够100%重现最终的打包步骤，需要检查构建的每一个环节，确保流水线能精确的完成整个构建过程）*

## 6. 安装专用容器

这个方法虽非本人原创，但却着实值得一提。 [nsenter and docker-enter tools](https://github.com/jpetazzo/nsenter) 是一款优秀的工具， 它出自流行的 `curl [some url you have no control over] | bash` 模式，它也提供了一个构建专用容器，但却更进一步。其在 Dockerfile 的最后部分下载并构建了一个合适版本的 nsenter （未对下载文件做完整性检查）。

```docker
ADD installer /installer
CMD /installer
```

其中 "installer" 文件如下所示：

```bash
#!/bin/sh
if mountpoint -q /target; then
       echo "Installing nsenter to /target"
       cp /nsenter /target
       echo "Installing docker-enter to /target"
       cp /docker-enter /target
else
       echo "/target is not a mountpoint."
       echo "You can either:"
       echo "- re-run this container with -v /usr/local/bin:/target"
       echo "- extract the nsenter binary (located at /nsenter)"
fi
```

这种情况下，虽然恶意攻击者可能利用权限问题攻击该容器，但攻击面却非常有限。本节的方法更多时候是为了避免 [开发者在安装脚本时不经意执行危险操作](https://github.com/MrMEEE/bumblebee-Old-and-abbandoned/issues/123) 。我喜欢这种方法，希望通过它能够减少大家对 `curl [something] | bash` 的厌恶（即时杯具发生，也仅会影响专用容器）。

## 7. 默认服务专用容器

当我认真对待一个 app 时，我会快速准备合适的容器，用于安装数据库等服务。对于项目，这一系列经过优化调整的 "基本" 容器都是无价的。与运行 `docker run [some-app-name]` 创建容器或者直接从 Docker Hub 拉取容器相比，我更倾向于先审核它们，搞清楚它们如何处理数据，然后优化调整容器，最后保存在自己的容器库中。下面是 Beanstalkd 的 Dockerfile 文件：

```docker
FROM debian:wheezy
ENV DEBIAN_FRONTEND noninteractive
RUN apt-get -q update
RUN apt-get -y install build-essential
ADD http://github.com/kr/beanstalkd/archive/v1.9.tar.gz /tmp/
RUN cd /tmp && tar zxvf v1.9.tar.gz
RUN cd /tmp/beanstalkd-1.9/ && make
RUN cp /tmp/beanstalkd-1.9/beanstalkd /usr/local/bin/
EXPOSE 11300
CMD ["/usr/local/bin/beanstalkd","-n"]
```
*（对比这个栗子，我有个更加复杂的 postgres 容器，其中包括持久化的数据和各种配置文件）*

我的目标是当需要 memcached ，postgres 和 beanstalk 时，只需运行三条 `docker run` 命令，三个根据我的宿主环境和个人偏好优化的容器将被创建并运行，这样建立“标准”基础设施组件只需一分钟。

## 8. 基础设施 / 胶水容器

上文介绍了很多用于开发环境的容器（这意味着讨论生产环境需要更多的篇幅），但还是漏了一类容器——胶水容器。胶水容器用于将各种环境粘合成一个整体，目前为止，这一领域还有待研究，这里我先举个栗子：

为了更加方便的访问容器，我构建了一个小小的 haproxy 容器，并在家用DNS服务器上添加一条A记录指向该容器，设置 iptable 防火墙使得 haproxy 容器可以正常访问，下面是对应的 Dockerfile 文件：

```docker
FROM debian:wheezy
ADD wheezy-backports.list /etc/apt/sources.list.d/
RUN apt-get update
RUN apt-get -y install haproxy
ADD haproxy.cfg /etc/haproxy/haproxy.cfg
CMD ["haproxy", "-db", "-f", "/etc/haproxy/haproxy.cfg"]
EXPOSE 80
EXPOSE 443
```

首先由脚本生成 `haproxy.cfg` 文件，然后根据 `docker ps` 的输出生成后端（backend）部分：

```
backend test
    acl authok http_auth(adminusers)
    http-request auth realm Hokstad if !authok
    server s1 192.168.0.44:8084
```

前端（frontend）定义中关于 acls 和 use_backend 部分的声明，可将来自 `[hostname].mydomain` 的请求转发到 `backend test` 后端。

如果希望优雅一些，可以部署类似 [AirBnB's Synapse](https://github.com/airbnb/synapse) 这样的服务发现系统，其提供的功能选项远超我的需求。以上这些容器已经能够满足我对家用环境的多数需求，但工作中，我还在不断的推出各种新容器，这些容器使得真实应用的部署轻而易举，就仿佛拥有一套私有云系统。

原文：[Eight Docker Development Patterns](http://www.hokstad.com/docker/patterns)