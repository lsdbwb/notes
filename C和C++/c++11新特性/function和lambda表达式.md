# std::function
std::function是可调用对象的封装器。可以把std::function看做一个函数对象，用于表示函数这个抽象概念。std::function可以**存储、复制和调用**可调用对象。存储的可调用对象称为std::function的目标，若std::function不含目标，则称它为空，调用空的std::function的目标会抛出std::bad_function_call异常。

可调用对象的概念：
>满足以下条件之一就称为可调用对象
>1. 是一个函数指针
>2. 是一个具有operator()成员函数的类对象（functor），lambda表达式
>3. 是一个可以被转换为函数指针的类对象
>4. 是一个类的成员函数指针
>5. bind表达式或者其他函数对象

std::function可以用来做回调函数

# std::bind
使用std::bind可以将可调用对象和部分参数一起绑定，绑定后的结果使用std::function进行保存，并延迟调用到实际需要调用的时候

std::bind通常有两大作用：

-   将可调用对象与参数一起绑定为另一个std::function供调用
  
-   将n元可调用对象转成m(m < n)元可调用对象，绑定一部分参数，这里需要使用std::placeholders

# lambda表达式
lambda表达式可以说是c++11引用的最重要的特性之一，它定义了一个匿名函数，可以捕获一定范围的变量在函数内部使用，一般有如下语法形式：

```c++
auto func = [capture] (params) opt -> ret { func_body; };
```

其中func是可以当作lambda表达式的名字，作为一个函数使用，capture是捕获列表，params是参数表，opt是函数选项(mutable之类)， ret是返回值类型，func_body是函数体。

## lambda的捕获规则
-   []不捕获任何变量
  
-   [&]引用捕获，捕获外部作用域所有变量，在函数体内当作引用使用

-   [=]值捕获，捕获外部作用域所有变量，在函数内内有个副本使用
  
-   [=, &a]值捕获外部作用域所有变量，按引用捕获a变量

-   [a]只值捕获a变量，不捕获其它变量
  
-   [this]捕获当前类中的this指针

## lambda的内部原理
编译器会把我们写的lambda表达式翻译成一个类，并重载 operator()来实现。比如我们写一个lambda表达式为

```text
auto plus = [] (int a, int b) -> int { return a + b; }
int c = plus(1, 2);
```

那么编译器会把我们写的表达式翻译为

```c++
// 类名是我随便起的
class LambdaClass
{
public:
    int operator () (int a, int b) const
    {
        return a + b;
    }
};

LambdaClass plus;
int c = plus(1, 2);
```

**捕获列表**，对应LambdaClass类的**private成员**。

**参数列表**，对应LambdaClass类的成员函数的operator()的形参列表

**mutable**，对应 LambdaClass类成员函数 **operator() 的const属性 ，但是只有在捕获列表捕获的参数不含有引用捕获的情况下才会生效，**因为捕获列表只要包含引用捕获，那operator()函数就一定是非const函数**。**

**返回类型**，对应 LambdaClass类成员函数 **operator() 的返回类型**

**函数体，**对应 LambdaClass类成员函数 **operator() 的函数体。**

**引用捕获和值捕获不同的一点就是，对应的成员是否为引用类型。**