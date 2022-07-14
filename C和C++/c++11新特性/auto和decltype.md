# auto和decltype
c++11引入了auto和decltype关键字，使得可以在编译期推导出变量或者表达式的类型，方便开发者编码也简化了代码

# auto
auto的主要作用是在编译期推导出变量的类型


>auto的使用规则
>1. auto在使用的时候必须马上初始化，否则无法推导出类型
>2. auto在一行定义多个变量时，各个变量的推导不能产生二义性
>3. auto不能用作函数参数
>4. auto不能用于定义数组，可以用于定义指针
>5. auto无法推导出模板参数

类型推导规则：
- auto在不声明为引用或者指针时，会忽略等号右边的引用和cv限定符
- auto在声明为引用或指针时，会保留等号右边的引用和cv限定符

什么时候使用auto：
1. 在不影响代码可读性的情况下尽量使用auto
2. 复杂类型使用auto，简单的基础类型如int,double就不要用了
3. 有时很难知道某些表达式的类型究竟是啥，例如lambda表达式的返回值

# decltype
decltype用于推导表达式的类型，这里只用于编译器分析表达式的类型，表达式实际不会进行运算

decltype不会像auto一样忽略引用和cv属性，decltype会保留表达式的引用和cv属性

>decltype的推导规则
>exp是表达式，则decltype(exp)的类型和表达式的类型相同
>exp是函数调用，则decltype(exp)的类型和函数返回值的类型相同
>其他情况，若exp是左值，则decltype(exp)返回exp的左值引用


# auto和decltype的配合使用
auto和decltype一般配合使用在*推导函数返回值类型*的问题上
```c++
template<typename T, typename U>
return_value add(T t, U u) { // t和v类型不确定，无法推导出return_value类型
    return t + u;
}
```

使用返回类型后置解决 函数返回值类型依赖于函数参数但却难以确定返回值类型的问题
```c++
template<typename T, typename U>
auto add(T t, U u) -> decltype(t + u) {
    return t + u;
}
```
