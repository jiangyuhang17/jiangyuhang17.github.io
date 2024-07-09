---
title: Google Cloud k8s 课程2 GKE架构：基础
date: 2024-06-24 23:00:00
categories: 云计算
tags: [云计算]
description: Google Cloud Skills Boost - DevSecOps Learning Path
---

## 虚拟机 VS 容器

* 虚拟机，针对硬件做虚拟化，隔离性强，安全性高

* 容器，针对OS做虚拟化，运行效率高；用于`打包`（便携性）和`运行`（高效利用资源，共享内核）代码

## K8S VS 微服务

K8S是容器的编排框架，在生产环境中大规模运行容器化应用

微服务，每个服务只做一件事，则需要成千上万的容器构成整个应用，依赖于K8S去编排容器。要考虑服务之间网络多跳，对延迟的影响。

## 云计算

云计算涉及以服务形式提供资源，有*五个特征*：按需自助（获得资源）、任意网络访问、资源池、快速弹性伸缩、按量计费。

### GCP计算资源

1. Compute Engine，云虚拟机，供用户自己最大限度管理服务

2. GKE，Google Kubernetes Engine，是K8S的托管式服务，供用户运行容器化应用，可以自己管理容器化应用

3. App Engine，由GCP完全管理的服务，用户只需要关注应用代码

4. Cloud Run，是App Engine的改进版，可以处理相同的工作负载，但提供了更大的灵活性

5. Cloud Functions，函数计算，用于部署事件驱动型函数代码，用户只需为代码真正运行时付费。注意，纯`冷启动`可能导致响应超时。

3-5都属于托管式`无服务平台`（`Serverless`），用户无需考虑管理服务器

### 存储资源

不同云存储区别在于`IO特征`

* `块存储`，以块为存储单位，块大小可为512B

* `文件存储`，也是以块为存储单位，特征是`可共享`

* `对象存储`，以对象（可以是文件）为存储单位

### GCP服务的网络拓扑

* Multi-region : Americas / Europe / Asia Pacific
* Region : europe-west2 (London)
* Zone : europe-west2-a europe-west2-b europe-west2-c

Global: HTTP(S) LB / VPC
Regional: Regional GKE cluster / Datastore
Zonal: Persistent Disk / GKE node / Compute Engine instance

Region以内，网络延迟小于1毫秒

### 资源层次结构

Zones和Regions物理上组织GCP资源；Projects逻辑上组织GCP资源

* Organzation: root node，是使用Folders的前提（但不是使用Projects的前提），允许设置权限 和 资源隔离。a fixed organization ID and a changeable display name。

* Folders: 反应企业组织层次，应用不同权限。可以嵌套。

* Projects: a unique project ID 和 project number，保持不变；也可以命名Project，添加labels用来过滤，这些可以改变。

IAM（identity and access management）支持管理所有你使用GCP资源的访问控制。访问控制是向下生效的，也可以通过deny policies来无视角色设置确切的访问控制。

### 账单

账单是按Project收费的，账单账号可以link一个或多个Projects。可以通过subaccounts划分project账单，一些用户使用subaccounts把GCP服务再次卖给他们自己的客户。

#### 控制账单费用

* Budgets and alerts，可配置Webhook自动化响应

* Billing export，指定BigQuery分析出详细的账单数据

* Reports，控制台可视工具，监控花费

`quotas`，阻止 `error`或`恶意攻击` 导致的资源过度消费，应用在Project层。

* rate quotas，每隔一段时间重制。例如 GKE API 3000 requests for each project per min

* allocation quotas，管理Project中资源的数量，可能要考虑及时释放资源。例如，each GCP no more than 15 VPC networks 

GCP也会给所有用户固定一些quotas,减少不可预见的使用spike（尖峰）的风险。

### GCP控制台 / Cloud Shell

## 地址

https://www.cloudskillsboost.google/paths/76
