# redis源码阅读-事务



redis的事务提供了一种将单个命令请求打包，然后一次性、按照顺序执行多个命令的机制，这种方式服务器会一次性把命令执行完，中间不会执行其他客户端的命令。不过redis的命令不支持错误命令执行后的回滚机制，也就是命令设计者要对命令的正确性负责，即使多个命令中存在部分错误的命令，剩余命令也会继续执行下去。

## 主要命令

|   命令    |            功能             |
| :-----: | :-----------------------: |
|  MULTI  |         开始一个新的事务          |
| DISCARD |          放弃执行事务           |
|  EXEC   |        执行事务中的所有命令         |
|  WATCH  | 监视key，如果在exec之前被修改，则不执行事务 |
| UNWATCH |         取消对所有键的监视         |

## 事务的实现

一个事务分为三个阶段：

- 事务开始
- 命令入队
- 事务执行

### 事务开始

使用multi开启事务，redis主要使用redisClient中flag成员记录状态。

```c

void multiCommand(redisClient *c) {

    // 已经开启事务
    if (c->flags & REDIS_MULTI) {
        addReplyError(c,"MULTI calls can not be nested");
        return;
    }

    // 标记事务开启
    c->flags |= REDIS_MULTI;
    addReply(c,shared.ok);
}
```
<!--more-->
### 事务入队

当客户端进入事务状态时，客户端不会立即执行命令EXEC、DISCARD、WATCH、MULTI之外的命令，这些命令先进入事务队列，在之后事务执行时候执行。

```c
// 客户端结构体
struct redisClient {
  	// ....
  	multiState mstate; // 事务状态
    // ....
}

// 事务状态
typedef struct multiState {
    multiCmd *commands; // 事务队列
    int count; // 命令计数
    int minreplicas; // 用于同步复制
    time_t minreplicas_timeout; // 超时时间
} multiState;

// 事务命令
typedef struct multiCmd {
    robj **argv; // 参数
    int argc; // 参数数量 
    struct redisCommand *cmd; // 命令指针
} multiCmd;
```

redis在执行客户端命令时，会判断事务是否开启，如果开启且不是上面提到的几个命令，那么就会将命令压入队列,在redis的命令处理函数processCommand()中。

```c

int processCommand(redisClient *c) {
  	// ...
    /* Exec the command */
    if (c->flags & REDIS_MULTI &&
        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
    {
        // 处在事务状态
        // 不是execCommand、discardCommand、multiCommand、watchCommand
        // 执行入队操作
        queueMultiCommand(c);
        addReply(c,shared.queued);
    } else {
        // 命令直接执行
        call(c,REDIS_CALL_FULL);
        c->woff = server.master_repl_offset;
        if (listLength(server.ready_keys))
            handleClientsBlockedOnLists();
    }
  	// ...
}
```

入队的实现:入队功能依靠queueMultiCommand()实现

```c
void queueMultiCommand(redisClient *c) {
    multiCmd *mc;
    int j;

    // 重新分配足够的空间
    c->mstate.commands = zrealloc(c->mstate.commands,
            sizeof(multiCmd)*(c->mstate.count+1));
    mc = c->mstate.commands+c->mstate.count; // 压入点

    // 初始化事务结构体
    mc->cmd = c->cmd; 
    mc->argc = c->argc;
    mc->argv = zmalloc(sizeof(robj*)*c->argc);
    memcpy(mc->argv,c->argv,sizeof(robj*)*c->argc);
    for (j = 0; j < c->argc; j++)
        incrRefCount(mc->argv[j]);
    c->mstate.count++; // 计数+1
}
```

### 事务的执行

当处于事务状态的客户端向服务器发送EXEC命令时，这个命令被立即执行，具体见processCommand()函数。

最终命令调用execCommand()执行exec命令。

```c
void execCommand(redisClient *c) {
    int j;
    robj **orig_argv;
    int orig_argc;
    struct redisCommand *orig_cmd;
    int must_propagate = 0; /* Need to propagate MULTI/EXEC to AOF / slaves? */

    // 判断是否执行事务中
    if (!(c->flags & REDIS_MULTI)) {
        addReplyError(c,"EXEC without MULTI");
        return;
    }

    // 判断监视键是否被修改
    // 命令在入队时发送错误
    // 均不执行命令
    // 取消事务
    if (c->flags & (REDIS_DIRTY_CAS|REDIS_DIRTY_EXEC)) {
        addReply(c, c->flags & REDIS_DIRTY_EXEC ? shared.execaborterr :
                                                  shared.nullmultibulk);
        discardTransaction(c); // 取消事务
        goto handle_monitor;
    }

    // 取消对键的监视
    unwatchAllKeys(c); /* Unwatch ASAP otherwise we'll waste CPU cycles */

    // 备份
    orig_argv = c->argv;
    orig_argc = c->argc;
    orig_cmd = c->cmd;
    addReplyMultiBulkLen(c,c->mstate.count);

    // 遍历事务中的命令，执行
    for (j = 0; j < c->mstate.count; j++) {

        // 备份
        c->argc = c->mstate.commands[j].argc;
        c->argv = c->mstate.commands[j].argv;
        c->cmd = c->mstate.commands[j].cmd;

        // 在事务中，发现了写命令，传播multi
        if (!must_propagate && !(c->cmd->flags & REDIS_CMD_READONLY)) {
            execCommandPropagateMulti(c);
            must_propagate = 1;
        }

        // 执行命令
        call(c,REDIS_CALL_FULL);

        // 恢复
        c->mstate.commands[j].argc = c->argc;
        c->mstate.commands[j].argv = c->argv;
        c->mstate.commands[j].cmd = c->cmd;
    }

    // 恢复
    c->argv = orig_argv;
    c->argc = orig_argc;
    c->cmd = orig_cmd;
    discardTransaction(c); // 关闭事务状态
    /* Make sure the EXEC command will be propagated as well if MULTI
     * was already propagated. */
    if (must_propagate) server.dirty++;

handle_monitor:
    /* Send EXEC to clients waiting data from MONITOR. We do it here
     * since the natural order of commands execution is actually:
     * MUTLI, EXEC, ... commands inside transaction ...
     * Instead EXEC is flagged as REDIS_CMD_SKIP_MONITOR in the command
     * table, and we do it here with correct ordering. */
    if (listLength(server.monitors) && !server.loading)
        replicationFeedMonitors(c,server.monitors,c->db->id,c->argv,c->argc);
}
```

### 事务取消

DISCARD函数取消客户端的事务状态

```c
// discardCommand命令处理函数
void discardCommand(redisClient *c) {

    // 没有在事务状态
    if (!(c->flags & REDIS_MULTI)) {
        addReplyError(c,"DISCARD without MULTI");
        return;
    }

    // 取消事务状态
    discardTransaction(c);
    addReply(c,shared.ok);
}

// 取消事务状态
void discardTransaction(redisClient *c) {
    freeClientMultiState(c); // 释放事务
    initClientMultiState(c); // 初始化事务

    // 取消事务状态
    c->flags &= ~(REDIS_MULTI|REDIS_DIRTY_CAS|REDIS_DIRTY_EXEC);
    unwatchAllKeys(c); // 取消键的监视
}

void freeClientMultiState(redisClient *c) {
    int j;

    for (j = 0; j < c->mstate.count; j++) {
        int i;
        multiCmd *mc = c->mstate.commands+j;

        for (i = 0; i < mc->argc; i++)
            decrRefCount(mc->argv[i]);
        zfree(mc->argv);
    }
    zfree(c->mstate.commands);
}

void initClientMultiState(redisClient *c) {
    c->mstate.commands = NULL;
    c->mstate.count = 0;
}
```

### WATCH命令的实现

WATCH命令用来监视键是否被修改。

```c
typedef struct redisDb {
	// ...
    dict *watched_keys; // 监视键的字典，字典的键为数据库的键，值为链表，保存所有监视的客户端
    // ...
} redisDb;

typedef struct redisClient {
    // ...
    list *watched_keys;  // 保存该客户端所有被监视的键,保存watchedKey结构
    // ...
}

typedef struct watchedKey {
    robj *key;  // 保存键
    redisDb *db;  // 保存键所在的数据库
} watchedKey;
```

#### WATCH的触发

所有的对数据库进行修改的命令，比如set、del等，在执行之后都会调用signalModifiedKey(redisDb *db, robj *key)，而该函数调用touchWatchedKey(redisDb *db, robj *key)。touchWatchedKey(redisDb *db, robj *key)查找监视字典，对被修改的键进行标记。

```c
void signalModifiedKey(redisDb *db, robj *key) {
    touchWatchedKey(db,key);
}

void touchWatchedKey(redisDb *db, robj *key) {
    list *clients;
    listIter li;
    listNode *ln;

    // 字典为空
    if (dictSize(db->watched_keys) == 0) return;

    // 获取进行监视的客户端
    clients = dictFetchValue(db->watched_keys, key);
    if (!clients) return;

    // 遍历所有客户端，进行标记
    listRewind(clients,&li);
    while((ln = listNext(&li))) {
        redisClient *c = listNodeValue(ln);

        c->flags |= REDIS_DIRTY_CAS;
    }
}
```

#### 监视的开启

监视的开启就是在字典里添加键

```c
void watchCommand(redisClient *c) {
    int j;

    if (c->flags & REDIS_MULTI) {
        addReplyError(c,"WATCH inside MULTI is not allowed");
        return;
    }
    for (j = 1; j < c->argc; j++)
        watchForKey(c,c->argv[j]);
    addReply(c,shared.ok);
}

// 客户端C监视键Key
void watchForKey(redisClient *c, robj *key) {
    list *clients = NULL;
    listIter li;
    listNode *ln;
    watchedKey *wk;

    // 判断是否已经被监视了
    // 发现则直接返回
    listRewind(c->watched_keys,&li);
    while((ln = listNext(&li))) {
        wk = listNodeValue(ln);
        if (wk->db == c->db && equalStringObjects(key,wk->key))
            return; /* Key already watched */
    }
    
    // 检查key是否存在数据库的watched_keys字典力
    clients = dictFetchValue(c->db->watched_keys,key);
    if (!clients) {

        // 不存在就增加一个链表
        clients = listCreate();
        dictAdd(c->db->watched_keys,key,clients);
        incrRefCount(key);
    }

    // 在链表末尾增加key
    // 前面已经保证没有被监视过，所以这里不需要再判断，直接插入到末尾
    listAddNodeTail(clients,c);
    /* Add the new key to the list of keys watched by this client */
    wk = zmalloc(sizeof(*wk));
    wk->key = key;
    wk->db = c->db;
    incrRefCount(key);
    listAddNodeTail(c->watched_keys,wk);
}
```

#### 监视的关闭

监视的关闭即将字典中键删除。

```c

void unwatchCommand(redisClient *c) {
    // 取消客户端对所有键的监视
    unwatchAllKeys(c);
    // 重置状态
    c->flags &= (~REDIS_DIRTY_CAS);
    addReply(c,shared.ok);
}

// 清除所有监视
void unwatchAllKeys(redisClient *c) {
    listIter li;
    listNode *ln;

    // 没有键被监视，直接返回
    if (listLength(c->watched_keys) == 0) return;

    // 遍历被监视的键
    listRewind(c->watched_keys,&li);
    while((ln = listNext(&li))) {
        list *clients;
        watchedKey *wk;

        wk = listNodeValue(ln); // 键
        clients = dictFetchValue(wk->db->watched_keys, wk->key); // 数据库中查找
        redisAssertWithInfo(c,NULL,clients != NULL);
        listDelNode(clients,listSearchKey(clients,c)); // 删除数据库中监视节点
        
        // 如果链表为空，删除键
        if (listLength(clients) == 0)
            dictDelete(wk->db->watched_keys, wk->key);
        
        // 删除客户端监视的节点key
        listDelNode(c->watched_keys,ln);
        decrRefCount(wk->key);
        zfree(wk);
    }
}
```


