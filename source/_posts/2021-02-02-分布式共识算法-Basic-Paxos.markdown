---
title:  "分布式共识算法Basic Paxos"
date:   2021-02-02 00:00:00
tags:
    - Doris
---



# Basic Paxos
## 什么是Paxos
* Paxos是解决分布式共识问题的一系列算法，由 [Leslie Lamport](https://lamport.azurewebsites.net/) 提出，大神的几篇关于Paxos的论文：

    [1998-The Part-Time Parliament](https://lamport.azurewebsites.net/pubs/lamport-paxos.pdf)

    [2001-Paxos Made Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)

    [2004-Cheap Paxos](http://lamport.azurewebsites.net/pubs/web-dsn-submission.pdf)

    [2005-Fast Paxos](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-2005-112.pdf)

* 共识(Consensus)是多个节点就某个提案(n, v)达成统一结果的过程。因为参与者本身会失败，或者参与者之间的通信会失败，比如网络分区，消息延迟，节点故障，所以这是个棘手的问题。

## Basic Paxos
* 就单个值达成共识，如果大多数节点就单个值达成共识，值就不会再变了。不涉及连续多个值的共识协商。

### 角色
* 客户端(Client)：向分布式系统发送请求，并接收回复。
* 提议者(Proposer)：接受客户端的一个请求并支持该请求，之后尝试让其他接受者支持相同的请求。
    接入和协调功能，收到客户端请求后，发起 *二阶段提交* ，进行共识协商
* 接受者(Acceptor)：对提案进行投票，存储接受的提案。
* 学习者(Learner)：不参与投票，在接受者达成共识之后，存储保存。
一个节点可以同时扮演多个角色，不影响算法正确性，一个节点扮演多个角色可以降低延迟，减少节点间消息发送

### 提案
* (n, v) n为提案编号，v为提案值
* 提案编号特性：原子，递增，持久存储

### Quorum
本义指的是出席议会议事的最低议员数量。
在Paxos中，接受者组成Quorums。发送给某一个接受者的消息必须发送给Quorum中的所有其他接受者。

## 二阶段提交
* Phase 1：准备阶段

    a. Prepare: 创建一个提案(n, v)，发起二阶段提交, n必须是递增的。将Prepare message(只包含提案编号n，不需要包含v)发给a quorum of acceptors

    b. Promise: 接受者比较n和自己已经响应的提案的最大提案编号：
    
        1. 如果小于等于已经响应的最大提案编号，则忽略，可以不响应，也可以响应以防止提议者反复提交提案编号为n的Prepare message。
    
        2. 如果大于已经响应的最大提案编号，接受者返回给提议者Promise，并忽略所有提案编号小于n的提案；如果接受者之前接受了某个提案(m, w)，则接受者必须将这个提案信息返回给提议者。
* Phase 2：接受阶段

    a. Accept: 
    
        1. 如果接受者已经达成共识(m, w)，则Accept message为(n, w)
        
        2. 如果提议者接收到a quorum of acceptors中大多数节点的Promises，会设置节点值为提案值v，提议者发送Accept message(n, v)
    
    b. Accepted: 
    
        1. 接受者之前尚未达成共识，Accept message(n, v)
        
        2. 接受者之前已达成共识(m, w), Accept message(n, w)


### Basic Paxos几个原则
* 如果准备请求的提案编号，小于等于接受者已经响应的准备请求的提案编号，则接受者不会响应这个准备请求

* 如果接受请求的提案编号，小于接受者已经响应的准备请求的提案编号，则接受者不会通过这个提案

* 如果接受者已经通过提案，那么接受者会在准备请求的响应中，包含已通过的最大编号的提案信息

* 如果大多数接受者对某个提案达成共识，那么提案值就不会再变了，而提案编号会因为少数节点(尚未通过任何提案)接收到新的准备请求而使用递增的更大的提案编号。


### 二阶段提交场景 
一个Quorum: A,B,C
* 场景1：节点A通过了准备请求(1,v1)，但还没有接受提案(1,v1)，如果此时节点A接受到新的准备请求(2, v2)，因为新的准备请求的提案编号2大于之前的提案编号1，则节点A保证不再响提案编号小于等于2的准备请求，也不会通过提案编号小于等于2的提案。

* 场景2：节点A,B已经通过了提案(2, v1),节点C未通过提案，此时客户端发起请求设置(3, v2), 因为AB已经通过提案(大多数节点)，所以提案值不会变动，而提案编号使用最大的3，则最终节点值(3, v1)


## 领导者选举
* 可以通过Basic Paxos算法实现领导者选举，也就是多个节点就提案达成共识的过程。在Chubby中，领导者选举就是通过Basic Paxos实现的。
