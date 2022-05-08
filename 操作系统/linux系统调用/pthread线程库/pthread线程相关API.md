# 线程状态相关
## pthread_create函数
当前进程创建一个新线程，新线程调用start_routine函数开始执行
```c++
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,                    void *(*start_routine) (void *), void *arg);
```
参数：
```
1. pthread_t *thread : 新线程的id
2. const pthread_attr_t *attr ：指定线程创建时的属性，可以使用pthread_attr_init()来初始化
3. void *(*start_routine) (void *) ：线程要运行的函数
4. void *arg ：要传递给线程函数的参数
```
## pthread_setname_np函数
设置线程的名字
```c++
 int pthread_setname_np(pthread_t thread, const char *name);
```
- 使用pthread_create创建的线程的名字默认是程序名。设置线程的名字可以方便多线程下程序的调试
- 设置的名字不能超过16个字符(最后一个字符为'\0')

## pthread_getname_np函数
获得线程的名字
```c++
int pthread_getname_np(pthread_t thread,
                              char *name, size_t len);
```

## pthread_join函数
等待线程标识为thread的线程终止
```c++
int pthread_join(pthread_t thread, void **retval);
```
- 如果thread已经终止，则直接返回
- thread必须是joinable的
- retval用来接受thread的状态（thread在调用pthread_exit()时提供的状态）

## pthread_detach函数
设置线程为detached状态
`int pthread_detach(pthread_t thread);`
- 设置为detached状态的线程自动将资源还回给系统，不用其他线程join
- 对已经处于detached状态的线程调用该函数会产生未定义行为

# 锁相关
## pthread_rwlock_t
读写锁

## pthread_mutex_t

## pthread_spinlock_t

# 信号量
类型 ： sem_t
- 初始化一个匿名信号量
`int sem_init(sem_t *sem, int pshared, unsigned int value);`
1. sem是要初始化的信号量
2. pshared=0 ：信号量在线程间共享 ； =1 ： 在进程间共享
3. value是信号量初始化的值
- 等待一个信号量就绪
```c++
int sem_wait(sem_t *sem);  //阻塞等待

int sem_trywait(sem_t *sem);  //非阻塞等待

//阻塞等待，有时间限制
int sem_timedwait(sem_t *sem, const struct timespec *abs_timeout);
```
- 信号量加一
```c++
int sem_post(sem_t *sem);
```
如果信号量大于0，则会唤醒等待队列上的线程