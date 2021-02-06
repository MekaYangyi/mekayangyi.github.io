---
title: redis源码阅读-链表
date: 2017-03-26 20:14:40
categories: 
- 源码阅读
tags:
- redis
- 源码阅读
---

C语言程序由于没有标准库的存在，各种造轮子。Redis为了满足需求，同样写了一个链表。

实现文件在adlist.h/adlist.c中。

## 结构体定义
和普通的C写的双向链表差不多。没有什么特点。

```c
typedef struct listNode {
    struct listNode *prev; // 前节点
    struct listNode *next; // 后节点
    void *value; // 值
} listNode;

// 迭代器
typedef struct listIter {
    listNode *next;// 下一个节点
    int direction;// 方向
} listIter;

typedef struct list {
    listNode *head; // 头
    listNode *tail; // 尾
    void *(*dup)(void *ptr); // 自定义节点复制函数
    void (*free)(void *ptr); // 自定义节点释放函数
    int (*match)(void *ptr, void *key); // 自定义节点匹配函数
    unsigned long len; // 链表长度
} list;
```
<!--more-->
### 宏

定义了一些快速操作的宏

```c

/* Functions implemented as macros */
#define listLength(l) ((l)->len) // 获取list长度
#define listFirst(l) ((l)->head) // 获取list头节点指针
#define listLast(l) ((l)->tail) // 获取list尾节点指针
#define listPrevNode(n) ((n)->prev) // 获取当前节点前一个节点
#define listNextNode(n) ((n)->next) // 获取当前节点后一个节点
#define listNodeValue(n) ((n)->value) // 获取当前节点的值

#define listSetDupMethod(l,m) ((l)->dup = (m)) // 设定节点值复制函数
#define listSetFreeMethod(l,m) ((l)->free = (m)) // 设定节点值释放函数
#define listSetMatchMethod(l,m) ((l)->match = (m)) // 设定节点值匹配函数

#define listGetDupMethod(l) ((l)->dup) // 获取节点值赋值函数
#define listGetFree(l) ((l)->free) // 获取节点值释放函数
#define listGetMatchMethod(l) ((l)->match) // 获取节点值匹配函数
```

## API

都是些链表常用的API，比较有特点的是迭代器的C语言实现。
```c
/* Prototypes */
list *listCreate(void);
void listRelease(list *list);
list *listAddNodeHead(list *list, void *value);
list *listAddNodeTail(list *list, void *value);
list *listInsertNode(list *list, listNode *old_node, void *value, int after);
void listDelNode(list *list, listNode *node);
listIter *listGetIterator(list *list, int direction);
listNode *listNext(listIter *iter);
void listReleaseIterator(listIter *iter);
list *listDup(list *orig);
listNode *listSearchKey(list *list, void *key);
listNode *listIndex(list *list, long index);
void listRewind(list *list, listIter *li);
void listRewindTail(list *list, listIter *li);
void listRotate(list *list);
```