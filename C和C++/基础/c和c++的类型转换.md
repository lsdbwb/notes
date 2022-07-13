# 隐式类型转换
隐式类型转换发生于将一个值赋值给其兼容的类型时，例如：
```c++
short a = 100;
int b;
b = a;   // a从short隐式转换为int型然后赋值给b
```

基础运算类型和一些指针类型以及bool类型可以发生隐式类型转换

## 基础运算类型的隐式转换
类型提升：
例如short转换为int，int转换为float或者double。这种转换能够保证转换后的值和原来的值相等。但是如果反过来进行转换，例如int转换为short，则无法保证（int 4字节， short 2字节，short表示的数的范围小很多）

一些会出现问题的转换：
1. 有符号数和无符号数的转换，例如int转换为unsigned int。int型是正数时没有问题，但是如果是负数，则转换为无符号数之后表示的值会变化(-1变成无符号数后会变成很大的值)
2. 和bool类型转换时，0或者空指针会转变为false，其余的都会转变为true
3. 如果是浮点型转换为整型(float->int)，小数部分会被舍弃

## 指针类型的隐式转换
数组和函数可以被隐式转换为指针

- 空指针能够被转换为任意指针类型
- 任意指针类型能够被转换为空指针
- 指针向上转型：子类的指针可以被转换为父类的指针（不修改const和volatile修饰符）（需要保证转换后指向的父类完整无歧义且可访问）

## 和c++类相关的隐式类型转换
c++中类相关的隐式类型转换由以下三个类的成员函数控制：
1. 只有一个参数的构造函数
2. 赋值运算符
3. 类型转换运算符

如下例所示：
```c++
// implicit conversion of classes:
#include <iostream>
using namespace std;

class A {};

class B {
public:
  // conversion from A (constructor):
  B (const A& x) { cout << "construct" << endl; }
  // conversion from A (assignment):
  B& operator= (const A& x) { cout << "operator = " << endl; return *this;}
  // conversion to A (type-cast operator)
  operator A() {cout << "operator A()" << endl; return A();}
};

int main ()
{
  A foo;
  B bar = foo;    // calls constructor
  bar = foo;      // calls assignment
  foo = bar;      // calls type-cast operator
  return 0;
}
```

### 使用explicit关键字禁止使用隐式类型转换构造对象
因为一个c++函数的每个参数都是允许隐式类型转换的，如果一个函数的定义如下：
`void fn (B x)`
则在上面B支持隐式转换的情况下，fn也可以传入A作为fn的参数，这和预期的行为不符。这时可以通过explict关键字避免这种情况

```c++
// explicit:
#include <iostream>
using namespace std;

class A {};

class B {
public:
  explicit B (const A& x) {} // 使用explict关键字禁止隐式转换
  B& operator= (const A& x) {return *this;}
  operator A() {return A();}
};

void fn (B x) {}

int main ()
{
  A foo;
  B bar(foo);
  // B bar = foo;  //not allowed for explicit ctor.
  bar = foo;
  foo = bar;
  
//  fn (foo);  // not allowed for explicit ctor.
  fn (bar);  

  return 0;
}
```

# c和c++中的显式类型转换
c++是强类型的语言，有将一种类型转换为另一种类型的需求。

## c中的显式类型转换
c有两种风格的强制类型转换
- (type)value
- type(value)
c中可以对以下类型使用上述强制转换
1. 基本的算术类型，例如int和float的相互转换
2. int型和指针类型相互转换
3. 两个指针类型相互转换
4. const/volitale类型和非 const/volatile类型的相互转换

## c++中显式类型转换
上面c中的类型转换的方式在c++中也是兼容的
在c++中新增了两个概念：
- 类和对象
- 模板

如果使用c风格的type-cast对类对象的指针进行转换，可能能够通过编译，但是会造成运行时错误。因为c风格的type-cast不保证转换前后类对象仍然是完整可访问的。

c++为了控制*类对象间*的type-cast，新增了以下四种类型转换操作：

### dynamic_cast
dynamic_cast只能用于类对象的指针或者引用之间的转换（或者和`void*`转换），确保转换后的指针指向的类对象是一个完整的对象。

- 指针向上转型(upcast): 继承类的指针转换为基类指针(和隐式类型转换一样)
- 指针向下转型(downcast):基类指针转换为继承类的指针

```c++
// dynamic_cast
#include <iostream>
#include <exception>
using namespace std;

class Base { virtual void dummy() {} };
class Derived: public Base { int a; };

int main () {
  try {
    Base * pba = new Derived;  // 隐式类型转换(子类指针 -> 基类指针)
    Base * pbb = new Base;
    Derived * pd;

	// 基类指针转换为子类指针
	// 可以正常转换，因为pba本身实际指向的是子类对象
    pd = dynamic_cast<Derived*>(pba);
    if (pd==0) cout << "Null pointer on first type-cast.\n";

	// 不能正常转换，因为pbb本身实际指向的基类对象
    pd = dynamic_cast<Derived*>(pbb);
    if (pd==0) cout << "Null pointer on second type-cast.\n";

  } catch (exception& e) {cout << "Exception: " << e.what();}
  return 0;
}

```

>dynamic_cast使用运行时类型信息(RTTI)来判断指针指向的对象的实际类型,RTTI通常存放在类的虚函数表中
>因此使用dynamic_cast会引入运行时的开销

### static_cast
static_cast同样可以进行指针的向上或者向下转型，但是它不会像dynamic_cast一样做运行时的安全检查。因此如果转换后的指针指向的对象不完整，可能会产生运行时错误。
```c++
class Base {};
class Derived: public Base {};
Base * a = new Base;
// 以下转换后可能会在运行时出错
Derived * b = static_cast<Derived*>(a);
```

所有允许的隐式转换都可以使用static_cast来显式地进行。除此之外，static_cast还支持以下一些转换：
- void* 和任意指针类型的互转
- enum类型和int,float类型的互转
- 可以显式地调用一个类对象的单参数构造函数或者type_cast运算符
- 将某个值转换为右值引用类型(std::move使用其实现)
- 转换任何类型为void类型

### reinterpret_cast
reinterpret_cast可以将任意一种指针类型转换为另一种，即使是两个不相干的类的指针。在进行指针类型的转换时，所做的操作只是简单的二进制的拷贝，不会做任何*指针本身类型检查和指针指向的对象的类型检查*

reinterpret_cast也可以将一个指针类型转换为一个int类型(其实就是把指针变量存的值拷贝到int类型变量中)

reinterpret_cast其实就是直接进行底层二进制的拷贝

### const_cast
为一个类型消除或者增加const修饰符

如下例所示,函数print接收 char* 作为参数，在想要传递一个const char * 的实参时，可以使用const_cast去除const
```c++
// const_cast
#include <iostream>
using namespace std;

void print (char * str)
{
  cout << str << '\n';
}

int main () {
  const char * c = "sample text";
  print ( const_cast<char *> (c) );
  return 0;
}
```


const_cast同样可以作用于`volitale`关键字
# typeid
typeid可以用来检查一个表达式的类型，它返回type_info类型(定义在< typeinfo >头文件)。多个type_info之间可以进行比较。

```c++
// typeid
#include <iostream>
#include <typeinfo>
using namespace std;

int main () {
  int * a,b;
  a=0; b=0;
  if (typeid(a) != typeid(b))
  {
    cout << "a and b are of different types:\n";
    cout << "a is: " << typeid(a).name() << '\n';
    cout << "b is: " << typeid(b).name() << '\n';
  }
  return 0;
}
```

当typeid应用于类对象时，会使用该类对象的RTII信息返回该类对象的实际类型