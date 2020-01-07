K8s CI/CD
=========

K8s进行CI/CD的优势
------------------

目前大多公司都采用 Jenkins 集群来搭建符合需求的 CI/CD 流程，然而传统的 Jenkins
Slave 一主多从方式会存在一些痛点，比如：

-   主 Master 发生单点故障时，整个流程都不可用了

-   每个 Slave
    的配置环境不一样，来完成不同语言的编译打包等操作，但是这些差异化的配置导致管理起来非常不方便，维护起来也是比较费劲

-   资源分配不均衡，有的 Slave 要运行的 job 出现排队等待，而有的 Slave
    处于空闲状态

-   资源有浪费，每台 Slave 可能是物理机或者虚拟机，当 Slave
    处于空闲状态时，也不会完全释放掉资源。

正因为上面的这些种种痛点，我们渴望一种更高效更可靠的方式来完成这个 CI/CD
流程，而 Docker 虚拟化容器技术能很好的解决这个痛点，又特别是在 Kubernetes
集群环境下面能够更好来解决上面的

Jenkins Master 和 Jenkins Slave 以 Pod 形式运行在 Kubernetes 集群的 Node
上，Master 运行在其中一个节点，并且将其配置数据存储到一个 Volume 上去，Slave
运行在各个节点上，并且它不是一直处于运行状态，它会按照需求动态的创建并自动删除。

这种方式的工作流程大致为：当 Jenkins Master 接受到 Build 请求时，会根据配置的
Label 动态创建一个运行在 Pod 中的 Jenkins Slave 并注册到 Master 上，当运行完 Job
后，这个 Slave 会被注销并且这个 Pod 也会自动删除，恢复到最初状态。

**服务高可用**，当 Jenkins Master 出现故障时，Kubernetes 会自动创建一个新的
Jenkins Master 容器，并且将 Volume
分配给新创建的容器，保证数据不丢失，从而达到集群服务高可用。

**动态伸缩**，合理使用资源，每次运行 Job 时，会自动创建一个 Jenkins Slave，Job
完成后，Slave 自动注销并删除容器，资源自动释放，而且 Kubernetes
会根据每个资源的使用情况，动态分配 Slave
到空闲的节点上创建，降低出现因某节点资源利用率高，还排队等待在该节点的情况。

**扩展性好**，当 Kubernetes 集群的资源严重不足而导致 Job
排队等待时，可以很容易的添加一个 Kubernetes Node 到集群中，从而实现扩展。

思路
----

1: jenkins部署在k8s中，并设置rs=1  
2: 每次构建时，启动一个slave pod来执行构建过程，构建结束后slave pod 会删除  
3: slave pod 所使用的docker file 需要自定义，里面安装了 jdk,maven,helm，docker，并在jenkins-master里面设置相关挂载。

安装jenkins
-----------

需要将 Jenkins 安装到 Kubernetes 集群当中，新建一个
Deployment：(jenkins.yaml)，通过 NodePort 的形式来暴露 Jenkins 的 web
服务，固定为30002端口，另外还需要暴露一个 agent 的端口，这个端口主要是用于
Jenkins 的 master 和 slave 之间通信使用的。
```

apiVersion: v1

kind: Service

metadata:

name: jenkins2

namespace: kube-ops

labels:

app: jenkins2

spec:

selector:

app: jenkins2

type: NodePort

ports:

\- name: web

port: 8080

targetPort: web

nodePort: 30002

\- name: agent

port: 50000

targetPort: agent
```

/var/jenkins_home 目录挂载到了一个名为 opspvc 的 PVC
对象上面，提前创建一个对应的 PVC 对象 (pvc.yaml)

使用一个拥有相关权限的 serviceAccount：jenkins2，只是给 jenkins
赋予了一些必要的权限（rbac.yaml）

![](media/9a1a322a234a1b519740fcbb5787d4f8.tmp)

![](media/a6b7d04007fe9fe070c2610acc699630.tmp)

查看生成的pod：

![](media/fef3bbc15229143a7c794daf507df78a.tmp)

根据任意节点的 IP:30002 端口可以访问 jenkins 服务，根据提示信息进行安装配置

![](media/9ef670bcc1552c267196eb75b77ea363.tmp)

![](media/d81a6394064afc370e99c775f87d2da1.tmp)

配置jenkins
-----------

1.  安装**kubernetes** 点击 Manage Jenkins -\> Manage Plugins -\> Available -\>
    Kubernetes plugin 勾选安装。

2.  点击 Manage Jenkins —\> Configure System —\> Add a new cloud —\> 选择
    Kubernetes填写 Kubernetes 和 Jenkins 配置信息。

![](media/c13c122c583c6c9aed36d1f5db8560d0.tmp)

3.   配置 Pod Template，就是配置 Jenkins Slave 运行的 Pod 模板，

![](media/1058868e02ad8ed0d6e01a3ff9ec1197.tmp)

测试jenkins
-----------

添加一个 Job 任务，看是否能够在 Slave Pod 中执行，任务执行完成后看 Pod
是否会被销毁。

创建一个测试的任务选择 Freestyle project 类型的任务

![](media/804cbcea11f80424af944d5f5b830827.tmp)

在Build 区域选择**Execute shell**

![](media/bbd4ecd083c0e7313dd14bd865be36a8.tmp)

输入测试命令

![](media/f5fc9eb5bdb564190e17da39157b0990.tmp)

在页面点击Build now 触发构建，观察 Kubernetes 集群中 Pod 的变化

![](media/68a34a6901e8acf8629d899b47084780.tmp)

任务构建完成后再去集群查看Pod 列表，发现 kube-ops 这个 namespace
下面已经没有之前的 Slave 这个 Pod 了

![](media/0348f03a2e6f940a2b3d51dfd2138400.tmp)
