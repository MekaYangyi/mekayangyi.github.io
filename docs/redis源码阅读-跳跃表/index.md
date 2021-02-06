# redis源码阅读-跳跃表


跳跃表一种有序数据结构，每个节点维护多个指向其他节点的指针，达到快速访问节点的目的。

大部分情况下跳跃表的效率可以和平衡数媲美，实现比平衡树简单，不少程序使用跳跃表来代替平衡树。

跳跃表有时会作为有序集合的实现。以分值排序。

## 结构体

```c
// 跳跃表节点
typedef struct zskiplistNode {
    robj *obj; // 保存的对象
    double score; // 分值 跳跃表按照分值进行排序
    struct zskiplistNode *backward; // 上一节点
    struct zskiplistLevel {
        struct zskiplistNode *forward; // 前进指针
        unsigned int span; // 跨度 记录两个节点之间的距离
    } level[]; // 层
} zskiplistNode;

// 跳跃表
typedef struct zskiplist {
    struct zskiplistNode *header, *tail; // 头、尾指针
    PORT_ULONG length; // 跳跃表长度
    int level; // 层数最大节点层数
} zskiplist;
```

## 跳跃表的创建

```c
// 创建节点
zskiplistNode *zslCreateNode(int level, double score, robj *obj) {
    // 申请内存
    zskiplistNode *zn = zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
    zn->score = score; // 赋值分数
    zn->obj = obj; // 设定成员对象
    return zn;
}

// 创建跳跃表
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    // 申请内存
    zsl = zmalloc(sizeof(*zsl));
    zsl->level = 1; // 设置层数初始为1
    zsl->length = 0; // 设置长度初始为0

    // 创建头节点 层数为32 分数为0
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);

    // 将每层的forward指针指向null，跨度0
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    // 设定backward指向null
    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}
```

<!--more-->

## 插入节点

逻辑是先找到节点插入位置，插入位置前一个节点的信息。
插入，并更新前一节点信息。
```c
// 创建一个成员为obj，分值为score的新节点
// 将新节点插入到跳跃表中
zskiplistNode *zslInsert(zskiplist *zsl, double score, robj *obj) {
    // updata[]数组记录每一层位于插入节点的前一个节点
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    // rank[]记录每一层位于插入节点的前一个节点的排名
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    redisAssert(!isnan(score));
    x = zsl->header; // 表头节点
    // 从最高层开始查找
    for (i = zsl->level-1; i >= 0; i--) {
        // i == (zsl->level-1) 为0
        //否则第i层起始rank值为i+1的rank值
        // 最终rank[0]的值+1就是新节点的前置节点排位
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        // 沿着前几指针遍历跳跃表
        while (x->level[i].forward &&
            // 比对分值
            (x->level[i].forward->score < score ||
                // 比对成员
                (x->level[i].forward->score == score &&
                compareStringObjects(x->level[i].forward->obj,obj) < 0))) {
            // 记录沿途跨越多少节点
            rank[i] += x->level[i].span;
            // 移动到下一个指针
            x = x->level[i].forward;
        }
        // 记录将要和新节点相连接的节点
        update[i] = x;
    }
       *
    // zslInsert() 的调用者会确保同分值且同成员的元素不会出现，
    // 所以这里不需要进一步进行检查，可以直接创建新元素。
    
    // 获取一个随机值作为新节点的层数
    level = zslRandomLevel();

    // 如果新节点的层数比其他节点层数大
    if (level > zsl->level) {
        // 初始化未使用层
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }

    // 创建新节点
    x = zslCreateNode(level,score,obj);

    // 将前面记录的指针指向新节点，并做相应的设置
    for (i = 0; i < level; i++) {

        // 设置新节点的前进指针
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x; // 将沿途记录的各个节点的前进指针指向新节点

        /* update span covered by update[i] as x is inserted here */
        // 计算新节点跨越的节点数量
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        // 更新新节点插入后，沿途节点的span值
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    /* increment span for untouched levels */
    // 未接触的节点的span值也需要增加1，这些节点从表头指向新节点
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    // 设置新节点的后退指针
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;

    // 长度+1
    zsl->length++;
    return x;
}
```

## 删除节点

Redis提供三种删除跳跃表节点的方式：

1. 根据给定分值和成员来删除节点，zslDelete。

2. 根据给定分值来删除节点，zslDeleteByScore。

3. 根据给定排名来删除节点，zslDeleteByRank。

删除操作均由zslDeleteNode执行。
```c
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;
    // 更新所有和被删除节点x有关的节点指针，解除它们之间的关系
    for (i = 0; i < zsl->level; i++) {
        if (update[i]->level[i].forward == x) {
            update[i]->level[i].span += x->level[i].span - 1;
            update[i]->level[i].forward = x->level[i].forward;
        } else {
            update[i]->level[i].span -= 1;
        }
    }
    // 更新被删除节点x的前进后退指针
    if (x->level[0].forward) {
        x->level[0].forward->backward = x->backward;
    } else {
        zsl->tail = x->backward;
    }
    // 更新跳跃表的最大层数
    while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
        zsl->level--;

    // 计数器-1
    zsl->length--;
}
```
根据节点的分值和成员删除节点,其余两种情况只是查找方法与判断方式不同。
```c
int zslDelete(zskiplist *zsl, double score, robj *obj) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    // 遍历跳跃表，查找目标节点，并记录所有沿途及节点
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                compareStringObjects(x->level[i].forward->obj,obj) < 0)))
            x = x->level[i].forward;
        update[i] = x;
    }
    
    // 分值和对象相同时，将其删除
    x = x->level[0].forward;
    if (x && score == x->score && equalStringObjects(x->obj,obj)) {
        zslDeleteNode(zsl, x, update);
        zslFreeNode(x);
        return 1;
    }
    return 0; /* not found */
}
```

## 查找给定分值和成员对象的节点在跳跃表中的排位

```c
unsigned long zslGetRank(zskiplist *zsl, double score, robj *o) {
    zskiplistNode *x;
    unsigned long rank = 0;
    int i;

    // 遍历整个跳跃表
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {

        // 遍历节点并对比元素
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                // 比对分值
                (x->level[i].forward->score == score &&
                // 比对成员对象
                compareStringObjects(x->level[i].forward->obj,o) <= 0))) {

            // 累积跨越的节点数量
            rank += x->level[i].span;

            // 沿着前进指针遍历跳跃表
            x = x->level[i].forward;
        }

        // 必须确保不仅分值相等，而且成员对象也要相等
        if (x->obj && equalStringObjects(x->obj,o)) {
            return rank;
        }
    }

    // 没找到
    return 0;
}
```


