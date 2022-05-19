# 动态库相关
 使用时需要链接dl库
## dlsym、dlvsym函数
获取共享库或者可执行文件中某个符号的地址
```c++
		#include <dlfcn.h>
		
       void *dlsym(void *handle, const char *symbol);

       #define _GNU_SOURCE
       #include <dlfcn.h>
       
		// dlvsym和dlsym作用相同，只是多了版本号
       void *dlvsym(void *handle, char *symbol, char *version);

```

**参数**
- handle : 使用dlopen打开的动态库的句柄
	有两种特殊的句柄：
	RTLD_DEFAULT :  使用默认动态库搜索顺序，搜索指定符号出现的第一次的地址
	RTLD_NEXT ： 搜索指定符号出现的下一次的地址
		在链接时指定 LD_PRELOAD选项加载的动态库会被优先查找
- symbol ：符号名

**返回值**
成功时返回symbol的地址，失败时返回NULL，可以使用dlerror来获取错误原因
