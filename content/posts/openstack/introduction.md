---
title: "OpenStack 简介"
date: 2020-11-26
description: ""
menu:
  sidebar:
    name: "OpenStack 简介"
    identifier: openstack-introduction
    parent: openstack
    weight: 100
tags: ["OpenStack"]
categories: ["OpenStack"]
---

根据相关资料整理 OpenStack 的相关概念。

<!--more-->

## OpenStack 的定义

以下内容翻译自 [Openstack 官网](https://www.openstack.org/software/)。

OpenStack 是一个开源的云操作系统，可以通过具有通用身份验证机制的 API 进行管理和配置，来控制整个数据中心的大量计算、存储和网络资源池。

OpenStack 提供了一个仪表板，可以让管理员进行控制，同时授权其用户通过 Web 界面配置资源。

OpenStack 还提供了编排、故障管理和服务管理等等其他服务，以确保用户应用程序的高可用性。

## OpenStack 组件概览

图片来源于官网：

![OpenStack landscape](/images/openstack/landscape.png)

下面只关注部分组件的功能，大部分内容翻译自 [OpenStack Tutorial](https://mindmajix.com/openstack-tutorial)：

- Nova：基本计算引擎，管理着大量处理计算任务的虚拟机和其他实例。
- Swift：提供对象存储的组件。
- Cinder：提供块存储的组件。
- Neutron：提供网络功能的组件，确保每个组件都与其他组件良好连接，从而在它们之间建立通信。
- Horizo​​n：仪表板，为系统管理员提供了访问和管理云的所有可能性。
- Keystone：提供身份验证的组件。
- Glance：提供映像服务的组件。
- Placement：实现统一的资源管理。
- Tempest：提供自动化测试的组件。

## OpenStack 的部署

OpenStack 的部署相对比较复杂，可以根据官网的教程。如果只是希望部署一个开发环境来学习 OpenStack，也可以使用官方的自动部署脚本 [DevStack](https://opendev.org/openstack/devstack)。

以部署 DevStack 为例，建议运行脚本前进行以下修改：

- 添加 pypi 镜像：比如设置[清华源](https://mirrors.tuna.tsinghua.edu.cn/help/pypi/)。
- 在 `local.conf` 中指定 Git 镜像：比如设置 `GIT_BASE` 为 `https://github.com`。
- 添加 `stack` 用户后修改目录访问权限：比如 `sudo chmod 755 /opt/stack`。

如果中途因为网络问题而失败，直接通过 `./unstack.sh && ./stack.sh` 再次运行脚本即可。

## 相关概念的补充说明

### 云计算模式

云计算的模式主要有三种：IaaS、PaaS 和 SaaS。其中，**OpenStack 主要部署为 IaaS 模式。**

云计算模式的比较可以用一张图片概括（图片来源于[阿里云](https://www.alibabacloud.com/zh/knowledge/difference-between-iaas-paas-saas)）：

![云计算模式的比较](/images/openstack/difference.png)

更多资料可以参考 [SaaS vs PaaS vs IaaS: What’s The Difference & How To Choose](https://www.bmc.com/blogs/saas-vs-paas-vs-iaas-whats-the-difference-and-how-to-choose/)。

### AMQP

AMQP（Advanced Message Queuing Protocol，高级消息队列协议）是为 MOM（Message-oriented middleware，面向消息中间件）设计的开放标准应用层协议。

更多内容可以参考 [RabbitMQ（一）：RabbitMQ快速入门](https://www.cnblogs.com/sgh1023/p/11217017.html)。

### REST

REST（Representational State Transfer，表述性状态传递）是一种软件架构风格，可以降低开发的复杂性，提高系统的可伸缩性。

更多内容可以参考 [rest api 介绍](https://www.jianshu.com/p/75389ea9a90b)。

### RPC

RPC（Remote Procedure Call，远程过程调用）是相对于本地过程调用而言的，能像调用本地函数一样调用远程函数的协议。

更多内容可以参考[谁能用通俗的语言解释一下什么是 RPC 框架？ - 洪春涛的回答](https://www.zhihu.com/question/25536695/answer/221638079)。

### ETCD

ETCD 是一个高可用的 Key/Value 存储系统，主要用于分享配置和服务发现。

更多内容可以参考 [ETCD 简介 + 使用](https://blog.csdn.net/bbwangj/article/details/82584988)。

### Django

Django 是一个 Python Web 框架，采用了 MVT 的软件设计模式，即模型（Model），视图（View）和模板（Template），被 Horizon 组件所使用。

更多内容可以参考 [Django 教程](https://www.runoob.com/django/django-tutorial.html)。

## 个人理解

OpenStack 是由 Python 编写的一系列工具的集合，它是由若干个项目组成的。OpenStack 社区也是仅次于 Linux 的第二大开源社区。

OpenStack 的各个组件并不是直接操控计算机系统的，而是借助于各种系统工具、函数库来管理计算机系统。相当于在现有操作系统上再进行了一层封装，来实现对多个计算机系统的管理。

OpenStack 对内使用 AMQP 进行调用，对外则提供 REST API。
