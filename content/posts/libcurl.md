---
title: 高性能libcurl
date: 2020-11-14 16:04:59
categories: 
- 技巧
tags:
- curl
---

本文主要介绍一下curl接口的使用方法，以及获取高性能的一些实践。

## 什么是libcurl？用处？

```
libcurl is a free and easy-to-use client-side URL transfer library, supporting DICT, FILE, FTP, FTPS, Gopher, HTTP, HTTPS, IMAP, IMAPS, LDAP, LDAPS, MQTT, POP3, POP3S, RTMP, RTMPS, RTSP, SCP, SFTP, SMTP, SMTPS, Telnet and TFTP. libcurl supports SSL certificates, HTTP POST, HTTP PUT, FTP uploading, HTTP form based upload, proxies, cookies, user+password authentication (Basic, Digest, NTLM, Negotiate, Kerberos), file transfer resume, http proxy tunneling and more!
```

官网摘的一段，意思大概是免费且容易使用的url传输库，支持下面一堆特性。

项目内用处：基于libcurl实现了一个单线程异步的httpc，用于msdk、防沉迷、信用分等需要接入外部http的服务。

## 接口介绍

curl一共有三种接口：

- [Easy Interface](https://curl.haxx.se/libcurl/c/libcurl-easy.html)
- [Multi Interface](https://curl.haxx.se/libcurl/c/libcurl-multi.html)
- [Share Interface](https://curl.haxx.se/libcurl/c/libcurl-share.html)

### Easy Interface

```
The easy interface is a synchronous, efficient, quickly used and... yes, easy interface for file transfers. Numerous applications have been built using this.
```

简单接口是一个同步接口，非常方便使用，接口以curl_easy_开头，下面一个简单的使用例子。

```cpp
CURL *curl = curl_easy_init(); // 创建一个handle
if(curl) {
  CURLcode res;
  curl_easy_setopt(curl, CURLOPT_URL, "https://example.com"); // 设置参数，参数有很多，header、cookie、接收回调函数、证书等等
  res = curl_easy_perform(curl); // 执行，注意这里是阻塞的
  curl_easy_cleanup(curl); // 清理
}
```

### Multi Interface

```
The multi interface is the asynchronous brother in the family and it also offers multiple transfers using a single thread and more. 
```

多重接口是异步接口，libcurl使用一个或者多个线程完成数据传输，通过多重接口可以再单线程下同时操作多个easy handle。

例子：https://gist.github.com/clemensg/4960504

```cpp
#define MAX_WAIT_MSECS 30*1000 /* Wait max. 30 seconds */

static const char *urls[] = {
  "http://www.microsoft.com",
  "http://www.yahoo.com",
  "http://www.wikipedia.org",
  "http://slashdot.org"
};
#define CNT 4

static size_t cb(char *d, size_t n, size_t l, void *p)
{
  /* take care of the data here, ignored in this example */
  (void)d;
  (void)p;
  return n*l;
}

static void init(CURLM *cm, int i)
{
  CURL *eh = curl_easy_init();
  curl_easy_setopt(eh, CURLOPT_WRITEFUNCTION, cb);
  curl_easy_setopt(eh, CURLOPT_HEADER, 0L);
  curl_easy_setopt(eh, CURLOPT_URL, urls[i]);
  curl_easy_setopt(eh, CURLOPT_PRIVATE, urls[i]);
  curl_easy_setopt(eh, CURLOPT_VERBOSE, 0L);
  curl_multi_add_handle(cm, eh);
}

int main(void)
{
    CURLM *cm = curl_multi_init(); // 创建一个multi handle

    // 创建一堆easy handle加入到multi handle里
    for (int i = 0; i < CNT; ++i) {
        init(cm, i);
    }

    // 执行，等待所有都执行完
    int still_running=0;
    curl_multi_perform(cm, &still_running);
    do {
      	// 等一段时间，MAX_WAIT_MSECS为最长时长
        // 有事件或者超时了返回
        int res = curl_multi_wait(cm, NULL, 0, MAX_WAIT_MSECS, &numfds);
        if(res != CURLM_OK) {
            fprintf(stderr, "error: curl_multi_wait() returned %d\n", res);
            return EXIT_FAILURE;
        }
        curl_multi_perform(cm, &still_running);
    } while(still_running);

    // 读取数据
    CURLMsg *msg=NULL;
    const char *szUrl;
    int msgs_left = 0;
    int http_status_code;
    while ((msg = curl_multi_info_read(cm, &msgs_left))) {
        if (msg->msg == CURLMSG_DONE) {
    		CURL *eh = msg->easy_handle;
            CURLcode return_code = msg->data.result;
            if(return_code!=CURLE_OK) {
                fprintf(stderr, "CURL error code: %d\n", msg->data.result);
                continue;
            }

            // Get HTTP status code
            http_status_code=0;
            szUrl = NULL;
            curl_easy_getinfo(eh, CURLINFO_RESPONSE_CODE, &http_status_code);
            curl_easy_getinfo(eh, CURLINFO_PRIVATE, &szUrl);

            if(http_status_code==200) {
                printf("200 OK for %s\n", szUrl);
            } else {
                fprintf(stderr, "GET of %s returned http status code %d\n", szUrl, http_status_code);
            }

            curl_multi_remove_handle(cm, eh);
            curl_easy_cleanup(eh);
        } else {
            fprintf(stderr, "error: after curl_multi_info_read(), CURLMsg=%d\n", msg->msg);
        }
    }

    // 关闭
    curl_multi_cleanup(cm);

    return EXIT_SUCCESS;
}
```

### Share Interface

```
The share interface was added to enable sharing of data between curl "handles".
```

用于再多个easy handle共享一些数据，比如dns cache、tls session。

### 其他

curl_global_init：进程启动时初始化curl

curl_global_cleanup：进程关闭时清理curl

使用异步DNS，防止同步解析DNS卡住主循环

## 实践

1. 最初实现了一个使用Multi Interface的HTTPC。

   - httpc/unittest/httpc.cpp

      ```cpp
      TEST_F(HTTPC_TEST, HTTP_GET)                                                 
      {
          HTTPC_CFG cfg;
          REQUEST_DRIVER driver(10);
          HTTPC c(cfg, driver);
          
          auto cb = [&](RESPONSE &rsp) {
              printf("result:%d code:%ld \n", rsp._result, rsp._http_status_code);
              printf("body:%s", rsp._body.c_str());
          };
      
          for (u32 i = 0; i < 10; ++i) {
              c.get("http://www.baidu.com", cb);
          }
      
          bool idle;
          while (!driver._ctx_map.empty()) {
              driver.loop(idle, 0);
          };
      }
      ```

2. 在压测过程中发现性能不足，DNS解析卡住主循环。根据https://moz.com/devblog/high-performance-libcurl-tips，增加了c-ares库，使得libcurl支持异步DNS，最终性能到了3000TPS。

3. 尝试使用Share Interface共享DNS，效果在接入异步DNS后不明显。

   ```cpp
   // 初始化共享
   pthread_mutex_t _dns_lock = PTHREAD_MUTEX_INITIALIZER
   CURLSH* _share_handler = curl_share_init();
   curl_share_setopt(_share_handler, CURLSHOPT_USERDATA, &_dns_lock);
   curl_share_setopt(_share_handler, CURLSHOPT_LOCKFUNC, _lock);
   curl_share_setopt(_share_handler, CURLSHOPT_UNLOCKFUNC, _unlock);
   curl_share_setopt(_share_handler, CURLSHOPT_SHARE, CURL_LOCK_DATA_DNS);
   
   // 设置共享
   curl_easy_setopt(ctx->_requst->_curl_handle, CURLOPT_SHARE, _share_handler);
   curl_easy_setopt(ctx->_requst->_curl_handle, CURLOPT_DNS_CACHE_TIMEOUT, 5 * 60);//5min
   ```

4. 请求处理完毕后，不适用curl_easy_cleanup，而是采用连接池的方式通过curl_easy_reset重用连接。

   - https://curl.se/libcurl/c/libcurl-tutorial.html#Persistence

     ```
     Re-cycling the same easy handle several times when doing multiple requests is the way to go.
     
     After each single curl_easy_perform operation, libcurl will keep the connection alive and open. A subsequent request using the same easy handle to the same host might just be able to use the already open connection! This reduces network impact a lot.
     
     Even if the connection is dropped, all connections involving SSL to the same host again, will benefit from libcurl's session ID cache that drastically reduces re-connection time.
     
     ....
     
     libcurl caches DNS name resolving results, to make lookups of a previously looked up name a lot faster.
     ```

   - https://curl.se/libcurl/c/curl_easy_reset.html

      ```
      Re-initializes all options previously set on a specified CURL handle to the default values. This puts back the handle to the same state as it was in when it was just created with curl_easy_init.
      
      It does not change the following information kept in the handle: live connections, the Session ID cache, the DNS cache, the cookies, the shares or the alt-svc cache.
      ```


