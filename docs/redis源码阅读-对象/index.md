# redis源码阅读-对象


之前阅读了redis用到的主要的数据结构，这些数据结构是redis对象基础。redis在这些基础数据结构之上创建了一个对象系统，这个系统包含字符串对象、列表对象、哈希对象、集合对象和有序集合对象五种类型的对象。

redis执行命令前，先判断命令是否能够执行给定命令。根据不同场合选择使用不同的数据结构。

## 对象的类型与编码

redis使用对象来表示数据库中的键值，创建一个键值对时，会创建至少两个对象，一个对象用作键值对的键，一个对象用作键值对的值。

### 对象的结构体

```c
typedef struct redisObject {
    unsigned type:4; // 类型
    unsigned encoding:4; // 编码
    unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */
    int refcount; // 引用计数
    void *ptr; // 值
} robj;
```

redis结构体使用位段结构节省空间

#### 类型type

记录redis对象类型，五种类型

```c
#define REDIS_STRING 0 // 字符串对象
#define REDIS_LIST 1 // 列表对象
#define REDIS_SET 2 // 哈希对象
#define REDIS_ZSET 3 // 集合对象
#define REDIS_HASH 4 // 有序集合对象
```

#### 编码encoding

记录redis对象的编码

```c
// 对象编码
#define REDIS_ENCODING_RAW 0     /* 简单动态字符串 */
#define REDIS_ENCODING_INT 1     /* long类型的整数 */
#define REDIS_ENCODING_HT 2      /* 字典 */
#define REDIS_ENCODING_ZIPMAP 3  /* zipmap 3.2.5不再使用 */
#define REDIS_ENCODING_LINKEDLIST 4 /* 双端队列 */
#define REDIS_ENCODING_ZIPLIST 5 /* 压缩列表 */
#define REDIS_ENCODING_INTSET 6  /* 整数集合 */
#define REDIS_ENCODING_SKIPLIST 7  /* 跳跃表 */
#define REDIS_ENCODING_EMBSTR 8  /* EMBSTR编码的简单字符串 */
```

每种类型对应至少两种不同的编码。

| 对象类型         | 编码方式                                     |
| ------------ | ---------------------------------------- |
| REDIS_STRING | REDIS_ENCODING_RAW ,REDIS_ENCODING_INT ,REDIS_ENCODING_EMBSTR |
| REDIS_LIST   | REDIS_ENCODING_LINKEDLIST ,REDIS_ENCODING_ZIPLIST |
| REDIS_SET    | REDIS_ENCODING_INTSET ,REDIS_ENCODING_HT |
| REDIS_ZSET   | REDIS_ENCODING_ZIPLIST ,REDIS_ENCODING_SKIPLIST |
| REDIS_HASH   | REDIS_ENCODING_ZIPLIST ,REDIS_ENCODING_HT |

#### 访问时间

表示对象的最后一次访问时间。

#### 引用计数

常见的管理方式，引用计数为0时回收。

<!--more-->

## 对象的API

```c
void decrRefCount(robj *o); // 引用计数-1，降为0时释放对象
void decrRefCountVoid(void *o); // 用于特定数据结构的释放
void incrRefCount(robj *o); // 引用计数+1
robj *resetRefCount(robj *obj); // 设置引用计数为0
void freeStringObject(robj *o); // 释放字符串对象
void freeListObject(robj *o); // 释放列表对象
void freeSetObject(robj *o);  // 释放集合对象
void freeZsetObject(robj *o); // 释放有序集合对象
void freeHashObject(robj *o); // 释放hash对象
robj *createObject(int type, void *ptr); // 创建一个新robj对象
robj *createStringObject(char *ptr, size_t len); // 创建一个字符串对象，根据大小选择编码
robj *createRawStringObject(char *ptr, size_t len);// 创建一个 REDIS_ENCODING_RAW 编码的字符对象
robj *createEmbeddedStringObject(char *ptr, size_t len);// 创建一个 REDIS_ENCODING_EMBSTR 编码的字符对象
robj *dupStringObject(robj *o); // 复制一个字符串对象
int isObjectRepresentableAsLongLong(robj *o, long long *llongval); // 检查对象的值是否为Long long
robj *tryObjectEncoding(robj *o); // 尝试对字符串对象编码，以节约内存
robj *getDecodedObject(robj *o); // 返回一个对象的编码版本
size_t stringObjectLen(robj *o); // 返回字符串对象的字符串值的长度
robj *createStringObjectFromLongLong(long long value); // 根据传入的值，创建一个字符串对象
robj *createStringObjectFromLongDouble(long double value, int humanfriendly); // 根据传入的值，创建一个字符串对象
robj *createListObject(void); // 创建一个linkedlist编码的列表对象
robj *createZiplistObject(void); // 创建一个ziplist编码的列表对象
robj *createSetObject(void); // 创建一个ht编码的集合对象
robj *createIntsetObject(void); // 创建一个intset编码的集合对象
robj *createHashObject(void); // 创建一个ziplist编码的哈希对象
robj *createZsetObject(void); // 创建一个skiplist编码的有序集合
robj *createZsetZiplistObject(void); // 创建一个ziplist编码的有序集合
int getLongFromObjectOrReply(redisClient *c, robj *o, long *target, const char *msg); // 尝试从对象中获取Long类型值
int checkType(redisClient *c, robj *o, int type); // 检查对象0的类型是否和type相同
int getLongLongFromObjectOrReply(redisClient *c, robj *o, long long *target, const char *msg); // 尝试从对象中取出整数值
int getDoubleFromObjectOrReply(redisClient *c, robj *o, double *target, const char *msg); // 尝试从对象中取出double值
int getLongLongFromObject(robj *o, long long *target); // 尝试从对象中获取整数值
int getLongDoubleFromObject(robj *o, long double *target); // 尝试从对象获取long double值
int getLongDoubleFromObjectOrReply(redisClient *c, robj *o, long double *target, const char *msg); // 尝试从对象获取long double值
char *strEncoding(int encoding); // 返回编码的字符串表示
int compareStringObjects(robj *a, robj *b); // 二进制方式比较两个字符串对象
int collateStringObjects(robj *a, robj *b); // 以collation方式比较两个字符串对象
int equalStringObjects(robj *a, robj *b); // 判断是否相同两个字符串对象
unsigned long long estimateObjectIdleTime(robj *o); // 计算对象的闲置时间
```

### 对象的创建

对象的创建都比较类似，一般创建底层数据结构，然后创建对象。然后初始化。

以string对象为例

```c
// 创建字符串对象
robj *createStringObject(char *ptr, size_t len) {
    // 长度小于39时使用EMBSTR
    if (len <= REDIS_ENCODING_EMBSTR_SIZE_LIMIT)
        return createEmbeddedStringObject(ptr,len);
    else
        return createRawStringObject(ptr,len);
}

 // 创建一个embstr编码的字符串对象
robj *createEmbeddedStringObject(char *ptr, size_t len) {
    // 直接分配一个连续空间长度为redis和字符串结构+字符串保存内存
    robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr)+len+1);
    struct sdshdr *sh = (void*)(o+1); // 找到字符串的起始位置

    // 初始化字符串
    o->type = REDIS_STRING;
    o->encoding = REDIS_ENCODING_EMBSTR;
    o->ptr = sh+1;
    o->refcount = 1;
    o->lru = LRU_CLOCK();

    sh->len = len;
    sh->free = 0;
    if (ptr) {
        memcpy(sh->buf,ptr,len);
        sh->buf[len] = '\0';
    } else {
        memset(sh->buf,0,len+1);
    }
    return o;
}

// 创建一个 REDIS_ENCODING_RAW 编码的字符对象
// 对象的指针指向一个 sds 结构
robj *createRawStringObject(char *ptr, size_t len) {
  	// sdsnewlen新建字符串
    return createObject(REDIS_STRING,sdsnewlen(ptr,len));
}

// 创建一个新对象
robj *createObject(int type, void *ptr) {

    robj *o = zmalloc(sizeof(*o));

    o->type = type;
    o->encoding = REDIS_ENCODING_RAW;
    o->ptr = ptr;
    o->refcount = 1;

    /* Set the LRU to the current lruclock (minutes resolution). */
    o->lru = LRU_CLOCK();
    return o;
}
```

### 对象的释放

以字符串对象为例子，redis采用引用计数进行对象的释放，当对象不再使用时调用decrRefCount减少引用计数，在引用计数减到0后，释放对象。

```c
void decrRefCount(robj *o) {
    if (o->refcount <= 0) redisPanic("decrRefCount against refcount <= 0");
    if (o->refcount == 1) {
        switch(o->type) {
        // 根据类型释放 各个函数会调用各自的释放函数释放
        case REDIS_STRING: freeStringObject(o); break;
        case REDIS_LIST: freeListObject(o); break;
        case REDIS_SET: freeSetObject(o); break;
        case REDIS_ZSET: freeZsetObject(o); break;
        case REDIS_HASH: freeHashObject(o); break;
        default: redisPanic("Unknown object type"); break;
        }
        zfree(o); // 释放对象内存
    } else {
        o->refcount--;
    }
}

// 如果时RAW编码调用sdsfree释放，否则在释放robj时就释放了，因为采用了embstr编码
void freeStringObject(robj *o) {
    if (o->encoding == REDIS_ENCODING_RAW) {
        sdsfree(o->ptr);
    }
}
```

## 字符串对象

字符串编码可以是Int、raw或者embstr。

字符串对象为整数值，可以用long long类型表示，则为int编码。

字符串对象为浮点数，能够用long double类型表示，使用embstr还是raw根据长度来定。

如果一个字符串对象小于等于REDIS_ENCODING_EMBSTR_SIZE_LIMIT则用embstr编码。

大于REDIS_ENCODING_EMBSTR_SIZE_LIMIT采用raw编码。



int编码在执行一个会将int转变为字符串值时，编码变为raw。

embstr为只读的，当尝试修改时会转换为raw。

## 列表对象

列表对象编码是ziplist或者linkedlist。



满足以下两个条件使用ziplist：

- 保存的字符串长度都小于64
- 元素数量小于512

## 哈希对象

哈希对象编码是ziplist或者hashtable。

如果采用的是ziplist那么添加键值时，先将键推入压缩列表表尾部，再将值推入压缩列表表尾。

如果采用hashtable编码，那么字典的键就是键值对的键的字符串对象，字典的值时键值对的值。



满足以下两个条件使用ziplist：

- 保存的字符串长度都小于64
- 元素数量小于512

## 集合对象

集合的编码是inset或者hashtable



满足以下条件使用intset编码：

- 集合对象保存的值都为整数
- 集合对象保存的元素不超过512个

## 有序集合对象

有序集合编码是ziplist或者skiplist。

skiplist编码使用一个zskiplist和dict作为底层实现。zskiplist按照分值从大到小保存集合元素。dict保存从成员到分值的映射。



满足以下条件使用ziplist编码：

- 有序集合保存的元素小于128个。
- 有序集合保存的所有元素成员的长度都小于64字节。


## 命令
对象的命令处理在redis.h中。
执行命令之前会检查对象的类型，是否能够执行该命令。
如果能就调用对象的命令处理函数,否则返回错误。

在调用了对象的命令处理函数之后，则根据命令具体的编码，去选择使用什么底层数据结构的接口来实现。

```c

/* Commands prototypes */
void authCommand(redisClient *c);
void pingCommand(redisClient *c);
void echoCommand(redisClient *c);
void setCommand(redisClient *c);
void setnxCommand(redisClient *c);
void setexCommand(redisClient *c);
void psetexCommand(redisClient *c);
void getCommand(redisClient *c);
void delCommand(redisClient *c);
void existsCommand(redisClient *c);
void setbitCommand(redisClient *c);
void getbitCommand(redisClient *c);
void setrangeCommand(redisClient *c);
void getrangeCommand(redisClient *c);
void incrCommand(redisClient *c);
void decrCommand(redisClient *c);
void incrbyCommand(redisClient *c);
void decrbyCommand(redisClient *c);
void incrbyfloatCommand(redisClient *c);
void selectCommand(redisClient *c);
void randomkeyCommand(redisClient *c);
void keysCommand(redisClient *c);
void scanCommand(redisClient *c);
void dbsizeCommand(redisClient *c);
void lastsaveCommand(redisClient *c);
void saveCommand(redisClient *c);
void bgsaveCommand(redisClient *c);
void bgrewriteaofCommand(redisClient *c);
void shutdownCommand(redisClient *c);
void moveCommand(redisClient *c);
void renameCommand(redisClient *c);
void renamenxCommand(redisClient *c);
void lpushCommand(redisClient *c);
void rpushCommand(redisClient *c);
void lpushxCommand(redisClient *c);
void rpushxCommand(redisClient *c);
void linsertCommand(redisClient *c);
void lpopCommand(redisClient *c);
void rpopCommand(redisClient *c);
void llenCommand(redisClient *c);
void lindexCommand(redisClient *c);
void lrangeCommand(redisClient *c);
void ltrimCommand(redisClient *c);
void typeCommand(redisClient *c);
void lsetCommand(redisClient *c);
void saddCommand(redisClient *c);
void sremCommand(redisClient *c);
void smoveCommand(redisClient *c);
void sismemberCommand(redisClient *c);
void scardCommand(redisClient *c);
void spopCommand(redisClient *c);
void srandmemberCommand(redisClient *c);
void sinterCommand(redisClient *c);
void sinterstoreCommand(redisClient *c);
void sunionCommand(redisClient *c);
void sunionstoreCommand(redisClient *c);
void sdiffCommand(redisClient *c);
void sdiffstoreCommand(redisClient *c);
void sscanCommand(redisClient *c);
void syncCommand(redisClient *c);
void flushdbCommand(redisClient *c);
void flushallCommand(redisClient *c);
void sortCommand(redisClient *c);
void lremCommand(redisClient *c);
void rpoplpushCommand(redisClient *c);
void infoCommand(redisClient *c);
void mgetCommand(redisClient *c);
void monitorCommand(redisClient *c);
void expireCommand(redisClient *c);
void expireatCommand(redisClient *c);
void pexpireCommand(redisClient *c);
void pexpireatCommand(redisClient *c);
void getsetCommand(redisClient *c);
void ttlCommand(redisClient *c);
void pttlCommand(redisClient *c);
void persistCommand(redisClient *c);
void slaveofCommand(redisClient *c);
void debugCommand(redisClient *c);
void msetCommand(redisClient *c);
void msetnxCommand(redisClient *c);
void zaddCommand(redisClient *c);
void zincrbyCommand(redisClient *c);
void zrangeCommand(redisClient *c);
void zrangebyscoreCommand(redisClient *c);
void zrevrangebyscoreCommand(redisClient *c);
void zrangebylexCommand(redisClient *c);
void zrevrangebylexCommand(redisClient *c);
void zcountCommand(redisClient *c);
void zlexcountCommand(redisClient *c);
void zrevrangeCommand(redisClient *c);
void zcardCommand(redisClient *c);
void zremCommand(redisClient *c);
void zscoreCommand(redisClient *c);
void zremrangebyscoreCommand(redisClient *c);
void zremrangebylexCommand(redisClient *c);
void multiCommand(redisClient *c);
void execCommand(redisClient *c);
void discardCommand(redisClient *c);
void blpopCommand(redisClient *c);
void brpopCommand(redisClient *c);
void brpoplpushCommand(redisClient *c);
void appendCommand(redisClient *c);
void strlenCommand(redisClient *c);
void zrankCommand(redisClient *c);
void zrevrankCommand(redisClient *c);
void hsetCommand(redisClient *c);
void hsetnxCommand(redisClient *c);
void hgetCommand(redisClient *c);
void hmsetCommand(redisClient *c);
void hmgetCommand(redisClient *c);
void hdelCommand(redisClient *c);
void hlenCommand(redisClient *c);
void zremrangebyrankCommand(redisClient *c);
void zunionstoreCommand(redisClient *c);
void zinterstoreCommand(redisClient *c);
void zscanCommand(redisClient *c);
void hkeysCommand(redisClient *c);
void hvalsCommand(redisClient *c);
void hgetallCommand(redisClient *c);
void hexistsCommand(redisClient *c);
void hscanCommand(redisClient *c);
void configCommand(redisClient *c);
void hincrbyCommand(redisClient *c);
void hincrbyfloatCommand(redisClient *c);
void subscribeCommand(redisClient *c);
void unsubscribeCommand(redisClient *c);
void psubscribeCommand(redisClient *c);
void punsubscribeCommand(redisClient *c);
void publishCommand(redisClient *c);
void pubsubCommand(redisClient *c);
void watchCommand(redisClient *c);
void unwatchCommand(redisClient *c);
void clusterCommand(redisClient *c);
void restoreCommand(redisClient *c);
void migrateCommand(redisClient *c);
void askingCommand(redisClient *c);
void readonlyCommand(redisClient *c);
void readwriteCommand(redisClient *c);
void dumpCommand(redisClient *c);
void objectCommand(redisClient *c);
void clientCommand(redisClient *c);
void evalCommand(redisClient *c);
void evalShaCommand(redisClient *c);
void scriptCommand(redisClient *c);
void timeCommand(redisClient *c);
void bitopCommand(redisClient *c);
void bitcountCommand(redisClient *c);
void bitposCommand(redisClient *c);
void replconfCommand(redisClient *c);
void waitCommand(redisClient *c);
void pfselftestCommand(redisClient *c);
void pfaddCommand(redisClient *c);
void pfcountCommand(redisClient *c);
void pfmergeCommand(redisClient *c);
void pfdebugCommand(redisClient *c);
```






