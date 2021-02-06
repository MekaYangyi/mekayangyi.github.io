# [读书笔记]深入探索C++对象模型 第四章 Function语意学

# 第三章：Function语意学

## Member的各种调用方式
### Nonstatic Member Functions
C++的设计准则之一就是:nonstatic member function 至少必须和一般的nonmember function有相同的效率。
nonstatic member function会转换为nonmember形式。
1. 改写函数原型，安插一个额外的参数，也就是this指针。
2. 将每一个对“nonstatic data member的存取操作"改为经由this指针来存取。
3. 将member function重写成一个外部函数，对函数名称进行“mangling"处理，使它在程序中成为一个独一无二的词汇。

```cpp
void normalize__7Point3dFv(register const Point3d *const this,
                                                Point3d &__result)
{
    register float mag = this->magnitude();
    __result.Point3d::Point3d();
    
    __result._x = this->_x/mag;
}
```

<!--more-->
### 名称的特殊处理
一般而言，member的名称前面会加上class名称，形成独一无二的命名。
```cpp
class Bar{ public: int ival; ...};
//其中ival有可能变成这样
ival__3Bar
```
主要考虑是存在继承的问题。
```cpp
//防止函数的重名
//加入类名
//加入参数链，使得支持重载
//cfront采用的编码方式
class Point{
public:
    void x__5PointFf(float newX);
    float x__5PointFv();
}
```
### Virtual Member Functions
如果normalize()是virtual member function
```cpp
//函数会变成
ptr-> normalize()
(*ptr->vptr[1])(ptr); //1为vtbl slot的索引值，关联到normalize
```
### Static Member Function
你可能会看到形如以下的调用
```cpp
((Point3d*)0)->test();
```
这种调用在test中没有对类对象的数据成员存储时候是不会出错的。
因为根据之前的转化形式看，没有是还用this指针进行操作。
这个式子的功能实际上就是实现static 成员函数的功能。在static member function成为c++的标准之前。
函数的转化
```cpp
void Point3d::object_cout()
{

}
//转化为
void object_cout__5Point3dSFv()
{
}
```
## Virtual Member Functions
对于多态的class object身上增加两个members
1. 一个字符串或者数字，表示class的类型
2. 一个指针，指向某表格，表格中带有程序的virtual functions的执行地址

一个class只会有一个virtual table，每个table内含其对应的class object中所有active virtual functions函数实体地址。这些active virtual functions包括：
>* 这个class所定义的函数实体。它会改写一个可能存在的Base class virtual function函数实体
>* 继承自base class的函数实体。这是派生类不改写的部分
>* 一个Pure_virtual_called()函数实体，它既扮演pure virtual function的空间保卫者角色，也可以作为执行期异常处理函数（有时候会用到）


<img src="/img/20160815 inside C++ 1.jpg" alt=""/>
￼
### 多重继承下的Virtual Functions
在多重继承中支持virtual functions，其复杂度围绕在第二个以及后继的base classes中，以及必须在执行期间调整的this指针这一点。
```cpp
class C : public A,B...
A* pA = new C;

B* pB = new C;
//
C* pC = new C;
B* pB = pC;

//转化
B* pB = pB ? pB + sizeof(A) : 0;
//
delete pB;
```
delete 操作会要求调用合适的虚析构函数，那么就要求指针再一次被调整。使得指针再次指向C对象的头。

thunk是小段assembly代码，来完成这个工作。
```cpp
pbase2_dtor_thunk:
    this += sizeof(base1);
    Derived::~Derived(this);
```
### 多重继承下的Virtual Functions
没讲明白
## 函数的效能
## 指向Member Function的指针
```cpp
//指向成员函数的指针声明
double (Point :: *coord) () = &Point :: x;
coord = &Point::y;

//要想要调用，需要
(origin.*coord)();
(ptr->*coord)();
//操作会自动被编译器转化
(coord) (&origin);
(coord)(ptr);
```
指向member function的指针的声明语法，以及指向member selection运算符的指针，其作用是作为this指针的空间保留者。这就是为什么static member functions的类型是函数指针，而不是指向member function指针的原因。
利用上述方式去获取一个虚函数的指针，一样能够支持多态。因为实际上获取的是一个索引值，指向虚表的内容。
### 在多重继承下，指向member funcitons的指针
为了让指向member funcitons的指针能够支持多重继承和虚拟继承，Stroustrup设计了下面一个机构体

index faddr分别带有virtual table和nonvirtual member function地址（为了方便，index不指向virtual table时候会被设为-1）。
```cpp
struct __mpter
{
    int delta;
    int index;
    union
    {
        ptrtofunc faddr;
        int            v_offset;
    }
}
//在该模型之下，像这样的操作
(ptr->*pmf)();
//转变为
(pmf.index < 0)
    ? (*pmf.faddr)(ptr)// nonvirtual invocation
    : (*ptr->vptr[pmf.index](ptr));//virtual invocation
```


