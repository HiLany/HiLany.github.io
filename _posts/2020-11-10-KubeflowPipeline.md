---
layout:     post
title:      Kubeflow-Pipeline
subtitle:   pipeline概念、架构、提交流程
date:       2020-11-10
author:     LANY
catalog: true
tags:
    - kubernetes
    - Kubeflow
---


# kubeflow-pipeline是什么？

## kubeflow pipelines平台组成

- 一个用来跟踪和管理job以及运行的web ui界面
- 一个用来调度多个ml工作流程步骤的引擎
- 用来定义和操作pipeline以及components的sdk
- 用sdk来与系统交互的notebook

## kubeflow pipeline的目标

- 端到端的编排方式：开启并简化机器学习pipelines的编排方式
- 简单的实现：可以很容易的尝试各种想法和技术并管理各种训练模型
- 简单的重用：允许重复使用components和pipeline去快速创建端到端的解决方式且不用每次重新构建。

## 什么是pipeline？

pipeline是对机器学习工作流的一种描述。包含了工作流中的所有组件以及如何以图的形式结合起来。管道中包含了运行pipeline需要定义的输出参数以及每个组件的输入输出。

当发布了一个pipeline之后，可以从kubeflow pipeliens ui中上传或者分享该pipeline。

一个pipeline是一组独立的用户代码集合，打包成镜像。

# pipeline架构

pipeline执行过程如下：

- Python SDK

通过kubeflow pipelines的DSL创建一个组件或者明确一个pipeline。

- DSL compiler

DSL compiler会转换定义pipeline的python code到一个静态配置(YAML)

- Pipeline Service

调用管道服务来创建从静态配置运行的管道。

- kubenertes resource

pipeline service调用k8s api服务器去创建用来运行pipeline所必须的k8s资源（kubernetes resources-CRDs）

- Orchestration controllers

一组用来编排运行pipeline所需要执行的容器的控制器。这些容器运行在k8s的pod中。

- Artiface storage

pods存储两种类型的数据：

1.Metadata(元数据): 实验、任务、运行的管道还有单梯度的指标。pipeline将这些元数据存放在mysql数据库中。

2.Artifacts(归档文件)：pipeline的包、视图以及大规模指标。大规模指标可以用来debug运行中的pipeline或者调查单个运行的pipelien的性能。pipeline存放这些归档文件到`minio server`或者云存储上。

- Persistence agent and ML metadata

管道持久化代理监控了pipeline service创建的k8s资源并持久化了在ml metadata service服务中资源的状态。管道持久化代理还记录了执行容器的输入以及输出。这些输入输出是由容器的参数以及数据归档后的url组成。

- Pipeline web server

web server会从各种各样的服务收集数据并展现出相关的视图，比如正在运行的pipelines列表，pipeline的执行历史，数据的归档文件列表，每个pipeline运行时的debbugging信息以及执行状态。

其整个pipeline架构以及执行流程如下：

![00b0d959520a54f0538c5b0316e47c9c.png](https://raw.githubusercontent.com/HiLany/HiLany.github.io/master/img/post-2020-1110-1.png)
