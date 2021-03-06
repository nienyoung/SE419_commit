Google Borg（超大规模集群管理系统Borg） 
========================================

Google Borg简介
---------------

Google是一个集群管理器，负责对成千上万个应用程序提交job进行接收、调试、启动、停止、重启、监控等。它可以同时管理者多个数据集群，每个集群上有成千上万台机器。Borg通过接入控制，高效的任务打包，超额的资源分配和进程级隔离的机器共享，来实现超高的资源利用率。它通过最小化故障恢复时间的运行时特性和减少相关运行时故障的调度策略来支持高可用的应用程序Borg通过提供一个作业声明的标准语言，命名服务的集成机制，实时的作业监控，以及一套分析和模拟系统行为的工具来简化用户的使用。

Borg主要有三大优势：
--------------------

隐藏了资源管理和错误处理的细节，让用户关注应用的开发；

具备高可用性和高可靠性，并支持应用的高可靠和高可用；

能够在数万个节点上高效运行任务；

解读：易用性、可用性、可靠性、可扩展性、高利用率是所有云平台都需要解决的问题，Borg在这几个方面做得更好。

borg工作机制
--------

### 集群和单元

在谷歌集群中，通常包含一个大规模的cell和若干个用于测试或者特殊用途的小cell。cell的规模大约是10K个机器，其中的机器在资源大小、处理器型号、性能、容量等方面异构。

个人解读：cell是集群和机器的中间层，这种方式在实践中被广泛使用，通常一个集群中都会预留一部分资源作为测试集群。

### job&task

一个Borg作业Job具有name、owner、task数量等属性。Job具有约束（constraint）来强制其任务运行在具有特定属性的机器上，例如处理器架构、OS版本、IP地址等。约束可以分为硬约束和软约束，前者是需求，必须满足；后者是偏好，尽量满足。每个任务，可以映射为一个container中的一组Linux程序。Borg绝大部分工作负载不运行在虚拟机中，这主要是由于虚拟机性能开销较大，此外一些硬件不支持虚拟化。

一个任务也具有属性，例如资源需求和task
index。Borg程序采用静态链接库，以减少对运行环境的依赖，软件包中主要包括二进制程序和数据文件。
用户向Borg发送RPC用以操作作业。用户可以提交作业、查询作业，也可以改变作业、任务属性。作业Job和任务task的状态图如下图所示。用户可以触发submit、kill和update事务。

任务Task在被抢占之前，会收到通知，二者都是通过Unix信号，抢占通过SIGKILL信号，程序直接退出；通知通过SIGTERM信号，可以被处理。这样做主要是为了让程序关闭前清理环境、保存状态、完成当前请求等。

解读：硬约束主要是为了解决资源异构问题，软约束则主要是为了解决类似于数据本地行的问题；目前将任务运行在容器中是趋势，因为虚拟机的开销太大，因为docker既可以支持镜像又能够简化部署
，所以非常风靡；以事件的方式进行任务状态管理是常用的做法，YARN中就有详细的状态机机制；对于抢占任务，云平台一般都会提供优雅的退出方式，一般情况都先通知应用由其自己决定如何处理，这样做主要是让程序在抢占前清理环境、保存状态等。

### 资源分配集合

一个Borg
alloc是一台machine上预留资源的集合，可以用于一个或多个任务运行。Alloc被用于为未来的task预留资源，用来在停止和再次启动任务之间保持资源，或者将不同作业Job的任务task聚合在一起运行（例如一个web服务器实例和一个相关的logsaver）。

一个Alloc集合类似一个job：它是在多个机器上预留资源的一组alloc。一旦一个alloc集合被创建，一个或者多个作业Job就可以被提交在其中运行。简而言之，一个task与一个alloc对应，一个job与一个alloc集合对应。

解读：alloc是资源单元，但其与YARN中的container不同，因为其中可以运行多个task，更像K8S中的pod（包含多个container）。

### 优先级、配额和准入控制

每个作业都有一个优先级，具体形式是一个小的正整数。Borg定义非重叠优先级，包括：monitoring,
production, batch, and best effort (also known as testing or
free)，生产型作业（prod job）包含前两种优先级。

为了避免“抢占洪流”，Borg不允许同类型作业之间的抢占，只允许生产型作业抢占非生产型作业。

资源配额Quota是一组资源量化表达式（CPU、RAM、Disk等），它与一个优先级和一个时间段对应。如果Quota不足，作业提交会被拒绝。

解读：在borg中如果配额不足会拒绝任务提交，而通常的调度系统如果资源不足会让任务排队，等到资源充足时系统再进行调度执行。这种差别可能是系统的用户群不同所导致的，borg的用户是高级工程师，而通常的调度系统使用者一般对分布式不太熟悉，所以需要将提交、调度和运行的细节封装起来。

### 名字服务和监控

Borg创建了一个稳定的Borg名字服务（BNS），用于为每个任务命名。任务名字包括cell
name、job name和task
number这个三元组。Borg将任务的主机名和端口号写入Chubby中的一致性、高可用文件。Borg也会将作业规模和任务健康信息写入Chubby，负载均衡器LoadBalancer据此来进行路由。

每个任务会构建HTTP
server来发布其健康信息和性能指标，Borg根据URL来监控任务，并重启异常任务。

解读：borg中不仅运行离线任务还运行在线任务，这就会导致集群中的服务相互之间可能会存在调度关系，这时候就需要名字服务。当客户端或者其他系统需要访问服务时，首先访问BNS根据任务名查找具体的访问方式进行访问；监控室每个调度系统都需要有的组件，对于一个数据节点，其需要定时的汇报自己所在节点的状态信息包括资源使用信息、将康状态信息和任务执行的汇总信息等，一般是设定好固定的时间间隔进行主动心跳上传。

borg架构
--------

borg的中心控制模块叫BorgMaster，每个节点模块叫Borglet，所有组件均使用C++编写。（基础平台使用C++书写属于惯例，能够从底层提高性能，而Java语言则比较重，不过使用Hadoop生态的还需要使用Java语言）

### borgmaster

borgmaster包含两个进程：主进程：负责处理客户端RPC调用，管理系统中所有对象的状态机，与borglet进行通信。以及一个分离的调度器。

borgmaster在逻辑上是一个进程，但是有5个副本，每个副本都维护cell状态的一份内存副本，cell状态同时在高可用、分布式、基于Paxos的存储系统中做本地磁盘持久化存储。一个单一的被选举的master既是Paxos
leader，也是状态管理者。当cell启动或者被选举master挂掉时，系统会选举Borgmaster，选举机制按照Paxos算法流程进行。选举master的过程通常会消耗10秒，在一些规模较大的cell会消耗1分钟左右。

borgmaster的状态会定时设置checkpoint，具体形式就是在Paxos
store中存储周期性的镜像snapshot和增量更改日志。其作用可以存储borgmaster的状态，修复问题，离线模拟等。

此外还提供了一个模拟器Fauxmaster，其是borgmaster的一个复制品，可以用来修复bug，模拟执行。

解读：master是一个调度系统的核心，为了保证可用性通常的做法就是进行副本维护，当一个master出现故障的时候可以由新的master来替代。

### scheduling

当job被提交时，borgmaster会将其记录到paxos
store中，并将task增加到等待队列中，调度器通过一步的方式扫描队列将机器分配给任务，调度过程主要包括可行性检查和打分。

可行性检查主要用于解决硬约束，找到一组满足硬约束的机器；打分则是满足软约束，在可行的机器中根据用户偏好为机器打分，用户偏好主要是系统内置的标准，如：被抢占的task的优先级和数量最小化，具有任务软件包的机器、分散任务到不同的失败域中，将高优先级和低优先级的task分配到同一个node上，使得高优先的任务可以抢占低优先级的任务等。borgmaster一开始使用E-PVN进行打分，其会将任务分散到不同的机器上，称为worst
fit;而best
fit会尽量使用紧凑的机器以减少资源碎片。考虑到系统中不同类型任务单额资源需求，采用混合的模型进行打分。

如果对于一个新task没有充足的资源，那么系统会杀死低优先级的任务，杀死的低优先级的任务会被放置在等待队列中。任务启动延迟平均在25秒，有80%的时间花费在包安装上。因此在调度的时候尽量把任务调度在含有包的机器上，同时因为包是不变得，所以可以共享和缓存，同时还提供了一些并行分发包的协议。

### borglet

borglet是运行在每台机器上的代理，负责启动和停止task,重启任务，管理本地资源，向master和监控模块汇报自己的状态。master会周期性的向borglet拉取其当前的状态，这样做能够控制通信速度避免“恢复风暴”。

为了性能扩展，master副本会运行一个无状态的link
shard去处理与部分borglet的通信，但重新选举时会重新分区，master副本会聚合和压缩信息，向被选举的master报告状态机的不同部分，减少更新负担。如果borglet多轮都没有进行响应会被标记为down。运行在上面的任务会被重新调度至其他机器，如果通信恢复，master会通知borglet杀死已经重调度的任务保证一致性。

解读：borglet的设计与其他调度系统相同，提供汇报功能。不同的是borg中采用拉取的方式进行，而在yarn中采取的是推的方式。另外borg中的副本会与一些borglet通信以此方式来减少更新负担。
