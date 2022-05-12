# 协程模块设计
- 有栈协程，使用ucontext[[ucontext_h相关api]]来保存协程上下文和进行上下文切换
- 使用非对称协程模型[[协程相关概念#非对称协程]]，子协程只能和主协程交互。子协程在yield让出时，必须回到主协程，让主协程继续后面的操作。并且子协程在运行结束时，也必须返回主协程，以便程序正常结束
![[Pasted image 20220511173738.png]]

# 协程模块实现
## 协程状态
通常可以划分为三种状态:
- ready ： 协程准备好，可以运行
- running ：协程正在运行
- term ： 协程已经运行完成
![[Pasted image 20220512103151.png]]
## 协程类Fiber
### 成员
```c++
	uint64_t m_id = 0;              // 协程id
    uint32_t m_stacksize = 0;       // 协程栈大小
    State m_state = INIT;           // 协程状态

    ucontext_t m_ctx;               // 协程上下文（寄存器状态）
    void* m_stack = nullptr;        // 协程栈的内存

    std::function<void()> m_cb;     // 该协程的入口函数
```

### 与协程相关的全局变量
- 两个原子变量
```c++
// 存放协程id， 递增生成
static std::atomic<uint64_t> s_fiber_id {0};
// 存放协程数量
static std::atomic<uint64_t> s_fiber_count {0};
```
- 两个线程局部变量
```c++
// 存放该线程当前正在运行的协程的指针
static thread_local Fiber* t_fiber = nullptr;
// 存放该线程的主协程的智能指针
static thread_local Fiber::ptr t_threadFiber = nullptr;
```
- 通过上面两个变量进行主协程，子协程之间的上下文的切换
- 初始化时，t_fiber存放的是主协程的指针

### 成员函数
1. 构造函数
	提供两个构造函数
	- 没有参数的构造函数，被定义为private方法，只能通过静态成员方法GetThis()调用。**主协程**使用该构造函数构造
	```c++
	Fiber::Fiber() {
		// 设置主协程的状态为正在执行
	    m_state = EXEC;
	    // 设置t_fiber = this
	    SetThis(this);
		// 获取上下文
	    if(getcontext(&m_ctx)) {
	        SYLAR_ASSERT2(false, "getcontext");
	    }
		// 协程数量+1
	    ++s_fiber_count;
	
	    SYLAR_LOG_DEBUG(g_logger) << "Fiber::Fiber";
	}

	Fiber::ptr Fiber::GetThis() {
		// 直接返回当前协程
	    if(t_fiber) {
	        return t_fiber->shared_from_this();
	    }
	    // 第一次调用GetThis时构造主协程
	    Fiber::ptr main_fiber(new Fiber);
	    SYLAR_ASSERT(t_fiber == main_fiber.get());
	    // 将主协程指针存到t_threadFiber中
	    t_threadFiber = main_fiber;
	    return t_fiber->shared_from_this();
	}

```

- 有参数的构造函数,public方法，用来生成子协程
```c++
Fiber::Fiber(std::function<void()> cb, size_t stacksize, bool use_caller)
    :m_id(++s_fiber_id)
    ,m_cb(cb) {
    ++s_fiber_count;  // 协程数量+1
    m_stacksize = stacksize ? stacksize : g_fiber_stack_size->getValue();
	// 分配协程栈内存空间
    m_stack = StackAllocator::Alloc(m_stacksize);
    // 使用makecontext构造该协程的上下文
    if(getcontext(&m_ctx)) {
        SYLAR_ASSERT2(false, "getcontext");
    }
    m_ctx.uc_link = nullptr;
    m_ctx.uc_stack.ss_sp = m_stack;
    m_ctx.uc_stack.ss_size = m_stacksize;

    if(!use_caller) {
        makecontext(&m_ctx, &Fiber::MainFunc, 0);
    } else {
        makecontext(&m_ctx, &Fiber::CallerMainFunc, 0);
    }

    SYLAR_LOG_DEBUG(g_logger) << "Fiber::Fiber id=" << m_id;
}
```

2. 协程原语的实现(yield和resume)
- resume
```c++
void Fiber::swapIn() {
	// 保存当前协程的指针到t_fiber
    SetThis(this);
    // 确保协程状态正确
    SYLAR_ASSERT(m_state != EXEC);
    // 设置协程为正在执行状态
    m_state = EXEC;
    // 使用swapcontext切换上下文,主协程换出，当前协程换入执行
    if(swapcontext(&(t_thread_fiber->m_ctx), &m_ctx)) {
        SYLAR_ASSERT2(false, "swapcontext");
    }
}
```
- yield
```c++
void Fiber::swapOut() {
	// 设置t_fiber为主协程
    SetThis(t_thread_fiber.get());
    // 使用swapcontext切换上下文,主协程换入执行，当前协程换出
    if(swapcontext(&m_ctx, &Scheduler::GetMainFiber()->m_ctx)) {
        SYLAR_ASSERT2(false, "swapcontext");
    }
}

//协程切换到后台，并且设置为Ready状态
void Fiber::YieldToReady() {
	// 返回当前协程
    Fiber::ptr cur = GetThis();
    // 将状态从running设置为ready
    cur->m_state = READY;
    // 切换上下文
    cur->swapOut();
}

//协程切换到后台，并且设置为Hold状态
void Fiber::YieldToHold() {
    Fiber::ptr cur = GetThis();
    cur->m_state = HOLD;
    cur->swapOut();
}
```

3. 协程入口函数
在协程运行函数结束之后自动执行yield操作
```c++
void Fiber::MainFunc() {
    Fiber::ptr cur = GetThis(); // GetThis()的shared_from_this()方法让引用计数加1
    SYLAR_ASSERT(cur);
    try {
	    // 执行真正有用的函数
        cur->m_cb();
        // 执行完后设置状态
        cur->m_cb = nullptr;
        cur->m_state = TERM;
	// 处理异常
    } catch (std::exception& ex) {
        cur->m_state = EXCEPT;
        SYLAR_LOG_ERROR(g_logger) << "Fiber Except: " << ex.what()
            << " fiber_id=" << cur->getId()
            << std::endl
            << sylar::BacktraceToString();
    } catch (...) {
        cur->m_state = EXCEPT;
        SYLAR_LOG_ERROR(g_logger) << "Fiber Except"
            << " fiber_id=" << cur->getId()
            << std::endl
            << sylar::BacktraceToString();
    }
	
    auto raw_ptr = cur.get();
    // 手动将该协程的智能指针的引用计数减一
    cur.reset();
    // 协程结束时让出，回到主协程
    raw_ptr->swapOut();

    SYLAR_ASSERT2(false, "never reach fiber_id=" + std::to_string(raw_ptr->getId()));
}
```

4. 协程的重置
	重复利用已经结束的协程，复用其栈空间，创建新协程
	```c++
void Fiber::reset(std::function<void()> cb) {
    SYLAR_ASSERT(m_stack);
    // 确保协程处于可以重置的状态
    SYLAR_ASSERT(m_state == TERM
            || m_state == EXCEPT
            || m_state == INIT);
    // 更新协程的函数
    m_cb = cb;
    //重新构造上下文
    if(getcontext(&m_ctx)) {
        SYLAR_ASSERT2(false, "getcontext");
    }

    m_ctx.uc_link = nullptr;
    // 复用协程栈
    m_ctx.uc_stack.ss_sp = m_stack;
    m_ctx.uc_stack.ss_size = m_stacksize;

    makecontext(&m_ctx, &Fiber::MainFunc, 0);
    m_state = INIT;
}
```

## 其他实现细节
- 关于协程id。sylar通过全局静态变量s_fiber_id的自增来生成协程id，每创建一个新协程，s_fiber_id自增1，并作为新协程的id（实际是先取值，再自增1）。

- 关于线程主协程的构建。线程主协程代表线程入口函数或是main函数所在的协程，这两种函数都不是以协程的手段创建的，所以它们只有ucontext_t上下文，但没有入口函数，也没有分配栈空间。

- 关于协程切换。子协程的resume操作一定是在主协程里执行的，主协程的resume操作一定是在子协程里执行的，这点完美和swapcontext匹配，参考上面协程原语的实现。

- 关于智能指针的引用计数，由于t_fiber和t_thread_fiber一个是原始指针一个是智能指针，混用时要注意智能指针的引用计数问题，不恰当的混用可能导致协程对象已经运行结束，但未析构问题