# C中的static
在C语言中，static是一个存储类说明符
**修饰变量：**
1. 使用static修饰的变量具有静态存储时期

2. 如果用static修饰全局变量，那么该变量就从外部链接变成了内部链接，只能在本文件中访问和操作

3. 如果用static修饰局部变量，那么该变量就从自动存储时期变成了静态存储时期
**修饰函数：**
函数默认都是静态存储时期和外部链接的
因此使用static修饰函数的作用就是使得函数只能在被定义的文件中被访问和操作

# C++中的static
C++中增加了类的概念，因此static可以修饰类对象相关的东西
1. 修饰类对象
和修饰变量一样，在代码块作用域内声明一个static类对象，也会使得该对象具有静态存储时期，同时static对象也是会使用构造函数进行初始化

c++支持局部静态对象和全局静态对象
- 局部静态对象
```c++
#include <iostream>
class Test 
{
public:
    Test()
    {
        std::cout << "Constructor is executed\n";
    }
    ~Test()
    {
        std::cout << "Destructor is executed\n";
    }
};
void myfunc()
{
	std::cout << "enter myfunc" << times << std::endl;
    static Test obj;
} // Object obj is still not destroyed because it is static
  
int main()
{
    std::cout << "main() starts\n";
    myfunc(1);    // Destructor will not be called here
    myfunc(2);
    std::cout << "main() terminates\n";
    return 0;
}
```
运行结果：
![[Pasted image 20220621105124.png]]
1. 局部静态对象具有静态存储时期（空链接），该对象在myfunc里被声明，但是出了myfunc函数作用域之后，仍然没有被析构，而是在main函数结束之后被析构
2. 局部静态对象的构造函数在*运行到myfunc的声明语句时才被调用*（c++的延迟初始化）
3. 多次调用myfunc，obj的构造函数只会被调用一次（如果多个线程并发调用myfunc，静态局部变量obj只会被初始化一次，其作用相当于使用std::once修饰了myfunc[[3. 在线程间共享数据#有时共享数据只需要在初始化时被保护，初始化完成后，共享数据是read-only的]]）

- 全局静态对象
```c++

#include <iostream>
class Test
{
public:
    int a;
    Test()
    {
        a = 10;
        std::cout << "Constructor is executed\n";
    }
    ~Test()
    {
        std::cout << "Destructor is executed\n";
    }
};
static Test obj;
int main()
{
    std::cout << "main() starts\n";
    std::cout << obj.a;
    std::cout << "\nmain() terminates\n";
    return 0;
}
```
运行结果：
![[Pasted image 20220621103548.png]]
全局静态对象具有静态存储时期（内部链接）
全局静态对象的构造函数在*main函数之前就被调用*

2. 修饰类的成员变量
static成员变量变成类拥有的而不是对象拥有的（需要使用类名::函数名来使用）
3. 修饰类的成员函数
static成员函数变成类拥有的而不是对象拥有的，函数不能访问类的非静态成员变量了（因为static成员函数没有this指针）