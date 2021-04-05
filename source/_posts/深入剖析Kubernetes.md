---
title: GeekTime -《深入剖析Kubernetes》笔记
date: 2021-03-17 21:04:39
tags:
- 笔记
- 极客时间
- 容器技术
- k8s集群
categories:
- 极客时间
---

# 深入剖析kubernetes

### 05 白话容器基础（一）：从进程说开去

- 进程：程序运行起来后的计算机执行环境的总和
- 容器技术的核心功能，就是通过约束和修改进程的动态表现，从而为其创造出一个”边界“
- 对于Docker等大多数Linux容器来说，两大基础技术：
  - Cgroups技术，用来制造约束的主要手段
  - Namespace技术，用来修改进程视图的主要方法。对被隔离的应用的进程空间做手脚，使得这些进程只能看到重新计算过的进程编号
- 除了 PID Namespace，Linux操作系统还提供了Mount，UTS，IPC，Network和User这些Namespace，用来对各种不同的进程上下文进行”障眼法“操作，这就是Linux容器最基本的实现原理
- 所以说，容器，其实是一种特殊的进程而已

### 06 白话容器基础（二）：隔离与限制

- Namespace技术实际上修改了应用进程看待整个计算机”视图“，即它的”视线“被操作系统做了限制，只能”看到“某些指定的内容，但对于宿主机来说，这些被”隔离“了的进程跟其他进程并没有太大区别
- Linux Cgroups就是Linux内核中用来为进程设置资源限制的一个主要功能
- Linux Cgroups，Linux Control Group，最主要的作用，就是限制一个进程组能够使用的资源上限，包括CPU，内存，磁盘，网络带宽等等
- 容器技术中非常重要的概念，即：容器是一个”单进程“模型
- Namespace和Cgroups都有很多不完善的地方

### 07 白话容器基础（三）：深入理解容器镜像

- Mount Namespace修改的，是容器进程对文件“挂载点”的认知
- Mount Namespace跟其他Namespace的使用略有不同的地方：它对容器进程视图的改变，一定是伴随着挂载操作（mount）才能生效
- chroot，change root file system，改变进程的根目录到指定的位置
- Mount Namespace正是基于对chroot的不断改良才被发明出来的，它也是Linux操作系统里的第一个Namespace
- 挂载在容器根目录上，用来为容器进程提供隔离后执行环境的文件系统，就是所谓的“容器镜像”，它还有一个更为专业的名字，叫做，rootfs（根文件系统）
- rootfs，只是一个操作系统所包含的文件，配置和目录，并不包括操作系统内核，在Linux操作系统中，这两部分是分开存放的，操作系统只有在开机启动时才会加载指定版本的内核镜像
- 正是由于rootfs的存在，容器才有了一个被反复宣传至今的重要特性：一致性
- 由于rootfs里打包的不只是应用，而是整个操作系统的文件和目录，也就意味着，应用以及它运行需要的所有依赖，都被封装在了一起
- 对一个应用来说，操作系统本身才是它运行所需要的最完整的”依赖库“
- 容器的rootfs由三部分组成：
  - 第一部分，只读层。这些层，都以增量的方式分别包含了操作系统的一部分
  - 第二部分，可读写层。
  - 第三部分，Init层。Docker项目单独生成的一个内部层，专门用来存放hosts，resolv.conf等信息

### 08 白话容器基础（四）：重新认识Docker容器

- 应用容器化的第一步，是制作容器镜像
- Dockerfile中的每个原语执行后，都会生成一个对应的镜像层
- Volumn机制，允许将宿主主机上指定的目录或者文件挂载到容器里面进行读取和修改操作

### 09 从容器到容器云：谈谈kubernetes的本质

- 最具代表性的容器编排工具
  - docker compose + Swarm
  - kubernetes
- Kubernetes是依托google的Borg项目的理论优势，由master和node两种节点组成，分别对应着控制节点和计算节点
- 控制节点，即Master节点，由三个独立组件组合而成，负责API服务的kube-apiserver，负责调度的kube-scheduler，负责容器编排的kube-controller-manager，整个集群的持久化数据，则由kube-apiserver处理后保存在Ectd中
- 计算节点上最核心的部分，叫做kubelet的组件。主要负责同容器运行时打交道
- kubelet还通过gRPC协议同一个叫做Device Plugin的插件进行交互，用来隔离GPU等宿主机物理设备的主要组件
- kubelet的另一个重要功能，调用网络插件和存储插件为容器配置网络和持久化存储
- kubernetes项目最主要的设计思想是，从更宏观的角度，以统一的方式来定义任务之间的各种关系，并且为将来支持更多种类的关系留有余地
- 几个在kubernetes中的对象
  - Pod
  - Job，用来描述一次性运行的pod
  - DaemonSet，用来描述每个宿主机上必须且只能运行一个副本的守护进程服务
  - CronJob，用于描述定时任务
  - Deployment，pod的多实例管理器
  - Service，主要作用，就是作为Pod的代理入口，从而代替Pod对外暴露一个固定的网络地址
  - Secret，保存在Etcd中的键值对数据，用于pod间的访问鉴权
- 在kubernetes项目中，推崇的使用方法是：
  - 通过“编排对象”，Pod，Job，CronJob等，来描述你试图管理的应用
  - 再为它定义一些“服务对象”，比如Service，Secret等，这些对象，会负责具体的平台级功能

### 10 kubernetes一键部署利器：kubeadm

- kubeadm的工作原理
  - kubeadm的妥协方案，把kubelet直接运行在宿主机上，然后使用容器部署其他的kubernetes组件
- kubeadm init的工作流程
  - 首先要做的是，一系列的检查工作，以确定这台机器可以用来部署kubernetes，称为“Preflight Checks”
  - 然后，生成kubernetes对外提供服务所需的各种证书和对应的目录
  - 证书生成后，kubeadm接下来会为其他组件生成访问kube-apiserver所需的配置文件
  - 接下来，kubeadm会为Master组件生成Pod配置文件
  - 然后，kubeadm就会为集群生成一个bootstrap token
  - token生成之后，kubeadm会将ca.crt等Master节点的重要信息，通过ConfigMap的方式保存在Etcd中，供后续部署Node节点使用
  - 最后一步，安装默认插件

### 12 第一个容器化应用

- 一个YAML文件，在kubernetes中，就是一个API Objetc，API对象
- 每一个API对象，都有一个叫做Metadata的字段，就是API对象的“标识”，即元数据，是我们从kubernetes中找到这个对象的主要依据
- 一个API对象，大多可以分为Metadata和Spec两个部分，前者是这个对象的元数据，后者存放的是属于这个对象独有的定义，用来描述它所要表达的功能
- 一些常用的命令
  - kubectl apply
  - kubectl exec
  - kubectl delete
  - kubectl get
  - kubectl create -f 我的配置文件

### 13 为什么我们需要Pod？

- Pod，是kubernetes项目的原子调度单位，即最小的编排单位
- Docker容器的本质：Namespace做隔离，Cgroups做限制，rootfs做文件系统
- Pod，最重要的一个事实是，它只是一个逻辑概念
- Pod里的所有容器，共享的是同一个Network Namespace，并且可以声明共享同一个Volume
- Pod的实现需要使用一个中间容器，这个容器叫做Infra容器，永远是第一个被创建的容器

### 14 深入解析Pod对象（一）：基本概念

- Pod和Container的关系：只是Pod属性里的一个普通的字段

- 凡是调度，网络，存储，以及安全相关的属性，基本上是Pod级别的

- Pod中几个重要的字段

  - NodeSelector，一个供用户将Pod与Node进行绑定的字段
  - NodeName，这个字段被赋值会被认为Pod已经经过了调度
  - HostAliases：定义了Pod的hosts文件里的内容，如：/etc/hosts
  - 凡是跟容器的Linux Namespace相关的属性，也一定是Pod级别的
  - 凡是Pod中的容器要共享宿主机的Namespace，也一定是Pod级别的定义

- Container中的重要字段

  - ImagePullPolicy字段，定义了镜像拉取的策略
  - Lifecycle字段，定义的是Container Lifecycle Hooks，例如postStart和preStop参数
  - Staus字段，包括以下几种可能的状态：
    - Pending
    - Running
    - Succeded
    - Failed
    - Unknown
  - Status还能细分出一组conditions，包括：
    - PodScheduled，Ready，Initialized，Unschedulable

  ### 15 深入解析Pod对象（二）：使用进阶

  - 特殊的Volumn，叫做Projected Volumn，可以翻译为，“投射数据卷”
  - kubernetes支持的Projected Volumn
    - Secret，典型场景，存放数据库的Credential信息
    - ConfigMap，用法与Secret几乎相同，保存的是不需要加密并且为应用所需的配置信息
    - Downward API，作用是让Pod里的容器能够直接获取到这个Pod API对象本身的信息，注意的是，获取的信息一定是Pod里的容器进程启动之前就能够确定下来的信息
    - ServiceAccountToken，只是一种特殊的Secret Volumn
  - PodPreset，即Pod预设置，可以用于对Pod批量化，自动化修改；其定义的内容只会在Pod API对象被创建之前追加在这个对象本身上，而不会影响任何Pod的控制器的定义

  ### 16 编排其实很简单：谈谈“控制器”模型

  - 控制器模式，control pattern
  - Pod这个看似复杂的API对象，实际上就是对容器的进一步抽象和封装而已
  - kubernetes的一个通用编排模式，控制循环，control loop，也叫调谐，reconcile，调谐的过程被称作调谐循环reconcile loop或同步循环sync loop
  - Deployment控制器，实际上都是由上半部分的控制器定义（包括期望状态），加上下半部分的被控制对象的模板组成的

  ### 17 经典PaaS的记忆：作业副本与水平扩展

  - Kubernetes中第一个重要的设计思想：控制器模式
  - Kubernetes中第一个控制器模式的完整实现：Deployment
  - 两个编排动作
    - 水平扩展/收缩
    - 滚动更新
      - 四个状态字段
        - DESIRED，用户期望的Pod副本个数
        - CURRENT，当前处于running状态的Pod的个数
        - UP-TO-DATE，当前处于最新版本的Pod的个数
        - AVAILABLE，当前已经可用的Pod的个数，既是running又是最新版本，处于Ready状态的Pod个数
      - 只有AVAILABLE字段描述的才是用户所期望的最终状态
  - Deployment只是在ReplicaSet的基础上，添加了UP-TO-DATE这个跟版本相关的状态字段
  - Deployment实际上是一个两层控制器，通过ReplicaSet的个数来描述应用的版本，再通过ReplicaSet的属性来保证Pod的副本数量
  - Deployment控制ReplicaSet（应用版本），ReplicaSet控制Pod（副本数）

### 18 StatefulSet（一）：拓扑状态

- 实例之间由不对等关系，以及实例对外部数据有依赖关系的应用，被称为“有状态应用”，Stateful Application

- StatefulSet将真实世界中的应用状态，抽象为了两种情况：

  - 拓扑状态，即不同节点间存在前后依赖关系
  - 存储状态，即应用的不同实例分别绑定了不同的存储数据

- StetfulSet的核心功能，旧是通过某种方式记录这些状态，然后在Pod被重新创建时，能够为新Pod恢复这些状态

- Headless Service不需要分配一个VIP，而是可以直接以DNS记录的方式解析出被代理Pod的IP地址

- Kubernetes项目为Pod分配的唯一的“可解析身份“，Resolvable Identity，如下所示DNS记录：

  ```bash
  <pod-name>.<svc-name>.<namespace>.svc.cluster.local
  ```

- StatefulSet将Pod的拓扑状态，按照Pod的名字+编号的方式固定下来