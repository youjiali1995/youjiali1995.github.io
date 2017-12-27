---
title: Redis源码阅读(十一) -- replication
excerpt: 介绍 Redis 主从复制相关实现。
layout: post
categories: Redis
---

{% include toc %}

## slaveof
配置`redis.conf`中`slaveof <masterip> <masterport>`或使用`slaveof`命令，会进行设置:
```c
server.masterhost = sdsnew(argv[1]);
server.masterport = atoi(argv[2]);
server.repl_state = REPL_STATE_CONNECT;
```

之后会进入`serverCron()`中每秒调用一次的`replicationCron()`与`master`建立连接:
```c
/* Check if we should connect to a MASTER */
if (server.repl_state == REPL_STATE_CONNECT) {
    serverLog(LL_NOTICE,"Connecting to MASTER %s:%d",
        server.masterhost, server.masterport);
    if (connectWithMaster() == C_OK) {
        serverLog(LL_NOTICE,"MASTER <-> SLAVE sync started");
    }
}
```

`connectWithMaster()`使用非阻塞套接字建立连接，并注册可读可写`syncWithMaster()`文件事件，更新`repl_state = REPL_STATE_CONNECTING`。

`connect()`默认的行为是等到接收到对端的`SYN + ACK`再返回，如果连接不能立即建立，会阻塞一段时间，直到出错或成功建立连接。使用非阻塞`connect()`会在不能立即建立连接的情况下，立即返回`EINPROGRESS`错误。
确定连接是否成功建立需要使用`select()`或`poll()`或`epoll()`，监控套接字返回可写条件，不过因为连接出错会同时返回可读和可写，
所以在可写时，要检查`SO_ERROR`：`getsockopt(fd, SOL_SOCKET, SO_ERROR, &error, &len)`，当建立成功时，`error`为0；建立失败，`error`为相应的错误，需要关闭套接字，重新创建再连接，不能再次调用`connect()`会返回`EADDRINUSE`。
当阻塞的`connect()`被中断时，也需要使用上面的方法:
```c
/* syncWithMaster() */

/* Check for errors in the socket: after a non blocking connect() we
    * may find that the socket is in error state. */
if (getsockopt(fd, SOL_SOCKET, SO_ERROR, &sockerr, &errlen) == -1)
    sockerr = errno;
if (sockerr) {
    serverLog(LL_WARNING,"Error condition on socket for SYNC: %s",
        strerror(sockerr));
    goto error;
}
```

连接成功建立后，会删除可写事件`syncWithMaster()`, `master`和`slave`会有下面几个交互:
1. `slave`发送`PING`
2. 如果配置`auth`，`slave`发送`AUTH`
3. `slave`发送`REPLCONF listening-port <port>`;
4. 如果使用`docker`等，可能会设置`slave-announce-ip`，`slave`发送`REPLCONF ip-address <ip>`
5. 发送`REPLCONF capca eof capa psync2`，主要是为了兼容不同版本，`master`根据这些对`slave`采取不同的操作
6. 开始`PSYNC`

## PSYNC
`Redis`使用`Replication ID, offset`唯一标识一个数据集:
* `Replication ID`: 在`Redis`初始化时调用`changeReplicationId`随机生成的长度为`40`的字符串`server.replid`，用于确定`Redis`实例
* `offset`: 已经同步的命令的`offset`

`syncWithMaster()`调用`slaveTryPartialResynchronization()`会发送`PSYNC`与`master`同步:
  * `slave`如果有`server.cached_master`，发送`PSYNC server.cached_master->replid server.cached_master->reploff+1`
  * 没有，发送`PSYNC ? -1`

`master`收到后，会在`syncCommand()`中调用`masterTryPartialResynchronization()`判断是否可以`partial resync`:
```c
/* Is the replication ID of this master the same advertised by the wannabe
 * slave via PSYNC? If the replication ID changed this master has a
 * different replication history, and there is no way to continue.
 *
 * Note that there are two potentially valid replication IDs: the ID1
 * and the ID2. The ID2 however is only valid up to a specific offset. */
if (strcasecmp(master_replid, server.replid) &&
    (strcasecmp(master_replid, server.replid2) ||
        psync_offset > server.second_replid_offset))
{
    /* ... */
    goto need_full_resync;
}

/* We still have the data our slave is asking for? */
if (!server.repl_backlog ||
    psync_offset < server.repl_backlog_off ||
    psync_offset > (server.repl_backlog_off + server.repl_backlog_histlen))
{
    /* ... */
    goto need_full_resync;
}
```
只有`slave`发送的`master_replid`和`master`的`replid`相同，且`repl_off`在`master`的`repl_backlog`有效数据范围内，才会进行`paritial resync`。

只有到`PSYNC`阶段才会标记`client`为`slave`:
* `Full sync`:
```c
    c->replstate = SLAVE_STATE_WAIT_BGSAVE_START;
    if (server.repl_disable_tcp_nodelay)
        anetDisableTcpNoDelay(NULL, c->fd); /* Non critical if it fails. */
    c->repldbfd = -1;
    c->flags |= CLIENT_SLAVE;
    listAddNodeTail(server.slaves,c);
```
* `Partial resync`:
```c
    c->flags |= CLIENT_SLAVE;
    c->replstate = SLAVE_STATE_ONLINE;
    c->repl_ack_time = server.unixtime;
    c->repl_put_online_on_ack = 0;
    listAddNodeTail(server.slaves,c);
```

### Full Sync
确定`Full Sync`后会尝试创建`rdb`，有这几种情况:
1. 如果有`disk`类型`rdb`并且有其他的`slave`处于`SLAVE_STATE_WAIT_BGSAVE_END`状态，会复制该`slave`的`output buffer`，等待同一个`rdb`完成，发送`+FULLRESYNC server.replid slave->psync_initial_offset`。
没有`slave`处于该状态会在`replicationCron()`等待触发`rdb`
2. 如果有`socket`类型`rdb`，等待完成，在`replicationCron()`中再触发`rdb`
3. 当前没有`rdb`:
  * `socket`: 等待`repl-diskless-sync-delay`秒，为了一次性`sync`多个`slave`
  * `disk`: 调用`startBgSaveForReplication()`开始`rdb`

#### disk
`master`调用`startBgSaveForReplication()`同时处理`disk`和`socket`类型的`rdb`，`disk`类型会调用`replicationSetupSlaveForResync()`设置
1. `slave->psync_initial_offset = server.master_repl_offset`
2. `server->replstate = SLAVE_STATE_WAIT_BGSAVE_END`
3. 发送`+FULLRESYNC server.replid server.master_repl_offset`

`slave`收到`FULLRESYNC replid offset`后:
1. 设置`server.master_replid = replid`
2. 设置`server.master_initial_offset = offset`
3. 创建`temp rdb`文件，注册可读事件`readSyncBulkPayload()`代替`syncWithMaster()`
4. 更新`server.repl_state = REPL_STATE_TRANSFER`

当`master` `rdb`完成后，会更新每个等待`rdb`完成的`slave`状态为`SLAVE_STATE_SEND_BULK`，注册写事件`sendBulkToSlave()`:
```c
/* serverCron()->backgroundSaveDoneHandler()->backgroundSaveDoneHandlerDisk()->updateSlavesWaitingBgsave() */

    if ((slave->repldbfd = open(server.rdb_filename,O_RDONLY)) == -1 ||
        redis_fstat(slave->repldbfd,&buf) == -1) {
        freeClient(slave);
        serverLog(LL_WARNING,"SYNC failed. Can't open/stat DB after BGSAVE: %s", strerror(errno));
        continue;
    }
    slave->repldboff = 0;
    slave->repldbsize = buf.st_size;
    slave->replstate = SLAVE_STATE_SEND_BULK;
    slave->replpreamble = sdscatprintf(sdsempty(),"$%lld\r\n",
        (unsigned long long) slave->repldbsize);

    aeDeleteFileEvent(server.el,slave->fd,AE_WRITABLE);
    if (aeCreateFileEvent(server.el, slave->fd, AE_WRITABLE, sendBulkToSlave, slave) == AE_ERR) {
        freeClient(slave);
        continue;
    }
```

`master`写事件`sendBulkToSlave()`会首先发送`slave->replpreamble: rdb文件的大小`，为了防止阻塞，然后每次发送最多`PROTO_IOBUF_LEN`的数据，当全部发送完毕后，调用`putSlaveOnline()`，
更新`slave->replstate = SLAVE_STATE_ONLINE`，注册写事件`sendReplyToClient()`。

`slave`读事件`readSyncBulkPayload()`可以处理`dist`和`socket`两种类型的`Full sync`。和`sendBulkToSlave()`对应着，首先读取`replpreamble`，保存在`server.repl_transfer_size`，然后每次
最多读取`4096 bytes`数据并写入`server.repl_transfer_fd`，当全部接收完会作如下操作:
1. `emptyDb(-1...)`: 清空所有`db`
2. 删除读事件
3. `rename(server.repl_transfer_tempfile, server.rdb_filename)`, 调用`rdbLoad()`
4. 调用`replicationCreateMasterClient()`创建`server.master`
5. 更新状态`sever.repl_state = REPL_STATE_CONNECTED`
6. 设置`server.replid`为`master`的`replid`
7. 清空`replid2`
8. 创建`repl_backlog`

此时，`Full sync`结束。

#### socket
`disk`类型需要先创建`rdb`文件，然后每次发送的时候都需要`lseek()`、`read()`，然后再发送给`slave`，当网卡速度比磁盘速度快的时候可以设置`repl-diskless-sync yes`使用`socket`类型`full sync`。
`socket`类型的`rdb`无法像`disk`一样服用`rdb`，所以为了服务多个`slave`，会等待`repl-diskless-sync-delay`秒:
```c
    /* replicationCron() */
    listRewind(server.slaves,&li);
    while((ln = listNext(&li))) {
        client *slave = ln->value;
        if (slave->replstate == SLAVE_STATE_WAIT_BGSAVE_START) {
            idle = server.unixtime - slave->lastinteraction;
            if (idle > max_idle) max_idle = idle;
            slaves_waiting++;
            mincapa = (mincapa == -1) ? slave->slave_capa :
                                        (mincapa & slave->slave_capa);
        }
    }

    if (slaves_waiting &&
        (!server.repl_diskless_sync ||
            max_idle > server.repl_diskless_sync_delay))
    {
        /* Start the BGSAVE. The called function may start a
            * BGSAVE with socket target or disk target depending on the
            * configuration and slaves capabilities. */
        startBgsaveForReplication(mincapa);
    }
```

和普通`rdb`过程不一样的是，`rio`不再以文件初始化，而是以`socket`初始化，每次调用`rioWrite()->rioFdsetWrite()`会给每个`socket`发送，只要有一个成功就算成功。
为了编程简单，每个`socket`会设为阻塞，并设置超时时间：
```c
/* Put the socket in blocking mode to simplify RDB transfer.
    * We'll restore it when the children returns (since duped socket
    * will share the O_NONBLOCK attribute with the parent). */
anetBlock(NULL,slave->fd);
anetSendTimeout(NULL,slave->fd,server.repl_timeout*1000);
```

`SO_SNDTIMEO`选项之前不知道，记录一下。当发送超时时，会返回`EWOULDBLOCK`:
```c
/* With blocking sockets, which is the sole user of this
 * rio target, EWOULDBLOCK is returned only because of
 * the SO_SNDTIMEO socket option, so we translate the error
 * into one more recognizable by the user. */

/* Set the socket send timeout (SO_SNDTIMEO socket option) to the specified
 * number of milliseconds, or disable it if the 'ms' argument is zero. */
int anetSendTimeout(char *err, int fd, long long ms) {
    struct timeval tv;

    tv.tv_sec = ms/1000;
    tv.tv_usec = (ms%1000)*1000;
    if (setsockopt(fd, SOL_SOCKET, SO_SNDTIMEO, &tv, sizeof(tv)) == -1) {
        anetSetError(err, "setsockopt SO_SNDTIMEO: %s", strerror(errno));
        return ANET_ERR;
    }
    return ANET_OK;
}
```

`rdb`格式也发生了变化，不像文件一样可以知道大小，增加了前缀和后缀用于判断结束:
* `prefix`: `$EOF:<40 bytes unguessable hex string>\r\n`
* `suffix`: `<40 bytes unguessable hex string>`

`slave`流程和`disk`一样。而`rdb`子进程结束也就意味着`socket`类型`full sync`结束，在`backgroundSaveDoneHandlerSocket`中会通过`pipe`判断所有`slave`的状态，然后恢复`slave`设置或关闭连接，设置`slave`状态:
```c
/* Note: we wait for a REPLCONF ACK message from slave in
 * order to really put it online (install the write handler
 * so that the accumulated data can be transfered). However
 * we change the replication state ASAP, since our slave
 * is technically online now. */
slave->replstate = SLAVE_STATE_ONLINE;
slave->repl_put_online_on_ack = 1;
slave->repl_ack_time = server.unixtime; /* Timeout otherwise. */
```

### Partial Resync
`Partial Resync`只需发送`slave`缺少的命令。`master`做如下操作:
1. 设置`slave`状态:
```c
    c->flags |= CLIENT_SLAVE;
    c->replstate = SLAVE_STATE_ONLINE;
    c->repl_ack_time = server.unixtime;
    c->repl_put_online_on_ack = 0;
    listAddNodeTail(server.slaves,c);
```
2. 发送`+CONTINUE server.replid`
3. 发送`repl_backlog`中部分命令

`slave`收到`+CONTINUE`后会更新相关状态，并注册事件:
1. `replid2 = cached_master.replid`，即前一个`master`的`id`
2. `replid`更新为当前`master`的`id`
3. 创建`server.master`，注册读事件`readQueryFromClient()`，所以当`master`发送增量写命令时，会像普通`client`一样对待

### sync过程中的write
在`sync`过程中，`master`会收到写命令，需要同步给`slave`。在`full sync`和`partial sync`中行为有些不同，在开始`sync`时，`master`都会将
`slave`追加到`server.slaves`，但是状态不同:
* `full sync`: `SLAVE_STATE_WAIT_BGSAVE_*`
* `partial resync`: `SLAVE_STATE_ONLINE`

`master`收到写操作会调用`propagate()->replicationFeedSlaves()`将命令发送给每个`slave`，真正写入`buffer`和普通`client`一样，会首先调用`prepareClientToWrite()`:
```c
/* addReply()->prepareClientToWrite() */

/* Schedule the client to write the output buffers to the socket only
    * if not already done (there were no pending writes already and the client
    * was yet not flagged), and, for slaves, if the slave can actually
    * receive writes at this stage. */
if (!clientHasPendingReplies(c) &&
    !(c->flags & CLIENT_PENDING_WRITE) &&
    (c->replstate == REPL_STATE_NONE ||
        (c->replstate == SLAVE_STATE_ONLINE && !c->repl_put_online_on_ack)))
{
    /* Here instead of installing the write handler, we just flag the
        * client and put it into a list of clients that have something
        * to write to the socket. This way before re-entering the event
        * loop, we can try to directly write to the client sockets avoiding
        * a system call. We'll only really install the write handler if
        * we'll not be able to write the whole reply at once. */
    c->flags |= CLIENT_PENDING_WRITE;
    listAddNodeHead(server.clients_pending_write,c);
}
```
可以看到只有处于`ONLINE`状态的才会真正发送，其余状态的只会写入`buffer`。所以`full sync`需要主动将累计在`buffer`的命令发送给`slave`，否则一直没有写命令的话，`slave`数据会一直滞后:
* `disk`: 在全部发送完毕后，调用`putSaveOnline()`， 更新`slave->replstate = SLAVE_STATE_ONLINE`，注册写事件`sendReplyToClient()`，将`buffer`中数据发送给`slave`
* `socket`: 更新`slave->replstate = SLAVE_STATE_ONLINE`，设置`slave->repl_put_online_on_ack = 1`(`disk`为`0`, `paritial`也为`0`)。`slave`会在`replicationCron()`中发送`REPLCONF ACK reploff`给`master`,
`master`会在这时调用`putSlaveOnline()`。为什么要多一步呢？没看出来原因。

当`sync`完成后(不管是`full sync`还是`partial resync`)，此时`master`对于`slave`来说会像普通`client`一样，读取写命令，保持`dataset`的一致(**最终一致性**)。

## 增量数据

简单来说，类似`aof`的机制，`master`收到写操作会调用`propagate()->replicationFeedSlaves()`将命令写入到`server.repl_backlog`中，
并且发送给每个`slave`进行同步。
`server.repl_backlog`是一个**环形缓冲区**，大小通过`repl-backlog-size`配置，默认为`1mb`:
```c
/* Add data to the replication backlog.
 * This function also increments the global replication offset stored at
 * server.master_repl_offset, because there is no case where we want to feed
 * the backlog without incrementing the offset. */
void feedReplicationBacklog(void *ptr, size_t len) {
    unsigned char *p = ptr;

    server.master_repl_offset += len;

    /* This is a circular buffer, so write as much data we can at every
     * iteration and rewind the "idx" index if we reach the limit. */
    while(len) {
        size_t thislen = server.repl_backlog_size - server.repl_backlog_idx;
        if (thislen > len) thislen = len;
        memcpy(server.repl_backlog+server.repl_backlog_idx,p,thislen);
        server.repl_backlog_idx += thislen;
        if (server.repl_backlog_idx == server.repl_backlog_size)
            server.repl_backlog_idx = 0;
        len -= thislen;
        p += thislen;
        server.repl_backlog_histlen += thislen;
    }
    if (server.repl_backlog_histlen > server.repl_backlog_size)
        server.repl_backlog_histlen = server.repl_backlog_size;
    /* Set the offset of the first byte we have in the backlog. */
    server.repl_backlog_off = server.master_repl_offset -
                              server.repl_backlog_histlen + 1;
}
```
从上面写入`server.repl_backlog`的代码可以看出:
* `server.master_repl_offset`: 写入到`repl_backlog`的全部字节数
* `server.repl_backlog_idx`: 下一个字节写入的位置
* `server.repl_backlog_histlen`: 缓冲区中有效数据长度
* `server.repl_backlog_off`: 有效数据的起始位置

`slave`收到`master`发送的命令，会在`readyQueryFromClient()`判断是否是`master`:
1. 更新`c->reploff`
2. 调用`replicationFeedSlavesFromMasterStream()`:
  * 调用`feedReplicationBacklog()`: 更新`server.master_repl_offset`
  * 作为代理，将`master`发送过来的数据发送给自己的`slave`(级联`slave`)

## replicationCron()

在`serverCron()`中每秒调用一次`replicationCron()`处理建立连接和一些周期性任务:
* 超时: 连接超时、`rdb`传输超时、`master` `ping`超时
* `server.repl_state == REPL_STATE_CONNECT`时，建立连接。连接使用非阻塞`connect`，并注册可读可写文件事件`syncWithMaster()`，设置`server.repl_state = REPL_STATE_CONNECTING`
* `slave`发送`REPLCONF ACK`。`master`用于更新`slave->repl_ack_off`，用在`Wait`命令中，实现同步复制
* `master`定期(`repl-ping-slave-period`)发送`ping`，`slave`可以根据`server.master.lastinteraction`判断`master`超时
* 处理超时`slave`
* 根据`repl-backlog-ttl`，释放`repl_back`
* 创建`rdb`

当出现超时时，会调用`freeClient()`释放连接，`master`和`slave`有不同的操作:
* `master`: 关闭连接，`slave`会读0，同时调用`freeClient()`释放`master`
* `slave`: 在释放`master`之前，会调用`replicationCacheMaster()`将`master`状态保存在`server.cached_master`，然后调用`replicationHandleMasterDisconnection()`更新状态为`REPL_STATE_CONNECT`，
会在之后尝试重连，并发送`server.cached_master`状态尝试`paritial resync`

可以看到，在连接已经成功建立后，`master`不会再对`slave`的`reploff`进行判断是否要进行`full sync`。`Redis`依赖`tcp`的“可靠性”，认为要么超时、要么报错，否则数据一定会到达`slave`。

## 重启
`rdbPopulateSaveInfo()`在下面几种情况下会返回非`NULL`:
* 是`master`
* 是`slave`
* 有`cached_master`

在`rdbSaveInfoAuxFields()`会写入`rsi`相关信息，以便重启后能进行`parital resync`:
```c
    if (rsi) {
        if (rdbSaveAuxFieldStrInt(rdb,"repl-stream-db",rsi->repl_stream_db)
            == -1) return -1;
        if (rdbSaveAuxFieldStrStr(rdb,"repl-id",server.replid)
            == -1) return -1;
        if (rdbSaveAuxFieldStrInt(rdb,"repl-offset",server.master_repl_offset)
            == -1) return -1;
    }
```

启动时，如果配置了`slaveof`，会设置`server`状态:
```c
    server.masterhost = sdsnew(argv[1]);
    server.masterport = atoi(argv[2]);
    server.repl_state = REPL_STATE_CONNECT;
```

在`loadDataFromDisk()`中会读取`rdb`的`rsi`恢复`replication`信息，同时以当前信息创建`cached_master`，在后续的`syncWithMaster()`会发送`cached_master`信息，尝试`resync`:
```c
rdbSaveInfo rsi = RDB_SAVE_INFO_INIT;
if (rdbLoad(server.rdb_filename,&rsi) == C_OK) {
    serverLog(LL_NOTICE,"DB loaded from disk: %.3f seconds",
        (float)(ustime()-start)/1000000);

    /* Restore the replication ID / offset from the RDB file. */
    if (server.masterhost &&
        rsi.repl_id_is_set &&
        rsi.repl_offset != -1 &&
        /* Note that older implementations may save a repl_stream_db
            * of -1 inside the RDB file in a wrong way, see more information
            * in function rdbPopulateSaveInfo. */
        rsi.repl_stream_db != -1)
    {
        memcpy(server.replid,rsi.repl_id,sizeof(server.replid));
        server.master_repl_offset = rsi.repl_offset;
        /* If we are a slave, create a cached master from this
            * information, in order to allow partial resynchronizations
            * with masters. */
        replicationCacheMasterUsingMyself();
        selectDb(server.cached_master,rsi.repl_stream_db);
    }
}
```

## failover
在`Redis4.0`之前，进行主从切换`failover`需要全量同步，耗费资源，而且大多数情况数据集都保持一直，没有必要重新同步。`Redis4.0`之后解决了这个问题。
假设`A`为`master`，`B`为`slave`，进行主从切换:
1. 发送`slaveof no one`给`B`:`B`会调用`shiftReplicationId()`:
```c
    memcpy(server.replid2,server.replid,sizeof(server.replid));
    server.second_replid_offset = server.master_repl_offset+1;
    changeReplicationId();
```
所以`server.replid2`和`server.second_replid_offset`保存了与之前`master`的同步进度，同时`server.repl_backlog`还保存着之前`master`同步过来的数据，用于之后的`parital resyn`。
2. 发送`slaveof B.ip B.port`给`A`: `A`会调用`replicationSetMaster()->replicationCacheMasterUsingMyself()`以自己状态创建`cached_master`，用于之后的同步:
```c
server.cached_master.reploff = server.master_repl_offset;
server.cached_master.repliid = server.master_replid;
```

再回顾一下`slave`发送`PSYNC`和`master`判断`Partial resync`的代码:
* `slave`:
```c
if (server.cached_master) {
    psync_replid = server.cached_master->replid;
    snprintf(psync_offset,sizeof(psync_offset),"%lld", server.cached_master->reploff+1);
    serverLog(LL_NOTICE,"Trying a partial resynchronization (request %s:%s).", psync_replid, psync_offset);
}
```
* `master`:
```c
    if (strcasecmp(master_replid, server.replid) &&
        (strcasecmp(master_replid, server.replid2) ||
         psync_offset > server.second_replid_offset))
    {
        /* ... */
        goto need_full_resync;
    }

    /* We still have the data our slave is asking for? */
    if (!server.repl_backlog ||
        psync_offset < server.repl_backlog_off ||
        psync_offset > (server.repl_backlog_off + server.repl_backlog_histlen))
    {
        /* ... */
        goto need_full_resync;
    }
```

可以看到`A`以自己的状态发送给`B`尝试重连，而`B`同样会根据之前`A`的状态，判断是否需要`full sync`，当然如果在这期间有很多写操作，更新了`B`的`repl_backlog`，还是会`full sync`。

## expire
`slave`不会主动进行`expire`，在调用`activeExpireCycle()`前会先判断是否存在`master`，存在则不进行。同样`expireIfNeeded()`只会返回逻辑上的过期，不会删`key`:
```c
    /* If we are running in the context of a slave, return ASAP:
     * the slave key expiration is controlled by the master that will
     * send us synthesized DEL operations for expired keys.
     *
     * Still we try to return the right information to the caller,
     * that is, 0 if we think the key should be still valid, 1 if
     * we think the key is expired at this time. */
    if (server.masterhost != NULL) return now > when;
```

`master`会调用`propagateExpire()`将相应的`del`操作发送给`slave`来达到一致，因为`del`操作同样需要查找`key`，所以`slave`对`master`的`client`做特殊对待:
```c
robj *lookupKeyReadWithFlags(redisDb *db, robj *key, int flags) {
    robj *val;

    if (expireIfNeeded(db,key) == 1) {
        /* Key expired. If we are in the context of a master, expireIfNeeded()
         * returns 0 only when the key does not exist at all, so it's safe
         * to return NULL ASAP. */
        if (server.masterhost == NULL) return NULL;

        /* However if we are in the context of a slave, expireIfNeeded() will
         * not really try to expire the key, it only returns information
         * about the "logical" status of the key: key expiring is up to the
         * master in order to have a consistent view of master's data set.
         *
         * However, if the command caller is not the master, and as additional
         * safety measure, the command invoked is a read-only command, we can
         * safely return NULL here, and provide a more consistent behavior
         * to clients accessign expired values in a read-only fashion, that
         * will say the key as non exisitng.
         *
         * Notably this covers GETs when slaves are used to scale reads. */
        if (server.current_client &&
            server.current_client != server.master &&
            server.current_client->cmd &&
            server.current_client->cmd->flags & CMD_READONLY)
        {
            return NULL;
        }
    }
    val = lookupKey(db,key,flags);
    if (val == NULL)
        server.stat_keyspace_misses++;
    else
        server.stat_keyspace_hits++;
    return val;
}
```

## evict
`slave`还是会进行`evict`的操作，同时`master`也会`propagate()`给`slave`它`evict`的`key`。这会有风险:
* 当`slave`和`master` `maxmemory`相同，且`slave`为`readonly`时：所有的写操作都是由`master`同步过来，所以`slave`不会`evcit`，数据集保持一致
* 当`slave` `maxmemory`比`master`小时，会造成数据集不一致

当然没有理由设置不同的`maxmemory`。

## readonly

`slave`默认为`readonly`，读写分离能提升`qps`:
```c
    /* Don't accept write commands if this is a read only slave. But
     * accept write commands if this is our master. */
    if (server.masterhost && server.repl_slave_ro &&
        !(c->flags & CLIENT_MASTER) &&
        c->cmd->flags & CMD_WRITE)
    {
        addReply(c, shared.roslaveerr);
        return C_OK;
    }
```
`slave`也可以配置为可写，可以用来操作比较耗时的操作，此时`master`和`slave`数据集可能会不一致，重启或重连后再次同步。
在`4.0`版本之前，可写`slave`无法设置`key`过期，因为`slave`只会等待`master`传递过期，这会导致`key` `leak`，不会获取到但是还存在。`Redis4.0`会将`slave`设置的`key`保存在
`dict *slaveKeyWithExpire`中，单独进行过期。

