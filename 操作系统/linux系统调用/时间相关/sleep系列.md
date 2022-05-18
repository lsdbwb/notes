## sleep函数
休眠一段时间(单位：秒)
```c++
unsigned int Sleep(unsigned int seconds);
```

## usleep函数
休眠一段时间(单位：微秒)
useconds_t  = unsigned int 
```c++
unsigned int Sleep(useconds_t usec);
```
- 成功时返回0
- 失败时返回-1，并设置errno