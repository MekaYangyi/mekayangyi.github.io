# Redis源码阅读-订阅与发布


Redis的发布与订阅主要是实现客户端订阅一个频道或者模式，当某客户端向一个频道发送消息时，该频道或者匹配模式订阅者都能够收到消息。

## 频道的订阅与退订

Redis在服务器结构体中的pubsub_channels字典中保存了所有的频道订阅关系。pubsub_channels键为频道，值为订阅的客户端组成的链表。

客户端结构体的pubsub_channels保存了客户端订阅的所有频道，pubsub_channels的键为频道，值为空。

```c
struct redisServer {
  // ...
  dict *pubsub_channels;  // 保存所有的频道订阅关系
  // ...
}

struct redisClient {
  // ...
  dict *pubsub_channels; // 记录客户端订阅的频道
  // ...
}
```

## 订阅

源码如下

```c
// 订阅命令处理函数
void subscribeCommand(redisClient *c) {
    int j;

    // 遍历指令中的所有频道
    for (j = 1; j < c->argc; j++)
        pubsubSubscribeChannel(c,c->argv[j]);
    c->flags |= REDIS_PUBSUB;
}

// 设置客户端c订阅频道channel
int pubsubSubscribeChannel(redisClient *c, robj *channel) {
    dictEntry *de;
    list *clients = NULL;
    int retval = 0;

    /* Add the channel to the client -> channels hash table */
    // 将channels加倒c->c->pubsub_channels的字典里
    if (dictAdd(c->pubsub_channels,channel,NULL) == DICT_OK) {
        retval = 1;
        incrRefCount(channel);
        /* Add the client to the channel -> list of clients hash table */
        // 找出服务器中的频道
        de = dictFind(server.pubsub_channels,channel);
        
        // 不存在就添加一个频道
        // 获取客户端链表
        if (de == NULL) {
            clients = listCreate();
            dictAdd(server.pubsub_channels,channel,clients);
            incrRefCount(channel);
        } else {
            clients = dictGetVal(de);
        }

        // 添加到客户端链表尾部
        listAddNodeTail(clients,c);
    }
    /* Notify the client */
    // 回复客户端
    addReply(c,shared.mbulkhdr[3]);
    addReply(c,shared.subscribebulk);
    addReplyBulk(c,channel);
    addReplyLongLong(c,clientSubscriptionsCount(c));
    return retval;
}
```

<!--more-->
## 退订

源码如下

```c
void unsubscribeCommand(redisClient *c) {
    if (c->argc == 1) {
        // 退订所有频道
        pubsubUnsubscribeAllChannels(c,1);
    } else {
        int j;

        // 遍历频道一一退订
        for (j = 1; j < c->argc; j++)
            pubsubUnsubscribeChannel(c,c->argv[j],1);
    }
    if (clientSubscriptionsCount(c) == 0) c->flags &= ~REDIS_PUBSUB;
}

// 退订所有频道
int pubsubUnsubscribeAllChannels(redisClient *c, int notify) {
    dictIterator *di = dictGetSafeIterator(c->pubsub_channels);
    dictEntry *de;
    int count = 0;

    // 遍历一一退订
    while((de = dictNext(di)) != NULL) {
        robj *channel = dictGetKey(de);

        count += pubsubUnsubscribeChannel(c,channel,notify);
    }
    /* We were subscribed to nothing? Still reply to the client. */
    if (notify && count == 0) {
        addReply(c,shared.mbulkhdr[3]);
        addReply(c,shared.unsubscribebulk);
        addReply(c,shared.nullbulk);
        addReplyLongLong(c,dictSize(c->pubsub_channels)+
                       listLength(c->pubsub_patterns));
    }
    dictReleaseIterator(di);
    return count;
}
// 客户端退订频道
int pubsubUnsubscribeChannel(redisClient *c, robj *channel, int notify) {
    dictEntry *de;
    list *clients;
    listNode *ln;
    int retval = 0;

    /* Remove the channel from the client -> channels hash table */
    incrRefCount(channel); /* channel may be just a pointer to the same object
                            we have in the hash tables. Protect it... */

    // 移除客户端字典中频道的订阅
    if (dictDelete(c->pubsub_channels,channel) == DICT_OK) {
        retval = 1;
        /* Remove the client from the channel -> clients list hash table */
        // 找到服务器频道
        de = dictFind(server.pubsub_channels,channel);
        redisAssertWithInfo(c,NULL,de != NULL);
        clients = dictGetVal(de); // 获取链表
        ln = listSearchKey(clients,c); // 寻找链表中的订阅
        redisAssertWithInfo(c,NULL,ln != NULL);
        listDelNode(clients,ln); // 删除节点
        if (listLength(clients) == 0) {
            /* Free the list and associated hash entry at all if this was
             * the latest client, so that it will be possible to abuse
             * Redis PUBSUB creating millions of channels. */
            // 删除节点后 链表为空 删除字典中的节点
            dictDelete(server.pubsub_channels,channel);
        }
    }
    /* Notify the client */
    // 回复客户端
    if (notify) {
        addReply(c,shared.mbulkhdr[3]);
        addReply(c,shared.unsubscribebulk);
        addReplyBulk(c,channel);
        addReplyLongLong(c,dictSize(c->pubsub_channels)+
                       listLength(c->pubsub_patterns));

    }
    decrRefCount(channel); /* it is finally safe to release it */
    return retval;
}
```

## 模式的订阅与退订
Redis在服务器结构体中的pubsub_patterns链表中保存了所有订阅模式关系。使用pubsubPattern结构的数据作为节点。

客户端结构体的pubsub_patterns保存了客户端订阅的所有模式，节点使用pubsubPattern结构的数据。

```c
struct redisServer {
  // ...
  list *pubsub_patterns;  // 所有订阅模式关系
  // ...
}

struct redisClient {
  // ...
  list *pubsub_patterns;  // 订阅模式关系
  // ...
}

typedef struct pubsubPattern {
    client *client; // 订阅模式的客户端
    robj *pattern;  // 被订阅模式
} pubsubPattern;
```

## 订阅

```c
// 订阅模式
void psubscribeCommand(redisClient *c) {
    int j;

    // 遍历命令中的模式
    for (j = 1; j < c->argc; j++)
        pubsubSubscribePattern(c,c->argv[j]);
    c->flags |= REDIS_PUBSUB;
}

// 设置客户端c订阅模式pattern
int pubsubSubscribePattern(redisClient *c, robj *pattern) {
    int retval = 0;

    // 客户端模式链表中查找模式
    // 为空则创建
    if (listSearchKey(c->pubsub_patterns,pattern) == NULL) {
        retval = 1;
        pubsubPattern *pat;
        // 客户端模式链表尾部添加模式
        listAddNodeTail(c->pubsub_patterns,pattern);
        incrRefCount(pattern);

        // 服务端模式链表尾部添加模式
        pat = zmalloc(sizeof(*pat));
        pat->pattern = getDecodedObject(pattern);
        pat->client = c;
        listAddNodeTail(server.pubsub_patterns,pat);
    }
    /* Notify the client */
    addReply(c,shared.mbulkhdr[3]);
    addReply(c,shared.psubscribebulk);
    addReplyBulk(c,pattern);
    addReplyLongLong(c,clientSubscriptionsCount(c));
    return retval;
}
```

## 退订

```c
// 退订模式
void punsubscribeCommand(redisClient *c) {
    if (c->argc == 1) {
        // 退订所有模式
        pubsubUnsubscribeAllPatterns(c,1);
    } else {
        int j;

        // 退订命令中的模式
        for (j = 1; j < c->argc; j++)
            pubsubUnsubscribePattern(c,c->argv[j],1);
    }
    if (clientSubscriptionsCount(c) == 0) c->flags &= ~REDIS_PUBSUB;
}

// 退订客户端c订阅的所有模式
int pubsubUnsubscribeAllPatterns(redisClient *c, int notify) {
    listNode *ln;
    listIter li;
    int count = 0;

    // 遍历，一一退订
    listRewind(c->pubsub_patterns,&li);
    while ((ln = listNext(&li)) != NULL) {
        robj *pattern = ln->value;

        count += pubsubUnsubscribePattern(c,pattern,notify);
    }
    if (notify && count == 0) {
        /* We were subscribed to nothing? Still reply to the client. */
        addReply(c,shared.mbulkhdr[3]);
        addReply(c,shared.punsubscribebulk);
        addReply(c,shared.nullbulk);
        addReplyLongLong(c,dictSize(c->pubsub_channels)+
                       listLength(c->pubsub_patterns));
    }
    return count;
}

 // 取消客户端c对模式pattern的订阅
int pubsubUnsubscribePattern(redisClient *c, robj *pattern, int notify) {
    listNode *ln;
    pubsubPattern pat;
    int retval = 0;

    incrRefCount(pattern); /* Protect the object. May be the same we remove */

    // 订阅了才进行操作
    if ((ln = listSearchKey(c->pubsub_patterns,pattern)) != NULL) {
        retval = 1;

        // 从客户端订阅中删除
        listDelNode(c->pubsub_patterns,ln);
        pat.client = c;
        pat.pattern = pattern;

        // 从服务端订阅中删除
        ln = listSearchKey(server.pubsub_patterns,&pat);
        listDelNode(server.pubsub_patterns,ln);
    }
    /* Notify the client */
    if (notify) {
        addReply(c,shared.mbulkhdr[3]);
        addReply(c,shared.punsubscribebulk);
        addReplyBulk(c,pattern);
        addReplyLongLong(c,dictSize(c->pubsub_channels)+
                       listLength(c->pubsub_patterns));
    }
    decrRefCount(pattern);
    return retval;
}
```


## 发送消息

```c
// 发布消息
void publishCommand(redisClient *c) {
    int receivers = pubsubPublishMessage(c->argv[1],c->argv[2]);

    // 暂时不考虑集群
    if (server.cluster_enabled)
        clusterPropagatePublish(c->argv[1],c->argv[2]);
    else
        forceCommandPropagation(c,REDIS_PROPAGATE_REPL);
    addReplyLongLong(c,receivers);
}

// 将消息发送到所有订阅了频道的客户端
int pubsubPublishMessage(robj *channel, robj *message) {
    int receivers = 0;
    dictEntry *de;
    listNode *ln;
    listIter li;

    /* Send to clients listening for that channel */
    // 先查找订阅了频道的
    de = dictFind(server.pubsub_channels,channel);
    if (de) {

        // 获取链表
        list *list = dictGetVal(de);
        listNode *ln;
        listIter li;

        // 遍历链表 发送消息
        listRewind(list,&li);
        while ((ln = listNext(&li)) != NULL) {
            redisClient *c = ln->value;

            addReply(c,shared.mbulkhdr[3]);
            addReply(c,shared.messagebulk);
            addReplyBulk(c,channel);
            addReplyBulk(c,message);
            receivers++;
        }
    }
    /* Send to clients listening to matching channels */

    if (listLength(server.pubsub_patterns)) {

        // 遍历模式链表
        listRewind(server.pubsub_patterns,&li);
        channel = getDecodedObject(channel);
        while ((ln = listNext(&li)) != NULL) {
            pubsubPattern *pat = ln->value;

            // 匹配模式 发送消息
            if (stringmatchlen((char*)pat->pattern->ptr,
                                sdslen(pat->pattern->ptr),
                                (char*)channel->ptr,
                                sdslen(channel->ptr),0)) {
                addReply(pat->client,shared.mbulkhdr[4]);
                addReply(pat->client,shared.pmessagebulk);
                addReplyBulk(pat->client,pat->pattern);
                addReplyBulk(pat->client,channel);
                addReplyBulk(pat->client,message);
                receivers++;
            }
        }
        decrRefCount(channel);
    }
    return receivers;
}

```


