
## 实验要求

- 阅读VirtualAlloc、VirtualFree、VirtualProtect等函数的官方文档
- 编程使用malloc分配一段内存，测试是否这段内存所在的整个4KB都可以写入读取
- 分配一段可读可写的虚内存，写入内存，然后将这段内存改为只读，再读数据和写数据，看是否会有异常情况。然后再释放这段内存，再测试对这段内存的读写释放正常
- 验证不同进程的相同的地址可以保存不同的数据

## 实验原理

### VirtualAlloc function

> 在调用进程的虚拟地址空间中保留，占用或更改页面区域的状态。该函数分配的内存将自动初始化为零。
> 
> --- 
> 首先，必须知道保留(Reserved)内存和占用（Committed）内存的含义。当内存放保留时，一段连续虚拟地址空间被留出。例如，假如我们的程序要使用5 -MB内存块（称为区域），但并不是要马上全部使用，则我们可以调用VirtualAlloc函数，使用MEM_RESERVE分配类型参数。Windows会以64 KB为边界计算该区域的起始地址，并防止进程在同一个范围内为其他内存保留。我们可以指定区域的起始地址，但更常见的是让Windows为区域分配地址。此时除了地址分配外，其他什么也没发生。没有RAM被分配，也没有交换文件空间被保留出来。
> 当我们对内存的需求更迫切时，我们可以再次调用函数VirtualAlloc来占用被保留的内存，调用时使用MEM_COMMIT分配类型参数。现在，区域的起始和结束地址都被计算到4KB边界，对应的交换文件页和所要求的页表被留出来。内存块可以被指定为只读或者可读写。然而，仍然没有RAM被分配；只有当程序访问这部分内存时RAM内存才会被真正分配。如果在此之前内存没有被保留，那就不会有问题；如果在此之前内存被占用了的话，也不会有问题。所以原则是，在使用内存之前一定要先占用。
> 我们可以调用VirtualFree函数“收回”(decommit)占用的内存，使指定的页回到保留的状态。VirtualFree也能够释放保留的内存区域，但我们必须指定其基地址，这个基地址是在前面调用VirtualAlloc保留内存时获得的。

```c
LPVOID VirtualAlloc(
  LPVOID lpAddress,
  SIZE_T dwSize,
  DWORD  flAllocationType,
  DWORD  flProtect
);
```

### 区别

#### VirtualAlloc

> A low-level, Windows API that provides lots of options, but is mainly useful for people in fairly specific situations. 
> - Can only allocate memory in larger chunks
> - One of the most common is if you have to share memory directly with another proces

#### HeapAlloc

> Allocates whatever size of memory you ask for, not in big chunks than VirtualAlloc. HeapAlloc knows when it needs to call VirtualAlloc and does so for you automatically. 
> - Like malloc, but is Windows-only

#### malloc

> The C way of allocating memory. Prefer this if you are writing in C rather than C++, and you want your code to work on e.g. Unix computers too, or someone specifically says that you need to use it.
> - Doesn't initialise the memory. Suitable for allocating general chunks of memory, like HeapAlloc.
> - Visual C++'s malloc calls HeapAlloc.

#### new

> - The C++ way of allocating memory. P
> - Visual studio's new calls HeapAlloc, and then maybe initialises the objects, depending on how you call it.

### VirtualProtect function

> Changes the protection on a region of committed pages in the virtual address space of the calling process.

### mmap

> Linux的虚拟内存管理是基于mmap来实现的。vm_area_struct是在mmap的时候创建的，vm_area_strcut代表了一段连续的虚拟地址，这些虚拟地址相应地映射到一个后备文件或者一个匿名文件的虚拟页。一个vm_area_struct映射到一组连续的页表项。页表项又指向物理内存page，这样就把一个文件和物理内存页相映射。
> - **如果没有匹配到，就报段错误，访问了一个没有分配的虚拟地址**。
> - 如果匹配到了vm_area_struct，根据虚拟地址和页表的映射关系，找到对应的页表项PTE，如果PTE没有分配，就报一个缺页异常，去加载相应的文件数据到物理内存，如果PTE分配，就去相应的物理页的偏移位置读取数据

- `struct vm_area_struct`

> struct vm_area_struct holds information about a contiguous virtual memory area
> - `cat /proc/1/maps`


#### malloc vs mmap

> - malloc，它指的是内存分配
> - mmap，它指的是内存映射系统，自带独一无二的I/O
> 
> ---
> mmap是一个系统调用，它负责并请求内核在应用程序的地址中找到一个未使用的、连续的区域，这个区域足够大，可以进行几页内存的映射。还有就是建立虚拟内存管理结构，不会导致段错误。

## 实验内容

### 测试4KB写入

```c
#include <stdio.h>
#include <malloc.h>

int main()
{
    int *l = (int *)malloc(sizeof(int) * 1024);
    for(int i =0;i<1024;i++)
    {
        l[i]=1;
    }
    
    for (int i = 0; i < 1200; i++)
    {
        l[i] = 1;
    }
}
```

#### 测试

```sh
gcc -Wall -g t.cpp -o t
valgrind --tool=memcheck --leak-check=full ./t
```

<img src='%E8%99%9A%E5%86%85%E5%AD%98/2020-06-28-14-39-57.png' width='' height='350'/>

- `Invalid write`

### 虚内存（虚拟空间）测试

#### 1

分配512B,可读可写
- 尝试将字串`test`拷贝到该段内存并读取

```c
const char s[] = "test";
// 分配512B,可读可写
char *l = (char *)mmap((void *)0x10000000, 4096, PROT_READ | PROT_WRITE, MAP_ANON | MAP_SHARED, -1, 0);
// 写测试
strcpy(l, s);
// 读测试
printf("%s\n", l);
// 释放
munmap(l, sizeof(l));
```

<img src='%E8%99%9A%E5%86%85%E5%AD%98/2020-06-28-11-22-47.png' width='' height='250'/>

查看虚拟空间分配
- 找到进程号pid：`ps -aux | grep ./t`
- `cat /proc/pid/maps`

<img src='%E8%99%9A%E5%86%85%E5%AD%98/2020-06-28-13-16-51.png' width='' height='250'/>

- `rw-s`:read/write/share

#### 2

在**1**的基础上改为只读

```c
// 用mprotect来修改刚才预留的内存的权限（可读不可）
if (mprotect(l, sizeof(l), PROT_READ) != 0)
{
    warn("mprotect error 1");
}
printf("%s\n", l);
strcpy(l, s);
munmap(l, sizeof(l));
```

<img src='%E8%99%9A%E5%86%85%E5%AD%98/2020-06-28-11-30-09.png' width='' height='250'/>

> signal SIGSEGV, Segmentation fault
> - 错误原因：段错误。指针声明时，指向的位置不确定，程序运行时，动态分配内存的时候指针指向异常位置
> - 段错误一般是访问的内存超出了系统给这个程序所设定的内存空间，例如
>   - 访问了不存在的内存地址（已被提前free掉等）
>   - 访问了系统保护的内存地址
>   - 访问了只读的内存地址
>   - 堆栈溢出等

#### 3

在**1**的基础上释放内存并再读写

```c
munmap(l, sizeof(l));
char *ll =(char *) 0x10000000;
printf("%s",ll);
strcpy(ll, s);
```
读

<img src='%E8%99%9A%E5%86%85%E5%AD%98/2020-06-28-11-46-04.png' width='' height='350'/>

写

<img src='%E8%99%9A%E5%86%85%E5%AD%98/2020-06-28-11-46-22.png' width='' height='350'/>

- 释放后无法再对该区域读写

> core dump(核心转储)
> 
> 核心文件（core file），也称磁芯倾印（core dump）[1]，是操作系统在进程收到某些信号而终止运行时，将此时进程地址空间的内容以及有关进程状态的其他信息写出的一个磁盘文件。这种信息往往用于调试。 

### 验证不同进程的相同的地址可以保存不同的数据

使用**1**的代码,分别创建t.cpp与tt.cpp
- 固定Entry point address
  > -Ttext-segment并不是指定.text段的加载位置，而是指定整个elf的加载位置。

```sh
gcc -Wall -g t.cpp -o t -Wl,-Ttext-segment,0x4200000

gcc -Wall -g tt.cpp -o tt -Wl,-Ttext-segment,0x420000

readelf -h ./t | grep Entry
readelf -h ./tt | grep Entry
```
- 固定基地址
  - ps: `0x420000`也是虚地址
<img src='%E8%99%9A%E5%86%85%E5%AD%98/2020-06-28-15-29-22.png' width='' height='100'/>

- 查看内存空间
<img src='%E8%99%9A%E5%86%85%E5%AD%98/2020-06-28-15-29-44.png' width='' height='300'/>

- 改为PROT_PRIVATE

<img src='%E8%99%9A%E5%86%85%E5%AD%98/2020-06-28-15-35-04.png' width='' height='200'/>

- 两个程序使用了相同的内存地址
- 同时运行却能使用相同的地址，是因为使用的是虚拟地址，映射于不同的物理地址

## 参考

- [VirtualAlloc函数使用总结_imJaron的博客-CSDN博客_c virtualalloc 函数返回](https://blog.csdn.net/imJaron/article/details/80157835)
- [winapi - What's the differences between VirtualAlloc and HeapAlloc? - Stack Overflow](https://stackoverflow.com/questions/872072/whats-the-differences-between-virtualalloc-and-heapalloc)
- [C语言mmap()函数：建立内存映射_C语言中文网](http://c.biancheng.net/cpp/html/138.html)
- [C语言基础-程序常见错误（一）_shuaixio的博客-CSDN博客_process terminating with default action of signal ](https://blog.csdn.net/baidu_35692628/article/details/72764829?utm_source=blogxgwz7)
- [ELF entry point和装载地址_ayu_ag的专栏-CSDN博客_elf entry point address](https://blog.csdn.net/ayu_ag/article/details/50737209)
