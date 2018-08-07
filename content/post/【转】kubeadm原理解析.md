---
title: "【转】Kubeadm原理解析"
date: 2018-08-07T11:19:03+08:00
weight: 170
keywords: ["kubernetes"]
description: "Kubeadm原理解析"
tags: ["kubernetes", "分布式"]
categories: ["kubernetes"]
author: "去去"
---  
【编者的话】根据所处环境的不同，Kubernetes集群的安装部署有多种不同的方式。如果是在公有云上还好，一般都会提供相应的安装方案。如果是自行搭建私有环境，可选的安装方案则不尽相同，有时可能需要从零一步步开始，这个过程很容易出错，这往往使人感觉耗时耗力，望而却步。kubeadm作为Kubernetes官方提供的集群部署管理工具，采用“一键式”指令进行集群的快速初始化和安装，极大地简化了部署过程，消除了集群安装的痛点。虽然kubeadm目前还不能用于生产，但社区一直在完善kubeadm的功能和稳定性，刚刚发布的1.10版本中便包含了一系列优化补丁，预计在下一个版本可进入GA，能够用于Kubernetes生产环境的安装部署。本次分享针对kubeadm的基本原理和一些特性进行讲解，和大家一起体会这个安装部署的“最佳实践”设计思想。讲解内容有不足之处，欢迎大家批评指正，共同探讨，谢谢大家！  

### 1\. 基本原理

kubeadm做为集群安装的“最佳实践”工具，目标是通过必要的步骤来提供一个最小可用的集群运行环境。它会启动集群的基本组件以及必要的附属组件，至于为集群提供更丰富功能（比如监控，度量）的组件，不在其安装部署的范围。在环境节点符合其基本要求的前提下，kubeadm只需要两条基本命令便可以快捷的将一套集群部署起来。这两条命令分别是：  

*   kubeadm init：初始化集群并启动master相关组件，在计划用做master的节点上执行。
*   kubeadm join：将节点加入上述集群，在计划用做node的节点上执行。

  
  
下面分别对这两条命令的流程进行详细的说明。在此之前需要特别说明的是，kubeadm并不负责kubectl和kubelet的安装，因此需要事先将kubectl和kubelet安装好。kubeadm、kubectl和kubelet的安装方式请参考：[https://kubernetes.io/docs/set ... bectl](https://kubernetes.io/docs/setup/independent/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)  

#### 1.1 kubeadm init

如果不需要对集群做任何定制化的配置，直接在master节点上执行该命令即可。该命令的内部流程如下图所示：  

[![1.png](http://dockone.io/uploads/article/20180328/d6705a59d2097d54e388d1177438f509.png "1.png")](http://dockone.io/uploads/article/20180328/d6705a59d2097d54e388d1177438f509.png)

  
在上图的第④步完成的时候，正常情况下，Kubernetes的master的各组件便启动起来了。这里会有一个雷区：由于kubeadm是以static pod的方式启动master各组件，这就涉及到组件容器镜像的拉取。kubeadm默认的镜像源是从Google的gcr.io镜像库，因此需要科学合理地去解决如何访问该镜像库的问题，或者为kubeadm指定可用的第三方镜像库（据个人所知，如下镜像库可用，比如要拉取apiserver的1.9.6版本镜像，地址为：registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64:v1.9.6）。由于该雷区的存在，可能会使得kubeadm的便捷性大打折扣。  
  
后面的步骤主要是进行master节点的属性设置、为node节点的加入做准备、启动附属组件等。需要注意的是，网络插件的安装也不在kubeadm管理的范围以内，需要根据自己对网络方案的选择来另行安装相应的CNI插件组件。  
  
**定制启动配置参数**  
  
如果在启动集群的时候希望定制kubeadm本身的参数（比如前文所提到的指定镜像库地址），或者定制master各组件的启动参数，可以通过给`kubeadm init`命令传入`--config`来指定本地的配置文件，在该文件中对期望的参数进行设置。这种方式为kubeadm部署集群提供了很大的配置灵活性，但它目前还是一个实验性的功能，后续可能会发生变化。  
  
另外，kubeadm还提供了phase命令，将init过程划分为模块化的各个阶段。如果想对部署过程进行细粒度的控制和干预，达到定制化的目的，则可以考虑使用该命令。phase命令组合起来与init的工作流是一致的，不过目前它也是一个alpha阶段的功能。  

#### 1.2 kubeadm join

join的流程就比较简单了，它主要用来启动节点并加入集群。该命令的内部流程如下图所示：  

[![2.png](http://dockone.io/uploads/article/20180328/a0d1e65f00f7ddab0e082ac41dc763fd.png "2.png")](http://dockone.io/uploads/article/20180328/a0d1e65f00f7ddab0e082ac41dc763fd.png)

  
在node节点加入集群时，需要建立双向信任，它分成两部分：discovery（node信任master）和TLS bootstrap（master信任node）。  
  
**discovery**  
  
node连接master时使用discovery token和CA public key hash，用来获取集群CA证书并完成对master的信任过程，确认所连接的master是所期望的合法master。  
  
除了token方式之外，还可以通过discovery file的方式来完成discovery过程。该文件是一个kubeconfig配置文件，其中只包含了apiserver的地址信息以及集群CA证书数据。  
  
**TLS bootstrap**  
  
node连接master时携带TLS bootstrap token（通常可以和discovery token使用同一个），用于kubelet向master发起certificate signing request（CSR）时的临时认证，kubeadm默认设置了master自动批准该CSR。当该CSR得到批准后，master便会为kubelet签发客户端证书，用于后续kubelet访问apiserver时的身份认证。  
  
在kubeadm init过程完成后，会打印出包含上述token参数的join命令，形式如下：  

kubeadm join --token 33737d.a1aade1da540d7a0 192.168.122.39:6443 --discovery-token-ca-cert-hash sha256:ba72cd9566cb3703337f6c443564a1c917640b86dbd8de2b9c847814af3f737c  

  
如果后面找不到该命令了，可以通过如下命令重新生成相应的命令：  

kubeadm token create --print-join-command  

  
其中token为discovery和TLS bootstrap共用的token，192.168.122.39:6443 为apiserver的IP地址和安全端口，discovery-token-ca-cert-hash为CA证书的HASH，用于校验master的证书。  
  
**定制启动配置参数**  
  
与`kubeadm init`命令类似，也可以通过传入`--config`来指定本地配置文件的方式给`kubeadm join`命令定制启动参数，不再赘述。需要特别说明的是，该配置文件中并不包含kubelet组件的配置，如果要修改kubelet的参数，需要修改kubelet对应的系统服务文件，位置一般在`/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`。  

### 2\. 让安全配置更简便

Kubernetes提供多种安全认证机制，为了对系统各组件间的通信进行加密，一般采用TLS进行双向认证，从前面的解析也已经可以零星的看到TLS和证书的影子。要部署一个TLS安全的Kubernetes集群，就需要生成并组织好CA、密钥、证书等一系列文件，并对Kubernetes组件的相关参数进行正确设置，而这些文件和参数数量繁多，稍不注意便容易配置错误，导致集群无法正常启动。  
  
kubeadm则把这些细节全部都隐藏起来了，它直接在代码流程中实现相关证书和密钥的生成过程，并为各组件设置对应的安全参数。这种方式使用户不用进行安全相关的任何配置，在无需感知的情况下便可以启动一个通信安全的集群，从而大大简化了集群安全配置，将此处的痛点降低为零。  
  
内置的证书生成流程如下图所示：  

[![3.png](http://dockone.io/uploads/article/20180328/312055eea5c125ced3904692676160d8.png "3.png")](http://dockone.io/uploads/article/20180328/312055eea5c125ced3904692676160d8.png)

  
注：对于上图中的front proxy，没有在官方文档中找到明确的说明，我的理解应该指的就是kubectl proxy。用户可以通过这个proxy来间接的访问apiserver，因此与它相关的通信也是需要进行加密认证的。  
  
kubeadm安装完集群后，生成的部分证书和密钥文件如下图所示。图中未显示etcd相关的证书，为etcd生成证书的功能是在1.10版本实现的。  

[![4.png](http://dockone.io/uploads/article/20180328/846ad76ed383949bee4337218b30bc9f.png "4.png")](http://dockone.io/uploads/article/20180328/846ad76ed383949bee4337218b30bc9f.png)

  

### 3\. self-hosting模式

kubeadm从1.8开始提供了一个实验性的功能，即支持self-hosting模式。这种模式简单来说就是以Kubernetes原生对象的方式来部署Kubernetes的master组件。（static pod模式下，虽然这些组件也是以Pod的形式存在的，但它们是由kubelet来管理的，在apiserver看到的只是它们的mirror pod，并不能进行实际管理，因此可以认为它们不是真正意义上的Kubernetes原生对象。）  
  
使用self-hosting模式的优点在于利用了Kubernetes原生对象的特性，使集群master组件的管理、升级、回滚、高可用配置等等更加方便。注意，目前self-hosting模式的实现也有其显著的缺点：当master节点重启后，这些组件无法自行启动。这个缺点以及其他可能的限制会在以后的版本中逐步得到解决。  
  
在self-hosting模式下，master组件是以DaemonSet对象来管理的。self-hosting模式的启动过程如下图所示（该图摘自社区@luxas的讲稿）：  

[![5.png](http://dockone.io/uploads/article/20180328/608287a4f392d884fbee1da221774e72.png "5.png")](http://dockone.io/uploads/article/20180328/608287a4f392d884fbee1da221774e72.png)

  
该流程描述如下：  

1.  通过feature gate启用该模式；
2.  读取master组件在本地对应的static pod manifests（即Pod配置文件）；
3.  根据上述文件生成对应的DaemonSet配置文件；
4.  创建DaemonSet资源对象，等待Pod启动；（在这里当apiserver Pod启动后会尝试绑定到6443端口，由于static pod的apiserver已经绑定对应的端口，因此DaemonSet的Pod会一直处于Crash Loopbackoff状态，直至后面的步骤完成）
5.  删除原来的静态Pod配置文件，kubelet随之删除对应的静态Pod。
6.  DaemonSet形式的master组件接管集群。

  
  
在self-hosting模式下，查看DaemonSet可以看到如下self-hosted的master组件：  

[![6.png](http://dockone.io/uploads/article/20180328/9949a4a68d783b35e786691990e927b9.png "6.png")](http://dockone.io/uploads/article/20180328/9949a4a68d783b35e786691990e927b9.png)

  

### 4\. 如何支持HA

实际上，kubeadm本身还并没有真正的原生支持HA，社区给出的借助kubeadm实现HA的方案参见：[https://kubernetes.io/docs/set ... lity/](https://kubernetes.io/docs/setup/independent/high-availability/)。  
  
其思路如下图所示（该图摘自社区@luxas的讲稿）：  

[![7.png](http://dockone.io/uploads/article/20180328/6250dbe4faaca9f7ccc87160dc8cfc2c.png "7.png")](http://dockone.io/uploads/article/20180328/6250dbe4faaca9f7ccc87160dc8cfc2c.png)

  
主要步骤：  

1.  首先建立一个HA的etcd集群；
2.  在首个master节点上使用`kubeadm init`启动；
3.  将首master节点上的证书等文件拷贝至其他master节点；
4.  在其他master节点使用`kubeadm init`启动；
5.  为master节点建立LB；
6.  将node节点加入集群。

  
  
我没有实践该HA方案，不太确定过程中是否有坑以及集群安装后的具体状态，因此这里就不进一步展开了。  
  
至于kubeadm原生支持HA的设计方案参见：[https://github.com/kubernetes/community/pull/1707](https://github.com/kubernetes/community/pull/1707) ，该方案还在review阶段，有兴趣的朋友可以阅读一下。  

### 5\. 集群升级

集群升级也往往是让人头疼的问题，处理起来会比较复杂。类似集群的安装过程，kubeadm也同样的把升级的细节隐藏起来，只需要使用一两条简单的命令便可轻松完成集群升级。  
  
首先可以通过`kubeadm upgrade plan`命令查看升级计划，该命令会输出可升级的最新目标版本。  

[![8.png](http://dockone.io/uploads/article/20180328/2723b7d6d2c4f88be6fc91500e34cb41.png "8.png")](http://dockone.io/uploads/article/20180328/2723b7d6d2c4f88be6fc91500e34cb41.png)

  
在上图中可以看到可升级的1.9系列最新版本为1.9.6。需要注意的有两点：1. 要执行升级，需要先将kubeadm升级到对应的版本；2. kubeadm并不负责kubelet的升级，需要在升级完master组件后，手工对kubelet进行升级。  

[![9.png](http://dockone.io/uploads/article/20180328/68d9e704d4c40430d94109c3ad4f262f.png "9.png")](http://dockone.io/uploads/article/20180328/68d9e704d4c40430d94109c3ad4f262f.png)

  
在上图中可以看到可升级的最新稳定版本为1.10.0。注意点同上。  
  
根据上面的结果以及自己的升级需求，选择相应的目标版本，执行`kubeadm upgrade apply [version]`即可启动升级过程。由于暂时未在个人环境上进行升级，这里就不贴出升级执行过程及结果了，大家可以在自己的环境上进行实验。  
  
补充：  
  
大家如果想实践kubeadm的功能但又受困于网络等问题的话，可以访问[Play with Kubernetes](https://labs.play-with-k8s.com/)来进行体验，个人感觉非常好用。目前它的kubeadm版本是1.8.4，默认部署的Kubernetes版本是1.8.10。由于该环境是基于Docker-in-Docker来实现的，因此在其节点主机上的体验和实际的主机会有所不同。该实验环境是社区第三方提供的，因此版本也会有所滞后。  

### Q&A

**Q：etcd的升级是怎么处理？**  

>   
> A：etcd也可以由kubeadm进行管理，这种情况下，etcd的升级也就可以由kubeadm来进行了。可以参考分享中kubeadm upgrade plan中的提示。如果kubeadm配置为使用外部etcd，则需要自行进行etcd的升级。

**Q：你们使用的网络插件是？**  

>   
> A：我们使用的是中兴通讯自研的网络插件Knitter，该插件可以原生支持多网络平面，适合NFV的场景。

**Q：kubeadm可以指定仓库地址，不然在国内就得先注入镜像，你可以试试。**  

>   
> A：对的，在分享中关于雷区的说明中有提到可以为kubeadm指定可用的第三方镜像库，需要在启动参数中进行设置。

**Q：Dashboard、网络组件、日志这些组件是不是都需要自己再安装？这些安装过程有没有介绍文档？有没有办法通过kubeadm导出每个版本的镜像名字，这样可以通过GitHub和Docker Hub关联导出镜像。**  

>   
> A：对于您提到的其他组件，都是需要自己再安装的，直接按各个组件官方的安装文档进行安装即可。kubeadm使用的master组件镜像名字一般为类似这样的名称：gcr.io/google_containers/kube-apiserver-amd64:v1.9.6。

**Q：一定要使用etcd吗，可以用其他的数据库代替吗？**  

>   
> A：目前Kubernetes只支持使用etcd，在大约两年前，社区有关于支持Consul的讨论，不过后来不了了之。

**Q：etcd默认有做故障恢复吗，如果没有，有没有好的方案？**  

>   
> A：由kubeadm启动的etcd应该是没有做故障恢复，个人理解可以通过建立外部etcd高可用集群的方式达到目的。

**Q：knitter能描述一下吗，跟Calico、Flannel有什么区别呢？你们有测试过多高的负载？**  

>   
> A：knitter已经开源，可以在其项目地址查看其说明：[https://github.com/ZTE/Knitter](https://github.com/ZTE/Knitter)。不过由于刚刚开源不久，上面的文档还不是特别完善，敬请保持关注。

**Q：CA证书过期怎么处理？**  

>   
> A：kubeadm创建的CA证书默认的有效期是10年，一般情况下不需要考虑CA证书过期问题。不过apiserver的证书有效期默认的是1年，这个需要考虑过期问题。我在1.9中提过一个PR，在升级集群的时候，如果发现apiserver的证书到期时间小于180天，会重新生成其证书。

**Q：我看过kubelet默认有主动注册的选项，如果提供证书密钥应该就不需要使用kubeadm join？**  

>   
> A：是的，不过这个前提就是像你所提到的，需要它已经有相应的证书密钥，如果不使用kubeadm join的话，就需要手动去为它创建证书和密钥，可能还需要一些其他配置，过程比较繁琐。所以如果集群是由kubeadm来管理的话，还是建议使用kubeadm join来加入。

**Q：kubeadm join使用的token是不是有时效的？**  

>   
> A：对，我记得这个时效应该是24小时，如果超过时效，可以使用kubeadm token create来重新生成一个token。

**Q：kubectl get daemonset -o wide --all-namespaces可以查到kube-apiserver-master的是self-hosting模式吗？**  

>   
> A：如果通过该命令可以查到，那应该就是了。kubeadm在这些DaemonSet的名字中统一加了“self-hosted-”前缀。

以上内容根据2018年3月27日晚微信群分享内容整理。 分享人**赵相鹏，中兴通讯高级工程师。2015年开始关注容器技术，目前专注于云计算开源生态圈的技术预研和社区参与，活跃于Kubernetes社区多个领域，对Kubernetes的架构原理及社区运作有较深的研究和理解。现为Kubernetes社区member，SIG-Cluster-Lifecycle组的核心成员，文档库维护者**。DockOne每周都会组织定向的技术分享，欢迎感兴趣的同学加微信：liyingjiesa，进群参与，您有想听的话题或者想分享的话题都可以给我们留言。
