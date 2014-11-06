Title: 利用 Vagrant 部署 CoreOS 和 Docker 容器（译）
date: 2014-10-30
url: getting-started-with-coreos-and-docker-using-vagrant
tags: docker, coreos, vagrant, etcd, fleet
toc: yes
status: draft

这是关于 [CoreOS](https://coreos.com/) 和 [Docker](https://docker.com/) 系列文中的第一篇，这篇文章主要介绍以下内容：

1. CoreOS 是什么？
2. 如何在 [Vagrant](https://www.vagrantup.com/) 虚拟机上运行 CoreOS 系统？
3. 如何使用 [fleet](https://github.com/coreos/fleet) 向 CoreOS  集群提交任务（比如：运行 Docker 容器）和执行基本管理任务？

这一系列文章基本阐明了 fleet 介绍视频的内容： [利用 fleet 部署 Docker 容器集群](https://coreos.com/blog/cluster-level-container-orchestration/) ，只是用 Vagrant 代替了 [Amazon EC2](https://aws.amazon.com/cn/ec2/) 。

## 一、为什么选择 CoreOS ?

我喜欢 Docker ，它非常有趣。但如果你希望在生产环境中大规模使用 Docker ，还需要施加一些魔法。我们希望不构建 [PaaS](https://zh.wikipedia.org/zh/%E5%B9%B3%E5%8F%B0%E5%8D%B3%E6%9C%8D%E5%8A%A1) 平台，也能充分释放 Docker 的潜力。在集群系统上运行服务，需要一个好的解决方案来解决服务发现、配置管理和部署等问题，单靠 Docker 无法完成这些。

市场上已有很多基于 Docker 的有趣项目，比如： [Dokku](https://github.com/progrium/dokku) ， [Flynn](https://flynn.io/) 和 [Deis](http://deis.io/) ， CoreOS 也算其中之一。但 CoreOS 并未努力成为另一个 PaaS 平台，它是一个专为大规模集群系统裁剪的 Linux 发行版，选择性吸收了一些 PaaS 平台特性。

## 二、CoreOS 是什么？

大多数 Linux 发行版自带很多软件包，有些包在服务部署时并不需要。 CoreOS 将不需要的统统裁掉，只保留系统核心，同时支持通过 Docker 按需创建应用容器。

CoreOS 提供 [etcd](https://github.com/coreos/etcd) （用于服务发现和配置管理的高可用、分布式键值存储服务）和 fleet （协助 etcd 提供分布式初始化服务，集群系统下的 [systemd](http://www.freedesktop.org/wiki/Software/systemd/) ）两个工具。有了 CoreOS 和这些工具，就可以对集群系统而非单个主机执行所希望的操作。通常 CoreOS ， etcd 和 fleet 不会制造麻烦（参考 [Apache ZooKeeper](http://zookeeper.apache.org/) ， [doozerd](https://github.com/ha/doozerd) 和 [Substack's Fleet](https://github.com/substack/fleet) 文档），有了 systemd 和 Docker 后，它们更加配合默契，已然是一个整体。

## 三、具体部署步骤

以下步骤均在 OS X 环境下进行， Linux 下步骤雷同。

### 1. 安装依赖软件

1. [VirtualBox](https://www.virtualbox.org/) -  使用版本 4.3.10
2. Vagrant - 使用版本 1.5.2
3. fleetctl - fleet 的命令行客户端

首先，直接下载解压安装 fleetctl 客户端：

```bash
$ wget https://github.com/coreos/fleet/releases/download/v0.3.2/fleet-v0.3.2-darwin-amd64.zip
$ unzip fleet-v0.3.2-darwin-amd64.zip
$ sudo cp fleet-v0.3.2-darwin-amd64/fleetctl /usr/local/bin/
```

或者通过 homebrew 安装：

```bash
$ brew update
$ brew install fleetctl
```

> **注意**：考虑到 CoreOS 正在快速开发中， fleetctl 可能随时更新， CoreOS 及其 Vagrantfile 也是如此。使用之前，请确认这些工具和 CoreOS 版本保持一致。

### 2. 克隆 CoreOS 的 Vagrantfile 仓库

```bash
$ git clone https://github.com/coreos/coreos-vagrant/
$ cd coreos-vagrant
$ DISCOVERY_TOKEN=`curl -s https://discovery.etcd.io/new`
$ perl -p -e "s@#discovery: https://discovery.etcd.io/<token>@discovery:$DISCOVERY_TOKEN@g" user-data.sample > user-data
$ export NUM_INSTANCES=1
```

将 `user-data.sample` 重命名为 `user-data` ，为让 Vagrantfile 启动 CoreOS 中的 etcd 和 fleet 工具；生成一个 discovery token 作为集群的唯一标识，为让所有节点知道自己所属集群。最后一行告诉 Vagrant 共启动多少台虚拟机，默认一台。

### 3. 启动 Vagrant 虚拟机

启动 Vagrant 虚拟机，并通过 SSH 登入。

```bash
$ vagrant up
```

> **Vagrant 小贴士**：执行 `vagrant ssh` 命令登入 Vagrant 虚拟机，执行 `Ctrl-d` 退出当前 SSH 会话，此时虚拟机仍在后台运行，通过 `vagrant ssh` 可重新登入。修改 Vagrantfile 后需要执行 `vagrant reload --provision` 命令重载配置文件并重启虚拟机。执行 `vagrant halt` 命令关闭虚拟机，执行 `vagrant destroy` 命令销毁虚拟机和所有内部数据。希望了解更多 Vagrant 相关知识，执行 `vagrant --help` 命令查看。

到此为止，一个基本可用的 CoreOS 虚拟机已准备就绪。

### 4. 在 Vagrant 虚拟机上运行基于 CoreOS 的 Docker 容器

本节将介绍如何在 CoreOS 虚拟机中远程拉取公共仓库（public registry）里的 Docker 镜像，这里以运行预制的 [dillinger.io](http://dillinger.io/)（一个基于 HTML5 的 Markdown 编辑器）镜像为例。首先在虚拟机中创建 dillinger.io 服务的 systemd unit 文件（告诉 systemd 如何启动和关闭一项服务的配置文件），命名为 `dillinger.service` ：

```bash
$ cat dillinger.service
[Unit]
Description=dillinger.io
Requires=docker.service
After=docker.service

[Service]
ExecStart=/usr/bin/docker run -p 3000:8080 dscape/dillinger
```

这个文件指导 systemd 如何通过 `docker run` 命令启动容器：基于 `dscape/dillinger` 镜像创建容器，绑定容器的 `8080` 端口到宿主（虚拟机）的 `3000` 端口。此外，我们需要创建环境变量，告诉 `fleetctl` 如何通过 SSH 与 虚拟机进行通信，简单测试一下：

```bash
$ export FLEETCTL_TUNNEL=127.0.0.1:2222
$ ssh-add ~/.vagrant.d/insecure_private_key
$ fleetctl list-machines
MACHINE      IP            METADATA
c9f8bd2f...  172.17.8.101  -
```

然后顺序输入下面的命令（以 `$` 开头的行）：

```bash
$ fleetctl submit dillinger.service
$ fleetctl list-units
UNIT                    LOAD    ACTIVE   SUB    DESC           MACHINE
dillinger.service       -       -        -      dillinger.io   -
$ fleetctl start dillinger.service
Job dillinger.service scheduled to c6d23f21.../172.17.8.101
$ fleetctl list-units
UNIT                    LOAD    ACTIVE  SUB     DESC           MACHINE
dillinger.service       loaded  active  running dillinger.io   c6d23f21.../172.17.8.101
$ fleetctl journal dillinger.service
-- Logs begin at Sat 2014-04-19 22:45:43 UTC, end at Sat 2014-04-19 22:48:53 UTC. --
Apr 19 22:47:12 core-01 systemd[1]: Starting dillinger.io...
Apr 19 22:47:12 core-01 systemd[1]: Started dillinger.io.
Apr 19 22:47:12 core-01 docker[3136]: Unable to find image 'dscape/dillinger' locally
Apr 19 22:47:12 core-01 docker[3136]: Pulling repository dscape/dillinger
```

`fleetctl list-units` 命令将列出所有提交过的容器，告诉我们它的状态和集群中所属机器。 `fleetctl journal  -f dillinger.service` 命令打印指定容器最近的日志，加上 `-f` 参数后自动刷新日志。如你所见，从远程仓库拉取 Docker 镜像会花费一些时间，等待之余，不妨通过 SSH 登入虚拟机，了解一些诊断工具。

```bash
$ vagrant ssh
$ docker ps -a
CONTAINER ID        IMAGE                     COMMAND                CREATED             STATUS              PORTS                    NAMES
018a1bbbaadd        dscape/dillinger:latest   /bin/sh -c 'forever    7 seconds ago       Up 7 seconds        0.0.0.0:3000->8080/tcp   agitated_babbage
$ sudo netstat -tulpn
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp6       0      0 :::22                   :::*                    LISTEN      1/systemd
tcp6       0      0 :::3000                 :::*                    LISTEN      3125/docker
tcp6       0      0 :::7001                 :::*                    LISTEN      3064/etcd
tcp6       0      0 :::4001                 :::*                    LISTEN      3064/etcd
```

`docker ps -a` 命令显示容器已经开始运行，正如上文配置文件：容器的 `8080` 端口映射到虚拟机的 `3000` 端口， `netstat -tulpn` 列出所有正在监听端口的进程（加上 `sudo` 后显示最后一列进程ID和名称），不出所料 Docker 正在监听 `3000` 端口。通过浏览器访问  [http://172.17.8.101:3000](http://172.17.8.101:3000/)  ，你会看到 dillinger.io 实例已经运行起来。

如果没有正常运行，检查步骤，查看日志，通过 `docker ps` 和 `netstat` 进行彻底排查。如果 `docker ps -a` 什么都没有返回，同时日志也未报错，也许虚拟机仍在下载镜像中。

## 四、结束语

我们开启了一个运行简单服务的 CoreOS 虚拟机，同时了解了一些简单的管理和诊断工具。你可能希望将其作为一个开发沙箱，在里面折腾 Docker 容器和创建 Docker 镜像。下一篇文章里面，我们将学习利用 etcd 和 fleet 管理集群系统中的多台虚拟机。

原文：[Getting Started with CoreOS and Docker using Vagrant](http://lukebond.ghost.io/getting-started-with-coreos-and-docker-using-vagrant/)
