---
title: redis源码阅读-动态字符串sds
date: 2017-03-26 19:45:58
categories: 
- 源码阅读
tags:
- redis
- 源码阅读
---

Redis没有使用c语言的字符串结构，自己设计了一个简单的动态字符串。特点是：修改时大小不足则扩容，大小足够直接使用不缩小。末尾使用‘\0’，与c语言字符串兼容。

sds的源代码在sds.h与sds.c中。

## sds的定义

```c
typedef char *sds;// 兼容C

struct sdshdr {
    unsigned int len;// 字符串长度
    unsigned int free;// 未分配的空间
    char buf[];// 末尾'/0'的C风格字符串
};// SDS的实际结构，兼容char*则返回buf地址
```

## SDS这样设计的优点：

1. 重用部分C字符串库函数的函数。
2. 在常数复杂度的情况下获取字符串长度(以下代码)。
3. 杜绝缓冲区溢出，通过获取空余空间函数，来进行处理(sdscat函数)。
4. 减少字符串内存的重分配。不足则分配更大的空间，足够也不减少空间，而是记录新的len、free值。
5. 二进制兼容。C字符串以空字符结尾，而某些二进制数据中间可能存在空字符。SDS兼容该种数据。

<!--more-->

## SDS API
### 获取长度
```c

// 获取字符串长度
static inline size_t sdslen(const sds s) {
    struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr)));
    return sh->len;
}

// 获取空余空间
static inline size_t sdsavail(const sds s) {
    struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr)));
    return sh->free;
}
```
### SDS创建

有两个函数，一个定长创建，一个是不定长创建。

```c
sds sdsnewlen(const void *init, size_t initlen) {
    struct sdshdr *sh;

    if (init) {
        //为空则使用malloc
        sh = zmalloc(sizeof(struct sdshdr)+initlen+1);
    } else {
        //不为空使用calloc分配
        sh = zcalloc(sizeof(struct sdshdr)+initlen+1);
    }
    if (sh == NULL) return NULL;// 分配失败处理
    //设定sds的参数
    sh->len = initlen;
    sh->free = 0;
    //值的复制
    if (initlen && init)
        memcpy(sh->buf, init, initlen);
    sh->buf[initlen] = '\0';// 尾部结束符
    return (char*)sh->buf;
}

// 复制一个char*
sds sdsnew(const char *init) {
    size_t initlen = (init == NULL) ? 0 : strlen(init);
    return sdsnewlen(init, initlen);
}

// 生成一个空sd
sds sdsempty(void) {
    return sdsnewlen("",0);
}

// 复制一个sds
sds sdsdup(const sds s) {
    return sdsnewlen(s, sdslen(s));
}
```

### sds释放函数

先获取sdshdr的首地址，使用zfree释放。

```c
void sdsfree(sds s) {
    if (s == NULL) return;
  	// 获取真实首地址释放
    zfree(s-sizeof(struct sdshdr));
}
```

### sds动态空间调整

```c
// 空间增长
sds sdsMakeRoomFor(sds s, size_t addlen) {
    struct sdshdr *sh, *newsh;
    size_t free = sdsavail(s);
    size_t len, newlen;

    if (free >= addlen) return s;// 空间足够直接返回
    len = sdslen(s);
    sh = (void*) (s-(sizeof(struct sdshdr)));
    newlen = (len+addlen);// 新的长度
    if (newlen < SDS_MAX_PREALLOC)// 不足1MB直接翻倍分配
        newlen *= 2;
    else// 超过1MB，多分配1MB空余空间
        newlen += SDS_MAX_PREALLOC;
    newsh = zrealloc(sh, sizeof(struct sdshdr)+newlen+1);// 分配空间
    if (newsh == NULL) return NULL; // 分配失败
    // 设置参数
    newsh->free = newlen - len;
    return newsh->buf;
}

// 空间的重分配
sds sdsRemoveFreeSpace(sds s) {
    struct sdshdr *sh;

    sh = (void*) (s-(sizeof(struct sdshdr)));
    sh = zrealloc(sh, sizeof(struct sdshdr)+sh->len+1);
    sh->free = 0;
    return sh->buf;
}
```

### sds连接操作

```c
sds sdscatlen(sds s, const void *t, size_t len) {
    struct sdshdr *sh;
    size_t curlen = sdslen(s); // 获取字符串长度

    s = sdsMakeRoomFor(s,len);// 扩展字符串
    if (s == NULL) return NULL;
    sh = (void*) (s-(sizeof(struct sdshdr)));
    memcpy(s+curlen, t, len);// 连接字符串到末尾
    sh->len = curlen+len;// 设置长度
    sh->free = sh->free-len;
    s[curlen+len] = '\0';// 设置尾部
    return s;
}

sds sdscat(sds s, const char *t) {
    return sdscatlen(s, t, strlen(t));
}
```

### sds复制

```c
sds sdscpylen(sds s, const char *t, size_t len) {
    struct sdshdr *sh = (void*) (s-(sizeof(struct sdshdr)));
    size_t totlen = sh->free+sh->len;

    // 空间不足，分配空间
    if (totlen < len) {
        s = sdsMakeRoomFor(s,len-sh->len);
        if (s == NULL) return NULL;
        sh = (void*) (s-(sizeof(struct sdshdr)));
        totlen = sh->free+sh->len;
    }
    // 复制
    memcpy(s, t, len);
    s[len] = '\0';
    sh->len = len;
    sh->free = totlen-len;
    return s;
}

sds sdscpy(sds s, const char *t) {
    return sdscpylen(s, t, strlen(t));
}
```

### 一些其他接口

```c

sds sdscatfmt(sds s, char const *fmt, ...);// 格式化输出
sds sdstrim(sds s, const char *cset); // 去除cset中所含字符
void sdsrange(sds s, int start, int end);// 获取指定区间字符串
void sdsupdatelen(sds s); // 更新字符串长度
void sdsclear(sds s); // 清空字符串
int sdscmp(const sds s1, const sds s2); // 字符串比较
// 依据sep将s分割，返回 一个二维数组
sds *sdssplitlen(const char *s, int len, const char *sep, int seplen, int *count);
// 释放由sdssplitlen函数解析的二维数组 
void sdsfreesplitres(sds *tokens, int count);
void sdstolower(sds s); // 转小写
void sdstoupper(sds s); // 转大写
sds sdsfromlonglong(long long value);// ll转sds
sds sdsjoin(char **argv, int argc, char *sep); // 以分隔符连接字符串子数组构成新的字符串
```