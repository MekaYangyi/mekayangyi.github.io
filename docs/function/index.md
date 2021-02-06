# 简易function


本文主要讲一下怎么实现一个简易版本的function<>模板，从c++ templates第二版摘出，相应的技巧在timer中一些应用。

## 出发点

一个简单的例子，下面这个模板上述能够接受任意可调用的对象，lambda表达式、函数指针、仿函数。

```cpp
template<typename F>
void for_int_up(int n, F f)
{
    for (int i = 0; i <= n; ++i) {
        f(i);
    }
}
```

```cpp
void print_int(int i)
{
    std::count << i << ' ';
}

int main()
{
    int sum = 0;
    for_int_up(5, [&sum](int i) {
        sum += i;
    });
    
    for_int_up(5, print_int);
    
    return 0;
}
```



像for_int_up这样的函数其实有两个问题：

- 使用了模板将函数内部实现暴露。
- 造成代码膨胀，for_int_up函数还比较小，如果是一个很大的函数，代码膨胀会厉害的多。



于是为了解决上面两个问题，可能尝试用下面这个方案，这也是我们代码中常见的解决方案。

```cpp
void for_int_up(int n, void (*f)(int))
{
    for (int i = 0; i <= n; ++i) {
        f(i);
    }
}
```

考虑到有可能有需要带上参数的需求，我们可能会写成这样。

```cpp
void for_int_up(int n, void (*f)(int, void *data), void *data)
{
    for (int i = 0; i <= n; ++i) {
        f(i, data);
    }
}

void sum(int i, void *data)
{
    int *sum = (int*)data;
    *sum += i;
}

int main()
{
    int j = 0;
    for_int_up(5, sum, &j);
    return 0;
}
```

使用函数指针替代模板，for_int_up只接受函数指针，无法接收lambda表达式、仿函数。

带参数的方案基本上能够满足需求，只是需要将函数与数据分离，代码相对难写，强转可能存在错误。

本质上带参数void*的方案是在抹除类型信息。



基于以上的需求标准库里的std::function<>就应运而生了。

```cpp
void for_int_up(int n, std::function<void(int)> f)
{
    for (int i = 0; i <= n; ++i) {
        f(i);
    }
}
```

std::function<void(int)>能够接收任意返回值是void，参数为int的函数指针、lambda表达式、仿函数。

上面两个问题一下就得到了解决，内部实现可以隐藏起来，同时模板范围缩小到std::function，即使for_int_up再大也不会出现代码膨胀很厉害的情况。



## 广义的函数指针

std::function实际上是一个广义上的C++函数指针，需要支持一下操作：

- 在调用者只知道入参与返回值的情况，可以调用执行，不需要理解内部具体实现。
- 支持复制、移动。
- 可以被入参与返回值相同的函数指针、lambda表达式、仿函数、std::function初始化
- 支持null状态

## 实现

下面我们会实现一个简易版本的std::function，FUNCTION_PTR。FUNCTION_PTR会支持上面提到的所有特性。



functionptr.hpp

```cpp
// 原始模板，因为实际上FUNCTION_PTR模板参数只有一个，所以需要这个原始模板:
// 模板参数是一个函数类型
template < typename T >
class FUNCTION_PTR;

// 偏特化，提取出变参Args为所有参数，R为返回值
template < typename R, typename... Args >
class FUNCTION_PTR< R( Args... ) > {
   private:
    FUNCTOR_BRIDGE< R, Args... > *bridge; // 核心类，参数类型信息，提供抽象的invoke。

   public:
    // 构造:
    FUNCTION_PTR() : bridge( nullptr ) {} // 默认为空
    FUNCTION_PTR(FUNCTION_PTR const &other );  // see functionptr-cpinv.hpp
    FUNCTION_PTR(FUNCTION_PTR &other )
        : FUNCTION_PTR( static_cast< FUNCTION_PTR const & >( other ) ) {}
    FUNCTION_PTR( FUNCTION_PTR &&other ) : bridge( other.bridge )
    {
         // 移动构造，所有权转移
        other.bridge = nullptr;
    }
    // 从任意函数对象构造:
    template < typename F >
    FUNCTION_PTR( F &&f );  // see functionptr-init.hpp
    
    // 赋值操作符:
    FUNCTION_PTR &operator=( FUNCTION_PTR const &other )
    {
        FUNCTION_PTR tmp( other );
        swap( *this, tmp );
        return *this;
    }
    FUNCTION_PTR &operator=( FUNCTION_PTR &&other )
    {
        delete bridge;
        bridge = other.bridge;
        other.bridge = nullptr;
        return *this;
    }
    // 从任意函数对象复制:
    template < typename F >
    FUNCTION_PTR &operator=( F &&f )
    {
        FUNCTION_PTR tmp( std::forward< F >( f ) ); // 先构造一个临时对象
        swap( *this, tmp ); // 调用swap
        return *this;
    }
    // 析构:
    ~FUNCTION_PTR( ) { delete bridge; }
    friend void swap( FUNCTION_PTR &fp1, FUNCTION_PTR &fp2 )
    {
        std::swap( fp1.bridge, fp2.bridge );
    }
    // bool隐式转换，便于if判断
    explicit operator bool( ) const { return bridge == nullptr; }
    // ()操作符重载:
    R operator( )( Args... args ) const;  // see functionptr-cpinv.hpp
};
```

可以看到FUNCTION_PTR包含一个FUNCTOR_BRIDGE< R, Args... >成员，它负责存储具体的功能对象，并将具体类型擦除了。



functorbridge.hpp

```cpp
// FUNCTOR_BRIDGE只是一个抽象类，定义了几个基本接口
template < typename R, typename... Args >
class FUNCTOR_BRIDGE {
   public:
    virtual ~FUNCTOR_BRIDGE( ) {}
    virtual FUNCTOR_BRIDGE *clone( ) const = 0; // 复制接口
    virtual R invoke( Args... args ) const = 0; // 调用接口
};
```



bridge/functionptr-cpinv.hpp

```cpp
// 基于FUNCTOR_BRIDGE的抽象接口，我们能够实现FUNCTION_PTR复制与调用
template < typename R, typename... Args >
FUNCTION_PTR< R( Args... ) >::FUNCTION_PTR( FUNCTION_PTR const &other )
    : bridge( nullptr )
{
    if ( other.bridge ) {
        bridge = other.bridge->clone( ); // 复制
    }
}
template < typename R, typename... Args >
R FUNCTION_PTR< R( Args... ) >::operator( )( Args... args ) const
{
    return bridge->invoke( std::forward< Args >( args )... ); // 调用，完美转发
}
```



bridge/specificfunctorbridge.hpp

```cpp
// 真正存储实际的可调用对象，继承自FUNCTOR_BRIDGE，利用多态实现类型的擦除
template < typename FUNCTOR, typename R, typename... Args >
class SPECIFIC_FUNCTOR_BRIDGE : public FUNCTOR_BRIDGE< R, Args... > {
    FUNCTOR functor;

   public:
    // 构造
    // 这里用了一个单独的FUNCTOR_FWD而不是FUNCTOR，因为有可能FUNCTOR_FWD类型与FUNCTOR
    // 不一样，存在隐式转换
    template < typename FUNCTOR_FWD >
    SPECIFIC_FUNCTOR_BRIDGE( FUNCTOR_FWD &&functor )
        : functor( std::forward< FUNCTOR_FWD >( functor ) ) 
    {
    }
    virtual SPECIFIC_FUNCTOR_BRIDGE *clone( ) const override
    {
        return new SPECIFIC_FUNCTOR_BRIDGE( functor );
    }
    virtual R invoke( Args... args ) const override
    {
        return functor( std::forward< Args >( args )... );
    }
};
```



functionptr-init.hpp

```cpp
// 任意类型转换为FUNCTION_PTR
template < typename R, typename... Args >
template < typename F >
FUNCTION_PTR< R( Args... ) >::FUNCTION_PTR( F &&f ) : bridge( nullptr )
{
    using FUNCTOR = std::decay_t< F >; // 类型退化
    using BRIDGE = SPECIFIC_FUNCTOR_BRIDGE< FUNCTOR, R, Args... >;
    bridge = new BRIDGE( std::forward< F >( f ) ); // 生成brige
}
```

到此基本上FUNCTION_PTR已经实现完成了。

## 其他

- 最终版本欠缺一个小功能判断FUNCTION_PTR内部存储的可调用对象是否相等。
- FUNCTION_PTR将所有可调用对象都转换为了一次虚函数的调用，降低了性能。
- 每创建一个FUNCTION_PTR都需要一次堆内存分配。
- gcc4.8中的std::function对于小对象不会进行堆内存的分配。



