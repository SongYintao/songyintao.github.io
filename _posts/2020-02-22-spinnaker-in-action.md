---
title: Spinnaker 持续发布系统
subtitle: spinnaker发布系统探索
layout: post
tags: [cd,kubernetes,spinnaker]
---

Spinnaker 持续发布系统探索研究。



https://www.spinnaker.io/concepts/



## 术语



### 1. 应用管理

![img](https://www.spinnaker.io/concepts/clusters.png)

- Server Group
- Cluster
- Application
- Load balancer
- FireWall







### 2. 应用发布



#### 1. 流水线

![img](https://www.spinnaker.io/concepts/pipelines.png)

You can start a pipeline manually, or you can configure it to be automatically triggered by an event, such as a Jenkins job completing, a new Docker image appearing in your registry, a CRON schedule, or a stage in another pipeline.

You can configure the pipeline to emit notifications, by email, Slack, or SMS, to interested parties at various points during pipeline execution (such as on pipeline start/complete/fail).



#### 2. Deployment strategies

![img](https://www.spinnaker.io/concepts/deployment-strategies.png)



# Netflex实践

## Netflex Cloud Model

### 1. 命名规则

Server groups are named according to a convention that helps organize them into *clusters*: 

*<name>-<stack>-<detail>-v<version>* 

- The *name* is the name of the application or service. 

- The (optional) *stack* is typically used to differentiate production, staging, and 

  test server groups. 

- The (optional) *detail* is used to differentiate special-purpose server groups. 

  For example, an application may run a Redis cache or a group of instances 

  dedicated to background work. 

- The *version* is simply a sequential version number. 









## Canary auto report and analysis



