---
layout: post
title: ActorSystem
categories: [powerjob] 
tags: [powerjob]
---

## PowerjobRemoteEngine

用来控制整个Powerjob的网络层

在work启动时创建的一个空对象，后续操作时会用到里面的方法

## EngineConfig

```java
/**
 * 服务类型
 */
private ServerType serverType;
/**
 * 需要启动的引擎类型
 */
private String type;
/**
 * 绑定的本地地址
 */
private Address bindAddress;
/**
 * actor实例，交由使用侧自己实例化以便自行注入各种 bean
 */
private List<Object> actorList;
```

重点是其中的actorList对象，其中包含三个Actor对象：TaskTrackerActor、ProcessorTrackerActor、WorkerActor。



## ActorInfo对象

所有的actor对象拆解后的对象。其中包含所有actor对象，和它下面所有的Handler方法

```java
//   actor对象本身
    private Object actor;
//    当前这个actor对象类上的@Actor注解信息（主要包含path信息）
    private Actor anno;
//当前actor类中所有的HandlerInfo对象
    private List<HandlerInfo> handlerInfos;
```



## HandlerInfo对象

ActorInfo中的属性，包含不同ActorInfo下的Handler修饰的注解，和Handler注解的属性

```java
private HandlerLocation location;
/**
 * handler 对应的方法
 */
private Method method;

/**
 * Handler 注解携带的信息
 */
private Handler anno;
```





## LightTaskTrackerManager

> 轻量级任务管理器

```java
//    用来存放所有轻量级任务，key为实例ID，value是任务对象
private static final Map<Long, LightTaskTracker> INSTANCE_ID_2_TASK_TRACKER = Maps.newConcurrentMap();
```





## HeavyTaskTrackerManager

> 重量级任务管理器

```java
//    用来存放所有的重量级任务
    private static final Map<Long, HeavyTaskTracker> INSTANCE_ID_2_TASK_TRACKER = Maps.newConcurrentMap();
```



## 初始化

![1722501205010actorSystem的初始化.png](https://fastly.jsdelivr.net/gh/thecoolboyhan/th_blogs@main/image/1722501205010actorSystem%E7%9A%84%E5%88%9D%E5%A7%8B%E5%8C%96.png)

以TaskTrackerActor为例

1. 从EngineConfig中取出TaskTrackerActor
2. 创建一个ActorInfo对象和HandlerInfo对象
3. 将所有的ActorInfo和对应的HandlerInfo交给PowerjobRemoteEngine来实现响应式编程（分为阻塞和非阻塞）两种处理方式。利用事件来触发
4. 后续所有的操作，均通过PowerjobRemoteEngine来触发worker和给server发消息。



## TaskTrackerActor



### 服务器任务调度处理器（onReceiveServerScheduleJobReq）

> 服务器触发”runJob“path的命令，worker检测到开始执行。

``` mermaid
graph TD
    A[服务器任务调度处理器] --> B{是否是轻量级任务}
    B --> |Yes| C[创建轻量级任务]
    C --> D{判断是否存在重复的任务}
    D --> |No| G{判断轻量级任务是否超载\n是否超过了1024*1.3}
    G --> |No| I{判断轻量级任务是否超过了1024个}
    I --> |Yes| J[告警提示轻量级任务超载]
    J --> K[原子性创建一个轻量级任务]
    B --> |No| M[创建重量级任务]
    M --> N{不存在重复实例id的重量级任务}
    N --> |No| P{判断重量级任务是否抄负荷是否超过64个}
    P --> |No| R[原子性创建重量级任务]

```

> 轻量级任务，重量级任务这里分析完全可以单独再做一次研究报告，暂时就不展开了。





### ProcessorTracker 心跳处理器

由"reportProcessorTrackerStatus"命令触发，请求参数中包含实例id

> 由代码推断，只有重量级任务需要上报心跳



``` mermaid
graph TD
	A[从重量级任务管理器中取出被检查的任务]-->B[处理心跳]
	B-->C[根据请求参数更新当前任务的任务状态]
	C-->D{检测当前任务是否长期处于空闲状态}
	D-->|yes|E[销毁目标地址的任务\n销毁方式在状态管理器中将目标机器重置为初始状态]
	E-->F[通过数据库查询长期处于空间机器上所有执行中的任务]
	F-->G[将空闲机器上的所有任务全部改成失败状态]
```

- 任务状态参数

```java
private static final int DISPATCH_THRESHOLD = 20;
private static final int HEARTBEAT_TIMEOUT_MS = 60000;

// 冗余存储一份 address 地址
private String address;
// 上次活跃时间
private long lastActiveTime;
// 等待执行任务数
private long remainTaskNum;
// 是否被派发过任务
private boolean dispatched;
// 是否接收到过来自 ProcessorTracker 的心跳
private boolean connected;
```



### 停止任务实例

由“stopInstance”命令触发，请求参数中包含实例id

``` mermaid
graph TD
	a[从请求中找出实例id]-->b{判断当前实例是什么实例?}
	b-->|重量级|c[关闭重量级任务]
	b-->|轻量级|d[关闭轻量级任务]
	c-->c1[将结束标识改为true]
	c1-->c2[0. 开始关闭线程池\n 1. 通知 ProcessorTracker 释放资源\n 2. 删除所有数据库数\n 3.移除顶层引,送去GC\n 4.强制关闭线程池]
	d-->d1{判断任务是否已经结束}
	d1-->|no|d2{判断结束标识是否为true}
	d2-->|no|d21[修改标识为true]
	d2-->|yes|d22{判断是否仍有未执行的任务}
	d22-->|yes|e1[执行销毁方法]
	d22-->|no|e{判断是否有在执行的任务}
	e-->e2[尝试打断任务]
```





### 查询任务的运行状态

由“queryInstanceStatus”命令触发，请求参数中包含实例id

查询任务状态的方法，方法设计2中重量级任务和一种轻量级任务的不同查询方式。





### 子任务状态上报处理器

由"reportTaskStatus"命令触发，请求参数中包含实例id

> 只有重量级任务存在子任务状态上报机制

``` mermaid
graph TD
	a{判断当前子任务是否需要广播}-->|yes|b[批量告诉所有节点当前节点任务处于'等待调度器调'状态]
	a-->|no|c
	b-->c[更新子任务的任务状态]
	c-->d[更新工作流上下文]
```



### 子任务 map 处理器

由"mapTask"命令触发，请求参数中包含实例id，和所有的子任务：List<SubTask> subTasks

> 只有重量级任务存在子任务map



``` mermaid
graph TD
	a[新建一个subTaskList用来存放所有子任务]-->b[从参数中取出所有分给当前节点的子任务]
	b-->c[把所有分给当前worker的子任务实例id设置为当前实例id\n把所有的任务状态设置为'等待调度器调度']
	c-->d[将所有任务全部持久化到当前worker的数据库or内存]
```



