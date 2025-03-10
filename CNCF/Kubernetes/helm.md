# Helm

## helm是什么

helm是kubernetes生态系统中的一个软件包管理工具，类似ubuntu的apt,centos的yum或python的pip一样，专门负责管理kubernetes应用资源；使用helm可以对kubernetes应用进行统一打包、分发、安装、升级以及回退等操作。

helm利用Chart来封装kubernetes原生应用程序的一些列yaml文件，可以在部署应用的时候自定义应用程序的一些Metadata，以便于应用程序的分发。

## helm为什么出现

利用Kubernetes部署一个应用，需要Kubernetes原生资源文件如deployment、replicationcontroller、service或pod 等。这些 k8s 资源过于分散，不方便进行管理，直接通过 kubectl 来管理一个应用，你会发现这十分蛋疼。

而对于一个复杂的应用，会有很多类似上面的资源描述文件，如果有更新或回滚应用的需求，可能要修改和维护所涉及的大量资源文件，且由于缺少对发布过的应用版本管理和控制，使Kubernetes上的应用维护和更新等面临诸多的挑战，而Helm可以帮我们解决这些问题:

* 如何统一管理、配置和更新这些分散的 k8s 的应用资源文件
* 如何分发和复用一套应用模板
* 如何将应用的一系列资源当做一个软件包管理

![helm架构](https://raw.githubusercontent.com/TDAkory/ImageResources/master/img/20220418104653.png)

## helm组件及相关基本概念

* helm有两个组件：helm客户端和Tiller服务端

    `Helm client`：是一个命令行客户端工具，主要用于k8s应用程序chart的创建、打包、发布以及创建和管理本地和远程的chart仓库。

    `Tiller service`：是Helm的一个服务端，部署在k8s集群中；Tiller用于接收Helm client的请求，并根据chart生成k8s的部署文件（称之为Release）,然后提交给k8s创建应用。同样Tiller还提供了Release的升级、删除、回滚等一些系列功能。
* `Chart`
Helm的软件包，采用TAR格式，类似apt的deb包或者yum的rpm包，chart包含了一组定义kubernetes资源相关的yaml文件。
* `Release`
采用helm install命令生成在kubernetes集群中部署的chart就叫Release。
* `Repository`
Helm的软件仓库，Repository本质上是一个web服务器，该服务器保存了一系列的Chart软件包以供用户下载，并且提供了一个该Repository的Chart包的清单文件以供查询；Helm可以同时管理多个不同的Repository。

## helm的工作原理

这张图描述了 Helm 的几个关键组件 Helm（客户端）、Tiller（服务器）、Repository（Chart 软件仓库）、Chart（软件包）之间的关系。

![helm的工作原理](https://raw.githubusercontent.com/TDAkory/ImageResources/master/img/20220418105121.png)

* Chart Install过程
  * Helm从指定的目录或者TAR文件中解析出chart结构信息.
  * Helm将指定的Chart结构和Values信息通过gRPC传递给Tiller.
  * Tiller根据Chart和Values生成一个Release.
  * Tiller将Release发送给kubernetes用于创建Release.
  
* Chart Update过程
  * Helm从指定的目录或者TAR文件中解析出chart结构信息.
  * Helm将需要更新的Release名称、Chart结构和Values信息通过gRPC传递给Tiller.
  * Tiller生成Release并更新指定名称的Release的History
  * Tiller将Release发送给kubernetes用于更新Release.
* Chart Rollback过程
  * Helm将需要回滚的Release名称传递给Tiller.
  * Tiller根据Release的名称查找History.
  * Tiller从history中获取上一个Release.
  * Tiller将上一个Release发送给Kubernetes用于替换当前的Release.

Helm客户端通过gRPC与Tiller服务端通信，Tiller服务端通过HTTP REST请求与kubernetes进行通信。