---
title: redis源码阅读-RDB持久化
date: 2017-07-12 16:04:59
categories: 
- 源码阅读
tags:
- redis
- 源码阅读
---

Redis是一个内存数据库，数据存储在内存之中。有一个问题就是如果数据不存储到硬盘，那么在服务器进程退出之后，服务器中所有的数据库数据就会丢失。

Redis为了解决这个问题，提供了持久化功能，目前有两种一种是RDB持久化，一种是AOF持久化。

RDB持久化是生成一个RDB文件，该文件是一个经过压缩的二进制文件，通过该文件可以还原数据库的状态。

## RDB文件的保存命令

Redis有两个命令可以生成RDB文件一个是SAVE，一个是BGSAVE。
SAVE命令调用saveCommand进行处理

```c
void saveCommand(redisClient *c) {

    // 判断是否正在执行BGSAVE，是则退出
    if (server.rdb_child_pid != -1) {
        addReplyError(c,"Background save already in progress");
        return;
    }

    //调用rdbSave生成RDB文件
    if (rdbSave(server.rdb_filename) == REDIS_OK) {
        addReply(c,shared.ok);
    } else {
        addReply(c,shared.err);
    }
}
```

BGSAVE调用bgsaveCommand进行处理

```c
void bgsaveCommand(redisClient *c) {

    // 判断是否已经在执行BGSAVE
    if (server.rdb_child_pid != -1) {
        addReplyError(c,"Background save already in progress");
    } else if (server.aof_child_pid != -1) {
        
    // 判断是否在执行BGREWRIEAOF
        addReplyError(c,"Can't BGSAVE while AOF log rewriting is in progress");
    
    // 执行rdbSaveBackground 生成RDB文件
    } else if (rdbSaveBackground(server.rdb_filename) == REDIS_OK) {
        addReplyStatus(c,"Background saving started");
    } else {
        addReply(c,shared.err);
    }
}

int rdbSaveBackground(char *filename) {
    pid_t childpid;
    long long start;

    // 如果BGSAVE正在执行直接返回
    if (server.rdb_child_pid != -1) return REDIS_ERR;

    // 获取dirty数据 执行时间
    server.dirty_before_bgsave = server.dirty;
    server.lastbgsave_try = time(NULL);

    // fork() 开始前时间
    start = ustime();

    // 调用fork，克隆该进程
    if ((childpid = fork()) == 0) {
        int retval;

        /* Child */
        closeListeningSockets(0);
        redisSetProcTitle("redis-rdb-bgsave");

        // 执行保存操作
        retval = rdbSave(filename);

        // 打印 copy-on-write 时使用的内存数
        if (retval == REDIS_OK) {
            size_t private_dirty = zmalloc_get_private_dirty();

            if (private_dirty) {
                redisLog(REDIS_NOTICE,
                    "RDB: %zu MB of memory used by copy-on-write",
                    private_dirty/(1024*1024));
            }
        }
        // 向父进程发送信号
        exitFromChild((retval == REDIS_OK) ? 0 : 1);
    } else {
        /* Parent */

        // 计算 fork() 执行的时间
        server.stat_fork_time = ustime()-start;
        server.stat_fork_rate = (double) zmalloc_used_memory() * 1000000 / server.stat_fork_time / (1024*1024*1024); /* GB per second. */
        latencyAddSampleIfNeeded("fork",server.stat_fork_time/1000);
        
        // 执行fork()错误信息
        if (childpid == -1) {
            server.lastbgsave_status = REDIS_ERR;
            redisLog(REDIS_WARNING,"Can't save in background: fork: %s",
                strerror(errno));
            return REDIS_ERR;
        }
        redisLog(REDIS_NOTICE,"Background saving started by pid %d",childpid);
        
        // 记录数据库开始BGSAVE时间
        server.rdb_save_time_start = time(NULL);

        // 子进程ID 类型
        server.rdb_child_pid = childpid;
        server.rdb_child_type = REDIS_RDB_CHILD_TYPE_DISK;

        // 关闭自动Rehash
        updateDictResizePolicy();
        return REDIS_OK;
    }
    return REDIS_OK; /* unreached */
}
```
<!--more-->
那么怎么才能够知道BGSAVE执行完毕，BGSAVE执行完毕后使用exitFromChild((retval == REDIS_OK) ? 0 : 1);向父进程发送信号。父进程调用serverCron接收该信号。

以下是处理函数的部分代码，处理BGREWRITEAOF与BGSAVE的完成信号。最终调用backgroundSaveDoneHandler根据返回信号信息进行对应处理。

```c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
\\ ...
    // 检查 BGSAVE 或者 BGREWRITEAOF 是否已经执行完毕
    if (server.rdb_child_pid != -1 || server.aof_child_pid != -1) {
        int statloc;
        pid_t pid;

        // 接收子进程发来的信号
        if ((pid = wait3(&statloc,WNOHANG,NULL)) != 0) {
            int exitcode = WEXITSTATUS(statloc);
            int bysignal = 0;
            
            if (WIFSIGNALED(statloc)) bysignal = WTERMSIG(statloc);

            // BGSAVE 执行完毕
            if (pid == server.rdb_child_pid) {
                backgroundSaveDoneHandler(exitcode,bysignal);

            // BGREWRITEAOF 执行完毕
            } else if (pid == server.aof_child_pid) {
                backgroundRewriteDoneHandler(exitcode,bysignal);

            } else {
                redisLog(REDIS_WARNING,
                    "Warning, detected child with unmatched pid: %ld",
                    (long)pid);
            }
            updateDictResizePolicy();
        }
    }
\\ ...
  
/* A background saving child (BGSAVE) terminated its work. Handle this.
 * This function covers the case of actual BGSAVEs. */
void backgroundSaveDoneHandlerDisk(int exitcode, int bysignal) {
    
    // 成功
    if (!bysignal && exitcode == 0) {
        redisLog(REDIS_NOTICE,
            "Background saving terminated with success");
        server.dirty = server.dirty - server.dirty_before_bgsave;
        server.lastsave = time(NULL);
        server.lastbgsave_status = REDIS_OK;

    // 出错
    } else if (!bysignal && exitcode != 0) {
        redisLog(REDIS_WARNING, "Background saving error");
        server.lastbgsave_status = REDIS_ERR;
    } else {
        mstime_t latency;

        redisLog(REDIS_WARNING,
            "Background saving terminated by signal %d", bysignal);
        latencyStartMonitor(latency);
        rdbRemoveTempFile(server.rdb_child_pid);
        latencyEndMonitor(latency);
        latencyAddSampleIfNeeded("rdb-unlink-temp-file",latency);
        /* SIGUSR1 is whitelisted, so we have a way to kill a child without
         * tirggering an error conditon. */
        if (bysignal != SIGUSR1)
            server.lastbgsave_status = REDIS_ERR;
    }

    // 更新状态
    server.rdb_child_pid = -1;
    server.rdb_child_type = REDIS_RDB_CHILD_TYPE_NONE;
    server.rdb_save_time_last = time(NULL)-server.rdb_save_time_start;
    server.rdb_save_time_start = -1;
    /* Possibly there are slaves waiting for a BGSAVE in order to be served
     * (the first stage of SYNC is a bulk transfer of dump.rdb) */
    updateSlavesWaitingBgsave((!bysignal && exitcode == 0) ? REDIS_OK : REDIS_ERR, REDIS_RDB_CHILD_TYPE_DISK);
}
```

### rdbSave

int rdbSave(char *filename);是SAVE与BGSAVE命令执行时正在用来生成RDB文件的函数

```c
int rdbSave(char *filename) {
    char tmpfile[256];
    FILE *fp;
    rio rdb;
    int error;

    // 生成临时文件
    snprintf(tmpfile,256,"temp-%d.rdb", (int) getpid());
    fp = fopen(tmpfile,"w");
    if (!fp) {
        redisLog(REDIS_WARNING, "Failed opening .rdb for saving: %s",
            strerror(errno));
        return REDIS_ERR;
    }

    // 初始化I/O
    rioInitWithFile(&rdb,fp);
    
    // 生成RDB文件
    if (rdbSaveRio(&rdb,&error) == REDIS_ERR) {
        errno = error;
        goto werr;
    }

    // 确保缓存中没有数据
    if (fflush(fp) == EOF) goto werr;
    if (fsync(fileno(fp)) == -1) goto werr;
    if (fclose(fp) == EOF) goto werr;

    // 使用RENAME改名
    if (rename(tmpfile,filename) == -1) {
        redisLog(REDIS_WARNING,"Error moving temp DB file on the final destination: %s", strerror(errno));
        unlink(tmpfile);
        return REDIS_ERR;
    }

    // 日志 设置状态
    redisLog(REDIS_NOTICE,"DB saved on disk");
    server.dirty = 0;
    server.lastsave = time(NULL);
    server.lastbgsave_status = REDIS_OK;
    return REDIS_OK;

werr:

    // 异常处理
    fclose(fp);
    unlink(tmpfile);
    redisLog(REDIS_WARNING,"Write error saving DB on disk: %s", strerror(errno));
    return REDIS_ERR;
}
```

实际是调用rdbSaveRio执行写入

```c
int rdbSaveRio(rio *rdb, int *error) {
    dictIterator *di = NULL;
    dictEntry *de;
    char magic[10];
    int j;
    long long now = mstime();
    uint64_t cksum;

    // 设置校验和
    if (server.rdb_checksum)
        rdb->update_cksum = rioGenericUpdateChecksum;

    // 写入REDIS版本号
    snprintf(magic,sizeof(magic),"REDIS%04d",REDIS_RDB_VERSION);
    if (rdbWriteRaw(rdb,magic,9) == -1) goto werr;

    // 遍历数据库
    for (j = 0; j < server.dbnum; j++) {
        redisDb *db = server.db+j;
        dict *d = db->dict;

        // 跳过空数据库
        if (dictSize(d) == 0) continue;

        // 键空间迭代器
        di = dictGetSafeIterator(d);
        if (!di) return REDIS_ERR;

        /* Write the SELECT DB opcode */
        // 写入DB选择器
        if (rdbSaveType(rdb,REDIS_RDB_OPCODE_SELECTDB) == -1) goto werr;
        if (rdbSaveLen(rdb,j) == -1) goto werr;

        /* Iterate this DB writing every entry */
      // 遍历数据库，写入键值对
        while((de = dictNext(di)) != NULL) {
            sds keystr = dictGetKey(de);
            robj key, *o = dictGetVal(de);
            long long expire;

            initStaticStringObject(key,keystr);
            expire = getExpire(db,&key);

            // 写入键值对
            if (rdbSaveKeyValuePair(rdb,&key,o,expire,now) == -1) goto werr;
        }
        dictReleaseIterator(di);
    }
    di = NULL; /* So that we don't release it again on error. */

    /* EOF opcode */
    // 写入 EOF 代码
    if (rdbSaveType(rdb,REDIS_RDB_OPCODE_EOF) == -1) goto werr;

    /* CRC64 checksum. It will be zero if checksum computation is disabled, the
     * loading code skips the check in this case. */
    // CRC64 校验和。
    cksum = rdb->cksum;
    memrev64ifbe(&cksum);
    if (rioWrite(rdb,&cksum,8) == 0) goto werr;
    return REDIS_OK;

werr:
    if (error) *error = errno;
    if (di) dictReleaseIterator(di);
    return REDIS_ERR;
}
```

## 自动保存

redis能够配置自动保存条件，当满足的情况下执行BGSAVE。

依赖上次保存时间和dirty计数。

```c
// serverCron函数
if (server.rdb_child_pid != -1 || server.aof_child_pid != -1 ||
    ldbPendingChildren())
{
  // 为之前处理BGSAVE完成的部分的代码
} else {
  // 检查所有保存条件
  for (j = 0; j < server.saveparamslen; j++) {
    struct saveparam *sp = server.saveparams+j;
    // 检查某个保存条件是否符合
    if (server.dirty >= sp->changes &&
        server.unixtime-server.lastsave > sp->seconds &&
        (server.unixtime-server.lastbgsave_try >
         CONFIG_BGSAVE_RETRY_DELAY ||
         server.lastbgsave_status == C_OK))
    {
      serverLog(LL_NOTICE,"%d changes in %d seconds. Saving...",
                sp->changes, (int)sp->seconds);
      
      // 保存
      rdbSaveBackground(server.rdb_filename);
      break;
    }
  }
  // ...
}
```

## RDB的文件载入

rdbload()用于将RDB文件从硬盘载入

```c
int rdbLoad(char *filename) {
    uint32_t dbid;
    int type, rdbver;
    redisDb *db = server.db+0;
    char buf[1024];
    long long expiretime, now = mstime();
    FILE *fp;
    rio rdb;

    // 打开rdb文件
    if ((fp = fopen(filename,"r")) == NULL) return REDIS_ERR;

    // 初始化rio
    rioInitWithFile(&rdb,fp);
    rdb.update_cksum = rdbLoadProgressCallback;
    rdb.max_processing_chunk = server.loading_process_events_interval_bytes;
    if (rioRead(&rdb,buf,9) == 0) goto eoferr;
    buf[9] = '\0';

    // 校验版本号
    if (memcmp(buf,"REDIS",5) != 0) {
        fclose(fp);
        redisLog(REDIS_WARNING,"Wrong signature trying to load DB from file");
        errno = EINVAL;
        return REDIS_ERR;
    }
    rdbver = atoi(buf+5);
    if (rdbver < 1 || rdbver > REDIS_RDB_VERSION) {
        fclose(fp);
        redisLog(REDIS_WARNING,"Can't handle RDB format version %d",rdbver);
        errno = EINVAL;
        return REDIS_ERR;
    }

    // 标记开始载入
    startLoading(fp);
    while(1) {
        robj *key, *val;
        expiretime = -1;

        /* Read type. */
        // 读取类型
        if ((type = rdbLoadType(&rdb)) == -1) goto eoferr;

        // 过期时间 秒为单位
        if (type == REDIS_RDB_OPCODE_EXPIRETIME) {

            // 过期时间
            if ((expiretime = rdbLoadTime(&rdb)) == -1) goto eoferr;
            /* We read the time so we need to read the object type again. */

            // 键值对
            if ((type = rdbLoadType(&rdb)) == -1) goto eoferr;
            /* the EXPIRETIME opcode specifies time in seconds, so convert
             * into milliseconds. */
            expiretime *= 1000;
        } else if (type == REDIS_RDB_OPCODE_EXPIRETIME_MS) {
            /* Milliseconds precision expire times introduced with RDB
             * version 3. */
             // 过期时间 毫秒但闻
            if ((expiretime = rdbLoadMillisecondTime(&rdb)) == -1) goto eoferr;
            /* We read the time so we need to read the object type again. */
            if ((type = rdbLoadType(&rdb)) == -1) goto eoferr;
        }

        // EOF
        if (type == REDIS_RDB_OPCODE_EOF)
            break;

        /* Handle SELECT DB opcode as a special case */
        // 读取切换数据库指示
        if (type == REDIS_RDB_OPCODE_SELECTDB) {

            // 数据库号
            if ((dbid = rdbLoadLen(&rdb,NULL)) == REDIS_RDB_LENERR)
                goto eoferr;

            // 校验
            if (dbid >= (unsigned)server.dbnum) {
                redisLog(REDIS_WARNING,"FATAL: Data file was created with a Redis server configured to handle more than %d databases. Exiting\n", server.dbnum);
                exit(1);
            }

            // 切换数据库
            db = server.db+dbid;
            continue;
        }
        /* Read key */
        // 读取键
        if ((key = rdbLoadStringObject(&rdb)) == NULL) goto eoferr;
        /* Read value */
        // 读取值
        if ((val = rdbLoadObject(type,&rdb)) == NULL) goto eoferr;
        /* Check if the key already expired. This function is used when loading
         * an RDB file from disk, either at startup, or when an RDB was
         * received from the master. In the latter case, the master is
         * responsible for key expiry. If we would expire keys here, the
         * snapshot taken by the master may not be reflected on the slave. */
        if (server.masterhost == NULL && expiretime != -1 && expiretime < now) {
            decrRefCount(key);
            decrRefCount(val);
            continue;
        }
        /* Add the new object in the hash table */
        // 将键值关联到数据库内
        dbAdd(db,key,val);

        /* Set the expire time if needed */
        // 设置过期时间
        if (expiretime != -1) setExpire(db,key,expiretime);

        decrRefCount(key);
    }
    /* Verify the checksum if RDB version is >= 5 */
    // 校验和比较
    if (rdbver >= 5 && server.rdb_checksum) {
        uint64_t cksum, expected = rdb.cksum;

        if (rioRead(&rdb,&cksum,8) == 0) goto eoferr;
        memrev64ifbe(&cksum);
        if (cksum == 0) {
            redisLog(REDIS_WARNING,"RDB file was saved with checksum disabled: no check performed.");
        } else if (cksum != expected) {
            redisLog(REDIS_WARNING,"Wrong RDB checksum. Aborting now.");
            exit(1);
        }
    }

    // 结束
    fclose(fp);
    stopLoading();
    return REDIS_OK;

eoferr: /* unexpected end of file is handled here with a fatal exit */
    redisLog(REDIS_WARNING,"Short read or OOM loading DB. Unrecoverable error, aborting now.");
    exit(1);
    return REDIS_ERR; /* Just to avoid warning */
}
```

