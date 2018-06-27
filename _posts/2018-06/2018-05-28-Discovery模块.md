---
layout: post
title: "Discovery发现模块原理剖析"
date: 2018-05-28
categories: Elasticsearch
excerpt: 自动发现节点并进行监控
---

* content
{:toc}


# 分布式集群

分布式体现：索引的数据会根据某种规则分摊到各个节点分片上(一个节点会有多个分片)

集群体现： 一个分片会有多个副本，分布在和主分片不同的节点上，提高了服务的高可用和提供了查询效率

## Discovery发现模块

该模块主要实现：发现集群中节点、集群主节点选举、集群状态的同步

整体发现流程如图：

![props](http://dymdmy2120.github.com//static/2018-06-images/discovery-flow-summary.png)


### ZenDiscovery 代码分析

#### 一、启动一个本地线程，尝试将本地节点加入集群(加入失败会有重试)

为了保证在多线程环境下安全，采用了一个内部类JoinThreadControl进行控制，内部使用了AtomicBoolean running表示是否开始， AtomicReference<Thread> 采用cms方法保证有且只有一个线程进行参数集群选举和加入等操作。其中每次操作之前都会进行判断当前线程是否为活跃加入线程 if (!joinThreadControl.joinThreadActive(currentThread))  return;

开始一个JoinThread 线程：

![props](http://dymdmy2120.github.com//static/2018-06-images/discovery-flow-start-thread.png)

#### 二、主节点的选举
	
1、首先会ping其他服务器，获取到一个PingResponse集合(集群名、节点类型(上次集群中的主节点，节点对象、集群版本号)) 采用TransportService模块进行和其他节点通讯

2、把自己的节点加入到fullPingResponse集合中

3、如果在配置文件中设置了discovery.zen.master_election.ignore_non_master_pings=true机会过滤掉非mater候选节点默认为false

4、从pingResponse集合中获取在其他节点当前的master节点和非本地节点(又可能不经过check和认证直接选举自己，这样会造成脑裂) 将这些节点放在activeMasters集合中

5、从pingResponse集合获取 node.isMasterNode() 候选主节点 放在masterCandidates集合中

**思考1：ElectMasterService类中的minimumMasterNodes属性为什么定义成volatile**

答： 因为当集群中增加了一个候选主节点时就通知更改minimumMasterNodes值，在多线程环境下可以及时发现此值已经改变过来，然后判断满足条件就可进行下一步选举操作

6、当activeMasters集合为空时，会判断候选主节点节点数目是否大于discovery.zen.minimum_master_nodes配置，若大于则进行下一步，否则会return null，然后继续下次选举。 如果满足就行进行选举，对候选节点进行排序
排序规则为：集群版本越大排在前面，可以为master节点排在前，最后按照node id进行字段排序，越小在前面。最后取出第一个元素
当activeMasters不为空时，同样也会按照以上排序规进排序，取出第一个元素

ping请求：

![props](http://dymdmy2120.github.com//static/2018-06-images/discovery-flow-ping-request.png)

选举master节点：

![props](http://dymdmy2120.github.com//static/2018-06-images/discovery-flow-Elect-Master.png)

#### 三、根据选择选举的master node和local node进行对比

##### 如果本地节点就是上面选举的master节点

1、会调用一个nodeController waitToBeElectedAsMaster方法，最终如果回调了 onElectedAsMaster方法表示当前节点为主节点了，然后开启 Node Fault Detection nodeFD,会ping其他各个节点，ping不通就会移除集群

如果回调了onFailure方法则 则结束当前线程，然后重新开始一个新的join thread重复上面的步骤

2、在waitToBeElectedAsMaster方法中使用了CountDownLatch 使用 await(等待其他节点加入集群的超时时间) 阻塞当前线程 直到 符合了条件或失败回调了，就会countDown，主线程继续运行

3、step2中说的条件是 requiredMasterJoins = minimumMasterNodes(最少候选主节点)-1，减1表示除去当前选举的master节点， 就是判断pendingMasterJoins >= requiredMasterJoins是否满足 ，其中pendingMasterJoins 会通过 Map<DiscoveryNode, List<MembershipAction.JoinCallback>> joinRequestAccumulator  if(node.isMasterNode()) pendingMasterJoins++; 进行计算

4、joinRequestAccumulator这个集合的数据是什么时候初始化的，在那里进行的，首先判断本地节点不是选举的master节点时，就会尝试joinElectedMaster，这个时候就会向master发送join请求 使用MembershipAction中的方法发送的，然后master会进行监听使用 JoinRequestRequestsHandler进行对接受请求做处理，重写messageReceived方法，其中该方法参数就会有要加入的node消息，最后会调用NodeJoinController 的handleJoinRequest，最后就会放入joinRequestAccumulator集合中

5、NodeFaultDetection,当已经确认了选举的master后，就会对集群中其他的节点进行ping,每隔一定discovery.zen.fd.ping_interval 默认为1s时间进行ping一次 ping的超时时间为discovery.zen.fd.ping_timeout 默认30s，如果ping失败了会进行尝试 discovery.zen.fd.ping_retries默认为3 ，如果尝试次数超过了，则从移除该节点，并通知集群
               

##### 如果本地节点不是master节点

1、停止选举master动作

2、joinElectedMaster 加入到主节点中 利用 transport模块 告诉主节点，那个节点需要加入进去，其中如果失败时会进行多少次重试，每隔多长时间重试一次，这些都可以进行配置

3、如果加入失败，则markThreadAsDoneAndStartNew 重新上面的步骤

4、如果发现当前集群中的master节点改变了，则会调用joinThreadControl.stopRunningThreadAndRejoin方法

开启对master节点的监控：

![props](http://dymdmy2120.github.com//static/2018-06-images/discovery-flow-Ping-MasterFaultDetection.png)

开启master对集群其他节点的监控

![props](http://dymdmy2120.github.com//static/2018-06-images/discovery-flow-Ping-NodeFaultDetection.png)


## 主节点发布集群状态分析

Discovery接口提供了publish 发布ClusterState Event的方法，在ZenDiscovery实现的，实际上具体实现是在PublishClusterStateAction中的

一、序列化状态数据

1、会根据配置是否每次发送full state discovery.zen.publish_diff.enable默认为true 发送diff state。 当之前状态为null时也会发送full state

2、封装发送的状态数据 将每个版本作为key value就是ClusterState序列化之后的字节

3、SendingController   主线程会一直阻塞直到 候选主节点ack数>= minMasterNode-1 这个会有个超时时间为commit timeout 当ack数>= minMasterNode-1时就会将state commit到这些节点，有个awaitAllNodes方法会一直阻塞 直到所有节点响应了commit state请求，这个超时时间为 pushTimeout-(send到所有节点到ack所用时间)

使用到CountDownLatch  committedOrFailedLatch: 用于阻塞直到候选主节点都ack 还有 latch count为 nodeToPingCount 用于阻塞直到所有node 都commit 响应

**注意：**

其他节点使用 pendingStatesQueue 队列存放 master commit 集群的状态	 	
集群状态的更新：

![props](http://dymdmy2120.github.com//static/2018-06-images/discovery-flow-cluster-state-update.png)

## 如何选举master节点

* 先获取配置文件中的 ping.unicast.host ip和port，如果没有配，默认使用多播协议 multicast，会ping同个网段的主机


* ping其他节点时会response 节点信息和当前集群的master节点


* 选举开始，先从各节点认为的master中选，规则很简单，按照id(node的id?)的字典序排序，取第一个


* 如果各节点都没有认为的master，则从所有master候选节点选择，规则同上。这里有个限制条件就是 

discovery.zen.minimum_master_nodes(可成为主节点/2+1，该参数可以防止集群发生脑裂)，如果节点数(可以成为master的节点)达不到最小值的限制，则循环上述过程，直到节点数足够可以开始选举


* 最后选举结果是肯定能选举出一个master，如果只有一个local节点那就选出的是自己

* 如果当前节点是master，则开始等待节点数达到 minimum_master_nodes，然后提供服务

* 如果当前节点不是master，则尝试加入到master集群中


### 和zookeeper比较

zookeeper要求可用节点数>总节点数/2 ，而且总节点数最好为奇数个 原因如下：

*  防止脑裂导致集群不可用 ：

  脑裂是指 部分节点之间不能通讯，这样会导致划分成多个小集群，而各个小集群选出各自的master节点，这样原有的集群就产生了多个master节点，这就是脑裂。 
  
  (1)、假如原有节点为5个，发生脑裂后产生了两个小集群 A、B
  
  第一种情况  A: 1个节点 B:4个节点 或者 A、B互换
  
  第二种情况  A：2个节点 B:3个节点 或A、B互换
  
  由于形成一个集群最基本要求是 可用节点>总节点数/2 所以 无然情况1 情况2 都会有一个小集群继续提供服务
  
  （2)、假如为偶数节点 4个 
  
  第一种情况  A: 1个节点 B:3个节点 或者 A、B互换
  
  第二种情况  A：2个节点 B:2个节点 或A、B互换
  
  第一种情况就可以继续提供服务，但是第二种情况就不能，因为A B集群可用节点为2个 都不满足 可用节点数>总节点数/2的选举条件，所以如果总节点数为偶数时，当脑裂成两个均等的子群时就会导致服务不可用，所以存在不能用的可能性。
  
*   在容错相同的情况下，更节省成本

	例如A集群节点数为 3。B集群节点数为4个  A允许1个节点挂掉，而B节点 可用节点数>总节点数/2 所以也允许1个节点挂掉。
	
可能会想为什么可用节点数要过半呢？ 这个和zookeeper选举leader算法有关，当启动时或leader挂掉时，都会进行投票节点 过程如下：
  
  1、首先会投自己一票(myid,zxid)
  
  2、当其他节点接受到集群中投票时处理投票，优先比较zxid，选更大的节点，如果相同时选myid大的
  
  3、统计投票，**发现超过一半以上都投了同个节点时**，那么就选择这个节点作为leader
  
  4、其他follower从leader同步数据

## 网络层 Transport

Elasticsearch 为了避免传输流量比较大的操作堵塞连接，当共用一个Channel，Channel就是Client 和Server端的一个tcp连接，会造成后面紧急的请求被前面大流量的操作阻塞了，而进行等待。所以会按优先级创建多个tcp连接称为Channel。

* recovery: 2个channel专门用做恢复数据。如果为了避免恢复数据时将带宽占满，还可以设置恢复数据时的网络传输速度。
* bulk: 3个channel用来传输批量请求等基本比较低的请求。
* regular: 6个channel用来传输通用正常的请求，中等级别。
* state: 1个channel保留给集群状态相关的操作，比如集群状态变更的传输，高级别。
* ping: 1个channel专门用来ping，进行故障检测。

节点A 会有13个来自节点B连接，同样节点B也有来自节点A的13个连接，当加入一个节点C时， 节点C启动服务，然后会来自 节点A B各13条连接，同样节点C也会13个连接 连接节点A的服务和13个连接连接节点B的服务

## 涉及到的类和接口

### 类图

![props](http://dymdmy2120.github.com//static/2018-06-images/discovery-class.png)

### 接口

Discovery: 发现节点、发布集群状态给其他节点、选举主节点并改变集群状态

ZenPing: 提供了ping请求接口，以及和一些相关的类 PingResponse PingCollection，这些类只能在同一包下使用


### 类

ZenDiscovery: 实现了Discovery接口

UnicastZenPing: 实现了ZenPing接口

JoinThreadControl: 用于维护一个线程进行选举主节点以及加入集群操作

ElectMasterService: 用作选举主节点，也就是指定排序规则

MembershipAction: 用户完成将将节点加入集群逻辑

MasterFaultDetection: 集群其他节点对master节点进行监测

NodeFaultDetection:   master对集群其他节点进行监测

PublishClusterStateAction: 由主节点进行调用，封装了将更新的状态同步到集群其他节点

ClusterService: **目前还不了解具体逻辑** 集群相关的服务

TransportService: 提供节点到节点进行通信的服务

EsThreadPoolExecutor: 自己封装了线程池逻辑

## 设计模式

### 模板模式

例如在每次启动一个线程时，都要创建一个实现 Ruunablle接口的对象，这样才会执行其中的run方法，为了统一实现处理运行时出现的异常，而具体的处理异常逻辑放在子类进行实现， 使用了一个抽象类实现了Runnable接口，并且在其中抽象类方法中提供了两个抽象方法供子类进行实现具体的逻辑。
 
     public abstract AbstractRunnable implements Runnable {
     
		   @Override
		    public final void run() {
		        try {
		            doRun();
		        } catch (Exception t) {
		            onFailure(t);
		        }
		    }
			
			// 处理异常的具体逻辑，在子类进行实现
			public abstract void onFailure(Exception e);
			
			//具体运行什么逻辑，在子类进行实现
			protected abstract void doRun() throws Exception;

## summary

### 主节点是如何选举的

首先会ping其他节点，获取PingResponse 集合，然后从中选出master 先排序，排序规则：版本大靠前，主节点候选节点靠前，nodeId字典排序小的靠前。 选出的主节点后，并不会立刻作为主节点，而是需要等到 minMasters - 1个节点加入进来后才会更新集群状态，成为主节点，开始工作。

### 集群中的节点是如何监测的

1、NodeFaultDetection 这个是主节点对集群的节点进行监测，如果任何一个节点ping 有问题并且retryCount超过了配置数，就会从集群中剔除该节点，然后将状态同步到集群中。

2、MasterFaultDetection 这个是其他节点进行对主节点监控的主类，在每次主节点对集群状态进行更新时其他节点都会restart ping的机制，如果ping 到master有问题并且retryCount超过了配置数,那么就会发起rejoin 请求重新开始JoinThread 流程。


### 同步状态到集群中是什么概念

状态可以看作是对集群的一种操作，这些操作必须只能主节点进行操作，然后进行同步到其他节点。
例如：

1、创建索引、删除索引

2、template的创建和删除

3、Mapping的创建和删除

4、打开和关闭索引

5、resharding 重新分片

6、节点关闭

7、集群信息、健康、配置、路由








	