---
title: '[读书笔记] stl源码剖析 第四章 序列式容器'
date: 2016-11-20 20:32:26
categories: 
- 读书笔记
tags:
- stl源码剖析
---

容器分类
<img src="/img/20161120124915.png" alt=""/>
# vector
vector还是比较简单，也就是一个动态数组，提供了些操作，带三个指针，start/finish/end
核心就是动态调整内存的方法，也就是满了开大一倍空间，拷贝过去。
push_back等操作都是值语义的，拷贝传进来的内容放入到vector中。
因为是连续的空间，所有元素连续存储在一整块内存上，迭代器直接用的原始指针。

# list
list复杂些.
模型和常见的list是一致的，动态创建节点，插入。
```cpp
struct _List_node_base {
  _List_node_base* _M_next;
  _List_node_base* _M_prev;
};

template <class _Tp>
struct _List_node : public _List_node_base {
  _Tp _M_data;
};

//迭代器base
struct _List_iterator_base {
  typedef size_t                     size_type;
  typedef ptrdiff_t                  difference_type;
  typedef bidirectional_iterator_tag iterator_category;

  _List_node_base* _M_node;//节点指针

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

//迭代器,重载了++ -- == * ->等操作
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

  _Self& operator++() { 
    this->_M_incr();
    return *this;
  }
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
# deque
双向开头的连续空间

<img src="/img/20161120163459.png" alt=""/>
比较特别的地方是用一个双指针维护了一个多个连续的空间

<img src="/img/20161120165939.png" alt=""/>
迭代器保持连续的方式
```cpp
inline size_t __deque_buf_size(size_t __size) {
  return __size < 512 ? size_t(512 / __size) : size_t(1);
}

template <class _Tp, class _Ref, class _Ptr>
struct _Deque_iterator {
  typedef _Deque_iterator<_Tp, _Tp&, _Tp*>             iterator;
  typedef _Deque_iterator<_Tp, const _Tp&, const _Tp*> const_iterator;
  static size_t _S_buffer_size() { return __deque_buf_size(sizeof(_Tp)); }

.....

  _Tp* _M_cur;
  _Tp* _M_first;
  _Tp* _M_last;
  _Map_pointer _M_node;

  _Deque_iterator(_Tp* __x, _Map_pointer __y) 
    : _M_cur(__x), _M_first(*__y),
      _M_last(*__y + _S_buffer_size()), _M_node(__y) {}
  _Deque_iterator() : _M_cur(0), _M_first(0), _M_last(0), _M_node(0) {}
  _Deque_iterator(const iterator& __x)
    : _M_cur(__x._M_cur), _M_first(__x._M_first), 
      _M_last(__x._M_last), _M_node(__x._M_node) {}

  reference operator*() const { return *_M_cur; }
#ifndef __SGI_STL_NO_ARROW_OPERATOR
  pointer operator->() const { return _M_cur; }
#endif /* __SGI_STL_NO_ARROW_OPERATOR */

  difference_type operator-(const _Self& __x) const {
    return difference_type(_S_buffer_size()) * (_M_node - __x._M_node - 1) +
      (_M_cur - _M_first) + (__x._M_last - __x._M_cur);
  }

//++
  _Self& operator++() {
    ++_M_cur;
    if (_M_cur == _M_last) {
    //到尾部了就进入下一个区域
      _M_set_node(_M_node + 1);//_M_node + 1就是下一个区域
      _M_cur = _M_first;
    }
    return *this; 
  }
......
//重置first等指针
  void _M_set_node(_Map_pointer __new_node) {
    _M_node = __new_node;
    _M_first = *__new_node;
    _M_last = _M_first + difference_type(_S_buffer_size());
  }
};
```
如果map不足，分配空间
```cpp
template <class _Tp, class _Alloc>
class _Deque_base {
....
protected:
  _Tp** _M_map;
  size_t _M_map_size;  //记录的map的大小
  iterator _M_start;
  iterator _M_finish;
  ....
};
```
空间不足分配空间
```cpp
  void push_back(const value_type& __t) {
    if (_M_finish._M_cur != _M_finish._M_last - 1) {
      construct(_M_finish._M_cur, __t);
      ++_M_finish._M_cur;
    }
    else
      _M_push_back_aux(__t);
  }
  ...
  template <class _Tp, class _Alloc>
void deque<_Tp,_Alloc>::_M_push_back_aux(const value_type& __t)
{
  value_type __t_copy = __t;
  _M_reserve_map_at_back();
  *(_M_finish._M_node + 1) = _M_allocate_node();
  __STL_TRY {
    construct(_M_finish._M_cur, __t_copy);
    _M_finish._M_set_node(_M_finish._M_node + 1);
    _M_finish._M_cur = _M_finish._M_first;
  }
  __STL_UNWIND(_M_deallocate_node(*(_M_finish._M_node + 1)));
}
```
# stack
以deuqe为底部结构，封闭头端开口，就是stack。
没有迭代器。
```cpp
//这里设定了了 以deque为基础
template <class _Tp, 
          class _Sequence __STL_DEPENDENT_DEFAULT_TMPL(deque<_Tp>) >
class stack;

template <class _Tp, class _Seq>
bool operator==(const stack<_Tp,_Seq>& __x, const stack<_Tp,_Seq>& __y);

template <class _Tp, class _Seq>
bool operator<(const stack<_Tp,_Seq>& __x, const stack<_Tp,_Seq>& __y);


template <class _Tp, class _Sequence>
class stack {
.....


#ifdef __STL_MEMBER_TEMPLATES
  template <class _Tp1, class _Seq1>
  friend bool operator== (const stack<_Tp1, _Seq1>&,
                          const stack<_Tp1, _Seq1>&);
  template <class _Tp1, class _Seq1>
  friend bool operator< (const stack<_Tp1, _Seq1>&,
                         const stack<_Tp1, _Seq1>&);
#else /* __STL_MEMBER_TEMPLATES */
  friend bool __STD_QUALIFIER
  operator== __STL_NULL_TMPL_ARGS (const stack&, const stack&);
  friend bool __STD_QUALIFIER
  operator< __STL_NULL_TMPL_ARGS (const stack&, const stack&);
#endif /* __STL_MEMBER_TEMPLATES */

.....

protected:
  _Sequence c;
public:
  stack() : c() {}
  explicit stack(const _Sequence& __s) : c(__s) {}

  bool empty() const { return c.empty(); }
  size_type size() const { return c.size(); }
  reference top() { return c.back(); }
  const_reference top() const { return c.back(); }
  void push(const value_type& __x) { c.push_back(__x); }
  void pop() { c.pop_back(); }
};

```
# queue
同样以deque为基础
```cpp
template <class _Tp, 
          class _Sequence __STL_DEPENDENT_DEFAULT_TMPL(deque<_Tp>) >
class queue;

template <class _Tp, class _Seq>
inline bool operator==(const queue<_Tp, _Seq>&, const queue<_Tp, _Seq>&);

template <class _Tp, class _Seq>
inline bool operator<(const queue<_Tp, _Seq>&, const queue<_Tp, _Seq>&);


template <class _Tp, class _Sequence>
class queue {

.....

#ifdef __STL_MEMBER_TEMPLATES 
  template <class _Tp1, class _Seq1>
  friend bool operator== (const queue<_Tp1, _Seq1>&,
                          const queue<_Tp1, _Seq1>&);
  template <class _Tp1, class _Seq1>
  friend bool operator< (const queue<_Tp1, _Seq1>&,
                         const queue<_Tp1, _Seq1>&);
#else /* __STL_MEMBER_TEMPLATES */
  friend bool __STD_QUALIFIER
  operator== __STL_NULL_TMPL_ARGS (const queue&, const queue&);
  friend bool __STD_QUALIFIER
  operator<  __STL_NULL_TMPL_ARGS (const queue&, const queue&);
#endif /* __STL_MEMBER_TEMPLATES */

.....

protected:
  _Sequence c;
public:
  queue() : c() {}
  explicit queue(const _Sequence& __c) : c(__c) {}

  bool empty() const { return c.empty(); }
  size_type size() const { return c.size(); }
  reference front() { return c.front(); }
  const_reference front() const { return c.front(); }
  reference back() { return c.back(); }
  const_reference back() const { return c.back(); }
  void push(const value_type& __x) { c.push_back(__x); }
  void pop() { c.pop_front(); }
};
```
# heap
堆也就是个完全二叉树。用个数组或者vector就能实现
stl中的heap是保证每个父节点大于等于子节点，每次插入删除节点需要重新调整树

## push_heap
push操作,后调整树，查看父节点是不是符合要求
```cpp
__push_heap(_RandomAccessIterator __first,
            _Distance __holeIndex, _Distance __topIndex, _Tp __value)
{
  _Distance __parent = (__holeIndex - 1) / 2;//父节点
  while (__holeIndex > __topIndex && *(__first + __parent) < __value) {
    //调整节点
    *(__first + __holeIndex) = *(__first + __parent);
    __holeIndex = __parent;
    __parent = (__holeIndex - 1) / 2;//再判断父节点
  }    
  *(__first + __holeIndex) = __value;//把新值移动到新位置
}

template <class _RandomAccessIterator, class _Distance, class _Tp>
inline void 
__push_heap_aux(_RandomAccessIterator __first,
                _RandomAccessIterator __last, _Distance*, _Tp*)
{
  __push_heap(__first, _Distance((__last - __first) - 1), _Distance(0), 
              _Tp(*(__last - 1)));
}

//push_heap算法，这个时候值已经插入heap的最尾端，调用此函数调整heap
template <class _RandomAccessIterator>
inline void 
push_heap(_RandomAccessIterator __first, _RandomAccessIterator __last)
{
  __STL_REQUIRES(_RandomAccessIterator, _Mutable_RandomAccessIterator);
  __STL_REQUIRES(typename iterator_traits<_RandomAccessIterator>::value_type,
                 _LessThanComparable);
  __push_heap_aux(__first, __last,
                  __DISTANCE_TYPE(__first), __VALUE_TYPE(__first));
}

```
## pop_heap
调整相关节点，使得上部全部填满，最底层空出区域。
```cpp
template <class _RandomAccessIterator, class _Distance, class _Tp>
void 
__adjust_heap(_RandomAccessIterator __first, _Distance __holeIndex,
              _Distance __len, _Tp __value)
{
  _Distance __topIndex = __holeIndex;
  _Distance __secondChild = 2 * __holeIndex + 2;
  while (__secondChild < __len) {
    if (*(__first + __secondChild) < *(__first + (__secondChild - 1)))
      __secondChild--;
    *(__first + __holeIndex) = *(__first + __secondChild);
    __holeIndex = __secondChild;
    __secondChild = 2 * (__secondChild + 1);
  }
  if (__secondChild == __len) {
    *(__first + __holeIndex) = *(__first + (__secondChild - 1));
    __holeIndex = __secondChild - 1;
  }
  __push_heap(__first, __holeIndex, __topIndex, __value);
}

template <class _RandomAccessIterator, class _Tp, class _Distance>
inline void 
__pop_heap(_RandomAccessIterator __first, _RandomAccessIterator __last,
           _RandomAccessIterator __result, _Tp __value, _Distance*)
{
  //把first放到last-1
  *__result = *__first;
  //重新整理first到last-1之间重新成为heap
  __adjust_heap(__first, _Distance(0), _Distance(__last - __first), __value);
}

template <class _RandomAccessIterator, class _Tp>
inline void 
__pop_heap_aux(_RandomAccessIterator __first, _RandomAccessIterator __last,
               _Tp*)
{
  //pop出的是最大值，也就是删除掉first，把first放到last-1，然后重新整理first到last-1之间重新成为heap
  //这样之后只要去取last-1的值，就是弹出的值了。
  __pop_heap(__first, __last - 1, __last - 1, 
             _Tp(*(__last - 1)), __DISTANCE_TYPE(__first));
}

//pop
template <class _RandomAccessIterator>
inline void pop_heap(_RandomAccessIterator __first, 
                     _RandomAccessIterator __last)
{
  __STL_REQUIRES(_RandomAccessIterator, _Mutable_RandomAccessIterator);
  __STL_REQUIRES(typename iterator_traits<_RandomAccessIterator>::value_type,
                 _LessThanComparable);
  __pop_heap_aux(__first, __last, __VALUE_TYPE(__first));
}
```
## make_heap
不断调用__adjust_heap调整heap的话，最终会是一个heap
```cpp
template <class _RandomAccessIterator, class _Tp, class _Distance>
void 
__make_heap(_RandomAccessIterator __first,
            _RandomAccessIterator __last, _Tp*, _Distance*)
{
  if (__last - __first < 2) return;
  _Distance __len = __last - __first;
  _Distance __parent = (__len - 2)/2;
    
  while (true) {
    __adjust_heap(__first, __parent, __len, _Tp(*(__first + __parent)));
    if (__parent == 0) return;
    __parent--;
  }
}

template <class _RandomAccessIterator>
inline void 
make_heap(_RandomAccessIterator __first, _RandomAccessIterator __last)
{
  __STL_REQUIRES(_RandomAccessIterator, _Mutable_RandomAccessIterator);
  __STL_REQUIRES(typename iterator_traits<_RandomAccessIterator>::value_type,
                 _LessThanComparable);
  __make_heap(__first, __last,
              __VALUE_TYPE(__first), __DISTANCE_TYPE(__first));
}
```
## sort_heap
对一个heap一直pop，最大值移到末尾，那么就是增序序列 了。
```cpp
template <class _RandomAccessIterator>
void sort_heap(_RandomAccessIterator __first, _RandomAccessIterator __last)
{
  __STL_REQUIRES(_RandomAccessIterator, _Mutable_RandomAccessIterator);
  __STL_REQUIRES(typename iterator_traits<_RandomAccessIterator>::value_type,
                 _LessThanComparable);
  while (__last - __first > 1)
    pop_heap(__first, __last--);
}
```
# priority_queue
优先弹出权值高的项目，用上面的max_heap就能够实现。
操作不多。
```cpp
template <class _Tp, 
          class _Sequence __STL_DEPENDENT_DEFAULT_TMPL(vector<_Tp>),
          class _Compare
          __STL_DEPENDENT_DEFAULT_TMPL(less<typename _Sequence::value_type>) >
class priority_queue {
......
protected:
  _Sequence c;
  _Compare comp;
public:
  priority_queue() : c() {}
  explicit priority_queue(const _Compare& __x) :  c(), comp(__x) {}
  priority_queue(const _Compare& __x, const _Sequence& __s) 
    : c(__s), comp(__x) 
    { make_heap(c.begin(), c.end(), comp); }
.....
  bool empty() const { return c.empty(); }
  size_type size() const { return c.size(); }
  const_reference top() const { return c.front(); }
  void push(const value_type& __x) {
    __STL_TRY {
      c.push_back(__x); 
      push_heap(c.begin(), c.end(), comp);
    }
    __STL_UNWIND(c.clear());
  }
  void pop() {
    __STL_TRY {
      pop_heap(c.begin(), c.end(), comp);
      c.pop_back();
    }
    __STL_UNWIND(c.clear());
  }
};
```
