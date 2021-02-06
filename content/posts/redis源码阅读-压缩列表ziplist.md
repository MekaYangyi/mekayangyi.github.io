---
title: redis源码阅读-压缩列表ziplist
date: 2017-07-08 21:49:02
categories: 
- 源码阅读
tags:
- redis
- 源码阅读
---

压缩列表时列表键和哈希键的底层实现之一。当一个列表键只包含少量列表项，并且每个列表项要么是小整数，要么是长度比较短的字符串，那么Redis就会使用压缩列表来做列表键的底层实现。

## 压缩列表的构成

压缩列表是redis为了节约内存而开发的，由一系列特殊编码的连续内存块组成的顺序型数据结构。一个压缩列表可以包含任意多个节点，每个节点可以保存一个字节数组或者一个整数组。

```
《redis设计与实现》
空白 ziplist 示例图

area        |<---- ziplist header ---->|<-- end -->|

size          4 bytes   4 bytes 2 bytes  1 byte
            +---------+--------+-------+-----------+
component   | zlbytes | zltail | zllen | zlend     |
            |         |        |       |           |
value       |  1011   |  1010  |   0   | 1111 1111 |
            +---------+--------+-------+-----------+
                                       ^
                                       |
                               ZIPLIST_ENTRY_HEAD
                                       &
address                        ZIPLIST_ENTRY_TAIL
                                       &
                               ZIPLIST_ENTRY_END

非空 ziplist 示例图

area        |<---- ziplist header ---->|<----------- entries ------------->|<-end->|

size          4 bytes  4 bytes  2 bytes    ?        ?        ?        ?     1 byte
            +---------+--------+-------+--------+--------+--------+--------+-------+
component   | zlbytes | zltail | zllen | entry1 | entry2 |  ...   | entryN | zlend |
            +---------+--------+-------+--------+--------+--------+--------+-------+
                                       ^                          ^        ^
address                                |                          |        |
                                ZIPLIST_ENTRY_HEAD                |   ZIPLIST_ENTRY_END
                                                                  |
                                                        ZIPLIST_ENTRY_TAIL
```

zlbytes记录整个压缩列表占用的字节数。

zltail记录压缩列表尾节点距离压缩列表的起始地址有多少字节。

zzlen记录压缩列表包含的节点数量。

entryX列表节点，数量不定。

zlend特殊值0xff，标记压缩列表末端。

<!--more-->

## 压缩列表节点的构成

每个压缩列表可以保存一个字节数组或者一个整数值，字节数组可以有三种长度：

1. 长度小于等于63字节的字节数组
2. 长度小于等于16383字节的字节数组
3. 长度小于等于4294967295字节的字节数组

每个压缩列表节点由previous_entry_length、encoding、content三个部分组成。

### previous_entry_length

节点previous_entry_length属性以字节为单位，记录一个压缩列表节点的长度。previous_entry_length属性的长度可以是1字节或者5字节：

- 如果前一节点的长度小于254字节，长度为1,前一节点长度保存在这个字节里面。
- 如果前一节点的长度大于 等于254字节，长度为5：属性的第一个字节被设置为0xfe，而之后的四个字节用于保存前一 节点的长度。

### encoding

记录节点的content属性所保存数据类型以及长度：

- 一字节、两字节或者五字节长，值的最高位为00、01或者10的是 字节数组编码：这种编码表示节点的content属性保存着字节数组，数组的长度由编码除去最高两位之后的其他位记录
- 一字节长，值的最高位以11开头的是整数编码：这种编码表示节点的content属性保存着整数数值，整数值的类型和长度由编码除去最高两位之后的其他位记录。

| 编码                                       | 编码长度    | content属性保存的值           |
| :--------------------------------------- | ------- | ----------------------- |
| 00bbbbbb                                 | 1 bytes | <= 63 bytes的字节数组        |
| 01bbbbbb xxxxxxxx                        | 2 bytes | <= 16383 bytes字节数组      |
| 10**__** aaaaaaaa bbbbbbbb cccccccc dddddddd | 5 bytes | <= 4294967295 bytes字节数组 |

| 编码       | 编码长度 | content属性保存的值        |
| -------- | ---- | -------------------- |
| 11000000 | 1    | int16_t（2 bytes）类型整数 |
| 11010000 | 1    | int32_t（4 bytes）类型整数 |
| 11100000 | 1    | int64_t（8 bytes）类型整数 |
| 11110000 | 1    | 24位有符整数              |
| 11111110 | 1    | 8位有符整数               |
| 1111xxxx | 1    | 0~12                 |

### content

节点的content属性负责保存节点的值，节点值可以是一个字节数组或者整数，值的类型和长度由节点的encoding属性决定。

## 连锁更新

previous_entry_length属性的长度可以是1字节或者5字节：

- 如果前一节点的长度小于254字节，长度为1,前一节点长度保存在这个字节里面。
- 如果前一节点的长度大于 等于254字节，长度为5：属性的第一个字节被设置为0xfe，而之后的四个字节用于保存前一 节点的长度。

那么一种情况：在压缩列表中，有多个连续的长度介于250字节到253字节之间的节点e1至eN。

在e1前插入一个大于254字节的节点，此时要更新e1的previous_entry_length属性，由于前一个节点大于254，那么要扩容，重新设置好压缩列表。之后又发现e2需要更新previous_entry_length属性，依旧大于254。产生了连锁反应。

同理删除节点也可能发生这种情况。

## 压缩列表API

### 创建空ziplist

```c
// 创建一个空的ziplist
unsigned char *ziplistNew(void) {
    // ZIPLIST_HEADER_SIZE 是 ziplist 表头的大小
    // 1 字节是表末端 ZIP_END 的大小
    unsigned int bytes = ZIPLIST_HEADER_SIZE+1;
    unsigned char *zl = zmalloc(bytes);
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes); // 设置ziplist所占字节数，如有必要进行大小端转换
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE); // 设定尾节点相对头部的偏移量
    ZIPLIST_LENGTH(zl) = 0; // 设定ziplist的节点数
    zl[bytes-1] = ZIP_END; // 设定尾部字节位0xff
    return zl;
}
```

### 插入节点

```c
unsigned char *ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
    return __ziplistInsert(zl,p,s,slen);
}

// 将长度为slen的字符串s插入到z1中，位置为p前
static unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
    size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), reqlen; // 当前ziplist长度，插入后的长度
    unsigned int prevlensize, prevlen = 0; // 前置节点长度和编码该长度所需要的长度
    size_t offset;
    int nextdiff = 0;
    unsigned char encoding = 0;
    long long value = 123456789; /* initialized to avoid warning. Using a value
                                    that is easy to see if for some reason
                                    we use it uninitialized. */
    zlentry tail;

    /* Find out prevlen for the entry that is inserted. */
    // 找到待插入节点的前置节点长度
    if (p[0] != ZIP_END) {
        // 不为末尾解码长度
        ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
    } else {
        // 指向末尾则表示ziplist为空
        unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl);
        if (ptail[0] != ZIP_END) {
            // 计算尾节点长度
            prevlen = zipRawEntryLength(ptail);
        }
    }

    // 判断编码是否为整数
    if (zipTryEncoding(s,slen,&value,&encoding)) {
        // 该节点编码为整数，通过encoding来获取编码长度
        reqlen = zipIntSize(encoding);
    } else {
        // 使用字符串来编码节点
        reqlen = slen;
    }
     // 计算前置节点长度所需的大小
    reqlen += zipPrevEncodeLength(NULL,prevlen);
    // 计算编码当前节点值所需要的大小
    reqlen += zipEncodeLength(NULL,encoding,slen);

    // 保存新旧编码之间的字节差
    nextdiff = (p[0] != ZIP_END) ? zipPrevLenByteDiff(p,reqlen) : 0;

    offset = p-zl; // 保存偏移
    // 重分配长度
    zl = ziplistResize(zl,curlen+reqlen+nextdiff);
    p = zl+offset;

    if (p[0] != ZIP_END) {
        
        // 移动元素，为新元素腾出位置
        memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);

        // 将新节点的长度编码到后置节点
        zipPrevEncodeLength(p+reqlen,reqlen);

        // 更新尾部的偏移量
        ZIPLIST_TAIL_OFFSET(zl) =
            intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+reqlen);

        /* When the tail contains more than one entry, we need to take
         * "nextdiff" in account as well. Otherwise, a change in the
         * size of prevlen doesn't have an effect on the *tail* offset. */
        tail = zipEntry(p+reqlen);
        if (p[reqlen+tail.headersize+tail.len] != ZIP_END) {
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
        }
    } else {
        /* This element will be the new tail. */
        ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(p-zl);
    }

    /* When nextdiff != 0, the raw length of the next entry has changed, so
     * we need to cascade the update throughout the ziplist */
    // 判断是不是需要连锁更新
    if (nextdiff != 0) {
        offset = p-zl;
        zl = __ziplistCascadeUpdate(zl,p+reqlen);
        p = zl+offset;
    }

    /* Write the entry */
    // 写入节点前置节点长度
    p += zipPrevEncodeLength(p,prevlen);
    // 节点值的长度写入节点
    p += zipEncodeLength(p,encoding,slen);
    // 写入节点值
    if (ZIP_IS_STR(encoding)) {
        memcpy(p,s,slen);
    } else {
        zipSaveInteger(p,value,encoding);
    }
    // 更新节点计数器
    ZIPLIST_INCR_LENGTH(zl,1);
    return zl;
}
```

### 根据给定索引，遍历列表，并返回索引指定节点的指针。

```c
unsigned char *ziplistIndex(unsigned char *zl, int index) {
    unsigned char *p;
    unsigned int prevlensize, prevlen = 0;
    // index为负从尾部，正从头部
    if (index < 0) {
        index = (-index)-1;
        // 获取尾部指针
        p = ZIPLIST_ENTRY_TAIL(zl);
        if (p[0] != ZIP_END) {
            // 解码前置节点长度
            ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
            while (prevlen > 0 && index--) {
                p -= prevlen; // 偏移
                // 解码前置节点长度
                ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
            }
        }
    } else {
        p = ZIPLIST_ENTRY_HEAD(zl); // 头部
        while (p[0] != ZIP_END && index--) {
            p += zipRawEntryLength(p); // 移动
        }
    }

    // 返回
    return (p[0] == ZIP_END || index > 0) ? NULL : p;
}
```

### 删除给定节点

```c
unsigned char *ziplistDelete(unsigned char *zl, unsigned char **p) {
    size_t offset = *p-zl;
    zl = __ziplistDelete(zl,*p,1);

    /* Store pointer to current element in p, because ziplistDelete will
     * do a realloc which might result in a different "zl"-pointer.
     * When the delete direction is back to front, we might delete the last
     * entry and end up with "p" pointing to ZIP_END, so check this. */
    *p = zl+offset;
    return zl;
}

static unsigned char *__ziplistDelete(unsigned char *zl, unsigned char *p, unsigned int num) {
    unsigned int i, totlen, deleted = 0;
    size_t offset;
    int nextdiff = 0;
    zlentry first, tail;

    // 计算被删除节点总共占用的内存字节数
    // 删除的节点总数
    first = zipEntry(p);
    for (i = 0; p[0] != ZIP_END && i < num; i++) {
        p += zipRawEntryLength(p);
        deleted++;
    }

    totlen = p-first.p; // 被删除节点总共占用内存字节数
    if (totlen > 0) {
        if (p[0] != ZIP_END) {
            
            // 计算新旧前置节点字节数差
            nextdiff = zipPrevLenByteDiff(p,first.prevrawlen);
            p -= nextdiff; // 有需要将p后退nextdiff字节，为新header空出空间
            zipPrevEncodeLength(p,first.prevrawlen); // 将first的前置节点长度编码至p中

            /* Update offset for tail */
            // 更新尾部偏移
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))-totlen);

            /* When the tail contains more than one entry, we need to take
             * "nextdiff" in account as well. Otherwise, a change in the
             * size of prevlen doesn't have an effect on the *tail* offset. */
            // 被删除节点之后还存在节点，就需要将nextdiff计算在内
            tail = zipEntry(p);
            if (p[tail.headersize+tail.len] != ZIP_END) {
                ZIPLIST_TAIL_OFFSET(zl) =
                   intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
            }

            /* Move tail to the front of the ziplist */
            // 将删除节点后面内存空间移动到删除节点之后
            memmove(first.p,p,
                intrev32ifbe(ZIPLIST_BYTES(zl))-(p-zl)-1);
        } else {
            /* The entire tail was deleted. No need to move memory. */
            // 被删除节点后无节点，不需要移动
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe((first.p-zl)-first.prevrawlen);
        }

        /* Resize and update length */
        // 更新ziplist长度
        offset = first.p-zl;
        zl = ziplistResize(zl, intrev32ifbe(ZIPLIST_BYTES(zl))-totlen+nextdiff);
        ZIPLIST_INCR_LENGTH(zl,-deleted);
        p = zl+offset;

        /* When nextdiff != 0, the raw length of the next entry has changed, so
         * we need to cascade the update throughout the ziplist */
         // 看看是否需要连锁更新
        if (nextdiff != 0)
            zl = __ziplistCascadeUpdate(zl,p);
    }
    return zl;
}
```

## 小结

整体上ziplist设计出来的目的是为了节省内存，采用了在连续内存空间上建立一个双向列表来实现。

