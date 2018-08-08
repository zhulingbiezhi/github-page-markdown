---
title: "【转】强大的Kubernetes包管理工具: Helm"
date: 2018-08-06T11:18:15+08:00
weight: 170
keywords: ["kubernetes"]
description: "Kubernetes工具"
tags: ["kubernetes", "分布式"]
categories: ["转载"]
author: "去去"
---
【编者的话】Helm是Kubernetes生态系统中的一个软件包管理工具。本文将介绍为何要使用Helm进行Kubernetes软件包管理，澄清Helm中使用到的相关概念，并通过一个具体的示例学习如何使用Helm打包，分发，安装，升级及回退Kubernetes应用。  

### **Kubernetes应用部署的挑战**

让我们首先来看看Kubernetes，Kubernetes提供了基于容器的应用集群管理，为容器化应用提供了部署运行、资源调度、服务发现和动态伸缩等一系列完整功能。  
  
Kubernetes的核心设计理念是：用户定义应用程序的规格，而Kubernetes则负责按照定义的规则部署并运行应用程序，如果应用系统出现问题导致偏离了定义的规格，Kubernetes负责对其进行自动修正。例如应用规格要求部署两个实例，其中一个实例异常终止了，Kubernetes会检查到并重新启动一个新的实例。  
  
用户通过使用Kubernetes API对象来描述应用程序规格，包括Pod，Service，Volume，Namespace，ReplicaSet，Deployment，Job等等。一般这些对象需要写入一系列的Yaml文件中，然后通过Kubernetes命令行工具Kubectl进行部署。  
  
以下面的WordPress应用程序为例，涉及到多个Kubernetes API对象，这些Kubernetes API对象分散在多个Yaml文件中。  

[![1.png](http://dockone.io/uploads/article/20180501/b5903b6ce59d8916b610e22809084f4d.png "1.png")](http://dockone.io/uploads/article/20180501/b5903b6ce59d8916b610e22809084f4d.png)

  
_图1： WordPress应用程序中涉及到的Kubernetes API对象_  
  
可以看到，在进行Kubernetes软件部署时，我们面临下述问题：  

*   如何管理，编辑和更新这些这些分散的Kubernetes应用配置文件？
*   如何把一套的相关配置文件作为一个应用进行管理？
*   如何分发和重用Kubernetes的应用配置？

  
  
Helm的引入很好地解决上面这些问题。  

### **Helm是什么？**
	
很多人都使用过Ubuntu下的ap-get或者CentOS下的yum，这两者都是Linux系统下的包管理工具。采用apt-get/yum，应用开发者可以管理应用包之间的依赖关系，发布应用；用户则可以以简单的方式查找、安装、升级、卸载应用程序。  
  
我们可以将Helm看作Kubernetes下的apt-get/yum。Helm是[Deis](https://deis.com/)开发的一个用于Kubernetes的包管理器。  
  
对于应用发布者而言，可以通过Helm打包应用，管理应用依赖关系，管理应用版本并发布应用到软件仓库。  
  
对于使用者而言，使用Helm后不用需要了解Kubernetes的Yaml语法并编写应用部署文件，可以通过Helm下载并在Kubernetes上安装需要的应用。  
  
除此以外，Helm还提供了Kubernetes上的软件部署、删除、升级、回滚应用的强大功能。  

### **Helm组件及相关术语**

开始接触Helm时遇到的一个常见问题就是Helm中的一些概念和术语非常让人迷惑，我开始学习Helm就遇到这个问题。  
  
因此我们先了解一下Helm的这些相关概念和术语。  
  
Helm，Kubernetes的应用打包工具，也是命令行工具的名称。  
  
Tiller，Helm的服务端，部署在Kubernetes集群中，用于处理Helm的相关命令。  
  
Chart，Helm的打包格式，内部包含了一组相关的Kubernetes资源。  
  
Repoistory，Helm的软件仓库，Repoistory本质上是一个Web服务器，该服务器保存了Chart软件包以供下载，并有提供一个该Repoistory的Chart包的清单文件以供查询。在使用时，Helm可以对接多个不同的Repository。  
  
Release，使用Helm install命令在Kubernetes集群中安装的Chart称为Release。  
  

>   
> 需要特别注意的是， Helm中提到的Release和我们通常概念中的版本有所不同，这里的Release可以理解为Helm使用Chart包部署的一个应用实例。  
> 其实Helm中的Release叫做Deployment更合适。估计因为Deployment这个概念已经被Kubernetes使用了，因此Helm才采用了Release这个术语。

下面这张图描述了Helm的几个关键组件Helm（客户端）、Tiller（服务器）、Repository（Chart软件仓库）、Chart（软件包）之前的关系。  

[![2.png](http://dockone.io/uploads/article/20180501/d460269f6b1369472a51bdd48400343a.png "2.png")](http://dockone.io/uploads/article/20180501/d460269f6b1369472a51bdd48400343a.png)

  
_图2： Helm软件架构_  

### **安装Helm**

下面我们通过一个完整的示例来介绍Helm的相关概念，并学习如何使用Helm打包、分发、安装、升级及回退Kubernetes应用。  
  
可以参考Helm的帮助文档[https://docs.helm.sh/using_helm/#installing-helm](https://docs.helm.sh/using_helm/#installing-helm) 安装Helm。  
  
采用二进制的方式安装Helm：  

1.  下载 Helm [https://github.com/kubernetes/helm/releases](https://github.com/kubernetes/helm/releases)
2.  解压 tar -zxvf helm-v2.0.0-linux-amd64.tgz
3.  拷贝到bin目录 mv linux-amd64/helm /usr/local/bin/helm
  

### **构建一个Helm chart**

让我们在实践中来了解Helm。这里将使用一个Go测试小程序，让我们先为这个小程序创建一个Helm chart。  

```
git clone https://github.com/zhaohuabing/testapi.git; 
cd testapi  
```

  
首先创建一个Chart的骨架：  

`helm create testapi-chart`  

  
该命令创建一个testapi-chart目录，该目录结构如下所示，我们主要关注目录中的这三个文件即可：Chart.yaml、values.yaml和NOTES.txt。  
```
testapi-chart  
├── charts  
├── Chart.yaml  
├── templates  
│   ├── deployment.yaml  
│   ├── _helpers.tpl  
│   ├── NOTES.txt  
│   └── service.yaml  
└── values.yaml  
```
  

*   Chart.yaml用于描述这个Chart，包括名字、描述信息以及版本。
*   values.yaml 用于存储templates目录中模板文件中用到的变量。 模板文件一般是Go模板。如果你需要了解更多关于Go模板的相关信息，可以查看[Hugo](https://gohugo.io)的一个关于Go模板的[介绍](https://gohugo.io/templates/go-templates/)。
*   NOTES.txt用于向部署该Chart的用于介绍Chart部署后的一些信息。例如介绍如何使用这个Chart，列出缺省的设置等。

  
  
打开Chart.yaml，填写你部署的应用的详细信息，以testapi为例：  

```
apiVersion: v1  
description: A simple api for testing and debugging  
name: testapi-chart  
version: 0.0.1 
``` 

  
然后打开并根据需要编辑values.yaml。下面是testapi应用的values.yaml文件内容。  

```
replicaCount: 2  
image:  
repository: daemonza/testapi  
tag: latest  
pullPolicy: IfNotPresent  
service:  
name: testapi  
type: ClusterIP  
externalPort: 80  
internalPort: 80  
resources:  
limits:  
cpu: 100m  
memory: 128Mi  
requests:  
cpu: 100m  
memory: 128Mi  
```
  
在testapi_chart目录下运行下面命令以对Chart进行校验。  

```
helm lint  
==> Linting .  
\[INFO\] Chart.yaml: icon is recommended  
  
1 chart(s) linted, no failures  
```
  
如果文件格式错误，可以根据提示进行修改；如果一切正常，可以使用下面的命令对Chart进行打包：  

`helm package testapi-chart --debug  `

  
这里添加了–debug参数来查看打包的输出，输出应该类似于：  

```
Saved /Users/daemonza/testapi/testapi-chart/testapi-chart-0.0.1.tgz to current directory  
Saved /Users/daemonza/testapi/testapi-chart/testapi-chart-0.0.1.tgz to /Users/daemonza/.helm/repository/local  
```

  
Chart被打包为一个压缩包testapi-chart-0.0.1.tgz，该压缩包被放到了当前目录下，并同时被保存到了Helm的本地缺省仓库目录中。  

### **Helm Repository**

虽然我们已经打包了Chart并发布到了Helm的本地目录中，但通过Helm search命令查找，并不能找不到刚才生成的Chart包。 

```
helm search testapi  
No results found  
```
  
这是因为Repository目录中的Chart还没有被Helm管理。我们可以在本地启动一个Repository Server，并将其加入到Helm repo列表中。  
  
通过Helm repo list命令可以看到目前Helm中只配置了一个名为stable的repo，该repo指向了Google的一个服务器。  
```
helm repo list  
NAME    URL  
stable  https://kubernetes-charts.storage.googleapis.com  
```
  
使用Helm serve命令启动一个repo server，该server缺省使用’$HELM_HOME/repository/local’目录作为Chart存储，并在8879端口上提供服务。  

```
helm serve&  
Now serving you on 127.0.0.1:8879  
```

  
启动本地repo server后，将其加入Helm的repo列表。  
```
helm repo add local http://127.0.0.1:8879  
"local" has been added to your repositories  
```
  
现在再查找testapi chart包，就可以找到了。  
```
helm search testapi  
  
NAME                    CHART VERSION   APP VERSION     DESCRIPTION  
local/testapi-chart     0.0.1                           A Helm chart for Kubernetes  
```
  

### **在Kubernetes中部署Chart**

Chart被发布到仓储后，可以通过Helm instal命令部署Chart，部署时指定Chart名及Release（部署的实例）名：  

`helm install local/testapi-chart --name testapi ` 

  
该命令的输出应类似：  
```
NAME:   testapi  
LAST DEPLOYED: Mon Apr 16 10:21:44 2018  
NAMESPACE: default  
STATUS: DEPLOYED  
  
RESOURCES:  
==> v1/Service  
NAME                   TYPE       CLUSTER-IP    EXTERNAL-IP  PORT(S)  AGE  
testapi-testapi-chart  ClusterIP  10.43.121.84  <none>       80/TCP   0s  
  
==> v1beta1/Deployment  
NAME                   DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE  
testapi-testapi-chart  1        1        1           0          0s  
  
==> v1/Pod(related)  
NAME                                   READY  STATUS   RESTARTS  AGE  
testapi-testapi-chart-9897d9f8c-nn6wd  0/1    Pending  0         0s  
```  
 ``` 
NOTES:  
1. Get the application URL by running these commands:  
export POD_NAME=$(kubectl get pods --namespace default -l "app=testapi-testapi-chart" -o jsonpath="{.items\[0\].metadata.name}")  
echo "Visit http://127.0.0.1:8080 to use your application"  
kubectl port-forward $POD_NAME 8080:80  
```
  
使用下面的命令列出所有已部署的Release以及其对应的Chart。  

`helm ls  `

  
该命令的输出应类似:  
```
NAME    REVISION        UPDATED                         STATUS          CHART                   NAMESPACE  
testapi 1               Mon Apr 16 10:21:44 2018        DEPLOYED        testapi-chart-0.0.1     default  
```
  
可以看到在输出中有一个Revision（更改历史）字段，该字段用于表示某一Release被更新的次数，可以用该特性对已部署的Release进行回滚。  

### **升级和回退**

修改Chart.yaml，将版本号从0.0.1修改为1.0.0，然后使用Helm package命令打包并发布到本地仓库。  
  
查看本地库中的Chart信息，可以看到在本地仓库中testapi-chart有两个版本：  
```
helm search testapi -l  
NAME                    CHART VERSION   APP VERSION     DESCRIPTION  
local/testapi-chart     0.0.1                           A Helm chart for Kubernetes  
local/testapi-chart     1.0.0                           A Helm chart for Kubernetes  
```
  
现在用Helm upgrade将已部署的testapi升级到新版本。可以通过参数指定需要升级的版本号，如果没有指定版本号，则缺省使用最新版本。  

`helm upgrade testapi local/testapi-chart   `

  
已部署的testapi release被升级到1.0.0版本。  
```
helm list  
NAME    REVISION        UPDATED                         STATUS          CHART                   NAMESPACE  
testapi 2               Mon Apr 16 10:43:10 2018        DEPLOYED        testapi-chart-1.0.0     default  
```
  
可以通过Helm history查看一个Release的多次更改。  
```
helm history testapi  
REVISION        UPDATED                         STATUS          CHART                   DESCRIPTION  
1               Mon Apr 16 10:21:44 2018        SUPERSEDED      testapi-chart-0.0.1     Install complete  
2               Mon Apr 16 10:43:10 2018        DEPLOYED        testapi-chart-1.0.0     Upgrade complete  
```
  
如果更新后的程序由于某些原因运行有问题，我们则需要回退到旧版本的应用，可以采用下面的命令进行回退。其中的参数1是前面Helm history中查看到的Release的更改历史。  

`helm rollback testapi 1 ` 

  
使用Helm list命令查看，部署的testapi的版本已经回退到0.0.1  
```
helm list  
NAME    REVISION        UPDATED                         STATUS          CHART                   NAMESPACE  
testapi 3               Mon Apr 16 10:48:20 2018        DEPLOYED        testapi-chart-0.0.1     default  
```
  

### 总结

Helm作为Kubernetes应用的包管理以及部署工具，提供了应用打包、发布、版本管理以及部署、升级、回退等功能。Helm以Chart软件包的形式简化Kubernetes的应用管理，提高了对用户的友好性。  

### Q&A

**Q：Helm结合CD有什么好的建议吗？**  

>   
> A：采用Helm可以把零散的Kubernetes应用配置文件作为一个Chart管理，Chart源码可以和源代码一起放到Git库中管理。Helm还简了在CI/CD Pipeline的软件部署流程。通过把Chart参数化，可以在测试环境和生产环境可以采用不同的Chart参数配置。  
> 下图是采用了Helm的一个CI/CD流程：  
> 
> [![3.png](http://dockone.io/uploads/article/20180501/278fd00686afbf76fbfa69bf4e7c8307.png "3.png")](http://dockone.io/uploads/article/20180501/278fd00686afbf76fbfa69bf4e7c8307.png)

**Q：请问下多环境（test、staging、production）的业务配置如何管理呢？通过Heml打包ConfigMap吗，比如配置文件更新，也要重新打Chart包吗？谢谢，这块我比较乱。**  

>   
> A：Chart是支持参数替换的，可以把业务配置相关的参数设置为模板变量。使用Helm install Chart的时候可以指定一个参数值文件，这样就可以把业务参数从Chart中剥离了。例子：helm install –values=myvals.yaml wordpress。

**Q：Helm能解决服务依赖吗？**  

>   
> A：可以的，在Chart可以通过requirements.yaml声明对其他Chart的依赖关系。如下面声明表明Chart依赖Apache和MySQL这两个第三方Chart。  
> 
> dependencies:  
>   - name: apache  
>   version: 1.2.3  
>   repository: http://example.com/charts  
>   - name: mysql  
>   version: 3.2.1  
>   repository: http://another.example.com/charts  
>   

**Q：Chart的reversion可以自定义吗，比如跟Git的tag？**  

>   
> A：这位朋友应该是把Chart的version和Release的reversion搞混了，呵呵。 Chart是没有reversion的，Chart部署的一个实例（Release）才有Reversion，Reversion是Release被更新后自动生成的。

**Q：这个简单例子并没有看出Helm相比Kubectl有哪些优势，可以简要说一下吗？**  

>   
> A：Helm将Kubernetes应用作为一个软件包整体管理，例如一个应用可能有前端服务器，后端服务器，数据库，这样会涉及多个Kubernetes部署配置文件，Helm就整体管理了。另外Helm还提供了软件包版本，一键安装、升级、回退。Kubectl和Helm就好比你手工下载安装一个应用 和使用apt-get安装一个应用的区别。

**Q：如何在Helm install时指定命名空间？**  

>   
> A：helm install local/testapi-chart –name testapi –namespace mynamespace。

以上内容根据2018年4月24日晚微信群分享内容整理。 分享人**赵化冰，开源软件爱好者，目前在NFV＆SDN编排开源社区ONAP担任MSB项目负责人，致力于高性能，高可用微服务架构在编排领域的应用。**DockOne每周都会组织定向的技术分享，欢迎感兴趣的同学加微信：liyingjiesa，进群参与，您有想听的话题或者想分享的话题都可以给我们留言。
