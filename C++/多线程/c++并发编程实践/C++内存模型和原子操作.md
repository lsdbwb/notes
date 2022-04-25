# 内存模型基础
内存模型有两个层面：
- structural aspects
	所有的东西在内存中如何布局
- concurrency aspects

## 对象和内存位置
- 在C++中，所有的数据都由对象组成
- 一个对象总要存储在一个或多个内存位置上
- 如下图所示，阐释了对象和内存位置的关系
![[Pasted image 20220421161740.png]]
1. my_data是一个struct对象，由许多子对象构成
2. 每一个变量都是一个对象
3. 每个变量都至少占据一个内存位置
4. 基础变量类型比如int,char，float都只占据一个内存位置（不管他们的大小是几个字节）
5. 临近的位域属于同一个内存位置
6. string对象s内部占据多个内存位置

## 对象、内存位置和多线程的关系
- 关系
1. 如果两个线程访问独立的内存位置，则不会产生竞争
2. 如果两个线程访问同一个内存位置，但是都只是**读**这个内存位置的对象，也不会产生竞争
3. 如果有某个线程会**修改**内存位置的数据，则可能产生race condiiton
- 为了避免race condition,必须强制两个线程访问同一个内存位置的顺序
	1. 使用锁来保证顺序
	2. 可以使用原子操作来保证顺序
- 如果既没有保证两个线程访问同一块内存位置的顺序，又没有使用原子操作，则可能产生数据竞争和**未定义行为**
- 使用原子操作可以避免未定义行为
	原子操作没有解决data race，因为不确定两个线程的原子操作访问同一块内存位置的顺序是怎样的，但是原子操作要么完成，要么不做，不会产生中间的未定义的状态，因此可以避免未定义行为

## 对象修改顺序
- c++程序的每个对象在多线程下从对象初始化开始到最终被销毁的整个生命周期内都应该有确定的**修改顺序**
- 每个对象的修改顺序在每个线程的视角下都应该是一样的

# c++原子操作和原子类型
- 原子操作是不可再切分的
- 使用原子读操作和原子写操作来对一个对象进行处理时，原子读操作要么读到对象的初始状态，要么读到某次修改完成后的状态，不会读到未定义的中间状态
## 标准原子类型
- 所有的原子类型都定义在atomic头文件中
- 在c++的定义中，只有在**原子类型**上进行的操作才是原子的，在非原子类型上进行的操作都不是原子的
- 可以用加锁来模拟原子操作
- 事实上，标准的原子类型也可能是使用锁来模拟的。大部分原子类型有一个is_lock_free()成员函数，如果调用该函数返回true，说明该原子类型是无锁的，是用**原子指令**做的；如果调用该函数返回false，说明该原子类型是编译器或者运行库用**内部锁**来模拟的
- std::atomic_flag
	1. 唯一一个没有is_lock_free()成员函数的原子变量，因为它是最简单的Boolean Flag,对该变量的所有操作都必须是lock-free的。
	2. 可以用它来实现一个简单的锁，然后作为实现其他所有原子类型的基础
	3. 只支持简单的 load,clear()操作，不支持赋值，不支持拷贝构造，不支持其他复杂的操作
- 所有剩下的原子类型都可以通过std::atomic<>类模板特化而来（大多数平台会尽量使得内置类型的原子类型比如std::atomic< int >是lock-free的）
- 除了使用std::atomic<>特化来使用原子类型外，还可以使用原子类型的别名
![[Pasted image 20220422152640.png]]
- c++对于一些typedef定义的类型比如std::size_t也会有相应的原子类型的名称，都是在前面加上atomic_
![[Pasted image 20220422152824.png]]
- 原子类型支持很多操作
- 原子类型支持的操作大多都有一个memory order参数

## std::atomic_flag支持的操作
- 属性
	1. std::atomic_flag只有两种状态,clear或者set，并且同一时刻只能处于其中一个
	2. 必须用ATOMIC_FLAG_INIT来进行初始化
	 `std::atomic_flag f=ATOMIC_FLAG_INIT;`
	3. 在第一次声明或者使用时一定会进行初始化
	4. 初始化后只能进行以下操作：
		 1. 销毁（deconstructor()）
		 2. 清除（clear()）
		 3. 设置和查询**上一个值**（test_and_set())
	5. clear()和test_and_set()操作都可以指定memory order
	6. 拷贝构造，赋值等操作都不是原子的，因为都涉及两个对象,(需要读一个对象的值然后赋值给另一个对象），因此都不被允许
- std::atomic_flag的属性使得其很适合当自旋锁
```c++
// 可以使用std::lock_guard<>来使用该自旋锁
// 因为有基本的lock()和unlock()操作
class spinlock_mutex
{
	std::atomic_flag flag;
public:
	spinlock_mutex():
		flag(ATOMIC_FLAG_INIT)
	{}
	void lock()
	{
	//当前线程调用test_and_set返回false时(说明flag被其他线程设置为false，当前线程设置为ture，并返回上一次的false状态)退出循环，获得锁
		while(flag.test_and_set(std::memory_order_acquire));
	}
	void unlock()
	{
	// 清空flag即为解锁（回到false的状态）
		flag.clear(std::memory_order_release);
	}
};
```
上面实现的简单的自旋锁不是最优的，因为线程调用lock()时，如果flag一直为true，说明其他线程正在占用该锁，当前线程会一直循环，一直占用cpu。
## std::atomic< bool >
- std::atomic_flag操作太过受限，甚至不能当作一个真正的布尔类型来使用，因为它不支持单纯的读操作
- std::atomic< bool >支持的操作
|操作|属性|举例|
|------|------|------|
|使用普通bool变量来构造| |std::atomic< bool > b(true);|
|使用普通变量来赋值|赋值运算符返回的不是reference，而是普通原子变量的value，这可以避免返回引用时需要显式地调用load()函数（不调用load()其他线程可能会在返回引用的中途修改该变量）|b=false;|
|load()|可以读内存中的原子变量到普通的bool变量|std::atomic< bool > b;  bool x=b.load(std::memory_order_acquire);|
|store()|  |b.store(true);|
|exchange()|read-modify-write operation|x=b.exchange(false,std::memory_order_acq_rel);|
|compare_exchange_weak()|CAS操作||
|compare_exchange_strong()|CAS操作||


## std::atomic< T* >
原子指针类型
std::atomic< T* >有std::atomic< bool >类型相同的操作和语义，同时多了一些指针的算术操作。
|操作|属性|
|-----|-----|
|fetch_add()|原子操作，对存储的指针地址加值|
|fetch_sub()|原子操作，对存储的指针地址减值|
|+=/++|fetch_add()的包装|
|-=/--|fetch_sub()的包装|
- fetch_add和fetch_sub都是原子的read-modify-write操作，fetch_add调用后，原子指针变量存放的地址会相应增加，fetch_add返回的是变量**原来存的地址的旧值**。运算符则返回的是新地址的值。
![[Pasted image 20220425150124.png]]
- fetch_add和fetch_sub都可以指定memory order
- +=/++和-=/--运算符都不能指定内存顺序，其memory order是memory_order_seq_cst

## 使用std::atomic<>模板自定义原子变量
使用std::atomic< UDT >自定义原子类型时，UDT(用户自定义类型)需要满足的条件：
1. UDT有trivial copy-assignment operator（不能有虚基类，虚函数，不然会有虚表），因此必须使用编译器生成的拷贝复制运算符；并且每一个基类和非静态成员变量都必须有trivial copy-assignment operator。这是必须的，使得编译器可以使用memcopy()或者类似的操作来进行拷贝赋值
2. UDT必须是bitwise equality comparable，即可以使用memcmp来比较两个UDT对象的值，这确保了compare and exanchge操作能够被正确执行
3. UDT类型越简单（可以看作一个字节数组），编译器越可能使用原子指令来实现原子操作
4. 对于std::atomic< float > or std::atomic< double >等浮点数类型，没有原子的arithmetic operations
5. 如果UDT类型的大小等于int或者void* 的大小，则大多数平台可以使用原子指令来实现std::atomic< UDT >
6. 如果UDT类型的size是两倍的int大小，有些平台还支持double-word-compare-and-swap (DWCAS) instruction来实现原子操作
7. 复杂的类型还是使用std::mutex来保护共享数据

## 各个原子类型支持的操作
![[Pasted image 20220425151627.png]]
- 以上都是原子类型的成员函数，C++还提供对应的非成员函数(free functions),通常都以atomic_开头，比如std::atomic_load()。
- free functions被设置为C兼容的风格，因此是传指针而不是传引用。std::atomic_load()调用时需要传入变量的指针。
- 如果要指定memory order,则需要调用std::atomic_load_explict()这类的函数。
- c++标准库对std::shared_ptr<>提供了原子操作，std::shared_ptr<>本身不是原子类型，c++标准委员会认为为std::shared_ptr<>提供原子操作是必须的
	std::shared_ptr<>支持load, store, exchange, and compare/exchange操作
```c++
std::shared_ptr<my_data> p;
void process_global_data()
{
	std::shared_ptr<my_data> local=std::atomic_load(&p);
	process_data(local);
}
void update_global_data()
{
	std::shared_ptr<my_data> local(new my_data);
	std::atomic_store(&p,local);
}
```
# 内存模型的并发层面
原子操作如何用来在多线程间同步数据和强制内存修改顺序
