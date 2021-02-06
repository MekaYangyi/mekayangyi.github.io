# redis源码阅读-内存分配


redis的内存分配主要就是对malloc和free进行了一层简单的封装。具体的实现在zmalloc.h和zmalloc.c中。

## 内存分配器的选择

redis支持多种内存分配器，可以选择的分配器tcmalloc、jemalloc、dlmalloc、malloc/malloc.h(apple)，这几个分配器自带内存大小的记录。如果不进行设置，则默认使用malloc进行分配。

<!--more-->

```cpp
// zmalloc.h
// 选择使用的内存分配器，分别为tcmalloc、jemalloc、dlmalloc、malloc/malloc.h
// 同时设置HAVE_MALLOC_SIZE为真，内存分配器自带大小统计。
// 如果不选择内存分配器，则使用默认的malloc，同时同时设置HAVE_MALLOC_SIZE为假。
// 指定zmalloc_size
#if defined(USE_TCMALLOC)
#define ZMALLOC_LIB ("tcmalloc-" __xstr(TC_VERSION_MAJOR) "." __xstr(TC_VERSION_MINOR))
#include <google/tcmalloc.h>
#if (TC_VERSION_MAJOR == 1 && TC_VERSION_MINOR >= 6) || (TC_VERSION_MAJOR > 1)
#define HAVE_MALLOC_SIZE 1
#define zmalloc_size(p) tc_malloc_size(p)
#else
#error "Newer version of tcmalloc required"
#endif

#elif defined(USE_JEMALLOC)
#define ZMALLOC_LIB ("jemalloc-" __xstr(JEMALLOC_VERSION_MAJOR) "." __xstr(JEMALLOC_VERSION_MINOR) "." __xstr(JEMALLOC_VERSION_BUGFIX))
#include <jemalloc/jemalloc.h>
#if (JEMALLOC_VERSION_MAJOR == 2 && JEMALLOC_VERSION_MINOR >= 1) || (JEMALLOC_VERSION_MAJOR > 2)
#define HAVE_MALLOC_SIZE 1
#define zmalloc_size(p) je_malloc_usable_size(p)
#else
#error "Newer version of jemalloc required"
#endif

#elif defined(__APPLE__)
#include <malloc/malloc.h>
#define HAVE_MALLOC_SIZE 1
#define zmalloc_size(p) malloc_size(p)

#elif defined(USE_DLMALLOC)
#include "Win32_Interop/win32_dlmalloc.h"
#define ZMALLOC_LIB ("dlmalloc-" __xstr(2) "." __xstr(8) )
#define HAVE_MALLOC_SIZE 1
#define zmalloc_size(p)  g_msize(p)
#endif

#ifndef ZMALLOC_LIB
#define ZMALLOC_LIB "libc"
#endif

// zmalloc.c
// 依据选择的内存分配器，设定好宏定义，否则使用系统默认分配。
#if defined(USE_TCMALLOC)
#define malloc(size) tc_malloc(size)
#define calloc(count,size) tc_calloc(count,size)
#define realloc(ptr,size) tc_realloc(ptr,size)
#define free(ptr) tc_free(ptr)
#elif defined(USE_JEMALLOC)
#define malloc(size) je_malloc(size)
#define calloc(count,size) je_calloc(count,size)
#define realloc(ptr,size) je_realloc(ptr,size)
#define free(ptr) je_free(ptr)
#elif defined(USE_DLMALLOC)
#define malloc(size) g_malloc(size)
#define calloc(count,size) g_calloc(count,size)
#define realloc(ptr,size) g_realloc(ptr,size)
#define free(ptr) g_free(ptr)
#endif
```

## 功能函数一栏

``` cpp
void *zmalloc(size_t size);// 调用malloc，分配size大小的空间
void *zcalloc(size_t size);// 调用calloc，分配size大小的空间
void *zrealloc(void *ptr, size_t size);// 调用realloc，重新分配size大小的空间
void zfree(void *ptr);// 释放ptr
char *zstrdup(const char *s);// c风格字符串copy
size_t zmalloc_used_memory(void); // 获取当前占用的内存大小
void zmalloc_enable_thread_safeness(void); // 设置线程安全
void zmalloc_set_oom_handler(void (*oom_handler)(size_t)); // 设置内存分配失败的处理方法
float zmalloc_get_fragmentation_ratio(size_t rss);// 获取所给内存和已经使用的内存大小之比
size_t zmalloc_get_rss(void); // 获取RSS信息（Resident Set Size）
size_t zmalloc_get_private_dirty(void);// 获取实际内存大小
size_t zmalloc_get_smap_bytes_by_field(char *field);// 获取/proc/self/smaps字段的字节数
void zlibc_free(void *ptr); // 获取物理内存大小
WIN32_ONLY(void zmalloc_free_used_memory_mutex(void);) //原始系统free的释放方法
```

## 统计使用的内存总数

redis每次分配内存、释放内存都会进行记录。用来统计redis使用的总内存。

```cpp
size_t zmalloc_used_memory(void) {// 获取使用的内存，直接获取used_memory的变量的值。
    size_t um;

    if (zmalloc_thread_safe) {// 线程安全
#if defined(__ATOMIC_RELAXED) || defined(HAVE_ATOMIC)
        um = update_zmalloc_stat_add(0);
#else
        pthread_mutex_lock(&used_memory_mutex);
        um = used_memory;
        pthread_mutex_unlock(&used_memory_mutex);
#endif
    }
    else {// 非线程安全
        um = used_memory;
    }

    return um;
}
```



```cpp
static size_t used_memory = 0;// 定义了一个全局变量，用来记录使用的内存数量。
static int zmalloc_thread_safe = 0;// 默认不线程安全，调用zmalloc_enable_thread_safeness进行设置为线程安全。
#ifdef _WIN32// 根据系统选择多线程锁。
pthread_mutex_t used_memory_mutex;
#else
pthread_mutex_t used_memory_mutex = PTHREAD_MUTEX_INITIALIZER;
#endif

// __ATOMIC_RELAXED提供原子加减操作
#if defined(__ATOMIC_RELAXED)
#define update_zmalloc_stat_add(__n) __atomic_add_fetch(&used_memory, (__n), __ATOMIC_RELAXED)
#define update_zmalloc_stat_sub(__n) __atomic_sub_fetch(&used_memory, (__n), __ATOMIC_RELAXED)
#elif defined(HAVE_ATOMIC)// GCC提供的原子加减操作
#define update_zmalloc_stat_add(__n) __sync_add_and_fetch(&used_memory, (__n))
#define update_zmalloc_stat_sub(__n) __sync_sub_and_fetch(&used_memory, (__n))
#else// 使用多线程锁来实现多线程加减操作
#define update_zmalloc_stat_add(__n) do { \
    pthread_mutex_lock(&used_memory_mutex); \
    used_memory += (__n); \
    pthread_mutex_unlock(&used_memory_mutex); \
} while(0)

#define update_zmalloc_stat_sub(__n) do { \
    pthread_mutex_lock(&used_memory_mutex); \
    used_memory -= (__n); \
    pthread_mutex_unlock(&used_memory_mutex); \
} while(0)

#endif

// 增加redis内存计数
#define update_zmalloc_stat_alloc(__n) do { \
    size_t _n = (__n); \
    if (_n&(sizeof(PORT_LONG)-1)) _n += sizeof(PORT_LONG)-(_n&(sizeof(PORT_LONG)-1)); \ // 将n调整为sizeof(PORT_LONG)的整数倍
    if (zmalloc_thread_safe) { \ // 开启线程安全
        update_zmalloc_stat_add(_n); \
    } else { \
        used_memory += _n; \ // 不开启线程安全
    } \
} while(0)

// 减少redis内存计数
#define update_zmalloc_stat_free(__n) do { \
    size_t _n = (__n); \
    if (_n&(sizeof(PORT_LONG)-1)) _n += sizeof(PORT_LONG)-(_n&(sizeof(PORT_LONG)-1)); \ // 将n调整为sizeof(PORT_LONG)的整数倍
    if (zmalloc_thread_safe) { \
        update_zmalloc_stat_sub(_n); \// 开启线程安全
    } else { \
        used_memory -= _n; \// 不开启线程安全
    } \
} while(0)
```

## 内存管理函数

### 异常处理函数

异常处理函数, 在内存分配失败时进行调用。

默认使用zmalloc_default_oom，也可以通过zmalloc_set_oom_handler进行设置异常处理方式。

```cpp
static void (*zmalloc_oom_handler)(size_t) = zmalloc_default_oom;

void zmalloc_set_oom_handler(void (*oom_handler)(size_t)) {
    zmalloc_oom_handler = oom_handler;
}

static void zmalloc_default_oom(size_t size) {
    fprintf(stderr, "zmalloc: Out of memory trying to allocate %Iu bytes\n",    WIN_PORT_FIX /* %zu -> %Iu */
        size);// 打印日志
    fflush(stderr);
    abort();// 中断退出
}
```




### zmalloc

zmalloc用来分配指定大小的内存。实际上对malloc进行了一层封装，加入了异常处理和内存统计。

```cpp
void *zmalloc(size_t size) {
    // 调用malloc进行内存分配
    // 多出的PREFIX_SIZE大内存用来记录该段内存大小。
    void *ptr = malloc(size+PREFIX_SIZE);

    // 分配失败的异常处理
    if (!ptr) zmalloc_oom_handler(size);

    // 统计内存使用
#ifdef HAVE_MALLOC_SIZE

    // 内存分配器自带内存大小
    update_zmalloc_stat_alloc(zmalloc_size(ptr));
    return ptr;
#else
    *((size_t*)ptr) = size;
    update_zmalloc_stat_alloc(size+PREFIX_SIZE);
    return (char*)ptr+PREFIX_SIZE;
#endif
}
```

### zcalloc

calloc是分配内存，并初始化为0。封装的和zmalloc类似。

```cpp
void *zcalloc(size_t size) {
    // 分配内存
    void *ptr = calloc(1, size+PREFIX_SIZE);

    // 分配失败的异常处理
    if (!ptr) zmalloc_oom_handler(size);

    // 统计内存使用
#ifdef HAVE_MALLOC_SIZE
    update_zmalloc_stat_alloc(zmalloc_size(ptr));
    return ptr;
#else
    *((size_t*)ptr) = size;
    update_zmalloc_stat_alloc(size+PREFIX_SIZE);
    return (char*)ptr+PREFIX_SIZE;
#endif
}
```

### zrealloc

zrealloc用来重新调整分配的内存大小。

```cpp
void *zrealloc(void *ptr, size_t size) {
#ifndef HAVE_MALLOC_SIZE
    void *realptr;
#endif
    size_t oldsize;
    void *newptr;

    // ptr为空，直接使用zmalloc进行分配size大小内存。
    if (ptr == NULL) return zmalloc(size);

#ifdef HAVE_MALLOC_SIZE
    oldsize = zmalloc_size(ptr); // 获取ptr指向的内存大小
    newptr = realloc(ptr,size); // 重新分配内存
    if (!newptr) zmalloc_oom_handler(size); // 异常处理

    update_zmalloc_stat_free(oldsize);// 先减
    update_zmalloc_stat_alloc(zmalloc_size(newptr));// 后加
    return newptr;
#else
    realptr = (char*)ptr-PREFIX_SIZE;
    oldsize = *((size_t*)realptr);// 获取ptr指向的内存大小
    newptr = realloc(realptr,size+PREFIX_SIZE);// 重新分配内存
    if (!newptr) zmalloc_oom_handler(size);// 异常处理

    *((size_t*)newptr) = size;
    update_zmalloc_stat_free(oldsize);// 先减
    update_zmalloc_stat_alloc(size);// 后加
    return (char*)newptr+PREFIX_SIZE;
#endif
}
```

### zfree

释放函数

```cpp
void zfree(void *ptr) {
#ifndef HAVE_MALLOC_SIZE
    void *realptr;
    size_t oldsize;
#endif

    if (ptr == NULL) return;// 空直接返回
#ifdef HAVE_MALLOC_SIZE
    update_zmalloc_stat_free(zmalloc_size(ptr));// 减少统计数量
    free(ptr);// 释放
#else
    realptr = (char*)ptr-PREFIX_SIZE; // 获取真实的指针
    oldsize = *((size_t*)realptr);// 获取大小
    update_zmalloc_stat_free(oldsize+PREFIX_SIZE);// 减少统计数量
    free(realptr);// 释放
#endif
}
```

### zmalloc_size

获取指针指向内存大小，在内存分配器不自带该函数时定义。

```cpp
// zmalloc.h
#ifndef HAVE_MALLOC_SIZE
size_t zmalloc_size(void *ptr);
#endif
// zmalloc.c
#ifndef HAVE_MALLOC_SIZE
size_t zmalloc_size(void *ptr) {
    void *realptr = (char*)ptr-PREFIX_SIZE;
    size_t size = *((size_t*)realptr);
    /* Assume at least that all the allocations are padded at sizeof(PORT_LONG) by
     * the underlying allocator. */
    if (size&(sizeof(PORT_LONG)-1)) size += sizeof(PORT_LONG)-(size&(sizeof(PORT_LONG)-1));
    return size+PREFIX_SIZE;
}
#endif
```



