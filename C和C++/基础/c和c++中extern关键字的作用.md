# 使用extern修饰变量或者函数
- extern修饰一个变量/函数时表示声明该变量/函数
- extern可以扩展一个变量/函数的可见性。使得编译器在编译当前源文件为目标文件时知道该变量/函数会在其他目标文件中进行定义，到时候进行链接时可以从定义该变量的模块中找到该变量/函数的定义(*函数默认是外部可见的*)
- 具体使用时，通常是在.c文件中定义了一个全局的变量，这个全局的变量如果要被引用，就放在.h中并用extern来声明。其他需要使用这个全局变量的模块include该头文件即可

**例子:**
在a.c文件中定义了一个全局变量x
```c++
//a.c
int x;

void func(){

};
```
在main.c中想要使用a.c中定义的全局变量,于是使用extern int x扩展x的可见性
```c++
//main.c
extern int x;
extern void func();
int main()
{
	//do something to x
	x = 10;
	func();
	printf("%d\n",x);
}
```
编译指令: gcc a.c main.c 

如果在main.c中不加上`extern int x;`,则会在编译a.c时报"x" undeclared的错误。`extern func();`同理。


# extern “C”
因为gcc在编译C模块时和g++在编译c++模块时解析函数名并生成函数符号的规则是不同的。
- c++支持函数重载，所以允许有同名但参数列表不同的函数。g++编译器使用Name mangling的技术为重载函数生成独一无二的符号表符号，以便链接器在链接的时候可以找到正确的函数地址
- c语言不支持函数重载，因此不需要对函数名做什么改变

**何时使用：** 
1. 当我们想要在c++项目中使用gcc编译的c模块时

```c++
//通过extern "C"，告诉g++编译器，不要对这些函数进行Name mangling，按照C编译器的方式去生成符号表符号

extern "C" {

void func1(int x, float y);

void func2(double z);

}
```

2. 当我们想要为c++库提供C接口时

首先有一个c++类，定义在robot.h中
```c++
#pragma once

#include <string>

class Robot
{
public:
    Robot(std::string name) : name_(name) {}

    void sayHi();

private:
    std::string name_;
};
```
成员函数定义在robot.cc中
```cpp
#include <iostream>

#include "robot.h"

void Robot::sayHi()
{
    std::cout << "Hi, I am " << name_ << "!\n";
}
```

我们用编译C++代码的方式，使用 **g++** 编译器对这个类进行编译，此时类 **Robot** 并不知道自己会被C程序调用。

```text
g++ -fpic -shared robot.cpp -o librobot.so
```

接下来用C++创建一个C的接口，定义在 **robot_c___api.h** 和 **robot_c_api.cpp** 中，这个接口会定义一个C函数 **Robot_sayHi(const char *name),** 这个函数会创建一个类 **Robot** 的实例，并调用 **Robot** 的成员函数 **sayHi()。**

**robot_c___api.h**

```text
#pragma once

#ifdef __cplusplus
extern "C" {
#endif

void Robot_sayHi(const char *name);

#ifdef __cplusplus
}
#endif
```

**robot_c_api.cpp**

```text
#include "robot_c_api.h"
#include "robot.h"

#ifdef __cplusplus
extern "C" {
#endif

// 因为我们将使用C++的编译方式，用g++编译器来编译 robot_c_api.cpp 这个文件，
// 所以在这个文件中我们可以用C++代码去定义函数 void Robot_sayHi(const char *name)（在函数中使用C++的类 Robot），
// 最后我们用 extern "C" 来告诉g++编译器，不要对 Robot_sayHi(const char *name) 函数进行name mangling
// 这样最终生成的动态链接库中，函数 Robot_sayHi(const char *name) 将生成 C 编译器的符号表示。

void Robot_sayHi(const char *name)
{
    Robot robot(name);
    robot.sayHi();
}

#ifdef __cplusplus
}
#endif
```

同样用编译C++代码的方式进行编译

```text
g++ -fpic -shared robot_c_api.cpp -L. -lrobot -o librobot_c_api.so
```

![](https://pic1.zhimg.com/80/v2-f2bb844e7b907395add9541dccd5244c_720w.jpg)

现在我们有了一个动态链接库 **librobot_c___api.so,** 这个动态链接库提供了一个C函数 **Robot_sayHi(const char *name)**，我们可以在C程序中调用它了。

**main.c**

```text
#include "robot_c_api.h"

int main()
{
    Robot_sayHi("Alice");
    Robot_sayHi("Bob");

    return 0;
}
```

使用C程序的编译方式，用 **gcc** 对 **main.c** 进行编译

```text
gcc main.c -L. -lrobot_c_api
```


![](https://pic4.zhimg.com/80/v2-7160eae6e0ee23b89952a8a37efc5277_720w.jpg)

可以看到 **gcc** 编译出的函数符号和 **librobot_capi.so**中 **g++** 编译器编译出的**函数符号一致**。这样最终在我们的C程序中可以正确的链接到动态库中的**Robot_sayHi(const char *name)** 函数。

![](https://pic2.zhimg.com/80/v2-56f53555b5829cfbd08b44197a3fd079_720w.jpg)