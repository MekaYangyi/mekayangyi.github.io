# [读书笔记]深入探索C++对象模型 第五章 构造 解构 拷贝语意学

# 构造、解构、拷贝语意学

### 纯虚函数的存在
纯虚函数能够被静态的调用，不能经过虚拟机制调用。
虚析构函数不能定义为纯虚的，一定要有定义，否则即使可以编译，但是链接的时候会有错误。因为其子类会静态调用上一层的析构函数。如果说上一层的析构函数是一个纯虚函数的话，那么链接的时候会失败。
### 虚拟规格的存在
不应该把所有的函数都声明为虚函数，然后靠编译器去优化操作吧virtual invocation去除。
### 虚拟规格中的const的存在
实际上你很难知道一个类的子类对于这个函数是不是应该定义为const，因为即使现在你不需要修改类的内容，但子类可能需要修改，你没法预料到。那么最好不要定义一个有const函数的基类了。
## “无继承”情况下的对象构造
```cpp
typedef struct
{
    float x,y,z;
} Point;
```
编译器会分析声明，贴上Plaint OI' Data的卷标，被贴上该卷标的类，不会有构造函数或者析构函数的调用了。直接使用C的方式。
<!--more-->
##  继承体系下的对象构造
```cpp
T object;
```
定义一个object如上时候，实际会发生什么。如果T有一个construct，它会被调用。
Constructor可能带有大量 隐藏码，因为编译器会扩充每一个constructor，扩充程度视class T的继承体系而定，一般而言编译器所做的扩充操作大约如下：
1. 记录在member initialization list中的data members初始化操作会被放进constructor函数本身，并以members的声明顺序为顺序。
2. 如果有一个member没有出现在member initialization list中，但是它有一个default constructor, 那么该default construtor必须被调用。
3. 在那之前，如果class object有virtual table pointer(s)，它们必须被设定初值。指向适合的virtual table(s)
4. 在那之前，所有上一层的base class constructirs必须被调用，以base class的声明顺序为顺序
> * 如果base class被列于初始化列表中，那么任何明确的指定参数都应该传递过去
> * 如果base class 没有被列于初始化列表中，而它有默认构造函数，那么调用之
> * 如果base class 是多重继承下的第二或者后继的base class那么this指针必须被调整
5. 在那之前，所有的virtual base class constructor必须被调用，从左到右，从最深到最浅
> * 如果class被列于初始化列表中，那么如果有任何明确指定的参数，都应该传递过去。若没有在list中，而class有默认构造函数，也应该调用
> * 此外，class中的每一个virtual base class subobject的偏移量必须在执行期间可被存取
> * 如果class object是最底层的class，其构造函数可能被调用。某些用以支持这个行为的机制被加入


虚函数不见得一定有运行期绑定，如果能够编译期确定的，编译期乐于去进行静态的调用。

### 虚拟继承


<img src="/img/20160818 inside C++ 0.jpg" alt=""/>
￼
有如上图的继承结构。
Vertex的构造函数必须调用Point的构造函数。但是Vertex和P哦Point3d同为Vertex3d的 subobjects的时候，它们对Point的构造函数的调用操作一定不可以发生，取而代之的是，作为底层的class，Vertex3d有责任将Point初始化，而更往后的继承，则由PVertex,而不是Vertex3d来负责完成共享的Point subobject的构造.
```cpp
Point3d*
Point3d::Point3d(Point3d *this, bool __most__derived,
                            float x, float y, float z)
{
    if(__most_derived != false)
        this->Point::Point(x,y);
    this->__vptr_Point3d = __vtbl_Pint3d;
    this->__vptr_Point3d__Point = __vtbl_Point3d__Point;
    this->_z = rhs._z;
    return this;
}
```

```cpp
Vertex3d*
Vertex3d::Vertex3d( Vertex3d *this, bool __most_deriver,
                                float x, float y, float z)
{
    if(__most_derived != false)
        this->Point::Point(x,y);
    this->Point3d::Point3d(false, x, y, z);
    this->Vertex::Vertex (false, x, y);
    //设定vptrs
    //安插USER CODE
    return this;
}
```
##vptr 初始化语意学
当我们定义一个PVertex object的时候，构造函数的调用顺序是
> * Point()
> * Point3d()
> * Vertex()
> * Vertex3d()
> * PVertex()

 构造函数调用成员函数会决议为静态的，不会使用多态机制。主要是考虑到构造函数中对象可能是不完整的，编译器需要找到合适的函数对象来调用，只能调用到此时能够起作用的函数。

1. 在派生类的构造函数中，所有的Virtual base classes以及上一层base class的constructors会被调用
2. 上述完成之后，对象的vptr(s)被初始化，指向相关的虚表
3. 如果有初始化列表的话，将在构造函数内扩展开来，这必须在vptr设定之后进行，以免一个virtual member function调用
4. 最后执行程序员所提供的的代码

构造函数：
```cpp
PVertex::PVertex(float x, float y, float z) : _next(0), Vertex3d(x, y, z), Point(x, y) {
    if (spyOn)
       cerr << "within Point3d::Point3d()" << " size: " << size() << endl;
}
```
会被编译器扩展为：
```cpp
PVertex* PVertex::PVertex( Pvertex* this, bool __most_derived, float x, float y, float z )  {
    //有条件地调用virtual base class的ctor
    if ( __most_derived != false )
       this->Point::Point( x, y );
    //无条件地调用上一层的base class的ctor
    this->Vertex3d::Vertex3d( x, y, z );
    //初始化vptr
    this->__vptr__PVertex = __vtbl__PVertex;
    this->__vptr__Point__PVertex = __vtbl__Point__PVertex;
    //显式的用户代码
    if ( spyOn )
        // 虚拟机制调用size()函数
       cerr << "within Point3d::Point3d()"<< " size: "
             << (*this->__vptr__PVertex[ 3 ].faddr)(this)
             << endl;
    return *this;
}
```
## 对象复制语意学
bitwise copy不够的情况和之前几章记录的一致。
```cpp
inline Point& Point::operator= (const Point &p)
{
    _x = p._x;
    _y = p._y;
}
//派生一个类
classPoint3d : virtual public Point
 {
    public:
       Point3d( float x = 0.0, y = 0.0, float z = 0.0 );
       ...
    protected:
       float _z;
};
//编译器会合成一个
inline Point3d& Point3d::operator=( Point3d *constthis, const Point3d &p ) {
   //调用base class的operator=
   this->Point::operator=( p );
   // memberwise copy the derived class members
   _z = p._z;
   return *this;
}
```
但是考虑到上面那个虚拟继承层次的结构的话。
怎么在虚拟继承中去处理复制这个问题。
事实上，copy assignment operator在虚拟继承的情况下行为不佳，需要小心的设计和说明，许多编译器甚至不尝试取得正取的语意，造成多次调用虚拟基类的copy assignment operator的多个实体被调用。（我的话，好像多次调用并不会产生大的问题，只是重复复制罢了，实际上没产生什么错误，效率貌似有下降。）

## 析构语意学
如果class没有定义析构函数，那么只有在class内带的成员对象有析构函数的情况下，编译器才会自动合成析构函数，否则析构函数视为不需要的。即使它拥有一个虚函数等。
也就是构造函数和析构函数并不是一定要成对的出现，没必要定义了构造函数，就定义出析构函数。

析构函数的扩展
1. 析构函数的本身被执行，user code
2. 如果class拥有成员类对象，而后者拥有析构函数，声明顺序的相反顺序调用其析构函数
3. 如果对象有vptr需要重新被设定，指向合适的base class的vtbl
4. 如果任何一个直接的nonvirtual base classes拥有析构函数，它们会以声明次序的相反顺序被调用
5. 如果任何virtual base classes拥有析构 函数，而当前讨论的这个class是最尾端的class,那么会以其原来的构造函数相反的顺序被调用
