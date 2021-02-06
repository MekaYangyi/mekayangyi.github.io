# redis源码阅读-字典



Redis字典由哈希表实现的保存键值对的抽象数据结构。

实现文件在dict.h\dict.c中。

## 哈希表

Redis字典结构体定义。

```c
typedef struct dictEntry {
    void *key;// key 键
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;// 值，支持多种类型,使用联合。
    struct dictEntry *next;// 下一个点 采用链式来解决索引冲突问题
} dictEntry;

typedef struct dictType {
    unsigned int (*hashFunction)(const void *key);// hash函数
    void *(*keyDup)(void *privdata, const void *key);// key复制函数
    void *(*valDup)(void *privdata, const void *obj);// value复制函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);// key比较函数
    void (*keyDestructor)(void *privdata, void *key);// key释放函数
    void (*valDestructor)(void *privdata, void *obj);// value释放函数
} dictType;

/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table. */
typedef struct dictht {
    dictEntry **table;// 指针的数组头的指针
    unsigned long size;// 大小
    unsigned long sizemask;// 大小的掩码 总是等于size-1
    unsigned long used;// 被使用的节点数
} dictht;// hash表

typedef struct dict {
    dictType *type;// 绑定的函数
    void *privdata;// 私有数据
    dictht ht[2];// hash表，一般只使用[0]，在rehash的时候使用[1]
    long rehashidx; // 记录rehash的进度，不进行rehash的时候为-1
    int iterators; /* number of iterators currently running */
} dict;// 字典

/* If safe is set to 1 this is a safe iterator, that means, you can call
 * dictAdd, dictFind, and other functions against the dictionary even while
 * iterating. Otherwise it is a non safe iterator, and only dictNext()
 * should be called while iterating. */
typedef struct dictIterator {
    dict *d;
    long index;
    int table, safe;
    dictEntry *entry, *nextEntry;
    /* unsafe iterator fingerprint for misuse detection. */
    long long fingerprint;
} dictIterator;// 迭代器
```

<!--more-->
## 哈希算法

确认一个键值插入到字典的位置是哪，需要调用hash算法

```c
//计算key的hash值
hash = dict->type->hashFunction(key);
//按位与确认在hash表中的位置
//根据情况，可能是ht[0]或者ht[1]
index = hash & dict->ht[x].sizemask;
```

字典在redis中使用MurmurHash2算法进行计算键的哈希值。有点在于计算速度非常快，即使输入的键有规律也能够很好的给出一个随机分布。

## 键的冲突解决

在key获取的index相同的情况下，产生了键的冲突。redis采用链式解决冲突，新的键值放在链的头部。

## rehash

随着hash表不断插入删除数据，hash表的负载因子会不断变化。当负载因子在一个不合理的范围内，则redis的会对hash表进行rehash。

## 渐进式 rehash

渐进式rehash的目的是为了防止一次性rehash的情况下，服务器停止响应。

redis渐进式rehash的步骤:

1）为ht[1]分配空间，让字典同时持有 ht[0]和ht[1]两个哈希表

2）在字典位置一个索引计数器变量rehashidx，并将它的值设置为0，表示rehash工作正式开始。

3）在rehash进行期间，每次对字典执行删除、添加、查找或者更新操作时候，程序除了执行指定的操作外，还顺便将ht[0]哈希表在rehashidx索引上的所有键值对rehash到ht[1]，当rehash工作完成后将rehashidx属性的值增加1

4）随着字典操作的不断执行，最终在某个时间点上，ht[0]的所有键值对都会被rehsh至ht[1]，这时程序将rehashidx属性值设为-1，表示rehash操作已完成。



渐进式rehash过程中，字典会同时对ht[0]和ht[1]两个哈希表进行操作，字典在删除、查找、更新等操作会在两个哈希表进行。

在渐进式rehash执行期间，新添加到字典的键值对一律会被保存在ht[1]里面，而ht[0]则不再进行任何添加操作，这一措施保证了ht[0]包含的键值对数量会只减少不增加，并随着rehash操作的执行最终变为空。


rehash有两种方式，一种是单步，在字典没有安全迭代器的情况下能够执行。一种是执行一段时间跳出。两种方法均调用int dictRehash(dict *d, int n) 完成。

```c
static void _dictRehashStep(dict *d) {
    if (d->iterators == 0) dictRehash(d,1);
}
```

```c
int dictRehashMilliseconds(dict *d, int ms) {
    // 开始时间
    long long start = timeInMilliseconds();
    int rehashes = 0;

    while(dictRehash(d,100)) {
        rehashes += 100;
        // 时间到达，跳出
        if (timeInMilliseconds()-start > ms) break;
    }

    return rehashes;
}
```

int dictRehash(dict *d, int n) 算法

```c
 //执行N步渐进式rehash操作，rehash之后如果旧表还存在数据，则返回1，不存在返回0
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; // 最大允许访问的空桶值
    if (!dictIsRehashing(d)) return 0; // 判断是否允许rehash

    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        // rehashidx不能大于哈希表的大小
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        // 跳过空节点
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            // 超过空节点最大值，直接跳出
            if (--empty_visits == 0) return 1;
        }
        // 获取需要rehash的节点
        de = d->ht[0].table[d->rehashidx];
        // 将该桶下所有节点移动到新表
        while(de) {
            unsigned int h;

            nextde = de->next;
            // 获取新表中hash索引
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }

    // 判断是否都迁移完成，完成返回0
    if (d->ht[0].used == 0) {
        // 释放旧表,将rehashidx设置为-1
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }

    // 未完成返回1
    return 1;
}
```

## dict部分API
```c
dict *dictCreate(dictType *type, void *privDataPtr); // 创建一个新字典
int dictExpand(dict *d, unsigned long size); // 在字典中创建一个新hash表
int dictAdd(dict *d, void *key, void *val); // 尝试将给定键值添加到字典中
dictEntry *dictAddRaw(dict *d, void *key); // 尝试将给定键插入到字典中，键已经存在则返回null
int dictReplace(dict *d, void *key, void *val); // 将给定键值添加到字典中，如果已经存在就替换
dictEntry *dictReplaceRaw(dict *d, void *key); // 将给定键值添加到字典中，如果已经存在则不添加，返回已经存在的值
int dictDelete(dict *d, const void *key); // 删除字典中给定键的节点
int dictDeleteNoFree(dict *d, const void *key);// 删除包含给定键的及诶单，但是不释放
void dictRelease(dict *d); // 删除并释放整个字典
dictEntry * dictFind(dict *d, const void *key); // 查找节点
void *dictFetchValue(dict *d, const void *key); // 获取包含给定键的节点值
int dictResize(dict *d); // 缩小字典，使得已用节点和字典大小比率接近1:1
dictIterator *dictGetIterator(dict *d); // 创建并返回给定字典的不安全迭代器
dictIterator *dictGetSafeIterator(dict *d); // 创建并返回给定节点的安全迭代器
dictEntry *dictNext(dictIterator *iter); // 返回当前节点，指向下个节点
void dictReleaseIterator(dictIterator *iter); // 释放迭代器
dictEntry *dictGetRandomKey(dict *d); // 随机返回字典中一个节点
void dictEmpty(dict *d, void(callback)(void*)); // 清空字典中所有哈希表节点，并重置属性
void dictEnableResize(void); // 开启自动rehash
void dictDisableResize(void); // 关闭自动rehash
```

### 创建dict

使用dictCreate创建字典。

```c
// 创建字典
dict *dictCreate(dictType *type,
        void *privDataPtr)
{
    // 分配空间
    dict *d = zmalloc(sizeof(*d));

    // 初始化字典
    _dictInit(d,type,privDataPtr);
    return d;
}

static void _dictReset(dictht *ht)
{
    ht->table = NULL;
    ht->size = 0;
    ht->sizemask = 0;
    ht->used = 0;
}

// 初始化字典
int _dictInit(dict *d, dictType *type,
        void *privDataPtr)
{
    // 重置hash表
    _dictReset(&d->ht[0]);
    _dictReset(&d->ht[1]);
    d->type = type; // 设置字典类型
    d->privdata = privDataPtr;
    d->rehashidx = -1; // 初始为-1，表明没有进行rehash
    d->iterators = 0; //正在使用的迭代器数量
    return DICT_OK;
}
```

### 添加键值对

使用int dictAdd(dict *d, void *key, void *val)添加键值对

```c
// 添加一个键值对到dict中
int dictAdd(dict *d, void *key, void *val)
{
    // 往字典中添加一个只有key的键值对
    dictEntry *entry = dictAddRaw(d,key);

    // 添加失败，则返回错误
    if (!entry) return DICT_ERR;
    //使用宏，添加成功则设置key键值对的值
    dictSetVal(d, entry, val);
    return DICT_OK;
}

 // 添加键到字典中
 // 键存在则返回null
 // 不存在则创建节点，与键关联，并返回节点
dictEntry *dictAddRaw(dict *d, void *key)
{
    int index;
    dictEntry *entry;
    dictht *ht;

    // 尝试进行单步式rehash
    if (dictIsRehashing(d)) _dictRehashStep(d);

    // 尝试获取hash表中的索引值，返回-1表示键已经存在
    if ((index = _dictKeyIndex(d, key)) == -1)
        return NULL;

    // rehash使用1号哈希表，不在rehash使用0号
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];

    // 分配空间，将节点添加到链表表头
    entry = zmalloc(sizeof(*entry));
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;

    // 使用宏，设置新节点的键
    dictSetKey(d, entry, key);
    return entry;
}
```

### 替换键值对

```c
 // 如果键值不存在，返回1
 //存在，更新键值，返回0
int dictReplace(dict *d, void *key, void *val)
{
    dictEntry *entry, auxentry;

    // 添加成功返回1
    if (dictAdd(d, key, val) == DICT_OK)
        return 1;
    
    // 查找键
    entry = dictFind(d, key);
    
    // 更新键值
    auxentry = *entry;
    dictSetVal(d, entry, val);

    // 释放原值
    dictFreeVal(d, &auxentry);
    return 0;
}
```

### 查找键值对

```c
dictEntry *dictFind(dict *d, const void *key)
{
    dictEntry *he;
    unsigned int h, idx, table;

    // hash表大小为0，表名无值
    if (d->ht[0].size == 0) return NULL; /* We don't have a table at all */
    // 如果在rehash，则单步rehash
    if (dictIsRehashing(d)) _dictRehashStep(d);

    // 查找索引
    h = dictHashKey(d, key);

    // 遍历索引下的键
    for (table = 0; table <= 1; table++) {
        idx = h & d->ht[table].sizemask;
        he = d->ht[table].table[idx];
        while(he) {
            if (dictCompareKeys(d, key, he->key))
                return he;
            he = he->next;
        }
        // 如果没有进行rehash，则不再查找ht[1]
        if (!dictIsRehashing(d)) return NULL;
    }
    return NULL;
}
```

### 删除键值对

```c

// 删除该键值对，并释放键和值
int dictDelete(dict *ht, const void *key) {
    return dictGenericDelete(ht,key,0);
}

// 删除该键值对，不释放键和值
int dictDeleteNoFree(dict *ht, const void *key) {
    return dictGenericDelete(ht,key,1);
}

// 查找并删除对应的键值对
static int dictGenericDelete(dict *d, const void *key, int nofree)
{
    unsigned int h, idx;
    dictEntry *he, *prevHe;
    int table;

    // 空则直接返回错误信息
    if (d->ht[0].size == 0) return DICT_ERR; /* d->ht[0].table is NULL */

    // rehash时，进行一次rehash
    if (dictIsRehashing(d)) _dictRehashStep(d);
    h = dictHashKey(d, key); // 获取hash索引

    for (table = 0; table <= 1; table++) {
        idx = h & d->ht[table].sizemask;
        he = d->ht[table].table[idx];
        prevHe = NULL;
        // 查找，遍历整个链表
        while(he) {
            if (dictCompareKeys(d, key, he->key)) {
                /* Unlink the element from the list */
                if (prevHe)
                    prevHe->next = he->next;
                else
                    d->ht[table].table[idx] = he->next;
                if (!nofree) {
                    // 根据传入参数是否释放键值
                    dictFreeKey(d, he);
                    dictFreeVal(d, he);
                }
                zfree(he);
                d->ht[table].used--;
                return DICT_OK;
            }
            prevHe = he;
            he = he->next;
        }
        // 没有进行rehash则不查找ht[1]
        if (!dictIsRehashing(d)) break;
    }
    return DICT_ERR; /* not found */
}
```

### 字典删除

```c
// 清理释放整个字典
void dictRelease(dict *d)
{
    // 释放ht[0]
    _dictClear(d,&d->ht[0],NULL);

    // 释放ht[1]
    _dictClear(d,&d->ht[1],NULL);

    // 释放字典
    zfree(d);
}

int _dictClear(dict *d, dictht *ht, void(callback)(void *)) {
    unsigned long i;

    // 释放所有元素
    for (i = 0; i < ht->size && ht->used > 0; i++) {
        dictEntry *he, *nextHe;

        if (callback && (i & 65535) == 0) callback(d->privdata);

        if ((he = ht->table[i]) == NULL) continue;
        while(he) {
            nextHe = he->next;
            // 释放键值对
            dictFreeKey(d, he);
            dictFreeVal(d, he);
            zfree(he);
            ht->used--;
            he = nextHe;
        }
    }
    
    // 释放hash表
    zfree(ht->table);
    
    // 重置hsh表
    _dictReset(ht);
    return DICT_OK; /* never fails */
}
```

