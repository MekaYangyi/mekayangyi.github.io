# [读书笔记]深入探索C++ 对象模型 第一章 关于对象

# 阅读的目的
在阅读《C++ primer》的时候，书里面写了各种各样的情况下，C++的处理方式。囫囵吞枣的记下后，也就不求甚解了。而C++在工作中的使用已经有了那么一段时间。但是对于C++的很多现象，却依旧是只知道是这样，却不知道为什么是这样。
那么阅读这本同样是 Lippman的书籍，就是为了解惑，为什么C++会导致我们看到的现象，而不是其他情况。
我希望，通过这么本书，能够解答我的一部分疑问。
# 序
工作里常听到的对于C++的抱怨是C++的编译器为程序员做了太多的服务，导致很多情况不受控制。不像C,大部分都需要手动去执行，可以明确的知道，什么时候做了什么。
我想这部分抱怨一方面来源于对于C++的不熟悉，一方面又来源于C++的特性。那么当对C++怎么实现各种特性了解后，对于编译器的行为有了概念后，我相信我应该能够解答很多疑问了。
就像Lippman在本书贴出的一封信件一般
<img src="/img/20160807 inside C++ 0.jpg" alt=""/>

他希望这本书是这些问题的解答。
<!--more-->

# 第一章：关于对象
本章主要是对于对象模型的一个大概浏览，但是对多重继承和虚拟继承等情况没有太多的观察
C语言，数据和处理数据操作是分开声明的，语言本身没有支持数据和函数的关联性。由一组分布在各个以功能为导向的函数中的算法所驱动，它们的处理的是共同的外部数据。
C++的实现使用的是ADT
```cpp
classPoint {
public:
    Point(float x);
private:
    float _x;
};
```
这种形式将数据和函数相关联，数据封装。
那么这样的数据封装有成本吗？没有
所有的Class只会生成出一个函数实体。data members则是直接包含在class object中与C的struct一致。
C++的布局和存储时间的额外负担是由virtual引起的：
> * virtual function机制 用来支持一个有效率的“执行期绑定”。 
> * virtual base class 用来实现“多次出现在继承体系中的base class,有一个单一而被共享的实体”。

此外，还有一些多重继承下的额外负担。

# C++的对象模型
C++中存在两种数据成员 static、nostatic，三种成员函数 static、nostatic、virtual。
```cpp
classPoint {
public:
    Point(float xval);
    virtual ~Point();
    float x() const;
    staticint PointCount();
protected:
    virtual ostream&
    print(ostream &os) const;
    float_x;
    staticint_point_count;
};
```
<img src="/img/20160807 inside C++ 1.jpg" alt=""/>

上图为C++的对象模型：
> * nostatic data members 被配置在每一个class object之内。
> * static data members 被存放在所有的class object之外。
> * staitic 和 nostatic function members存放在class object之外。
> * virtual functions以两个步骤支持：
> * ①每一个class产生出一堆指向virtual functions 的指针，放在表格之中。也就是虚表。
> * ②每一个class object被添加了一个指针，指向相关的virtual table。通常这个指针被称为vptr。vptr的设和重置由每一个classd constructor destructor 和 copy assignmemt运算符自动完成。每一个class所关联的type_info object用意支持runtiome tyoe identification，rtti也经由virtual table被指出来，通常是放在第一个slot处。

## 加上继承
c++支持单一继承、多重继承、虚拟继承。
<img src="/img/20160807 inside C++ 2.jpg" alt=""/>

具体的讨论需要3.4中见到。

## 对象模型如何影响程序
class X定义了一个拷贝构造函数，一个虚析构函数，一个虚函数foo();
```cpp
X foobar()
{
    X xx;
    X *px = new X;
    xx.foo();
    px->foo();
    delete px;
    return x;
}
```
这个函数可能在内部转换为
```cpp
//可能的内部转换结果
//虚拟C++码
void foobar(X& _result)
{
    //使用引用返回，属于编译器的优化了。
    //构造
    _result.X::X();

    //申请内存
    px = _new(sizeof(X));
    //调用构造函数
    if( px != 0)
        px->X::X();

    //成员函数的形式的转换，成员函数就是普通函数不过有一个this指针
    foo(&_result);

    //虚函数的基本调用方式，通过vptr来调用
    (*px->vtbl[2])(px);

    //调用虚析构函数
    if( px != 0)
    {
        (*px->vtbl[1])(px);
        _delete(px);
    }

    //不需要使用named return statement
    //不需要摧毁Local object xx
    //而是使用了传入参数_result
    return;
}
```

# 关键词所带来的差异

## 策略性正确的struct
把单一元素的数组放在一个struct的尾端，于是每个 struct objects 可以拥有可变大小的数组：
```cpp
struct mumble {
    char pc[1];
};

//获取一个字符串，然后为struct本身和该字符串配置足够的内存
struct mumble *pmumbl = (struct mumble*)malloc(sizeof(struct
mumble) + strlen(string) + 1);
strcpy(pmumbl->pc, string);
```
这在C++中是很有问题的。不一定在pc后面就没有存放数据。

# 对象的差异
C++支持多种程序设计典范：程序模型、抽象数据类型模型、面向对象模型
