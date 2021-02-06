---
title: '[读书笔记] stl源码剖析 第三章 迭代器'
date: 2016-11-17 20:32:26
categories: 
- 读书笔记
tags:
- stl源码剖析
---

迭代器是一种抽象的设计概念。iterator模式：提供一种方法，使能够依序寻访某个聚合物所含的各个元素，而又无需暴露该聚合物的内部表述方式。
# 迭代器的设计思维-stl关键所在
STL的中心思想在于：将数据容器和算法分开，彼此独立设计。容器和算法的泛型化。
迭代器就是扮演着粘胶角色。
# 迭代器是一种smart pointer
list迭代器stl的实现
```cpp
//listnode的基础类
struct _List_node_base {
  _List_node_base* _M_next;
  _List_node_base* _M_prev;
};
//listnode
template <class _Tp>
struct _List_node : public _List_node_base {
  _Tp _M_data;
};

//迭代器基础类
struct _List_iterator_base {
  typedef size_t                     size_type;
  typedef ptrdiff_t                  difference_type;
  typedef bidirectional_iterator_tag iterator_category;

  _List_node_base* _M_node;//包含一个node

  _List_iterator_base(_List_node_base* __x) : _M_node(__x) {}
  _List_iterator_base() {}

  void _M_incr() { _M_node = _M_node->_M_next; }
  void _M_decr() { _M_node = _M_node->_M_prev; }

  bool operator==(const _List_iterator_base& __x) const {
    return _M_node == __x._M_node;
  }
  bool operator!=(const _List_iterator_base& __x) const {
    return _M_node != __x._M_node;
  }
};  
//迭代器
template<class _Tp, class _Ref, class _Ptr>
struct _List_iterator : public _List_iterator_base {
  typedef _List_iterator<_Tp,_Tp&,_Tp*>             iterator;
  typedef _List_iterator<_Tp,const _Tp&,const _Tp*> const_iterator;
  typedef _List_iterator<_Tp,_Ref,_Ptr>             _Self;

  typedef _Tp value_type;
  typedef _Ptr pointer;
  typedef _Ref reference;
  typedef _List_node<_Tp> _Node;

  _List_iterator(_Node* __x) : _List_iterator_base(__x) {}
  _List_iterator() {}
  _List_iterator(const iterator& __x) : _List_iterator_base(__x._M_node) {}

  reference operator*() const { return ((_Node*) _M_node)->_M_data; }

#ifndef __SGI_STL_NO_ARROW_OPERATOR
  pointer operator->() const { return &(operator*()); }
#endif /* __SGI_STL_NO_ARROW_OPERATOR */
  //重载了几个操作实现了迭代器，不是很复杂
  //++i
  _Self& operator++() { 
    this->_M_incr();
    return *this;
  }
  //i++
  _Self operator++(int) { 
    _Self __tmp = *this;
    this->_M_incr();
    return __tmp;
  }
  _Self& operator--() { 
    this->_M_decr();
    return *this;
  }
  _Self operator--(int) { 
    _Self __tmp = *this;
    this->_M_decr();
    return __tmp;
  }
};
```

<!--more-->
# Traits编程技法
之前就见到用过，通过类型获取，其余的类型。
```cpp
template <class _Category, class _Tp, class _Distance = ptrdiff_t,
          class _Pointer = _Tp*, class _Reference = _Tp&>
struct iterator {
  typedef _Category  iterator_category;
  typedef _Tp        value_type;
  typedef _Distance  difference_type;
  typedef _Pointer   pointer;
  typedef _Reference reference;
};

template <class _Iterator>
struct iterator_traits {
  typedef typename _Iterator::iterator_category iterator_category;
  typedef typename _Iterator::value_type        value_type;
  typedef typename _Iterator::difference_type   difference_type;
  typedef typename _Iterator::pointer           pointer;
  typedef typename _Iterator::reference         reference;
};
//对原生指针特化
template <class _Tp>
struct iterator_traits<_Tp*> {
  typedef random_access_iterator_tag iterator_category;
  typedef _Tp                         value_type;
  //ptrdiff_t是C/C++标准库中定义的一个与机器相关的数据类型。ptrdiff_t类型变量通常用来保存两个指针减法操作的结果。ptrdiff_t定义在stddef.h（cstddef）这个文件内。ptrdiff_t通常被定义为long int类型。
  typedef ptrdiff_t                   difference_type;
  typedef _Tp*                        pointer;
  typedef _Tp&                        reference;
};

template <class _Tp>
struct iterator_traits<const _Tp*> {
  typedef random_access_iterator_tag iterator_category;
  typedef _Tp                         value_type;
  typedef ptrdiff_t                   difference_type;
  typedef const _Tp*                  pointer;
  typedef const _Tp&                  reference;
};
```
迭代器的分类：
input iter：只读iter
output iter:只写iter
forward iter:允许写入型算法在这种迭代器所形成的区间操作，前向迭代器
bidirectional iterator：双向移动iter。
random access iter：前三种支持++,第四种支持++ --。这种支持所有指针的算术能力。


<img src="/img/20161116205706.png" alt=""/>
typedef typename _Iterator::iterator_category iterator_category;的作用就是能够在编译器发现迭代器的类型，选用最合适的函数版本。
```cpp
struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator_tag : public input_iterator_tag {};
struct bidirectional_iterator_tag : public forward_iterator_tag {};
struct random_access_iterator_tag : public bidirectional_iterator_tag {};
```
例子
```cpp
template <class _InputIter, class _Distance>
inline void __advance(_InputIter& __i, _Distance __n, input_iterator_tag) {
  while (__n--) ++__i;
}

template <class _BidirectionalIterator, class _Distance>
inline void __advance(_BidirectionalIterator& __i, _Distance __n, 
                      bidirectional_iterator_tag) {
  __STL_REQUIRES(_BidirectionalIterator, _BidirectionalIterator);
  if (__n >= 0)
    while (__n--) ++__i;
  else
    while (__n++) --__i;
}

template <class _RandomAccessIterator, class _Distance>
inline void __advance(_RandomAccessIterator& __i, _Distance __n, 
                      random_access_iterator_tag) {
  __STL_REQUIRES(_RandomAccessIterator, _RandomAccessIterator);
  __i += __n;
}

template <class _InputIterator, class _Distance>
inline void advance(_InputIterator& __i, _Distance __n) {
  __STL_REQUIRES(_InputIterator, _InputIterator);
  //通过下面的函数，选用合适的版本
  //不需要传递参数，能够在编译器选定版本
  __advance(__i, __n, iterator_category(__i));
}
```
# __type_traits
提供一种机制，允许针对不同的型别熟悉，完成函数派送。
在内存配置器中就用到了，比如有没有拷贝构造函数等

```cpp
struct __true_type {
};

struct __false_type {
};
//选择使用__false_type为struct的目的是为了类型推倒，而不是使用bool值作为参数的话只是运行时的选择。
template <class _Tp>
struct __type_traits { 
   typedef __true_type     this_dummy_member_must_be_first;
                   /* Do not remove this member. It informs a compiler which
                      automatically specializes __type_traits that this
                      __type_traits template is special. It just makes sure that
                      things work if an implementation is using a template
                      called __type_traits for something unrelated. */

   /* The following restrictions should be observed for the sake of
      compilers which automatically produce type specific specializations 
      of this class:
          - You may reorder the members below if you wish
          - You may remove any of the members below if you wish
          - You must not rename members without making the corresponding
            name change in the compiler
          - Members you add will be treated like regular members unless
            you add the appropriate support in the compiler. */
 

   typedef __false_type    has_trivial_default_constructor;
   typedef __false_type    has_trivial_copy_constructor;
   typedef __false_type    has_trivial_assignment_operator;
   typedef __false_type    has_trivial_destructor;
   typedef __false_type    is_POD_type;
};

```
然后在这个文件里定义了大量的特化，主要是特化C++标准类型。
```cpp
__STL_TEMPLATE_NULL struct __type_traits<bool> {
   typedef __true_type    has_trivial_default_constructor;
   typedef __true_type    has_trivial_copy_constructor;
   typedef __true_type    has_trivial_assignment_operator;
   typedef __true_type    has_trivial_destructor;
   typedef __true_type    is_POD_type;
};
//等。。。
```
