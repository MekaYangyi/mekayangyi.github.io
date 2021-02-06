---
title: redis源码阅读-一个命令的旅程
date: 2017-07-18 15:28:23
categories: 
- 源码阅读
tags:
- redis
- 源码阅读
---

redis服务器会与多个客户端建立网络连接，处理客户端发送的命令请求，在数据库中保存客户端执行的命令，并通过资源管理来维持整个服务器自身的运转。

本文目的是为了理清楚服务器对于命令请求的整个处理过程，说明这个过程中服务器与客户端如何互相交互，服务器内部如何调用内部组件达到对命令的执行。

## 命令的处理流程

之前在事务代码部分的阅读，已经对于服务器接收客户端的连接并创建命令请求处理器等待命令请求的过程。

现在以一个简单的set命令为例子，看服务器在接收请求之后如何处理。

```c
redis> SET KEY VALUE
ok
```

从客户端发送set key value命令到接收回复ok，都做了如下操作：

- 客户端发送命令。
- 服务器命令请求处理器接收，引发命令的执行，并产生命令回复处理器。
- 命令回复处理器发送ok给客户端。
- 客户端接收ok，并打印。

<!--more-->
## 发送命令

客户端将键入的命令转换为协议格式并套接字发送到服务器

## 读取命令

服务器发现客户端连接套接字变为可读时，通过命令请求处理器readQueryFromClient()函数来进行处理。

### 读取命令到缓冲区并处理

```c
// 读取客户端的查询缓冲区内容
void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {
    redisClient *c = (redisClient*) privdata;
    int nread, readlen;
    size_t qblen;
    REDIS_NOTUSED(el);
    REDIS_NOTUSED(mask);

    server.current_client = c; // 设置服务器的当前客户端
    readlen = REDIS_IOBUF_LEN; // 读取的默认长度
    /* If this is a multi bulk request, and we are processing a bulk reply
     * that is large enough, try to maximize the probability that the query
     * buffer contains exactly the SDS string representing the object, even
     * at the risk of requiring more read(2) calls. This way the function
     * processMultiBulkBuffer() can avoid copying buffers to create the
     * Redis Object representing the argument. */
    if (c->reqtype == REDIS_REQ_MULTIBULK && c->multibulklen && c->bulklen != -1
        && c->bulklen >= REDIS_MBULK_BIG_ARG)
    {
        int remaining = (unsigned)(c->bulklen+2)-sdslen(c->querybuf);

        if (remaining < readlen) readlen = remaining;
    }

    // 获取缓存区遗留数据长度
    qblen = sdslen(c->querybuf);
    if (c->querybuf_peak < qblen) c->querybuf_peak = qblen; // 查询缓冲区长度峰值更新
    c->querybuf = sdsMakeRoomFor(c->querybuf, readlen); // 重新分配查询缓冲区空间
    nread = read(fd, c->querybuf+qblen, readlen); // 读取fd中数据，在遗留数据之后
    if (nread == -1) {
        // 读取错误处理
        if (errno == EAGAIN) {
            nread = 0;
        } else {
            redisLog(REDIS_VERBOSE, "Reading from client: %s",strerror(errno));
            freeClient(c);
            return;
        }
    } else if (nread == 0) {
        // 遇到EOF,关闭客户端
        redisLog(REDIS_VERBOSE, "Client closed connection");
        freeClient(c);
        return;
    }

    if (nread) {
        // 读取成功
        sdsIncrLen(c->querybuf,nread); // 正确更新 free 和 len 属性的。
        c->lastinteraction = server.unixtime; // 记录最后一次互动时间
        if (c->flags & REDIS_MASTER) c->reploff += nread; // 客户端为master,更新复制偏移量
        server.stat_net_input_bytes += nread;
    } else {
        // 在 nread == -1 且 errno == EAGAIN 时运行
        server.current_client = NULL;
        return;
    }

    // 缓冲区长度超过服务器最大缓冲区长度
    // 清空缓冲区并释放客户端
    if (sdslen(c->querybuf) > server.client_max_querybuf_len) {
        sds ci = catClientInfoString(sdsempty(),c), bytes = sdsempty();

        bytes = sdscatrepr(bytes,c->querybuf,64);
        redisLog(REDIS_WARNING,"Closing client that reached max query buffer length: %s (qbuf initial bytes: %s)", ci, bytes);
        sdsfree(ci);
        sdsfree(bytes);
        freeClient(c);
        return;
    }

    // 从缓冲区中读取内容，创建参数，并执行命令
    // 直到缓冲区所有的内容被处理完为止
    processInputBuffer(c);
    server.current_client = NULL;
}

```

### 从缓冲区中读取命令并处理

redis的readQueryFromClient()函数在将命令读取到缓冲区之后，调用processInputBuffer()对缓冲区内的命令进行处理，直到所有命令处理完毕。

```c
// 处理客户端输入的内容
void processInputBuffer(redisClient *c) {
    /* Keep processing while there is something in the input buffer */
    while(sdslen(c->querybuf)) {
        // 客户端处于暂停状态，直接返回
        if (!(c->flags & REDIS_SLAVE) && clientsArePaused()) return;

        // 客户端被阻塞直接返回
        if (c->flags & REDIS_BLOCKED) return;

        // 客户端被设置为关闭，返回
        if (c->flags & REDIS_CLOSE_AFTER_REPLY) return;

        // 判断请求类型
        if (!c->reqtype) {
            if (c->querybuf[0] == '*') {
                c->reqtype = REDIS_REQ_MULTIBULK;
            } else {
                c->reqtype = REDIS_REQ_INLINE;
            }
        }

        // 将缓冲区中的内容转换为命令，以及命令参数
        // processMultibulkBuffer()处理一般客户端发送的信息
      	// processInlineBuffer()处理TELNET发送的信息
        // 命令转换失败跳出循环，也就是可能在没有处理完缓冲区所有数据的情况下跳出。
        if (c->reqtype == REDIS_REQ_INLINE) {
            if (processInlineBuffer(c) != REDIS_OK) break;
        } else if (c->reqtype == REDIS_REQ_MULTIBULK) {
            if (processMultibulkBuffer(c) != REDIS_OK) break;
        } else {
            redisPanic("Unknown request type");
        }

        if (c->argc == 0) {
            // 命令参数为0，不需要执行
            resetClient(c);
        } else {
            // 执行命令，在成功执行之后重置客户端
            if (processCommand(c) == REDIS_OK)
                resetClient(c);
        }
    }
}

// 命令的转换介绍下processMultibulkBuffer()，此为处理客户端发送来命令，相对协议更复杂。
// processInlineBuffer()相对协议简单就不介绍了。
/*
 * 将 c->querybuf 中的协议内容转换成 c->argv 中的参数对象
 * 
 * 比如 *3\r\n$3\r\nSET\r\n$3\r\nMSG\r\n$5\r\nHELLO\r\n
 * 将被转换为：
 * argv[0] = SET
 * argv[1] = MSG
 * argv[2] = HELLO
 */
int processMultibulkBuffer(redisClient *c) {
    char *newline = NULL;
    int pos = 0, ok;
    long long ll;

    // 读取命令参数个数
    if (c->multibulklen == 0) {
        redisAssertWithInfo(c,NULL,c->argc == 0);

        // 校验命令参数中"\r\n"的存在
        newline = strchr(c->querybuf,'\r');
        if (newline == NULL) {
            if (sdslen(c->querybuf) > REDIS_INLINE_MAX_SIZE) {
                addReplyError(c,"Protocol error: too big mbulk count string");
                setProtocolError(c,0); // 异步关闭客户端
            }
            return REDIS_ERR;
        }

        if (newline-(c->querybuf) > ((signed)sdslen(c->querybuf)-2))
            return REDIS_ERR;

        // 第一个字符必须时*
        redisAssertWithInfo(c,NULL,c->querybuf[0] == '*');

        // 转换出参数的个数
        ok = string2ll(c->querybuf+1,newline-(c->querybuf+1),&ll);

        // 检测参数个数是否超限
        if (!ok || ll > 1024*1024) {
            addReplyError(c,"Protocol error: invalid multibulk length");
            setProtocolError(c,pos); // 异步关闭客户端
            return REDIS_ERR;
        }

        pos = (newline-c->querybuf)+2;
        if (ll <= 0) {
            // 参数小于等于0，删除c->querybuf中从pos到-1的内容
            // 返回读取成功
            sdsrange(c->querybuf,pos,-1);
            return REDIS_OK;
        }

        // 设置参数个数
        c->multibulklen = ll;

        // 分配参数空间
        if (c->argv) zfree(c->argv);
        c->argv = zmalloc(sizeof(robj*)*c->multibulklen);
    }

    redisAssertWithInfo(c,NULL,c->multibulklen > 0);
    while(c->multibulklen) {
        // 读取参数长度
        if (c->bulklen == -1) {

            // 校验命令参数中"\r\n"的存在
            newline = strchr(c->querybuf+pos,'\r');
            if (newline == NULL) {
                if (sdslen(c->querybuf) > REDIS_INLINE_MAX_SIZE) {
                    addReplyError(c,
                        "Protocol error: too big bulk count string");
                    setProtocolError(c,0);
                    return REDIS_ERR;
                }
                break;
            }

            if (newline-(c->querybuf) > ((signed)sdslen(c->querybuf)-2))
                break;

            // 确认格式
            if (c->querybuf[pos] != '$') {
                addReplyErrorFormat(c,
                    "Protocol error: expected '$', got '%c'",
                    c->querybuf[pos]);
                setProtocolError(c,pos);
                return REDIS_ERR;
            }

            // 读取长度
            ok = string2ll(c->querybuf+pos+1,newline-(c->querybuf+pos+1),&ll);
            if (!ok || ll < 0 || ll > 512*1024*1024) {
                addReplyError(c,"Protocol error: invalid bulk length");
                setProtocolError(c,pos);
                return REDIS_ERR;
            }
            pos += newline-(c->querybuf+pos)+2;
            
            // 参数太长的特殊处理
            if (ll >= REDIS_MBULK_BIG_ARG) {
                size_t qblen;

                /* If we are going to read a large object from network
                 * try to make it likely that it will start at c->querybuf
                 * boundary so that we can optimize object creation
                 * avoiding a large copy of data. */
                sdsrange(c->querybuf,pos,-1);
                pos = 0;
                qblen = sdslen(c->querybuf);
                /* Hint the sds library about the amount of bytes this string is
                 * going to contain. */
                if (qblen < (size_t)ll+2)
                    c->querybuf = sdsMakeRoomFor(c->querybuf,ll+2-qblen);
            }
            c->bulklen = ll;
        }

        // 读取参数
        if (sdslen(c->querybuf)-pos < (unsigned)(c->bulklen+2)) {
            // 确认协议内容
            break;
        } else {
            //复制参数
            if (pos == 0 &&
                c->bulklen >= REDIS_MBULK_BIG_ARG &&
                (signed) sdslen(c->querybuf) == c->bulklen+2)
            {
                c->argv[c->argc++] = createObject(REDIS_STRING,c->querybuf);
                sdsIncrLen(c->querybuf,-2); /* remove CRLF */
                c->querybuf = sdsempty();
                /* Assume that if we saw a fat argument we'll see another one
                 * likely... */
                c->querybuf = sdsMakeRoomFor(c->querybuf,c->bulklen+2);
                pos = 0;
            } else {
                c->argv[c->argc++] =
                    createStringObject(c->querybuf+pos,c->bulklen);
                pos += c->bulklen+2;
            }
            c->bulklen = -1;
            c->multibulklen--; // 继续读取
        }
    }

    // 清除已经读取的内容
    if (pos) sdsrange(c->querybuf,pos,-1);

    // 读取完毕返回
    if (c->multibulklen == 0) return REDIS_OK;

    // 可能内容不符合协议返回失败
    return REDIS_ERR;
}
```
## 命令的执行

processInputBuffer()在解析成功命令之后，调用processCommand()对命令进行执行。

```c
void processInputBuffer(redisClient *c) {
  		// ....
        if (c->argc == 0) {
            // 命令参数为0，不需要执行
            resetClient(c);
        } else {
            // 执行命令，在成功执行之后重置客户端
            if (processCommand(c) == REDIS_OK)
                resetClient(c);
        }
  		// ...
}
```

### 命令的查找

在processCommand()执行的第一步就是查询命令表，找到对于的命令实现信息。

```c
 // 命令的执行
int processCommand(redisClient *c) {
    // quit命令特殊处理，异步关闭服务器
    if (!strcasecmp(c->argv[0]->ptr,"quit")) {
        addReply(c,shared.ok);
        c->flags |= REDIS_CLOSE_AFTER_REPLY;
        return REDIS_ERR;
    }

    // 查找命令
    c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
    if (!c->cmd) {
        // 没有查找倒
        flagTransaction(c);
        addReplyErrorFormat(c,"unknown command '%s'",
            (char*)c->argv[0]->ptr);
        return REDIS_OK;
    } else if ((c->cmd->arity > 0 && c->cmd->arity != c->argc) ||
               (c->argc < -c->cmd->arity)) {
        // 命令实现与输入的参数数量不匹配
        flagTransaction(c);
        addReplyErrorFormat(c,"wrong number of arguments for '%s' command",
            c->cmd->name);
        return REDIS_OK;
    }
// ...
}
```

redis维护了一个命令表，该命令表为一个字典，键为命令名字，值是一个redisCommand结构，该结构记录了一个Redis的命令实现。

服务器启动时，调用初始化服务器配置函数initServerConfig()，该函数会进行命令表的初始化，保存在两个字典中commands、orig_commands。原始的命令表初始参数保存在redis.c文件中的redisCommandTable定义。

```c
struct redisServer {
 	// ...
    // 命令表（受到 rename 配置选项的作用）
    dict *commands;
    // 命令表（无 rename 配置选项的作用）
    dict *orig_commands;
  	// ...
};
int main(int argc, char **argv) {
  	// ...
    initServerConfig();
}

void initServerConfig() {
  	// ...
  	
    // 初始化命令表
  	// 创建命令字典
    server.commands = dictCreate(&commandTableDictType,NULL);
    server.orig_commands = dictCreate(&commandTableDictType,NULL);
    populateCommandTable(); // 初始化命令表
  
  	// 初始化常用命令快捷
    server.delCommand = lookupCommandByCString("del");
    server.multiCommand = lookupCommandByCString("multi");
    server.lpushCommand = lookupCommandByCString("lpush");
    server.lpopCommand = lookupCommandByCString("lpop");
    server.rpopCommand = lookupCommandByCString("rpop");
  	// ...
}

void populateCommandTable(void) {
    int j;

    // 命令数量
    int numcommands = sizeof(redisCommandTable)/sizeof(struct redisCommand);

    for (j = 0; j < numcommands; j++) {
        struct redisCommand *c = redisCommandTable+j; // 命令
        char *f = c->sflags;
        int retval1, retval2;

        while(*f != '\0') {
            switch(*f) {
            case 'w': c->flags |= REDIS_CMD_WRITE; break;
            case 'r': c->flags |= REDIS_CMD_READONLY; break;
            case 'm': c->flags |= REDIS_CMD_DENYOOM; break;
            case 'a': c->flags |= REDIS_CMD_ADMIN; break;
            case 'p': c->flags |= REDIS_CMD_PUBSUB; break;
            case 's': c->flags |= REDIS_CMD_NOSCRIPT; break;
            case 'R': c->flags |= REDIS_CMD_RANDOM; break;
            case 'S': c->flags |= REDIS_CMD_SORT_FOR_SCRIPT; break;
            case 'l': c->flags |= REDIS_CMD_LOADING; break;
            case 't': c->flags |= REDIS_CMD_STALE; break;
            case 'M': c->flags |= REDIS_CMD_SKIP_MONITOR; break;
            case 'k': c->flags |= REDIS_CMD_ASKING; break;
            case 'F': c->flags |= REDIS_CMD_FAST; break;
            default: redisPanic("Unsupported command flag"); break;
            }
            f++;
        }

        // 添加
        retval1 = dictAdd(server.commands, sdsnew(c->name), c);
        /* Populate an additional dictionary that will be unaffected
         * by rename-command statements in redis.conf. */
        retval2 = dictAdd(server.orig_commands, sdsnew(c->name), c);
        redisAssert(retval1 == DICT_OK && retval2 == DICT_OK);
    }
}
```

redisCommandTable定义

```c
// Redis 命令
struct redisCommand {
    char *name; // 命令名
    redisCommandProc *proc; // 实现函数
    int arity; // 参数个数
    char *sflags; // 字符串表示FLAG
    int flags;   // 实际FLAG
    /* Use a function to determine keys arguments in a command line.
     * Used for Redis Cluster redirect. */
    redisGetKeysProc *getkeys_proc;
    // 指定哪个参数为key
    int firstkey; /* The first argument that's a key (0 = no keys) */
    int lastkey;  /* The last argument that's a key */
    int keystep;  /* The step between first and last key */
    long long microseconds, calls; // 统计信息
};

struct redisCommand redisCommandTable[] = {
    {"get",getCommand,2,"r",0,NULL,1,1,1,0,0},
    {"set",setCommand,-3,"wm",0,NULL,1,1,1,0,0},
  	// ...
};
```

这样通过一个命令表，能够快捷的找到命令实现及相关参数。

### 命令的执行

找到命令之后即可执行命令。继续看processCommand()函数。在经过一系列特殊情况处理之后，开始执行命令。

```c
// 命令的执行
int processCommand(redisClient *c) {
  	// ...查找命令
  
    // 一系列特殊情况处理

    /* Exec the command */
    if (c->flags & REDIS_MULTI &&
        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
    {
        // 事务状态下的特殊处理
        queueMultiCommand(c);
        addReply(c,shared.queued);
    } else {

        // 执行命令
        call(c,REDIS_CALL_FULL);
        c->woff = server.master_repl_offset;

        // 处理那些解除阻塞的键
        if (listLength(server.ready_keys))
            handleClientsBlockedOnLists();
    }
    return REDIS_OK;
}

// 执行命令
void call(redisClient *c, int flags) {
    long long dirty, start, duration;
    int client_old_flags = c->flags;

    // 命令发送倒MONITOR
    if (listLength(server.monitors) &&
        !server.loading &&
        !(c->cmd->flags & (REDIS_CMD_SKIP_MONITOR|REDIS_CMD_ADMIN)))
    {
        replicationFeedMonitors(c,server.monitors,c->db->id,c->argv,c->argc);
    }

    // 执行命令
    c->flags &= ~(REDIS_FORCE_AOF|REDIS_FORCE_REPL);
    redisOpArrayInit(&server.also_propagate);
    dirty = server.dirty;
    start = ustime();
    c->cmd->proc(c); // 命令实现函数
    duration = ustime()-start; // 执行时间
    dirty = server.dirty-dirty; // 命令执行dirty的数量
    if (dirty < 0) dirty = 0;
	// ... 一系列附加操作
}
```

最终调用到setCommand(redisClient *c)函数。

```c
/* SET key value [NX] [XX] [EX <seconds>] [PX <milliseconds>] */
void setCommand(redisClient *c) {
    int j;
    robj *expire = NULL;
    int unit = UNIT_SECONDS;
    int flags = REDIS_SET_NO_FLAGS;

    // 设置选项参数
    for (j = 3; j < c->argc; j++) {
        char *a = c->argv[j]->ptr;
        robj *next = (j == c->argc-1) ? NULL : c->argv[j+1];

        if ((a[0] == 'n' || a[0] == 'N') &&
            (a[1] == 'x' || a[1] == 'X') && a[2] == '\0') {
            flags |= REDIS_SET_NX;
        } else if ((a[0] == 'x' || a[0] == 'X') &&
                   (a[1] == 'x' || a[1] == 'X') && a[2] == '\0') {
            flags |= REDIS_SET_XX;
        } else if ((a[0] == 'e' || a[0] == 'E') &&
                   (a[1] == 'x' || a[1] == 'X') && a[2] == '\0' && next) {
            unit = UNIT_SECONDS;
            expire = next;
            j++;
        } else if ((a[0] == 'p' || a[0] == 'P') &&
                   (a[1] == 'x' || a[1] == 'X') && a[2] == '\0' && next) {
            unit = UNIT_MILLISECONDS;
            expire = next;
            j++;
        } else {
            addReply(c,shared.syntaxerr);
            return;
        }
    }

    // 尝试编码转换
    c->argv[2] = tryObjectEncoding(c->argv[2]);

    // set命令通用的实现
    setGenericCommand(c,flags,c->argv[1],c->argv[2],expire,unit,NULL,NULL);
}

void setGenericCommand(redisClient *c, int flags, robj *key, robj *val, robj *expire, int unit, robj *ok_reply, robj *abort_reply) {
    long long milliseconds = 0; /* initialized to avoid any harmness warning */

    // 取出过期时间 expire为过期时间参数
    if (expire) {
        if (getLongLongFromObjectOrReply(c, expire, &milliseconds, NULL) != REDIS_OK)
            return;
        if (milliseconds <= 0) {
            addReplyErrorFormat(c,"invalid expire time in %s",c->cmd->name);
            return;
        }
        if (unit == UNIT_SECONDS) milliseconds *= 1000;
    }

    // 如果REDIS_SET_NX REDIS_SET_XX 判断是否符合规范
    if ((flags & REDIS_SET_NX && lookupKeyWrite(c->db,key) != NULL) ||
        (flags & REDIS_SET_XX && lookupKeyWrite(c->db,key) == NULL))
    {
        addReply(c, abort_reply ? abort_reply : shared.nullbulk);
        return;
    }

    // 设置键
    setKey(c->db,key,val);
    server.dirty++; // 脏计数增加
    if (expire) setExpire(c->db,key,mstime()+milliseconds); // 设置过期时间
    notifyKeyspaceEvent(REDIS_NOTIFY_STRING,"set",key,c->db->id); // 发送事件通知
    if (expire) notifyKeyspaceEvent(REDIS_NOTIFY_GENERIC,
        "expire",key,c->db->id);// 发送事件通知
    addReply(c, ok_reply ? ok_reply : shared.ok); // 回复
}
```

## 回复客户端

在执行命令出错或者成功后使用addReply()生成回复信息，该函数将通过prepareClientToWrite()产生回复客户端的文件事件，同时将回复内容复制到回复缓存区。

```c
void addReply(redisClient *c, robj *obj) {
    if (prepareClientToWrite(c) != REDIS_OK) return; // 生成回复客户端的文件事件

    // 根据不同情况，生成回复内容,写入不同的缓冲区
    if (sdsEncodedObject(obj)) {
        if (_addReplyToBuffer(c,obj->ptr,sdslen(obj->ptr)) != REDIS_OK)
            _addReplyObjectToList(c,obj);
    } else if (obj->encoding == REDIS_ENCODING_INT) {
        /* Optimization: if there is room in the static buffer for 32 bytes
         * (more than the max chars a 64 bit integer can take as string) we
         * avoid decoding the object and go for the lower level approach. */
        if (listLength(c->reply) == 0 && (sizeof(c->buf) - c->bufpos) >= 32) {
            char buf[32];
            int len;

            len = ll2string(buf,sizeof(buf),(long)obj->ptr);
            if (_addReplyToBuffer(c,buf,len) == REDIS_OK)
                return;
            /* else... continue with the normal code path, but should never
             * happen actually since we verified there is room. */
        }
        obj = getDecodedObject(obj);
        if (_addReplyToBuffer(c,obj->ptr,sdslen(obj->ptr)) != REDIS_OK)
            _addReplyObjectToList(c,obj);
        decrRefCount(obj);
    } else {
        redisPanic("Wrong obj->encoding in addReply()");
    }
}
int prepareClientToWrite(redisClient *c) {
    // lua 脚本伪客户端
    if (c->flags & REDIS_LUA_CLIENT) return REDIS_OK;

    // 客户端为REDIS_MASTER REDIS_MASTER_FORCE_REPLY
    if ((c->flags & REDIS_MASTER) &&
        !(c->flags & REDIS_MASTER_FORCE_REPLY)) return REDIS_ERR;

    if (c->fd <= 0) return REDIS_ERR; /* Fake client for AOF loading. */

    // 一般的客户端生成写事件
    if (c->bufpos == 0 && listLength(c->reply) == 0 &&
        (c->replstate == REDIS_REPL_NONE ||
         (c->replstate == REDIS_REPL_ONLINE && !c->repl_put_online_on_ack)))
    {
        /* Try to install the write handler. */
        if (aeCreateFileEvent(server.el, c->fd, AE_WRITABLE,
                sendReplyToClient, c) == AE_ERR)
        {
            freeClientAsync(c);
            return REDIS_ERR;
        }
    }

    /* Authorize the caller to queue in the output buffer of this client. */
    return REDIS_OK;
}
```

最终文件事件调用sendReplyToClient()回复客户端。

```c
void sendReplyToClient(aeEventLoop *el, int fd, void *privdata, int mask) {
    redisClient *c = privdata;
    int nwritten = 0, totwritten = 0, objlen;
    size_t objmem;
    robj *o;
    REDIS_NOTUSED(el);
    REDIS_NOTUSED(mask);

    // 从缓存区获取数据，写入，直到写完
    while(c->bufpos > 0 || listLength(c->reply)) {
        if (c->bufpos > 0) {

            // 先写入回复缓冲区数据
            nwritten = write(fd,c->buf+c->sentlen,c->bufpos-c->sentlen);
            if (nwritten <= 0) break; // 出错跳出

            // 成功计数
            c->sentlen += nwritten;
            totwritten += nwritten;

            // 如果内容写完则清空两个计数器
            if (c->sentlen == c->bufpos) {
                c->bufpos = 0;
                c->sentlen = 0;
            }
        } else {

            // 回复缓冲区为空的话，在回复链表查找
            o = listNodeValue(listFirst(c->reply));
            objlen = sdslen(o->ptr);
            objmem = getStringObjectSdsUsedMemory(o);

            // 跳过空对象
            if (objlen == 0) {
                listDelNode(c->reply,listFirst(c->reply));
                c->reply_bytes -= objmem;
                continue;
            }

            // 写入
            nwritten = write(fd, ((char*)o->ptr)+c->sentlen,objlen-c->sentlen);
            if (nwritten <= 0) break; // 出错跳出

            // 计数
            c->sentlen += nwritten;
            totwritten += nwritten;

            // 如果汉冲去内容写入完毕，删除已经写入的节点
            if (c->sentlen == objlen) {
                listDelNode(c->reply,listFirst(c->reply));
                c->sentlen = 0;
                c->reply_bytes -= objmem;
            }
        }
        // 写入量超过限制 在最大内存没设或者最大内存没使用完的情况下跳出
        server.stat_net_output_bytes += totwritten;
        if (totwritten > REDIS_MAX_WRITE_PER_EVENT &&
            (server.maxmemory == 0 ||
             zmalloc_used_memory() < server.maxmemory)) break;
    }

    // 写入出错
    if (nwritten == -1) {
        if (errno == EAGAIN) {
            nwritten = 0;
        } else {
            redisLog(REDIS_VERBOSE,
                "Error writing to client: %s", strerror(errno));
            freeClient(c);
            return;
        }
    }
    if (totwritten > 0) {
        /* For clients representing masters we don't count sending data
         * as an interaction, since we always send REPLCONF ACK commands
         * that take some time to just fill the socket output buffer.
         * We just rely on data / pings received for timeout detection. */
        if (!(c->flags & REDIS_MASTER)) c->lastinteraction = server.unixtime;
    }

    if (c->bufpos == 0 && listLength(c->reply) == 0) {
        c->sentlen = 0;

        // 写完了删除write handler
        aeDeleteFileEvent(server.el,c->fd,AE_WRITABLE);

        // 必要的话关闭客户端
        if (c->flags & REDIS_CLOSE_AFTER_REPLY) freeClient(c);
    }
}
```

## 结束

至此一个命令的旅程就结束了。

