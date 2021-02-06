---
title: '[读书笔记] stl源码剖析 第一章、第二章 概论、内存配置器'
date: 2016-11-16 08:32:26
categories: 
- 读书笔记
tags:
- stl源码剖析
---

# 概论
第一章基本都是C++基础的知识，读了《C++ Primer》的话都懂。关于各种C++的特性、STL特性都有。
## STL六大组件 功能与运用
STL提供六大组件，彼此可以组合套用
1. 容器：各种数据结构。Vector,list,deque,set,map
2. 算法：各种常用算法如sort,search,copy,erase
3. 迭代器：扮演容器与算法之间的粘合剂，所谓的泛型指针。五种类型
4. 仿函数：行为类似函数，可以作为算法的某种策略。仿函数是重载了()的class或者class template，一般函数指针可视为狭义的仿函数
5. 配接器：一种用来修饰容器或者仿函数或迭代器接口的东西。
6. 配置器：负责控件的配置与管理。

<!--more-->

<img src="/img/20161108212017.png" alt=""/>
# 空间配置器
SGI STL时间了一个专属的，拥有次层配置能力的、效率优越的特殊配置器。
SGI STL的缺省分配器都是其自己的分配器。
## SGI特殊的空间配置器 std::alloc
使用::construct() ::destroy()构造和析构
使用alloc::allocate() alloc::deallocate()分配 释放
```cpp
//直接利用这个类能够用指定类型指针，转换为其他引用等
template <class _Tp>
struct iterator_traits<const _Tp*> {
  typedef random_access_iterator_tag iterator_category;
  typedef _Tp                         value_type;
  typedef ptrdiff_t                   difference_type;
  typedef const _Tp*                  pointer;
  typedef const _Tp&                  reference;
};
```
```cpp
//如果有non-trivial 析构函数
template <class _ForwardIterator>
void
__destroy_aux(_ForwardIterator __first, _ForwardIterator __last, __false_type)
{
  for ( ; __first != __last; ++__first)
    destroy(&*__first);
}
//如果没有non-trivial 析构函数
template <class _ForwardIterator> 
inline void __destroy_aux(_ForwardIterator, _ForwardIterator, __true_type) {}

template <class _ForwardIterator, class _Tp>
inline void 
__destroy(_ForwardIterator __first, _ForwardIterator __last, _Tp*)
{
  typedef typename __type_traits<_Tp>::has_trivial_destructor
          _Trivial_destructor;
  __destroy_aux(__first, __last, _Trivial_destructor());//_Trivial_destructor()将会是_true_type 或者_false_type
  //利用模板和特化
}

template <class _ForwardIterator>
inline void _Destroy(_ForwardIterator __first, _ForwardIterator __last) {
  __destroy(__first, __last, __VALUE_TYPE(__first));//__VALUE_TYPE(__first)获取到了类型的指针，然后调用上面函数
  //利用模板和特化
}
```
## 空间的配置与释放 std::alloc
sgi设计了双层级配置器，第一级配置器直接使用malloc()和free()，第二级配置器则视情况采用不同的策略。
当分配大于128bytes时候调用第一级，小于128时候，为了降低额外负担，使用负责的memory pool整理方式。
### 第一级配置器 __malloc_alloc_template剖析
以malloc free realloc实现。
然后自己实现了一个new handler机制。
```cpp
template <int __inst>
void*
__malloc_alloc_template<__inst>::_S_oom_malloc(size_t __n)
{
    void (* __my_malloc_handler)();
    void* __result;

    for (;;) {
        __my_malloc_handler = __malloc_alloc_oom_handler;
        if (0 == __my_malloc_handler) { __THROW_BAD_ALLOC; }
        (*__my_malloc_handler)();
        __result = malloc(__n);
        if (__result) return(__result);
    }
}
```
### 第二级配置器
SIG第二级配置器的做法，如果区块够大，超过128bytes时候，就移交第一级配置器处理。当区块小鱼128bytes时候，则以内存池管理。每次配置一大块内存，并维护对应之自由链表，下次若再有相同大小的内存需求从free-lists中拨出。如果客端释放小额区块，就由配置器回收到fres-lists中。
分配区块时候，自动上调到8的倍数，并维护16个fres-lists,分别管理大小为8/16/24/32/40/48/56/64/72/80/88/96/104/112/120/128的小额区块。
> 其实这里和公司用的NG_malloc是一样的策略。ng_malloc之前对于大小的区分是256k的样子，更大。后来又改写了，支持更大的内存块了。

节点如下
```cpp
__PRIVATE:
  union _Obj {
        union _Obj* _M_free_list_link;
        char _M_client_data[1];    /* The client sees this.        */
  };
```
使用union的话，可以不用每次都去转换。char _M_client_data的存在是，方便去取首地址。实际上下面用到的就是_M_free_list_link，用来作为链表。这样的话，就可以用一个4字节的空间(32位机器指针大小)，实现两个功能。而不是使用struct，因为那样就会浪费空间。

#### 空间配置函数allocate()
功能：判断大小，大于128用第一级分配，小于128就查找free list。如果free list中有可用的区块，就直接拿来用，如果没有就将区块大小上调至8的倍数边界，然后调用refill()，准备为free list重新填充空间。
```cpp
  static void* allocate(size_t __n)
  {
    void* __ret = 0;

    //超过设定的最大值就调用第一级配置器，STL设置为128
    if (__n > (size_t) _MAX_BYTES) {
      __ret = malloc_alloc::allocate(__n);
    }
    else {
    //寻找合适的free lists中适当的一个
      _Obj* __STL_VOLATILE* __my_free_list
          = _S_free_list + _S_freelist_index(__n);
      // Acquire the lock here with a constructor call.
      // This ensures that it is released in exit or during stack
      // unwinding.
#     ifndef _NOTHREADS
      /*REFERENCED*/
      //多线程锁
      _Lock __lock_instance;
#     endif
      _Obj* __RESTRICT __result = *__my_free_list;
      if (__result == 0)
        //没找到的话，就重新填充free list
        __ret = _S_refill(_S_round_up(__n));
      else {
      //指向后一个成员
        *__my_free_list = __result -> _M_free_list_link;
        __ret = __result;
      }
    }

    return __ret;
  };
```
#### 空间释放函数 deallocate()
先判断大小，大于128调用第一级，小于128找出对应的free list，将区块回收。
```cpp
/* __p may not be 0 */
  static void deallocate(void* __p, size_t __n)
  {
    if (__n > (size_t) _MAX_BYTES)
      malloc_alloc::deallocate(__p, __n);
    else {
      _Obj* __STL_VOLATILE*  __my_free_list
          = _S_free_list + _S_freelist_index(__n);
      _Obj* __q = (_Obj*)__p;

      // acquire lock
#       ifndef _NOTHREADS
      /*REFERENCED*/
      _Lock __lock_instance;
#       endif /* _NOTHREADS */
      __q -> _M_free_list_link = *__my_free_list;
      *__my_free_list = __q;
      // lock is released here
    }
  }
```
#### 重新填充free lists
```cpp
template <bool __threads, int __inst>
void*
__default_alloc_template<__threads, __inst>::_S_refill(size_t __n)
{
    int __nobjs = 20;
    //尝试分配空间 __nobjs是引用传递，作为返回值
    char* __chunk = _S_chunk_alloc(__n, __nobjs);
    _Obj* __STL_VOLATILE* __my_free_list;
    _Obj* __result;
    _Obj* __current_obj;
    _Obj* __next_obj;
    int __i;
    
    //对只分配出一个的时候的优化
    if (1 == __nobjs) return(__chunk);
    __my_free_list = _S_free_list + _S_freelist_index(__n);

    //形成链表
    /* Build free list in chunk */
      __result = (_Obj*)__chunk;
      *__my_free_list = __next_obj = (_Obj*)(__chunk + __n);
      for (__i = 1; ; __i++) {
        __current_obj = __next_obj;
        __next_obj = (_Obj*)((char*)__next_obj + __n);
        if (__nobjs - 1 == __i) {
            __current_obj -> _M_free_list_link = 0;
            break;
        } else {
            __current_obj -> _M_free_list_link = __next_obj;
        }
      }
    return(__result);
}
```
#### 内存池
从内存池中取空间给free list 使用，是chunk_alloc的工作
```cpp
template <bool __threads, int __inst>
char*
__default_alloc_template<__threads, __inst>::_S_chunk_alloc(size_t __size, 
                                                            int& __nobjs)
{
    char* __result;//返回值
    size_t __total_bytes = __size * __nobjs;//需要分配的空间大小
    size_t __bytes_left = _S_end_free - _S_start_free;//内存池剩余空间

    if (__bytes_left >= __total_bytes) {
        __result = _S_start_free;//返回
        _S_start_free += __total_bytes;//内存池可用空间起始处后移
        return(__result);
    } else if (__bytes_left >= __size) {//能够分配一部分空间
        __nobjs = (int)(__bytes_left/__size);//判断能够分配的块数
        __total_bytes = __size * __nobjs;
        __result = _S_start_free;
        _S_start_free += __total_bytes;//与上同
        return(__result);
    } else {//不能够分配一块的大小
        size_t __bytes_to_get = 
	  2 * __total_bytes + _S_round_up(_S_heap_size >> 4);
        // Try to make use of the left-over piece.
        if (__bytes_left > 0) {//把剩余空间，分配到合适的free list
            _Obj* __STL_VOLATILE* __my_free_list =
                        _S_free_list + _S_freelist_index(__bytes_left);

            ((_Obj*)_S_start_free) -> _M_free_list_link = *__my_free_list;
            *__my_free_list = (_Obj*)_S_start_free;
        }
        
        //从堆上重新分配出部分空间
        _S_start_free = (char*)malloc(__bytes_to_get);
        if (0 == _S_start_free) {
            size_t __i;
            _Obj* __STL_VOLATILE* __my_free_list;
	    _Obj* __p;
            // Try to make do with what we have.  That can't
            // hurt.  We do not try smaller requests, since that tends
            // to result in disaster on multi-process machines.
            for (__i = __size;
                 __i <= (size_t) _MAX_BYTES;
                 __i += (size_t) _ALIGN) {
                __my_free_list = _S_free_list + _S_freelist_index(__i);
                __p = *__my_free_list;
                //malloc失败的话，在现有的free list中找未用的、足够大的fee list分配
                if (0 != __p) {
                    *__my_free_list = __p -> _M_free_list_link;
                    _S_start_free = (char*)__p;
                    _S_end_free = _S_start_free + __i;
                    return(_S_chunk_alloc(__size, __nobjs));
                    // Any leftover piece will eventually make it to the
                    // right free list.
                }
            }
	    _S_end_free = 0;	// In case of exception.
            _S_start_free = (char*)malloc_alloc::allocate(__bytes_to_get);
            // This should either throw an
            // exception or remedy the situation.  Thus we assume it
            // succeeded.
        }
        //拿到新空间
        _S_heap_size += __bytes_to_get;
        _S_end_free = _S_start_free + __bytes_to_get;
        return(_S_chunk_alloc(__size, __nobjs));
    }
}
```
### 内存基本处理工具
uninitialized_copy、uninitialized_fill、uninitialized_fill_n用来处理一些特殊的复制构造的情况。
#### uninitialized_fill_n实现
```cpp

template <class _ForwardIter, class _Size, class _Tp>
inline _ForwardIter 
uninitialized_fill_n(_ForwardIter __first, _Size __n, const _Tp& __x)
{
  return __uninitialized_fill_n(__first, __n, __x, __VALUE_TYPE(__first));
}
```
```cpp
template <class _ForwardIter, class _Size, class _Tp, class _Tp1>
inline _ForwardIter 
__uninitialized_fill_n(_ForwardIter __first, _Size __n, const _Tp& __x, _Tp1*)
{
  typedef typename __type_traits<_Tp1>::is_POD_type _Is_POD;
  return __uninitialized_fill_n_aux(__first, __n, __x, _Is_POD());//判断有没有复制构造函数，调用不同的函数处理
}
```

```cpp
//没有复制构造函数的版本
template <class _ForwardIter, class _Size, class _Tp>
inline _ForwardIter
__uninitialized_fill_n_aux(_ForwardIter __first, _Size __n,
                           const _Tp& __x, __true_type)
{
  return fill_n(__first, __n, __x);
}
//有复制构造函数的版本
template <class _ForwardIter, class _Size, class _Tp>
_ForwardIter
__uninitialized_fill_n_aux(_ForwardIter __first, _Size __n,
                           const _Tp& __x, __false_type)
{
  _ForwardIter __cur = __first;
  __STL_TRY {
    for ( ; __n > 0; --__n, ++__cur)
      _Construct(&*__cur, __x);
    return __cur;
  }
  __STL_UNWIND(_Destroy(__first, __cur));
}
```
uninitialized_copy、uninitialized_fill的实现类似

