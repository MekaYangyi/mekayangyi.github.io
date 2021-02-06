# [读书笔记] 深入探索C++对象模型 第六章 执行期语意学


# 执行期语意学
以下一个简单的式子
```cpp
if (yy == xx.getValue)
```
xx  yy定义如下
```cpp
X xx;
Y yy;
class Y
{
public:
  Y();
 ~Y();
  bool operator==(const& Y) const;
};
class X
{
public:
  X();
  ~X();
  operator Y() const;
  X getValue();
};
```
那么编译器在我们之后做了什么呢
```cpp
if(yy == xx.getValue())

//转换为
if(yy.operator==(xx.getValue())

//接着转换
if(yy.operator==(xx.getValue().operator Y()))

//接着转换
X temp1 = xx.getValue();
Y temp2 = temp1.operator Y();
int temp3 = (yy.operator==(temp2));
if(temp3)

temp2.y::~Y();
temp1.x::~X();
```

<!--more-->
## 对象的构造与析构

### 局部对象
```cpp
{
 Point p;
 // p.Point::Point();
 ...
 //p.Point::~Point();
 }
 ```
 如果一个函数拥有多个离开点，那么会在每一个离开点之前对对象进行析构。
### 全局对象
```cpp
Matrix identity;
main()
{
    Matrix ml = identity;
    ...
    return 0;
}
```
C++ 保证一定会在main()中第一次用到identity之前把 didentity构造出来，在main()函数结束之前销毁。
C++程序中所有全局对象都被防止在程序的data segment中，如果明确指定给它一个值,object将以该值为初值。否咋object所配置到的内存内容为0。
> * class object在编译器可以被放置与data sement中并且为0,但是它的构造函数需要在程序激活的时候才会被实施。(也就是说全局对象的初始化的问题，对于类对象的有些门道。)

<img src="/img/20160820 inside C++ 0.jpg" alt=""/>
￼
（还是不是很清楚全局对象是如何初始化的）
### 局部静态对象
```cpp
const Matrix&
identity()
{
    static Matrix mat_identity;
    ...
    return mat_identity;
}
```
对于局部静态的变量，他们的构造和析构必须只施行一次。
编译器的策略是，导入一个临时性的对象以保护mat_identity的初始化操作。第一次处理identity()时候，这临时对象被评估为false，于是构造函数被调用，然后临时对象改为true。同理析构也是如此。
（但是具体现代编译器怎么操作的我还是不清楚。）。
### 对象数组
```cpp
Point knots[10];
```
需要做什么。如果是一个没有构造函数的，也没有析构函数的。那么工作不会比建立一个内建类型所组成的数组更多。
如果有的话，那么整齐的操作必须施行与每一个元素上。
在cfront中，使用一个命名为vec_new()的函数，产生以class objects构造而成的数组。
```cpp
void* 
ver_new(
    void *array,                            //数组的起始位置
    size_t elem_size,                       //一个对象的大小
    int elem_count,                         //数组的元素个数
    void (*constructor)( void*),
    void (*destructor)(void*, char)
)
```
调用操作

```cpp
Point knots[10];
ver_new(&knots, sizeof(Point), 10, &Point::Point, 0);
```
同样如果Point有一个析构函数会有一个类似ver_delete()的函数
```cpp
void* 
ver_delete(
    void *array,                            //数组的起始位置
    size_t elem_size,                       //一个对象的大小
    int elem_count,                         //数组的元素个数
    void (*destructor)(void*, char)
)
```
不同的编译器会有不同的实现。
如果数组部分被赋予了初值的，那么会产生什么转换
```cpp
Point knots[10] = {
    Point(),
    Point(1.0, 1.0, 0.5),
    -1.0
};
```
对于有了初值的元素ver_new不必要，但是未被初始化的部分会调用vec_new。
```cpp
Point knots[10] = {
    Point(),
    Point(1.0, 1.0, 0.5),
    -1.0
};

//明确的初始化前三个
Point::Point(&knots[0]);
Point::Point(&knots[1], 1.0, 1.0, 0.5);
Point::Point(&knots[2], -1.0, 0.0, 0.0);
ver_new(&knots + 3, sizeof(Point), 7, &Point::Point, 0);
```
### default Constructors和数组
为了支持
```cpp
complex:: complex(double = 0.0, double = 0.0);
complext c_array[10];
//编译器最终调用
vec_new(&c_array, sizeof(complex), 10, &complex::complex, 0);
//cfront采用如下方法支持
//产生一个默认构造函数 调用带默认参数的构造函数
complex::complex()
{
    complex(0.0,0.0);
}
//来完成调用
```
（有一个问题，那么这个不就是产生了两个不带参数的构造函数吗，虽然一个有参数，但是都用默认的。怎么解决的。不过大部分构造过程都是在编译期间，那么都是静态指定调用的话，还是解决掉了的。不是很清楚这个问题。）
## new delete运算符
运算符new的使用，之前的几章一直都有。
会转换成两步，一步是使用适当的函数，分配内存。
后一步是给对象设置初值，类对象的话，调用的对应的构造函数等等
```cpp
extern void* operator new( size_t size )   
{   
    if( size == 0 )   
    size = 1; // 这里保证像 new T[0] 这样得语句也是可行的   
   
    void *last_alloc;   
    while( !(last_alloc = malloc( size )) )   
    {   
       if( _new_handler )   
           ( *_new_handler )(); //调用handler函数  
        else   
           return 0;   
    }   
    return last_alloc;         
}   
extern void operator delete( void *ptr )   
{   
    if(ptr) // 从这里可以看出，删除一个空指针是安全的   
    free( (char*)ptr );   
}  
```
### 针对数组的new语意
内建的或者没有默认构造函数的，直接默认的new就能完成任务。
对于有默认构造函数的，某些版本的vec_new()就会被调用。

## 临时性对象
很多简短的代码实际上都会产生一些临时对象。
是不是真的产生，需要看编译器的具体实现了。
### 临时性对象的迷思

