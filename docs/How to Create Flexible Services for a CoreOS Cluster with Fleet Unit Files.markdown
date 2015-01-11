如何通过fleet unit files 来构建灵活的服务
===

## 简介

CoreOS利用一系列的工具，大大简化了集群和Docker服务的管理。其中，Etcd将独立的节点连接起来，并提供一个存储全局数据的地方。大部分实际的服务管理和管理员任务则有fleet来负责。

在上一个guide中，我们过了一遍fleetctl的基本使用：操纵服务和集群中的节点。在那个guide中，我们简单的介绍了fleet unit文件，fleet使用它来定义服务。还创建一个简单的unit 文件，展示了如何使用fleetctl提供一个工作的service。

在本guide中，我们深度学习fleet unit文件，了解一些使你的service更加强壮的技巧。

## 准备


为了完成本教程，我们假设你已经根据[我们以往的教程][]创建了一个CoreOS集群，假定集群包含以下主机：

- coreos-1
- coreos-2
- coreos-3

尽管本教程的大部分内容是关于unit文件的创建，但在描述unit文件中指令（属性）的作用时会用到这些机器。我们同样假设你阅读了[如何使用fleetctl][]，你已经可以使用fleetctl将unit文件部署到集群中。

当你已经做好这些准备后，请继续阅读本教程。

## unit文件的section和类型

因为fleet的服务管理方面主要依赖于集群节点本地的systemd，所以systemd unit file基本就是 fleet unit file。

fleet unit file 有许多类型，service是最常用的一种。这里列出一些 systemd unit file支持的类型，每个类型（文件）以"文件名.类型名"标识（命名）。比如example.service

- service
- socket
- device
- mount
- automount
- timer
- path

尽管这些类型都是有效的，但service类型用的最多。在本guide中，我们也只讨论service类型的配置。

unit 文件是简单的文本文件，以"文件名.类型名（上面提到的）"命名。在文件内部，它们以section 组织，对于fleet，大部分unit 文件是以下格式：

    [Unit]
    generic_unit_directive_1
    generic_unit_directive_2

    [Service]
    service_specific_directive_1
    service_specific_directive_2
    service_specific_directive_3
    
    [X-Fleet]
    fleet_specific_directive
    

每个小节的头和其他部分都是大小写敏感的。unit小节用来定义一个unit文件的通用信息，在unit小节定义的信息对所有类型（比如service）都通用。

Service小节用来设置一些只用Service类型文件用到的指令，上面提到的每一个类型（但不是全部）（比如service和socket）都有相应的节来定义本类型的特定信息。

fleet根据X-Fleet小节来判定如何调度unit文件。在该小节中，你可以设定一些条件来将unit文件部署到某台换主机上。

## Building the Main Service
在本节，我们的unit文件将和 basic guide on running services on CoreOS 中提到的有所不同。这个文件叫apache.1.service，具体内容如下：

    [Unit]
    Description=Apache web server service
    
    # Requirements
    Requires=etcd.service
    Requires=docker.service
    Requires=apache-discovery.1.service
    
    # Dependency ordering
    After=etcd.service
    After=docker.service
    Before=apache-discovery.1.service
    
    [Service]
    # Let processes take awhile to start up (for first run Docker containers)
    TimeoutStartSec=0
    
    # Change killmode from "control-group" to "none" to let Docker remove
    # work correctly.
    KillMode=none
    
    # Get CoreOS environmental variables
    EnvironmentFile=/etc/environment
    
    # Pre-start and Start
    ## Directives with "=-" are allowed to fail without consequence
    ExecStartPre=-/usr/bin/docker kill apache
    ExecStartPre=-/usr/bin/docker rm apache
    ExecStartPre=/usr/bin/docker pull username/apache
    ExecStart=/usr/bin/docker run --name apache -p ${COREOS_PUBLIC_IPV4}:80:80 \
    username/apache /usr/sbin/apache2ctl -D FOREGROUND
    
    # Stop
    ExecStop=/usr/bin/docker stop apache
    
    [X-Fleet]
    # Don't schedule on the same machine as other Apache instances
    X-Conflicts=apache.*.service
    
我们先说unit section，在本seciton，描述了unit 的基本信息和依赖情况。服务启动前有一系列的requirements，并且本例中使用的是hard requirements。如果我们想让fleet在启动本服务时启动其它服务，并且如果其他服务启动失败不影响本服务的启动，我们可以使用Wants来代替requirements。

接下来，我们明确的列出了requirements 启动的顺序。his is important so that the prerequisite services are available when they are needed. It is also the way that we automatically kick off（使开始） the sidekick（伙伴） etcd announce service that we will be building.

在service section中，我们关闭了服务启动间隔时间。因为当服务开始在一个节点上运行时，相应镜像需要先从doker registry中pull下来，pull的时间会被计入startup timeout。这个值默认是90秒，通常是足够的。但对于比较复杂的容器来说，可能需要更长时间。

我们把killmode 设置为none，这是因为正常的kill 模式有时会导致容器移除命令执行失败（尤其是执行是docker --rm containerID/containername时），这回导致下次启动时出问题。

我们把environment 文件也拉下来，这样我们就可以访问COREOS_PUBLIC_IPV4。并且，if private networking was enabled during creation, the COREOS_PRIVATE_IPV4 environmental variables. These are very useful for configuring Docker containers to use their specific host's information.

ExecStartPre 行被用来清除上次运行的残留，确保本次的执行环境是干净的。对于头两个ExecStartPre 使用“=-”，来告诉systemd：即使这两个命令执行失败了，也继续执行接下来的命令。这样，docker 将尝试kill 并且 移除先前的容器，没有发现容器也没有关系。最后一个ExecStartPre用来确保将要运行的container是最新的。

The actual start command boots the Docker container and binds it to the host machine's public IPv4 interface. This uses the info in the environment file and makes it trivial（不重要的） to switch interfaces and ports. The process is run in the foreground because the container will exit if the running process ends. The stop command attempts to stop the container gracefully.

X-fleet section包含一个简单的条件，强制fleet将service调度某个没有运行Apache 服务的机器上。这样可以很容易的提高服务的可靠性，因为它们运行在不同的机器上（一个机器挂了，还有另外的机器运行在该服务）。

## Basic Take-Aways（熟食，可以理解为忠告） For Building Main Services

在以上的例子中，我们搞了一个相当简单基本的配置。然而，我们可以从中学到许多知识点，并应用到构建服务的过程中。

在构建main service时候要记住下面的一些点：

- 将服务的依赖和启动顺序独立出来：Lay out your dependencies with Requires= or Wants= directives dependent on whether the unit you are building should fail if the dependency cannot be fulfilled. Separate out the ordering with separate After= and Before= lines so that you can easily adjust if the requirements change. Separating the dependency list from ordering can help you debug in case of dependency issues.

- 用一个独立的进程实现服务注册：你的服务应该注册到etcd，它可以帮你实现服务发现和动态配置。但你必须用一个独立的“伙伴”容器来做这件事，以保持逻辑的独立性。这样做可以帮你更加精准的报告服务的健康状态，并从一个外部视图查看这些状态。而其他组件也会需要这些数据（健康状态）。

- 要意识到你的服务启动超时的可能性：Consider adjusting the TimeoutStartSec directive in order to allow for longer start times. Setting this to "0" will disable a startup timeout. This is often necessary because there are times when Docker has to pull an image (on first-run or when updates are found), which can add significant time to initializing the service.

- Adjust the KillMode if your service does not stop cleanly：Be aware of the KillMode option if your services or containers seem to be stopping uncleanly. Setting this to "none" can sometimes resolve issues with your containers not being removed after stopping. This is especially important when you name your containers since Docker will fail if a container with the same name has been left behind from a previous run. Check out the documentation on KillMode for more information

- Clean up the environment before starting up: Related to the above item, make sure to clean up previous Docker containers at each start. You should not assume that the previous run of the service exited as expected. These cleanup lines should use the =- specifier to allow them to fail silently if no cleanup is needed. While you should stop containers with docker stop normally, you should probably use docker kill during cleanup.

- Pull in and use host-specific information for service portability: If you need to bind your service to a specific network interface, pull in the /etc/environment file to get access to COREOS_PUBLIC_IPV4 and, if configured, COREOS_PRIVATE_IPV4. If you need to know the hostname of the machine running your service, use the %H systemd specifier. To learn more about possible specifiers, check out the systemd specifiers docs. In the [X-Fleet] section, only the %n, %N, %i, and %p specifiers will work.

## Building the Sidekick Announce Service

现在当我们构建主unit文件时候，已经有了一些不错的想法。接下来，我们看一下传统的从service。这个从 service和主 serivce有一定联系，被用来向etcd注册服务。

这个文件，就像它在主unit文件中被引用的那样，叫做`apache-discovery.1.service`，内容如下：

    [Unit]
    Description=Apache web server etcd registration
    
    # Requirements
    Requires=etcd.service
    Requires=apache.1.service
    
    # Dependency ordering and binding
    After=etcd.service
    After=apache.1.service
    BindsTo=apache.1.service
    
    [Service]
    
    # Get CoreOS environmental variables
    EnvironmentFile=/etc/environment
    
    # Start
    ## Test whether service is accessible and then register useful information
    ExecStart=/bin/bash -c '\
      while true; do \
        curl -f ${COREOS_PUBLIC_IPV4}:80; \
        if [ $? -eq 0 ]; then \
          etcdctl set /services/apache/${COREOS_PUBLIC_IPV4} \'{"host": "%H", "ipv4_addr": ${COREOS_PUBLIC_IPV4}, "port": 80}\' --ttl 30; \
        else \
          etcdctl rm /services/apache/${COREOS_PUBLIC_IPV4}; \
        fi; \
        sleep 20; \
      done'
    
    # Stop
    ExecStop=/usr/bin/etcdctl rm /services/apache/${COREOS_PUBLIC_IPV4}
    
    [X-Fleet]
    # Schedule on the same machine as the associated Apache service
    X-ConditionMachineOf=apache.1.service

和讲述主service一样，先从文件的unit节开始。依赖关系和启动顺序就不谈了。

第一个新的指令（参数）是`BindsTo=`，这个指令意味着
当fleet start、stop和restart `apache.1.service`时，`apache-discovery.1.service`也做同样操作。这意味着，我们使用fleet 只操作一个主服务，即同时操作了主服务与与之"BindsTo"的服务。并且，这个机制是单向的，fleet操作`apache-discovery.1.service`对 `apache.1.service`没有影响。

对于service节，我们应用了`/etc/environment`文件，因为我们需要它包含的环境变量（如果将unit文件视作c程序，引入environment文件，有点引入全局变量的意思）。`ExecStart=`在此处是一个bash脚本，它企图使用暴漏的ip和端口访问主服务。

如果连接成功，使用etcdctl设置etcd中`/services/apache/{COREOS_PUBLIC_IPV4}`为一个json串：`{"host": "%H", "ipv4_addr": ${COREOS_PUBLIC_IPV4}, "port": 80}`。这个key值的有效期为30s，一次来确保一旦这个节点down掉，etcd中不会残留这个key值的信息。如果连接失败，则立即从etcd移除key值，因为主服务已经失效了。

循环有20s的休息时间，每隔20秒（在30s内etcd中key失效之前）重新检查一下主服务是否有效并重置key。这将更新ttl时间，确保在下一个30秒内key值是有效的。

本例中，stop指令仅仅是将key从etcd中移除，当stop主服务时，因为`BIndsTo=`，本服务也将被执行，从而从etcd移除注册信息。

对于X-fleet小节，我们需要确保该unit和主unit运行在一台机器上。结合`BindsTo=`指令，这样做可以将主节点信息汇报到远程主机上。


## Basic Take-Aways For Building Sidekick Services

通过创建这个从unit，当创建类似的unit时，要记住下面一些点：

- 校验主unit是否有效: It is important to actually check the state of main unit. Do not assume that the main unit is available just because the sidekick has been initialized. This is dependent on what the main unit's design and functionality is, but the more robust(强健的) your check, the more credible your registration state will be. The check can be anything that makes sense for the unit, from checking a /health endpoint to attempting to connect to a database with a client.

- 循环检测: Checking the availability of the service at start is important, but it is also essential that you recheck at regular intervals. This can catch instances of unexpected service failures, especially if they somehow result in the container not stopping. The pause between cycles will have to be tweaked according to your needs by weighing the importance of quick discovery against the additional load on your main unit.

- 使用ttl flag来防止校验失效: Unexpected failures of the sidekick unit can result in stale discovery information in etcd. To avoid conflicts between the registered and actual state of your services, you should let your keys time out. With the looping construct above, you can refresh each key before the timeout interval to make sure that the key never actually expires while the sidekick is running. The sleep interval in your loop should be set to slightly less than your timeout interval to ensure this functions correctly.

- 向etcd存入更多有用的信息，而不只是确认信息: During your first iteration of a sidekick, you may only be interested in accurately registering with etcd when the unit is started. However, this is a missed opportunity to provide plenty of useful information for other services to utilize. While you may not need this information now, it will become more useful as you build other components with the ability to read values from etcd for their own configurations. The etcd service is a global key-value store, so don't forget to leverage this by providing key information. Storing details in JSON objects is a good way to pass multiple pieces of information.


By keeping these considerations in mind, you can begin to build robust registration units that will be able to intelligently ensure that etcd has the correct information.

## Fleet-Specific Considerations
While fleet unit files are for the most part no different from conventional systemd unit files, there are some additional capabilities and pitfalls.

The most obvious difference is the addition of a section called [X-Fleet] which can be used to direct fleet on how to make scheduling decisions. The available options are:

- X-ConditionMachineID: 
它可以指定一个特定的machine来加载unit。
The provided value is a complete machine ID. This value can be retrieved from an individual member of the cluster by examining the /etc/machine-id file, or through fleetctl by issuing the list-machines -l command. The entire ID string is required. This may be needed if you are running a database with the data directory kept on a specific machine. Unless you have a specific reason to use this, try to avoid it because it decreases the flexibility of the unit.

- X-ConditionMachineOf:  
它用来设定将unit部署到运行某个特定unit的machine上。This is helpful for sidekick units or for lumping associated units together.

X-Conflicts: 这个和上一个参数的作用恰恰相反, 它指定了本unit不想和哪些unit运行在同一台机器上. This is useful for easily configuring high availability by starting multiple versions of the same service, each on a different machine.

X-ConditionMachineMetadata: 它基于machine的metadata来确定调度策略。 In the "METADATA" column of the fleetctl list-machines output, you can see the metadata that has been set for each host. To set metadata, pass it in your cloud-config file when initializing the server instance.

Global: 是一个布尔值，用来确定你是否想把unit部署到集群的每一台machine上。 Only the metadata conditional can be used alongside this directive.

这些额外的directives（指示）让管理员灵活的在集群上部署服务，在unit被提交到machinie的systemd实例之前，（directives）指定的策略将会被执行。

还有一个问题，什么时候使用fleet来部署相关的unit。除了X-fleet 小节外，fleetctl工具不考虑units之间的任何依赖问题。因此在使用fleet部署相关的unit是，会有一些有趣的问题。

这意味着，尽管fleetctl工具会执行必要的步骤直到unit变成期望的状态，提交，加载，执行unit中的命令。但不会解决unit之间的依赖问题。

所以，如果你同时提交了主从两个unit:A和B，但还没有加载。执行`fleetctl start A.service`将加载并执行`A.service` unit。然而，因为`B.service` unit还没有加载，并且因为fleetctl 不会考虑unit之间的依赖问题，`A.service`将会执行失败。因为一旦machine的systemd开始执行`A.service`，却没有找到`B.service`，而systemd会考虑依赖关系。

为了避免这种伙伴unit执行失败的情况，你可以手动同时启动两个unit`fleet start A.service B.service`  ，不要依赖`BindsTo=`执令。

另一种方法是当运行主unit的时候，确保从unit至少已经被加载。**被加载意味着一台machine已经被选中，并且unit file已被提交到systemd实例**，此时满足了依赖条件，`BindsTo`参数也能够正确执行。

    fleetctl load A.service B.service
    fleetctl start A.service
    
如果fleetctl执行多个unit时失败，请记起这一点。

## Instances and Templates

unit template是fleet一个非常有用的概念。

unit templates依赖systemd一个叫做"instance"的特性。systemd运行时通过计算template unit文件实例化的unit file。template文件和普通unit文件大致相同，只有一点改进。但如果正确使用，威力无穷。

你可以在文件名中加入"@"来标识一个template 文件，比如一个传统的service文件名：`unit.service`，一个template文件则可以叫做`unit@.service`。
当template文件被实例化的时候，在"@"和".service"之间将有一个用户指定的标识符：`unit@instance_id.service`。在template文件内部，可以用%p来指代文件名（即unit），类似的，%i可以指代标识符（即instance_id）。

## Main Unit File as a Template

我们可以创建一个template文件:apache@.service。

    [Unit]
    Description=Apache web server service on port %i
    
    # Requirements
    Requires=etcd.service
    Requires=docker.service
    Requires=apache-discovery@%i.service
    
    # Dependency ordering
    After=etcd.service
    After=docker.service
    Before=apache-discovery@%i.service
    
    [Service]
    # Let processes take awhile to start up (for first run Docker containers)
    TimeoutStartSec=0
    
    # Change killmode from "control-group" to "none" to let Docker remove
    # work correctly.
    KillMode=none
    
    # Get CoreOS environmental variables
    EnvironmentFile=/etc/environment
    
    # Pre-start and Start
    ## Directives with "=-" are allowed to fail without consequence
    ExecStartPre=-/usr/bin/docker kill apache.%i
    ExecStartPre=-/usr/bin/docker rm apache.%i
    ExecStartPre=/usr/bin/docker pull username/apache
    ExecStart=/usr/bin/docker run --name apache.%i -p ${COREOS_PUBLIC_IPV4}:%i:80 \
    username/apache /usr/sbin/apache2ctl -D FOREGROUND
    
    # Stop
    ExecStop=/usr/bin/docker stop apache.%i
    
    [X-Fleet]
    # Don't schedule on the same machine as other Apache instances
    X-Conflicts=apache@*.service


就像你看到的，我们将`apache-discovery.1.service`改为`apache-discovery@%i.service`。即如果我们有一个unit文件`apache@8888.service`，它将需要一个从服务`apache-discovery@8888.service`。%i 曾被 实例化的标识符（即8888） 替换过。在这个例子中，%i也可以被用来指代服务的一些信息，比如apahce server运行时占用的端口。

为了使其工作，我们改变了`docker run`的参数，将containr的80端口，暴漏给主机的某一个端口。在静态的unit 文件中，我们使用`${COREOS_PUBLIC_IPV4}:80:80`,将container的80端口，映射到主机${COREOS_PUBLIC_IPV4}网卡的80端口。在这个template 文件中，我们使用`${COREOS_PUBLIC_IPV4}:%i:80`，使用`%i`来说明我们使用哪个端口。在template文件中选择合适的instance identifier会带来很大的灵活性。

container的名字被改为了基于instance ID的`apache.%i`，**记住container的名字不能够使用`@`标识符**。现在，我们校正了运行container用到的所有指令（参数）。

在X-Fleet小节，我们同样改变了部署信息来替代原先的配置。

## Sidekick Unit as a Template

将从unit文件模板化也是类似的过程，新的从unit文件叫`apache-discovery@.service`，内容如下：

    [Unit]
    Description=Apache web server on port %i etcd registration
    
    # Requirements
    Requires=etcd.service
    Requires=apache@%i.service
    
    # Dependency ordering and binding
    After=etcd.service
    After=apache@%i.service
    BindsTo=apache@%i.service
    
    [Service]
    
    # Get CoreOS environmental variables
    EnvironmentFile=/etc/environment
    
    # Start
    ## Test whether service is accessible and then register useful information
    ExecStart=/bin/bash -c '\
      while true; do \
        curl -f ${COREOS_PUBLIC_IPV4}:%i; \
        if [ $? -eq 0 ]; then \
          etcdctl set /services/apache/${COREOS_PUBLIC_IPV4} \'{"host": "%H", "ipv4_addr": ${COREOS_PUBLIC_IPV4}, "port": %i}\' --ttl 30; \
        else \
          etcdctl rm /services/apache/${COREOS_PUBLIC_IPV4}; \
        fi; \
        sleep 20; \
      done'
    
    # Stop
    ExecStop=/usr/bin/etcdctl rm /services/apache/${COREOS_PUBLIC_IPV4}
    
    [X-Fleet]
    # Schedule on the same machine as the associated Apache service
    X-ConditionMachineOf=apache@%i.service

当主unit和从unit都是静态文件的时候，我们已经讲述了如何将从unit绑定到主unit，现在我们来讲下如何将实例化的从 unit绑定到同样根据模板实例化的主unit。

`curl`命令可以检查服务的有效性，为了让其连接到正确的url，我们用instance ID来代替80端口。这是非常必要的，因为在主unit中，我们改变了container暴漏的端口。

我们改变了写入到etcd的端口信息，同样是使用instance id来替换80端口。这样，设置到etcd的json信息就可以是动态的。它将采集service服务所在主机的hostname,ip地址和端口信息。

最后，我们也更改了X-Fleet小节，我们需要确定这个进程和其对应的主unit运行在一台主机上。

## Instantiating Units from Templates

实例化template文件有多种方法：

fleet和systemd都可以处理链接，我们可以创建模板文件的链接文件：

    ln -s apache@.service apache@8888.service
    ln -s apache-discovery@.service apache-discovery@8888.service
   
这将创建两个链接，叫做`apache@8888.service` 和 `apache-discovery@8888.service`，每个文件都包含了fleet和systemd运行它们所需要的所有信息，我们可以使用`fleetctl`提交、加载和启动这些服务`fleetctl start apache@8888.service apache-discovery@8888.service`。

如果你不想使用链接文件的方式，可以直接使用`fleetctl`来提交模板文件：`fleetctl submit apache@.service apache-discovery@.service`，你可以在运行时为其赋予一个instance identifier，比如：`fleetctl start apache@8888.service apache-discovery@8888.service`

这种方式不需要链接文件，然而一些系统管理员偏好于使用链接文件的方式。因为链接文件一旦创建完毕便可以在任意时刻使用，并且，如果你将所有的连接文件放在一个地方，便可以在同一时刻启动所有实例。

假设我们将静态unit文件全放在static目录下，模板文件放在templates目录下，根据模板文件创建的链接文件全放在instances目录下，你就可以启动所有的实例：`fleetctl start instances/*`。如果你的服务很多的话，这将非常方便。

## 小结

通过本章，相信你对如何创建unit文件以及fleet的使用有了深入的了解。利用unit文件中灵活的指令（参数），你可以将将你的服务分布化，将相互依赖的服务部署在一起，并将它们的信息注册到etcd中。

## 小结

一个主服务，我们应该为它设计一个从服务，这个两个服务之间有一定的关系。从服务和etcd交互也有一定的技巧。

还可以使用模板，进一步提高使用的灵活性。模板文件的使用：

写一个模板文件， 创建它们的链接（链接名的标识符要有具体的值），这个链接就可以是一个具体的unit文件了。

本文首先介绍了unit 文件的构成。然后介绍了

静态方式下

* 构建主服务
* 构建从服务
* 模板文件

模板文件方式下

* 构建主服务 
* 构建从服务


[我们以往的教程]: https://www.digitalocean.com/community/tutorials/how-to-set-up-a-coreos-cluster-on-digitalocean
[如何使用fleetctl]: https://www.digitalocean.com/community/tutorials/how-to-use-fleet-and-fleetctl-to-manage-your-coreos-cluster