---
title: Docker源码剖析（一）api-server
subtitle: api server解析
layout: post
tags: [docker]
---

根据上一篇「容器概述」里面，dockcer整体的架构，我们逐一解析相关组件的核心逻辑。今天，我们要搞的是Docker的api也就是，docker server提供对外的rest api接口，其中定义了相关的访问接口（比如，容器查询、exec、inspect等操作）。其底层实现是，调用daemon组件，该组件实现了在api中定义好了的backend接口。

好了，我们来瞅瞅吧。

首先，我们来简单的回顾一下HttpServer的原理，以及HttpServer在Docker server上面是如何使用实现的。

## 一、HttpServer 



### Backend接口的定义





## 二、 Daemon中的实现