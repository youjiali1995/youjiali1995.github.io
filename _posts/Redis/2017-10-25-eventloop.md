---
title: Redis源码阅读(一) -- EventLoop
excerpt: 介绍Redis提供服务的主要流程——事件循环。
layout: post
categories: Redis
---

{% include toc %}

看到过很多文章都是以Redis的数据结构为入口进行分析，我个人认为还是先知道大体的流程比较好。所以首先写一些关于Redis的主流程相关的，
主要是`EventLoop`。

## 启动流程
`redis-server`的入口函数`main`在`server.c`中，主要做了下面工作(具体看源码):
  1. 初始化配置为默认值: `initServerConfig()`
  2. 解析命令行参数
  3. 加载配置文件: `loadServerConfig(configfile, options)`
  4. 初始化`server`: `initServer()`
  5. 启动事件循环: `aeMain(server.el)`

### server初始化
在`server.c`中定义了一个全局变量`struct redisServer server`保存了`Redis`运行时的状态和相关配置。`server`初始化流程如下:
  1. `initServerConfig()`: 使用常量默认值初始化`server`
  2. 解析`option`: 解析命令行参数中格式`--[name] value`为`name value\n`，生成`options`
  3. `loadServerConfig(configfile, options)`: 加载配置文件，并`append``options`生成`config`，解析`config`设置`server`
  4. `initServer()`: 初始化需要动态设置的部分:
        * 设置信号处理函数
        * 创建`eventloop`: `aeCreateEventLoop(server.maxclients + CONFIG_FDSET_INCR)`
        * 初始化`db`
        * 添加事件

## EventLoop
`Redis`使用`I/O`多路复用来处理两类事件：
  * `TimeEvent`
  * `FileEvent`
在`redisServer`结构体中包含`aeEventLoop *el`这一成员，结构如下:

```c
/* State of an event based program */
typedef struct aeEventLoop {
    int maxfd;   /* highest file descriptor currently registered */
    int setsize; /* max number of file descriptors tracked */
    long long timeEventNextId;
    time_t lastTime;     /* Used to detect system clock skew */
    aeFileEvent *events; /* Registered events */
    aeFiredEvent *fired; /* Fired events */
    aeTimeEvent *timeEventHead;
    int stop;
    void *apidata; /* This is used for polling API specific data */
    aeBeforeSleepProc *beforesleep;
    aeBeforeSleepProc *aftersleep;
} aeEventLoop;
```
`server.el`使用`AeEventLoop *aeCreateEventLoop(int setsize)`创建，需要关注的是`aeApiCreate`函数，`Redis`为了实现跨平台，
封装了多个平台下相应的系统调用供`EventLoop`使用，见`ae.c`和`config.h`。生产环境中最常用的还是`Linux`，所以主要关注`epoll`。
`ae_epoll.c`中有如下实现：
```c
typedef struct aeApiState {
    int epfd;
    struct epoll_event *events;
} aeApiState;
```
`aeApiState`中封装了`epoll`所需要的结构，`aeApiPoll()`会调用对应的底层事件循环。

### TimeEvent
时间事件在`aeEventLoop`中以链表保存，`aeCreateTimeEvent()`会将新创建的时间事件添加在链表头：
```c
/* Time event structure */
typedef struct aeTimeEvent {
    long long id; /* time event identifier. */
    long when_sec; /* seconds */
    long when_ms; /* milliseconds */
    aeTimeProc *timeProc;
    aeEventFinalizerProc *finalizerProc;
    void *clientData;
    struct aeTimeEvent *next;
} aeTimeEvent;
```
在`initServer()`中添加了几个事件，其中包含一个时间事件——`serverCron`。
`serverCron`以每秒`server.hz`次数执行，主要做了以下工作：  
  * 信息统计
  * 客户端连接的处理：`clientsCron()`
  * 数据库的调整：`databasesCron()`
  * 持久化相关工作
  * 还有一些更复杂的工作，如复制和集群相关工作

### FileEvent
文件时间处理与套接字相关的工作，在`aeEventLoop`中以数组的形式保存，`redis.conf`中可以设置`maxClients`，根据这个配置分配数组的大小，对应`fd`的时间就保存
在`events[fd]`中，因为`fd`会按照从小到大分配。结构如下：  
```c
/* File event structure */
typedef struct aeFileEvent {
    int mask; /* one of AE_(READABLE|WRITABLE) */
    aeFileProc *rfileProc;
    aeFileProc *wfileProc;
    void *clientData;
} aeFileEvent;

typedef struct aeFiredEvent {
    int fd;
    int mask;
} aeFiredEvent;
```
`aeFileEvent`、`aeFiredEvent`和`aeApiState`三个结构实现了通用的事件循环机制：
  1. 创建事件会添加到`server.el.events`和`server.el.apidata`中
  2. 底层调用依赖对应的`aeApiState`，并将触发事件的套接字还有相应事件保存在`server.el.fired`
  3. 处理事件时通过`server.el.fired`，在`server.el.events`中找到对应的`handler`执行

在`initServer()`中创建了多个监听套接字，并创建了`FileEvent`用于接收客户端连接。

### 事件处理
在初始化完成后，`Redis`就一直在`aeMain()`中处理事件循环：  
```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
```
`aeProcessEvents()`为处理事件的函数，流程如下：
  1. 查找最近的一个`TimeEvent`，因为以链表形式保存，耗时O(n)
  2. 以最近的`TimeEvent`时间间隔为参数调用`aeApiPoll()`，保证能及时处理
  3. 处理`FileEvents`
  4. 处理`TimeEvents`: `processTimeEvents()`会处理所有到时的事件，并重新添加到链表头
