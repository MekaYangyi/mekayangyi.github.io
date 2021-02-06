# [读书笔记]深入理解C++对象模型 第二章 构造函数的语意学

#  第二章：构造函数语意学
由于C++的编译器在程序员之外做了太多事情，导致会产生很多意料之外的错误。
例子 Conversion运算符。
```cpp
//为了支持
if(cin);
//定义了一个perator int()
//但是导致了以下错误的代码能够正常运行
int inVal;
cin << inVal;
//此处<<被解释为左移操作符
int temp = cin.operator int();
temp << intVal;
```

#  Default Constructor的构建操作
默认构造函数只在编译器认为需要的时候才创建。
```cpp
class Foo{public: int val; Foo *pnext};
int main()
{
    Foo test;
    return 0;
]
```
这种情况实际没有默认构造函数，编译器什么都没做。
<!--more-->

## 带有Default Constructor的 Member class Object
如果一个Class没有任何构造函数，但是包含一个member object，而后者又有default constructor，那么会在constructor真正被调用时合成出一个默认构造函数。
```cpp
classFoo {public: Foo(),Foo(int);...};
classBar {public: Foofoo; char *str;};
void foo_bar(){
   Bar bar; //Bar::foo应在此处被初始化
   if(str){...}
}
```
此时的Bar合成默认构造函数会调用Foo的默认构造函数。
```cpp
Bar::Bar()
{ 
    foo.Foo::Foo(); //合成出的
    //但是str等成员是不会管的
}
```
如果已经写了一个默认构造函数，那么就会扩张该函数。
```cpp
Bar::Bar()
{ 
    foo.Foo::Foo(); //扩展的
    str=0;//程序员代码
}
```
C++会以member objects在class中声明的次序来调用各个构造函数，由编译器完成。如果成员没有默认构造函数的话，就不会有扩张。

## 带有Default Constructor的 Base Class
一个没有任何构造函数的类派生子一个带有默认构造函数的基类，那么会合成出一个一个默认构造函数，调用上一层的默认构造函数（根据声明次序）。
如果是一个有多个构造函数的类，但是没有默认构造函数，则会扩张每一个构造函数，加入有必要的基类部分的默认构造。
但是不会合成默认构造函数，因为已经存在了程序员编写的构造函数。
如果也存在带有构造函数的成员，那么会在基类的部分构造之后，调用这些成员的构造函数。

## 带有一个Virtual Function的 Class
另有两种情况，也需要合成出default constructor
> * class声明（或者继承）一个virtual function
> *  class派生自一个继承串链，其中有一个或者更多的virtual base classes

如果程序员没有声明自己的构造函数，编译器就会详细记录合成一个default constructor的必要信息。
编译器需要做以下的几个功能：
> * 一个virtual function table会被编译器产生出来，内放class的virtual function地址。
> * 在每一个class object中，一个额外的pointer member会被编译器合成，内含相关的classs vtbl的地址。

此外如果必要，虚函数表的部分会被重写，以改变为需要的情况。
为了支持这种功能，编译器必须为每个w对象设置它的vptr（这是成员变量，此时需要指向合适的vtbl），因此编译器需要在default ctor中安插一些代码来完成这种工作。
## 带有一个Virtual Base Class 的Class
```cpp
classX { public: inti; };
classA : publicvirtualX   { public: intj; };
classB : publicvirtualX   { public: doubled; };
classC : publicA, publicB { public: intk; };
//无法在编译期间解析出 pa->i 的位置（给一个pa无法确定i的地址）。
void foo( constA* pa ) { pa->i = 1024; }
main() {
   foo( new A );
   foo( new C );
   // ...
}
//由于pa的真正类型不确定，所以某些编译器会记录一个指针例（如 __vbcX）来记录X，然后通过这个指针来定位pa指向的i。
//上述
void foo( constA* pa ) { pa->i = 1024; }
//变成了：
void foo( constA* pa ) { pa-> __vbcX ->i = 1024; }
```
其中__vbcX标示编译器产生的指针，指向 virtual base class X。
__vbcX需要在每一个构造函数按错那么“允许每一个 virtual base class的执行期间存取操作“的码。如果class没有声明任何构造函数，就需要合成一个默认构造函数。

# Copy Constructor的构建操作
有三种情况会执行拷贝构造函数：
> * 显式的使用 =
> * 传参
> * 返回

## Default Memberwise Initialization
一个class没有提供显式的拷贝构造函数的话，那么利用Memberwise Initialization(对每一个成员将源对象所有的member复制给目标对象)

## 非逐位拷贝
> * 这个类的某个member object有拷贝构造函数。（不管成员的拷贝构造函数是合成的还是显式的定义的，都需要合成一个拷贝构造函数）。
> * 这个类继承自某个有copy ctor的base class。（同上）。
> * 这个类声明了若干个virtual function。（如果继承的基类有virtual function那么一定有拷贝构造函数，符合第二条。）
> * 这个类派生自的继承链中有virtual base class。

第三四种情况需要合成的复制构造函数构建正确的虚表指针给每一个对象。
因为复制构造函数可以 派生类赋值给基类。那么派生类的虚表指针是没法用逐位拷贝来赋值，需要一个合成的复制构造函数正确的设定虚表指针。

# 程序转化语意学
```cpp
X foo()
{
    X xx;
    return xx;
}
```
一个人可能会做出以下假设：
> * 每次foo()调用，就会传回xx的值.
> * 如果class X定义了一个拷贝构造函数，那么每次调用foo()，保证该拷贝构造函数也会被调用。

均不一定。move语意和外面没有接收的可能导致以上假设不一定。

## 明确的初始化操作
```cpp
X x0;
void foo_bar(){
   X x1(x0);             //定义了x1
   X x2 = x0;           //定义了x2
   X x3 = X(x0);       //定义了x3
}
```

//转化的两个动作
//即变成了
```cpp
void foo_bar(){
   X x1;    //定义被重写，初始化操作被剥除
   X x2;    //定义被重写，初始化操作被剥除
   X x3;    //定义被重写，初始化操作被剥除

   //编译器安插X copy ctor。
   x1.X::X( x0 );
   x2.X::X( x0 );
   x3.X::X( x0 );
}
```
其中:
x1.X::X( x0 );
会表现为对拷贝构造函数的调用：
 X::X( constX& xx);）
 
## 参数的初始化
 如下代码的变化
```cpp
void foo(X x0);
//...
X xx;
foo(xx)
```
这种方式把函数的参数变成了引用，然后将拷贝构造函数构造的参数传入。
变成了
```cpp
void foo(X& x0);
//...
X __tmp;
__tmp.X::X( XX );
foo(__tmp);
```
其中X声明了destructor，它在foo调用完成后销毁暂时性的对象。

## 返回初始值
 ```cpp
 X bar(){
   X xx;
   //...
   return xx;
}
```
先添加一个额外的引用参数，然后在返回之前调用一个复制构造函数构造这个返回对象。
于是变成了
```cpp
void bar(X& _result){
   X xx;
   //...
   _result.X::X(xx);
   return;
}
```
```cpp
X xx = bar();
//转换为
X xx;
bar(xx);
```

```cpp
bar().memfunc();
//可能转换为
X _temp0;
(bar(_temp0), _temp0).memfunc();
```
## 在使用者层面的优化
 ```cpp
X bar(const T &y,  const T &z)
{
    X xx;
    //...
    return xx;
} 
 ```
 
而是定义一个新的构造函数，这样在转换之后效率更高
 ```cpp
X bar(const T &y,  const T &z)
{
    return X(y , z);
} 
 ```
 转换后
  ```cpp
void bar(X &_result, const T &y,  const T &z)
{
    _result X::X(y, z);
    return;
} 
 ```

## 在编译器层面优化
  就是上面那个转换代码
  
  理论上合成的拷贝构造函数就能够正常的工作了。某些情况不会有合成的拷贝构造函数，而是直接逐位拷贝。这样的情况下，就没法实施，上面的编译器层面的优化。
那么我们应该预见这个类是不是有很多传值的操作，比如上面的函数的参数，返回值。如果有，那么提供一个，编译器才能够实施优化。

  **成员们的初始化队伍**
构造函数初始化列表
> * 初始化一个引用成员（不这样做出错）
> * 初始化一个const成员（不这样做出错）
> * 调用一个base class的构造函数，而它拥有一组参数时（不这样做出错）
> * 调用一个member class的构造函数，而它拥有一组参数时（比赋值更有效率）

编译器会一一操作初始化列表，以适当次序（成员的声明次序）在构造函数之内安插初始化操作，并且在任何显式写的代码之前完成操作。
初始化列表能够使用成员函数来初始化一个成员。（这里如果比较复杂的初始化能够使用这种方式提高初始化的效率，但是要注意该函数使用的变量是不是在调用前都被初始化了。合法的原因是此时this指针已经创建好了。）

