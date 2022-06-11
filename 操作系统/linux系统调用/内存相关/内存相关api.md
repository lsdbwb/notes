
## mmap和unmmap
### map
为当前进程创建新的虚拟内存区域，并且将对象映射到这些虚拟内存区域
```c++
#include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags,
                  int fd, off_t offset);
```
**参数**
![[Pasted image 20220611171031.png]]
- addr
	新的虚拟内存区域的起始地址（只是一个对内核的建议，实际由内核来选择），如果为空，则由内核选择一个
- length
	映射的虚拟内存区域的大小
- prot
	包含描述新映射的虚拟内存区域的访问权限位（即在相应区域结构中的
prot 位。不能和打开文件时的mode冲突
	![[Pasted image 20220611171315.png]]
- flags
	包含了被映射对象的各种类型
	MAP_SHARED : 表示被映射对象是一个共享对象
	MAP_PRIVATE : 表示被映射对象是一个私有对象
	MAP_ANON ： 表示被映射对象是一个匿名对象，而相应的虚拟页面是请求二进制零的
	
- fd
	要映射的磁盘文件的文件描述符
- offset
	映射从文件的起始偏移量offset开始

**返回值**
成功时返回映射成功的虚拟内存区域的起始地址，失败时，MAP_FAILED (that is, (void * ) -1)被返回，同时设置errno


### munmap
munmap 函数删除从虚拟地址start 开始的，由接下来length 字节组成的区域。接下来对已删除区域的引用会导致段错误

当进程终止时，会自动取消映射。close文件时不会取消映射
```
	#include <sys/mman.h>
 int munmap(void *addr, size_t length);
```

**参数**
- addr : 要取消映射的虚拟内存区域的起始地址
- length :要取消映射的虚拟内存区域的大小

**返回值**
成功时返回0，失败时返回-1并设置错误码errno