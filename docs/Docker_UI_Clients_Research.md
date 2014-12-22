---
date: 2014-12-22
tags: docker, ui
author: fundon
---

# Docker UI 客户端调研


## 调研


### [DockerUI][]

  DockerUI，目前功能比较简单，docker 的一个 web 接口

  - 作者：[crosbymichael][]，Docker 员工

  - 功能：

    * Dashboard
      - 列出正在运行的 Containers
      - 统计 Containers
      - 统计 Images

    * Containers
      - 操作
        - Start/Stop
        - Pause/Unpause
        - Kill
        - Remove
      - 基本信息、运行状态

    * Images
      - 操作
        - 删除
        - 创建 Container
        - 添加 Tag
      - 基本信息


### [Shipyard][]

  Shipyard，可组合的 Docker 管理工具

  - 作者：[ehazlett][]

  - 功能：

      * Dashboard
        - 列出集群中CPU总共使用率
        - 列出集群中内存总共使用率
        - 最新操作日志

      * Containers
        - 列出集群中所有运行的容器(ID，Name，Image，Engine，CPUs，Memory，State，Type)
        - 操作
         - 发布新的容器，支持所有Docker参数，并支持调度，以及容器初始个数。
         - Restart/Stop
         - Destory
         - Scale
         - Logs

      * Engines
        - 列出集群中所有引擎（主机）
        - 操作
         - 添加新引擎（使用docker TCP绑定，CPUs，Memory信息）
         - 删除引擎

      * Events
        - 列出集群中所有引擎中容器的操作日志
        - 清除日志


### [Kitematic][]

  Kitematic，Mac 上的一个简单的 Docker 管理器

  - 作者：[@kitematic][]

  - 功能：

      * Containers
        - 创建 Container，从 Images 列表中选择需要的 Source Image
        - Container
          * 查看暴露的端口
          * Volumes
          * Terminal，打开终端，访问 Container bash
          * Stop
          * Restart
          * Logs
          * Settings
            - Ports
            - 设置环境变量
            - Volumes，开启/禁用
            - 删除 Container

      * Images
        - 创建 Image，可以导入包含 `Dockerfile` 文件的目录，进行 Build Image
        - 列出 Images
        - Image
            - 创建 Container
            - 日志信息
            - 删除
            - 改变目录，重新 Build Image
            - 基本状态信息

      * Settings
        - Boot2Docker VM 磁盘、内存使用率
        - Report issue

  - 技术栈:

    * [boot2docker][]

    * [Atom-Shell][]

    * [Meteor][]


### [Gaudi][]

  Gaudi，一个可视化的结构生成器，一个管弦乐演奏家   
  使用它可以简单快速启动任何类型的应用，不需要 Docker 和系统配置知识，通过 **link** 的方式关联它们   

  - 作者：[marmelab][]

  - 功能：

    * Builder

      - 组件（component），多个组件模板供选择

        * 可托拽
        * 可配置
        * 可删除
        * 可关联（**link**）其他组件

      - `.gaudi.yml`（yml）配置文件

        * 组件变动，实时更新
        * 复制、剪贴
        * 下载

    * `gaudi` cli

      根据 `.gaudi.yml` 文件描述，自动创建 Containers，并且 link 它们，快速构建一个 app 或者 服务，这点类似 [fig][]


### [Juju][]

  Juju，自动化你的云基础设施   
  通过 GUI 可视化工具或者命令行工具设计、创建、配置、部署、管理你的基础设施，使你能够专注的创造神奇的应用

  - 作者：[ubuntu][]

  - 功能：

    * Charms
        有个 Charms Store，你可以选择你需要的应用

    * Configure
        把需要的 Charm 托拽都画布中，在部署前，可以修改这些服务的配置

    * Deploy
        简单单击（或者执行命令从命令行中）就可以在几秒钟把这些服务部署到目标云   
        可以看到启动进度和错误

    * Build relations
        可以关联这些服务，构建一个应用架构，每个 Charm 都知道如何去连接其他 Charms

    * Monitor

    * Diagnose

    * Scaling

    * Added intelligence with Landscape

    * Import and export your environments

  - [Try Juju][]


### [Seagull][] 海鸥

  海鸥是Docker的最佳小伙伴，它为Docker实现了一套Web监控页面

  - 作者：[tobegit3hub][]

  - 功能：

    * Containers
        - 列出
        - 筛选
        - Container
            * Stop
            * Refresh
            * 基本信息
            * 运行状态

    * Images
        - 列出
        - 筛选
        - 删除 Image
        - Image
            * 基本信息

    * Configuration
        Go，Docker，HTTP API 信息

    * DockerHub
        - 列出当前使用到 DockerHub 中的镜像
        - 筛选

    * Export JSON File

    * More
        - English
        - Chinese

  - [Try Seagull][]



### [Dockerboard][]

  Dockerboard，一个可视化的平台，让你的 Dockers 操作变的更简单。
    用可视化的方式，来构建的 Docker 服务。

    **已经支持 Containers， Images 的基本操作； 其他功能还在开发中，不但只是单机操作。**

  - 创建者：[fundon][]，[DockerPool][] 成员

  - 功能：

      * 后端 - [dockerboard APIs][]

      * 前端 - [bluewhale Web][]
        * Dashboard
        * Containers
            - 列表
            - 删除
            - Start/Stop
            - Pause/Unpause
        * Images
            - 列表
            - 删除
        * Canvas

  - [dockerboard.wiki][] - 更多详情请看 wiki



## 笔者

- [@fundon](https://twitter/_fundon)
- [GitHub](https://github.com/fundon)
- cfddream#gmail.com


[dockerui]: https://github.com/crosbymichael/dockerui
[crosbymichael]: https://github.com/crosbymichael
[shipyard]: https://github.com/shipyard/shipyard
[ehazlett]: https://github.com/ehazlett
[Kitematic]: https://kitematic.com
[Kitematic Team]: https://github.com/kitematic
[@kitematic]: https://twitter.com/kitematic
[Atom-Shell]: https://github.com/atom/atom-shell
[boot2docker]: https://github.com/boot2docker
[meteor]: https://www.meteor.com
[Gaudi]: http://gaudi.io
[marmelab]: https://github.com/marmelab
[fig]: http://fig.sh
[juju]: http://juju.ubuntu.com/
[ubuntu]: http://juju.ubuntu.com/
[try juju]: https://demo.jujucharms.com
[seagull]: https://github.com/tobegit3hub/seagull
[tobegit3hub]: https://github.com/tobegit3hub
[try seagull]: http://96.126.127.93:10086
[dockerboard]: https://github.com/dockerboard
[dockerpool]: https://github.com/dockerpool
[dockerboard apis]: https://github.com/dockerboard/dockerboard
[dockerboard.wiki]: https://github.com/dockerboard/dockerboard/wiki
[bluewhale web]: https://github.com/dockerboard/bluewhale
[fundon]: https://github.com/fundon