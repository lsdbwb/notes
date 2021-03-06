# std::enable_shared_from_this< T >
## 定义
- std::enable_shared_from_this< T >是一个模板类，定义在< memory >头文件中
- 使用时需要继承该类，继承该类后就可以调用该类的shared_from_this()方法
## 作用
- 如果某个对象Obj obj继承了std::enable_shared_from_this< Obj >并且是**使用std::shared_ptr< Obj > p_obj来管理**,那么调用shared_from_this()会返回一个和p_obj一样共享obj对象的智能指针，如果obj对象没有被std::shared_ptr管理，调用该方法会出错
## 使用场景
- shared_from_this()是成员函数
- 当一个类被shared_ptr管理，并且在该类的**成员函数中**需要把当前类的对象作为参数传递给其他函数时，这时需要一个指向自身的shared_ptr
```c++
    // 该函数接收Good的智能指针作为参数
void func1(std::shared_ptr<Good> p_good){}

class Good : std::enable_shared_from_this<Good> // note: public inheritance
{
    void func() {
	    // 成员函数func要调用func1
        func1(shared_from_this());
    }
 
};
```
- 在成员函数需要返回this指针的场景下，由于对象使用智能指针来管理，直接返回this指针会破坏智能指针的作用。因此可以使用shared_from_this()返回shared_ptr给调用者
```c++
// 比较推荐的写法
class Good : std::enable_shared_from_this<Good> // note: public inheritance
{
    std::shared_ptr<Good> getptr() {
        return shared_from_this();
    }
};
```