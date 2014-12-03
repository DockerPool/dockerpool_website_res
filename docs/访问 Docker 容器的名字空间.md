熟悉 Linux 技术的人都知道，容器只是利用名字空间进行隔离的进程而已，Docker 在容器实现上也是利用了 Linux 自身的技术。

有时候，我们需要在宿主机上对容器内进行一些操作，当然，这种绕过 Docker 的操作方式并不推荐。

如果你使用的是比较新的 Docker 版本，会尴尬的发现，直接使用系统命令，会无法访问到容器名字空间。


这里，首先介绍下 `ip netns` 系列命令。这些命令负责操作系统中的网络名字空间。

首先，我们使用 `add` 命令创建一个临时的网络名字空间
```
$ip netns add test
```
然后，使用 `show` 命令来查看系统中的网络名字空间，会看到刚创建的 test 名字空间。
```
$ip netns show
test
```

另外，一个很有用的命令是 `exec`，会在对应名字空间内执行命令。例如
```
$ ip netns exec test ifconfig
```

使用 `del` 命令删除刚创建的 test 名字空间。
```
$ip netns del test
```

接下来运行一个 Docker 容器，例如

```sh
$ docker run -it ubuntu
```

再次执行 `ip netns show`命令。很遗憾，这里什么输出都没有。


原因在于，Docker 启动容器后仍然会以进程号创建新的名字空间，但在较新的版本里面，默认删除了系统中的名字空间信息文件。

网络名字空间文件位于 /var/run/netns 下面，比如我们之前创建的 test 名字空间，则在这个目录下有一个 test 文件。诸如 netns 类似的系统命令依靠这些文件才能获得名字空间的信息。

在容器启动后，查看这个目录，会发现什么都没有。

OK，那让我们手动重建它。

首先，使用下面的命令查看容器进程信息，比如这里的1234。
```
$ docker inspect --format='{{ .State.Pid }}' $container_id
1234
```

接下来，在 /proc 目录（保存进程的所有相关信息）下，把对应的网络名字空间文件链接到 /var/run/netns 下面

```
$ ln -s proc/1234/ns/net /var/run/netns/
```

然后，就可以通过正常的系统命令来查看或访问容器的名字空间了。例如

```
$ip netns show
1234
$ ip netns exec 1234 ifconfig eth0 172.16.0.10/16
...
```