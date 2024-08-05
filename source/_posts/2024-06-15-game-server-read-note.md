---
title: 大型游戏服务端读书笔记
date: 2024-06-15 17:00:00
categories: 游戏服务器
tags: [开发经验]
description: 百万在线：大型游戏服务端开发 读书笔记
---

## 第一章 引入

### 服务器承载量

从`CPU负载`、`内存占用`、`网络流量`等多个角度考量，也需要考虑不同游戏类型的`逻辑复杂度`。

可以做个假设，在MMORPG中，玩家平均每3秒操作一次（走路，购物），服务器平均处理一条消息花费2毫秒。从CPU角度来看，如果同一时刻只处理一个客户端请求，服务器每秒可以处理500条消息，最高承载1500人。按经验推算，最高1000多人在线的游戏，DAU大概是三五千，这种承载量对于多数独立游戏和小型手游是足够的。

上面是仅计算了CPU负载，实际上"走路"这类`广播消息`受网络影响很大，不同类型游戏对服务器`响应速度`有不同要求。如果不做进一步优化，广播的消息量与在线玩家呈`指数增长`关系，通常这类`单线程程序`只能支持几十名玩家进行广播。

### 强弱交互

* 同一场景角色交互很强，比如有"走路"消息，可以在`同一进程`处理。

* 不同场景角色交互较弱，只有聊天、好友、公会这些功能需要交互，可以将同一个服的玩家放在`一台机器`上处理，进程间通信会比同一进程共享数据慢数百倍。

* 不同服务器的玩家交互很少，可以放到`不同机器`上。

### 一致性问题

分布式程序要处理`断线重连`、断线期间的`消息重发`，以及 断线后进程间`状态不一致`的问题。

在游戏业务中，开发者一般会把一致性问题抛给`具体业务`去处理。

### 分布式与单点

#### 难以分割的业务

实现分布式程序的前提是`游戏逻辑能够分割`。

如果游戏规则复杂，各个功能紧密相连，则不容易找到分割的方案，部分功能依然要靠单点的性能支撑，那么`单点`（单个进程、单个线程）的运算能力依然会限制服务端的承载量。

#### 延迟和承载的权衡

多个程序协作意味着`消息延迟`，某些功能对`消息即时性`要求很高，比如帧同步。

### Actor模型

`合理分割功能`是分布式模型一大难点，Actor模型既能符合游戏逻辑的表达，又能让计算机高效执行。

Actor模型中，每个Actor`相互隔离`，只通过`消息通信`，具有天然的`并发性`。理念是万物皆Actor，它是更进一步的面向对象，即把世间万物都当作Actor对象。Actor可以代表一个角色、一只动物，也可以代表整个游戏场景。

每个Actor都会包含`自身状态`（Items，HP），以及一个`信箱`（消息队列），Actor通过给其他Actor寄信来实现通信。至于收到信件后的反应，取决于`收信Actor`。

## 第二章 Skynet入门

### 一些API

目录结构，Service即Actor，放在cservice和service；lib库放在luaclib和lualib。

``` lua 基础API
-- 启动一个新服务
newservice(name, ...) 
-- 用func初始化服务
start(func) 
-- 为type类型的消息设定处理函数func
-- Skynet支持多种消息类型，Lua服务间的消息类型是"lua"
-- func形式 function(session, source, cmd, ...) session为消息的唯一ID，source为发送消息的服务地址，cmd代表消息名
dispatch(type, func) 
-- 向地址为addr的服务发送一条type类型的消息，消息名为cmd
send(addr, type, cmd, ...)
-- send对应的阻塞方法，要等待对方回应
call(addr, type, cmd, ...)
```

``` lua 网络API
-- 返回 listenfd
socket.listen(host, port)
-- 为listenfd设置新客户端连接时的回调方法connect
socket.start(listenfd, connect)
-- 回调方法connect获得新连接后，不会立即接收它的数据，需要再次调用socket.start(fd)开始接受
-- 回调方法connect的完整写法
function connect(fd, addr)
    socket.start(fd)
    ...
end
-- read write close
socket.read(fd)
socket.write(fd, data)
socket.close(fd)
```

``` lua 数据库API
-- 连接数据库
mysql.connect(args)
-- 执行sql语句
-- "insert into msgs (text) values (\'\hello')"
db.query(sql)
```

``` lua 集群API
-- 本节点（重新）加载节点配置
-- {node1 = "127.0.0.1:7001", node2 = "127.0.0.1:7002"}
cluster.reload(cfg)
-- 启动本地节点
cluster.open(node)
-- 跨节点推送消息
cluster.send(node, address, cmd, ...)
cluster.call(node, address, cmd, ...)
-- 为远程节点上的服务（address）创建一个本地代理服务，可以直接用skynet.send和call操作本地代理
cluster.proxy(node, address)
```

### 协程同步问题

服务当前协程挂起时，还可以接受并处理其他消息，如果多个协程改到同一份数据，就会有`同步问题`。

解决方案，加多一个`state标识`和一个`协程列表`，操作执行时，将state置doing，其他协程判断state=doing时就将自己加到协程列表，然后skynet.wait。在操作执行完后，重置state，然后遍历协程列表依次skynet.wakeup(co)，最后将协程列表置空。

## 第三章 案例 球球大作战

### 封装易用的API

Skynet的API提供了偏底层的功能，不方便使用，通过`snax框架`给出了一套更简单的API。

本节在service模块封装了`更简洁的API`，service模块是对Skynet服务的一种封装，还封装了`重复调用`的方法。

``` lua GitHub的service.lua
local M = {
    -- 服务类型 服务ID
    name = "", -- gateway
    id = 0, -- 1
    -- 回调函数
    exit = nil,
    init = nil,
    -- dispatch方法
    resp = {}
}
```

### 分布式登录流程



## 代码地址

https://github.com/luopeiyu/million_game_server