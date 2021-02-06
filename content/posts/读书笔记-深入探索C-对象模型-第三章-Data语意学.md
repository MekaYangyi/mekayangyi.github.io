---
title: '[读书笔记] 深入探索C++对象模型 第三章 Data语意学'
date: 2016-08-14 20:35:34
categories: 
- 读书笔记
tags:
- C++
- 理论
- 读书笔记
---

# 第三章：Data语意学
class的大小：
内存对齐
空Class需要1byte来占位，说明是独一无二的存在
有虚函数时候会有虚表指针
static 成员不属于类对象，不占空间

## Data Member的绑定
早期的编译器可能看不到Class后面的内容，导致数据成员的用了外层的同名的。现在已经没有这种情况了。
```cpp
typedef long long length;
class Point3d
{
public:
    length x;//是long long
    typedef int length;    
    length y;//是 int
};
```

<!--more-->
## Data Member的布局
Nonstatic data members在class object中的排列顺序是和声明顺序一致的。
但是没有规定多个access sections中的数据成员的排列，可以自由排列。实际上的编译器处理来看，还是按照顺序来的。这些顺序成员依靠声明次序在一个连续的区域里。
虚表也有强制的规定放在尾部还是头部等位置。

## Data Member的存取
下面的一段代码中的x的存取成本
```cpp
    Point3d origin;
    origin.x = 0.0;
```
需要视x和Point3d如何声明而定。

## Static Data Member的存取
Static Data Member实际上是一种全局变量，放在类对象之外。每一个Static Data Member对象的存取操作都会转化为：
```cpp
Point3d::x == 250;//不论是. 或者->来存取
```
不论该class是单一的类还是继承有虚函数的类或者多重继承，Static Data Member的路径仍然是这么直接。

编译器会给每一个Static Data Member暗中编码指定独一无二的名字。

## NonStatic Data Member的存取
NonStatic Data Member是存放在每一个类对象中的，除非经由明确的或者暗喻的类对象，否则么有办法直接存取它们。

```cpp
void Point3d::translate()
{
    x = 1;
    y = 2;
    z = 3;
}
```
实际上都会转化为
```cpp
void Point3d::translate(Point3d *const this)
{
    this-> x = 1;
    this->y = 2;
    this->z = 3;
}
```
欲对一个NonStatic Data Member访问必须在类对象的起始地址加上一个数据成员的偏移值。

```cpp
origin._y = 0.0;
//origin._y的地址实际上等于
&origin + (&Point3d::_y - 1);
```
存取这种数据成员和一个struct的成员是一致的。除了虚拟继承，此种情况下会引入一层间接性。

## 继承与Data Member
在C++中的继承模型中，一个derived class object所表现出来的东西，是其自己的members加上其base lass members的总和。

### 只要继承不要多态
base class的padding也会随之而来。这种设计是为了防止把派生类赋值给基类对象的时候导致基类对象的后面部位有数据而不是0（如果不保留padding的话，而是直接接上去）

<img src="/img/20160814 inside C++ 0.jpg" alt=""/>

### 加上多态
加上多态的额外负担：
> * 导入一个相关的virtual table,用来存放它所声明的每一个虚函数的地址。这个table元素的数目一般而言是被声明的虚函数的数目，再加上一个或者两个slots（用来支持runtime type idetification）。
> *  在每一个类对象中导入一个vptr，提供执行器的连接，使每一个对象能够找到对应的vittual table。
> * 加强构造函数
> * 加强虚构函数

vptr的放置没有强制规定。

### 多重继承
多重继承实际上对于第一继承没什么影响，主要是对于第二或者后继的base class
```cpp
Vertex3d v3d;
Vertex *pv;
Point2d *p2d;
Point3d *p3d;

pv = &v3d;
//需要这样的内部转化
pv = (Vertex*)(((char*)&v3d + sizeof(Point3d));
//下面的只要简单的拷贝地址就行
p2d = &v3d;

pv = pv3d;
//要考虑pv3d位0的情况
pv = pv3d
        ? (Vertex*)(((char*)&v3d + sizeof(Point3d
        : 0;
```
多重继承的存储模型，没有规定怎么存储，但是大部分编译器都是采用下面的方式存储的。

<img src="/img/20160814 inside C++ 1.jpg" alt=""/>

### 虚拟继承
一般的实现方法：class如果含有一个或者多个virtual base class，分割为亮哥部分:一个是不变的局部，一个是共享的局部。不变的局部中的数据，不管后继如何衍化，总有固定的offset，所以这一部分数据可以被直接存取。至于共享局部，所表现的是virtual base class suboject.这一部分的数据，其位置会因为每次的派生操作而变化，所以它们只可以被间接存取。

<img src="/img/20160814 inside C++ 2.jpg" alt=""/>

例子继承体系
微软的解决方案：引入所谓的virtual base class table.每一个类对象如果有一个或者多个virtual base classes就会由编译器安插一个指针，指向virtual base class table。至于真正的virtual base class指针，被安放在这个表格中。
第二种解决方案

<img src="/img/20160814 inside C++ 3.jpg" alt=""/>

在virtula function table中放置 virtula base class的offset。
如同虚函数的指针，虚函数指针的在虚表中是正的offset，负的部分就给了虚基类。
```cpp
(this +__vptr__Point3d[-3]->_x += (&rhs + +__vptr__Point3d[-3]->_x)
```

```cpp
Point3d *p2d = pv3d;
//转换为
Point3d *p2d = pv3d ? pv3d +__vptr__Point3d[-3] : 0;
```
一般而言，virtual base class 最有效的一种运行形势是：一个抽象的virtual base class，没有任何data members。
## 对象成员的效率

## 指向Data Members的指针
& Point3d::z;的值为z在class object中的偏移量。

书上的代码试了在VS2015，并没有偏移量增加1的出现
```cpp
    printf("%d", (&Point3d::x));//4
    printf("%d", (&Point3d::y));//8
    printf("%d", (&Point3d::z));//c
```
