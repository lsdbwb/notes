## 概念上
- 指针也是一个变量，存放的某个对象的地址
- “引用”可以理解为一个变量的别名

## 使用时
- ”引用“初始化时必须绑定一个变量，并且后续不可以更改绑定的变量。因此引用可以看作一个**指针常量**
- 引用一定绑定了一个对象，因此引用一定不为空，使用时不用每次做检查，相比指针效率更高


## 实际实现
- 在汇编层面，引用和指针并无区别
- 引用只是c++的一个语法糖，可以看作编译器自动完成取地址、解引用的常量指针

