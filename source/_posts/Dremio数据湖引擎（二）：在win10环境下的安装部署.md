---
title: Dremio数据湖引擎（二）：在win10环境下的安装使用
date: 2019-09-28 00:51:39
tags: 
- Dremio
categories:
- Dremio
id: 29c5c69a-f6a1-4b89-9b43-81c9b11d56b0
---

&emsp;&emsp;由于博主日常使用的OS为windows10，故本文将简单展示如何在win10基于Docker容器安装部署Dremio。另外，Dremio的官网也给出诸如AWS版本、Azure版本等的安装部署包，有兴趣的话可通过以下链接前往了解：[dremio deploy](https://www.dremio.com/deploy/)



- 环境准备
  - win10环境下的Docker容器服务

- 拉取dremio-oss的docker镜像

  这里拉取下来的社区版本的Dremio镜像，商业版本的Dremio需要联系Dremio官方了。当然，作为个人开发使用，社区版本的Dremio已完全够用了。

  ```shell
  docker pull dremio/dremio-oss
  ```

- 单节点Dremio服务的部署

  ```shell
  docker run -p 9047:9047 -p 31010:31010 -p 45678:45678 dremio/dremio-oss
  ```

  利用以上命令可在Docker上启用一个单节点的Dremio服务，节点还包括以下服务：

  - Embedded Zookeeper
  - Master Coordinator
  - Executor

- 注册账号

  Dremio服务启动成功后，在浏览器访问本地9047端口，http://localhost:9047/。

  将出现以下账号注册界面，如下图：

  ![这是代替图片的文字，随便写](register.png)

  填写完信息并注册成功后会自动跳转到Dremio的管理面板首页，如下图：

  ![这是代替图片的文字，随便写](index.png)

- 创建一个内置简单数据源

  点击左下角的`Add Data Lake`，，可以看到Dremio已支持的基于表的存储源以及基于文件的存储源，这里我们先直接新增一个内置的数据源即可

  ![这是代替图片的文字，随便写](sample.png)

  点击文件路径一直到如下图的界面，可以看到这个数据源已经内置了csv、json、parquet等文件格式的多个数据文件或目录（对应基于表存储系统中的表）

  ![这是代替图片的文字，随便写](sample2.png)











