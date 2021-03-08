---
title:  "Yarn Application开发"
date:   2020-11-29 00:00:00
tags:
    - YARN
---


# Yarn Application开发

## 1. Yarn架构

![yarn](/images/yarn_application/yarn.png)

YARN的本质在于将资源管理和作业调度监控的功能分离，在编写Application之前，需要熟悉YARN中的几个重要角色：RM，NM，AM等。



### 1.1 ResourceManager(RM)
#### 1. Scheduler

"纯"资源Scheduler，只负责调度和分配资源，不会监控Application状态，包括FairScheduler和CapacityScheduler等。

#### 2. ApplicationsManager(ASM)

负责接收管理整个系统中所有应用程序的提交、与调度器协商资源以启动ApplicationMaster、监控ApplicationMaster运行状态并在失败时重新启动它等。RM只负责监控AM，在AM运行失败时候启动它，RM并不负责AM内部任务的容错，这由AM来完成。


### 1.2 NodeManager(NM)
运行容器，监控容器资源(CPU，内存，磁盘，网络)使用情况，并报告给RM。

接收并处理来自AM的Container启动/停止等各种请求。

### 1.3 ApplicationMaster（AM）

每个Application都有一个ApplicationMaster，负责向Scheduler申请运行Application需要的Container，协调Application运行，监控Application执行状态和进度，ApplicationMaster负责将一个应用分割成多个任务。RM只负责监控AM，在AM运行失败时候启动它，RM并不负责AM内部任务的容错，这由AM来完成。

### 1.4 Container

YARN中的资源抽象，封装了NM的资源，如内存、CPU、磁盘、网络等，当AM向RM申请资源时，RM返回的资源的用Container表示，YARN会为每个任务分配一个Container。现在YARN仅支持CPU和内存两种资源，使用轻量级资源隔离机制Cgroups进行资源隔离。


## 2. YARN应用开发流程
### 2.1 YARN Client

示例代码：

https://github.com/apache/hadoop/blob/trunk/hadoop-yarn-project/hadoop-yarn/hadoop-yarn-applications/hadoop-yarn-applications-distributedshell/src/main/java/org/apache/hadoop/yarn/applications/distributedshell/Client.java

**Client<-->ResourceManager**

**Interface YarnClient**

1. 初始化并启动Yarn Client

2. 创建application，并获取application id

3. 设置application的上下文-ApplicationSubmissionContext

```
    1. Application info: id， name

    2. Queue, priority info：application提交队列， application分配的priority

    3. User：提交application的用户

    4. ContainerLaunchContext

        4.1 R: local Resources (binaries, jars, files etc.)

        4.2 E: Environment settings (CLASSPATH etc.)

        4.3 C: the Command to be executed

        4.4 T: security Tokens (RECT)
```

4. 提交application：submitApplication

5. 追踪进度：getApplicationReport

6. 杀掉application：killApplication


### 2.2 Application Master

示例代码：

https://github.com/apache/hadoop/blob/trunk/hadoop-yarn-project/hadoop-yarn/hadoop-yarn-applications/hadoop-yarn-applications-distributedshell/src/main/java/org/apache/hadoop/yarn/applications/distributedshell/ApplicationMaster.java

**ApplicationMaster<-->ResourceManager**

**Interface AMRMClientAsync**

1. AM启动后，分别启动AMRMClient和NMClient

2. AM通过AMRMClient向RM注册自己，和RM保持心跳：registerApplicationMaster

3. 申请Container：addContainerRequest

4. 运行Containers并通过心跳将进度报告给RM

5. ApplicationMaster确认作业结束后，通过AMRMClient从RM取消注册：unregisterApplicationMaster



**ApplicationMaster<-->NodeManager**

**Interface NMClientAsync**

1. 和启动AM一样，初始化ContainerLaunchContext的RECT

2. 通过NMClientAsync启动执行Task的Container：startContainerAsync

3. NMClientAsync处理Container启动，停止，状态更新


## 示例

```bash
hadoop jar \
/opt/cloudera/parcels/CDH/jars/hadoop-yarn-applications-distributedshell-2.6.0-cdh5.16.1.jar \
 org.apache.hadoop.yarn.applications.distributedshell.Client \
   --jar /opt/cloudera/parcels/CDH/jars/hadoop-yarn-applications-distributedshell-2.6.0-cdh5.16.1.jar \
   --shell_command ls \
   --num_containers 10 \
   --container_memory 350 \
   --master_memory 350 \
   --priority 10
```








