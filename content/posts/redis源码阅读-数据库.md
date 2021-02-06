---
title: redis源码阅读-服务器
date: 2017-07-10 16:21:21
categories: 
- 源码阅读
tags:
- redis
- 源码阅读
---

## 服务器

redis运行存在一个redis服务器结构，一个服务器中保存着n个数据库。

dbnum由服务器配置决定，默认值为16。

```c
struct redisServer {
    redisDb *db;  // Redis的数据库
    // ...
    int dbnum;  // 表明数据库的数量
    // ...
}

typedef struct redisDb {
    dict *dict;                 /* 数据库键字典 */
    dict *expires;              /* 键过期时间字典 */
    dict *blocking_keys;        /* 处于阻塞状态的键 */
    dict *ready_keys;           /* 可以解除阻塞的键 */
    dict *watched_keys;         /* 被watch的键 */
    struct evictionPoolEntry *eviction_pool;    /* Eviction pool of keys */
    int id;                     /* 数据库编号 */
    long long avg_ttl;          /* 数据库键的平均时间*/
} redisDb;
```

## 切换数据库

每个redis客户端都有自己的目标数据库，当客户端执行数据库读写命令，目标数据库是这些命令的操作对象。

redis提供select命令来切换数据库，redisClient

```c
typedef struct redisClient { 
    int fd;  // 套接字描述符
    redisDb *db; // 当前正在使用的数据库
    // ...
}

int selectDb(redisClient *c, int id) {

    // 校验id
    if (id < 0 || id >= server.dbnum)
        return REDIS_ERR;

    // 切换客户端数据库
    c->db = &server.db[id];

    return REDIS_OK;
}

```

## 数据库键空间

Redis数据库存放的数据都是以键值对形式存在，redisDB结构的dict字典保存数据库中的所有键值对，这个字典被成为键空间。

```c

typedef struct redisDb {
    dict *dict;                 /* 数据库键字典 */
    // ...
} redisDb;
```

键空间的键就是数据库的键，每个键都是一个字符串对象。

键空间的值就是数据库的值，每个值可以是字符串对象、列表对象、哈希表对象、集合对象、有序集合对象中任意一种。

## 键空间的操作

```c
/* db.c -- Keyspace access API */
int removeExpire(redisDb *db, robj *key);// 移除键的过期时间
void propagateExpire(redisDb *db, robj *key); // 
int expireIfNeeded(redisDb *db, robj *key); // 检查是否过期，是则删除键
long long getExpire(redisDb *db, robj *key); // 获取过期时间
void setExpire(redisDb *db, robj *key, long long when); // 设定过期时间
robj *lookupKey(redisDb *db, robj *key); // 从db中取出键key的值
robj *lookupKeyRead(redisDb *db, robj *key); // 从db中取出键key的值
robj *lookupKeyWrite(redisDb *db, robj *key); // 从db中取出键key的值
robj *lookupKeyReadOrReply(redisClient *c, robj *key, robj *reply); // 从db中取出键key的值
robj *lookupKeyWriteOrReply(redisClient *c, robj *key, robj *reply);// 从db中取出键key的值
void dbAdd(redisDb *db, robj *key, robj *val);// 尝试将键值对key\val添加到数据库中
void dbOverwrite(redisDb *db, robj *key, robj *val); // 重写指定键的值,键不存在的话终止
void setKey(redisDb *db, robj *key, robj *val);// 设定指定键的值，不管存不存在
int dbExists(redisDb *db, robj *key); // 判断指定键是否存在  
robj *dbRandomKey(redisDb *db); // 随机从数据库中取出一个键，并以字符串对象的方式返回这个键
int dbDelete(redisDb *db, robj *key); // 从数据库中删除给定的键
robj *dbUnshareStringValue(redisDb *db, robj *key, robj *o);
long long emptyDb(void(callback)(void*));// 情况所有数据
int selectDb(redisClient *c, int id); // 切换db
void signalModifiedKey(redisDb *db, robj *key);
void signalFlushedDb(int dbid);
unsigned int getKeysInSlot(unsigned int hashslot, robj **keys, unsigned int count);
unsigned int countKeysInSlot(unsigned int hashslot);
unsigned int delKeysInSlot(unsigned int hashslot);
int verifyClusterConfigWithData(void);
void scanGenericCommand(redisClient *c, robj *o, unsigned long cursor);
int parseScanCursorOrReply(redisClient *c, robj *o, unsigned long *cursor);
```

<!--more-->

### 键空间的初始化

```c
/* Db->dict, keys are sds strings, vals are Redis objects. */
// 键空间的类型
dictType dbDictType = {
    dictSdsHash,                /* hash function */
    NULL,                       /* key dup */
    NULL,                       /* val dup */
    dictSdsKeyCompare,          /* key compare */
    dictSdsDestructor,          /* key destructor */
    dictRedisObjectDestructor   /* val destructor */
};

// 服务器初始化的同时初始化键空间

void initServer() {
	// ...
    /* Create the Redis databases, and initialize other internal state. */
    // 创建并初始化数据库结构
    for (j = 0; j < server.dbnum; j++) {
        server.db[j].dict = dictCreate(&dbDictType,NULL);
		// ...
        server.db[j].id = j;
		// ...
    }
	// ...
};
```

### 查找

有五个和查找相关的接口，代码如下。

```c
robj *lookupKeyWriteOrReply(redisClient *c, robj *key, robj *reply) {

    robj *o = lookupKeyWrite(c->db, key);

    if (!o) addReply(c,reply);

    return o;
}
robj *lookupKeyReadOrReply(redisClient *c, robj *key, robj *reply) {

    // 查找
    robj *o = lookupKeyRead(c->db, key);

    // 发送信息
    if (!o) addReply(c,reply);

    return o;
}
robj *lookupKeyWrite(redisDb *db, robj *key) {

    // 删除过期键
    expireIfNeeded(db,key);

    // 查找并返回对象
    return lookupKey(db,key);
}
robj *lookupKeyRead(redisDb *db, robj *key) {
    robj *val;

    // 删除过期键
    expireIfNeeded(db,key);

    // 查找对象
    val = lookupKey(db,key);

    // 更新命中/不命中信息
    if (val == NULL)
        server.stat_keyspace_misses++;
    else
        server.stat_keyspace_hits++;

    // 返回值
    return val;
}
robj *lookupKey(redisDb *db, robj *key) {

    // 查找
    dictEntry *de = dictFind(db->dict,key->ptr);

    if (de) {

        // 取出值
        robj *val = dictGetVal(de);

        // 更新时间信息
        if (server.rdb_child_pid == -1 && server.aof_child_pid == -1)
            val->lru = LRU_CLOCK();

        // 返回值
        return val;
    } else {
        return NULL;
    }
}
```

### 添加新键

```c
void dbAdd(redisDb *db, robj *key, robj *val) {
    sds copy = sdsdup(key->ptr); // 复制键名
    int retval = dictAdd(db->dict, copy, val); // 尝试添加

    redisAssertWithInfo(NULL,key,retval == REDIS_OK); // 已经存在则停止
    if (val->type == REDIS_LIST) signalListAsReady(db, key);
    if (server.cluster_enabled) slotToKeyAdd(key);
 }
```

### 修改键

两种方式一种重写，一种设定不管存不存在。

```c
void dbOverwrite(redisDb *db, robj *key, robj *val); // 重写指定键的值,键不存在的话终止
void setKey(redisDb *db, robj *key, robj *val);// 设定指定键的值，不管存不存在

void dbOverwrite(redisDb *db, robj *key, robj *val) {
    dictEntry *de = dictFind(db->dict,key->ptr); // 查找

    redisAssertWithInfo(NULL,key,de != NULL); // 不存在，终止
    dictReplace(db->dict, key->ptr, val); // 修改旧值
}

void setKey(redisDb *db, robj *key, robj *val) {
    if (lookupKeyWrite(db,key) == NULL) {
        dbAdd(db,key,val); // 找不到就添加
    } else {
        dbOverwrite(db,key,val); // 找到就重写
    }
    incrRefCount(val); // 增加引用计数
    removeExpire(db,key); // 移除过期时间
    signalModifiedKey(db,key); // 发送键修改通知
}
```

### 删除键

```c
int dbDelete(redisDb *db, robj *key) {
    /* Deleting an entry from the expires dict will not free the sds of
     * the key, because it is shared with the main dictionary. */
    // 删除键的过期时间
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);

    // 删除键值对
    if (dictDelete(db->dict,key->ptr) == DICT_OK) {
        if (server.cluster_enabled) slotToKeyDel(key);
        return 1;
    } else {
        return 0;
    }
}
```

## 键的生存时间或过期时间

与键空间类似redis建立了一个字典，存放每个键的对应的过期时间。在初始化的时候创建

```c
dictType keyptrDictType = {
    dictSdsHash,               /* hash function */
    NULL,                      /* key dup */
    NULL,                      /* val dup */
    dictSdsKeyCompare,         /* key compare */
    NULL,                      /* key destructor */
    NULL                       /* val destructor */
};

void initServer() {
	// ...
    // 创建并初始化数据库结构
    for (j = 0; j < server.dbnum; j++) {
        server.db[j].expires = dictCreate(&keyptrDictType,NULL);
		// ...
        server.db[j].id = j;
		// ...
    }
	// ...
};
```

### 设定键的过期时间

```c
void setExpire(redisDb *db, robj *key, long long when) {
    dictEntry *kde, *de;

    kde = dictFind(db->dict,key->ptr); // 查找键
    redisAssertWithInfo(NULL,key,kde != NULL);
    // 在过期时间字典中查找，没有则添加
    de = dictReplaceRaw(db->expires,dictGetKey(kde));
    dictSetSignedIntegerVal(de,when); // 设置过期时间
}
```

### 获取键的过期时间

```c
long long getExpire(redisDb *db, robj *key) {
    dictEntry *de;

    //  如果不存在直接返回
    if (dictSize(db->expires) == 0 ||
       (de = dictFind(db->expires,key->ptr)) == NULL) return -1;

    redisAssertWithInfo(NULL,key,dictFind(db->dict,key->ptr) != NULL);

    // 返回过期时间
    return dictGetSignedIntegerVal(de);
}
```

### 删除键的过期时间

```c
int removeExpire(redisDb *db, robj *key) {
     // 确保键有过期时间
    redisAssertWithInfo(NULL,key,dictFind(db->dict,key->ptr) != NULL);
    return dictDelete(db->expires,key->ptr) == DICT_OK; // 删除
}
```

### 过期键的删除策略

三种策略：

- 定时删除。定时删除占用cpu，可能使服务器长期无响应。但是对内存友好。
- 惰性删除，对键进行操作时，才删除。缺点是对内存不友好，过期键过多的话，没有及时清理。
- 定期删除。间隔依据算法确定。两者结合，主要看算法选择。

redis采用定期和惰性两种删除方式。

#### 惰性删除

redis在很多操作前都会调用expireIfNeeded进行惰性删除。例如lookupKeyRead。

```c
int expireIfNeeded(redisDb *db, robj *key) {
    mstime_t when = getExpire(db,key);
    mstime_t now;

    // 无过期时间
    if (when < 0) return 0; /* No expire for this key */

    // 正在加载不删除
    if (server.loading) return 0;

    /* If we are in the context of a Lua script, we claim that time is
     * blocked to when the Lua script started. This way a key can expire
     * only the first time it is accessed and not in the middle of the
     * script execution, making propagation to slaves / AOF consistent.
     * See issue #1525 on Github for more information. */
    now = server.lua_caller ? server.lua_time_start : mstime();

    /* If we are running in the context of a slave, return ASAP:
     * the slave key expiration is controlled by the master that will
     * send us synthesized DEL operations for expired keys.
     *
     * Still we try to return the right information to the caller,
     * that is, 0 if we think the key should be still valid, 1 if
     * we think the key is expired at this time. */
    if (server.masterhost != NULL) return now > when;

    // 没过期，返回0
    if (now <= when) return 0;

    // 删除
    server.stat_expiredkeys++;
    propagateExpire(db,key);
    notifyKeyspaceEvent(REDIS_NOTIFY_EXPIRED,
        "expired",key,db->id);
    return dbDelete(db,key);
}
```

#### 定期删除

redis服务器周期性操作serverCron函数执行时，activeExpireCycle被调用，它在规定时间内分多次遍历服务器中的各个数据库，从数据库的expires字典中随机检查一部分键的过期时间，并删除其中的过期键。







