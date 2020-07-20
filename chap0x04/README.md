

## 实验要求

- 安装KLEE，完成官方tutorials（前三个）

<!--more-->

## 实验原理

### 符号执行

- 参考：[对符号执行的理解（1） - 知乎](https://zhuanlan.zhihu.com/p/42831910)

> 符号执行(Symbolic Execution)是指在不执行程序的前提下，用符号值表示程序变量的值，然后模拟程序执行来进行相关分析的技术
> 
> 如果是具体执行(concrete execution，对变量赋值进行测试)，虽然仅有一个变量quantity，但取值范围很大，如果是暴力遍历，则时间复杂度不允许；如果是随机取值，不一定能覆盖所有情况。如果是其他复杂的情况，这样的方法更不可取。那如何能保证可靠性(soundness)和完备性(completeness)呢

### clang-llvm

> Clang 是一个C、C++、Objective-C和Objective-C++编程语言的编译器前端。它采用了LLVM作为其后端
> 
> ---
> LLVM是一个自由软件项目，它是一种编译器基础设施，以C++写成，包含一系列模块化的编译器组件和工具链，用来开发编译器前端和后端。它是为了任意一种编程语言而写成的程序，利用虚拟技术创造出编译时期、链接时期、运行时期以及“闲置时期”的最优化。它最早以C/C++为实现对象，而目前它已支持包括ActionScript、Ada、D语言、Fortran、GLSL、Haskell、Java字节码、Objective-C、Swift、Python、Ruby、Crystal、Rust、Scala以及C#等语言
> 
> LLVM的命名最早源自于底层虚拟机（Low Level Virtual Machine）的首字母缩写，由于这个项目的范围并不局限于创建一个虚拟机，这个缩写导致了广泛的疑惑。LLVM开始成长之后，成为众多编译工具及低端工具技术的统称，使得这个名字变得更不贴切，开发者因而决定放弃这个缩写的意涵

## 实验内容

### 安装和启动klee

```sh
# 启动 docker
sudo systemctl start docker
# 安装 KLEE
sudo docker pull klee/klee:2.1

# 创建一个临时容器(为了测试实验用)
sudo docker run --rm -ti --ulimit='stack=-1:-1' klee/klee:2.1
```
- pull docker时可先换docker源

<img src='%E7%AC%A6%E5%8F%B7%E6%89%A7%E8%A1%8C/2020-07-01-13-38-32.png' width='' height='200'/>

### 1. Testing a Small Function

- 位于`klee_src/examples/get_sign/`下

#### 把输入符号化
 
```c
int main() {
  int a;
  klee_make_symbolic(&a, sizeof(a), "a");
  return get_sign(a);
}
```
> 为了使用KLEE测试这个函数，我们首先需要用符号化的输入去运行这个函数,klee提供了klee_make_symbolic()函数（定义在klee/klee.h中）将变量符号化，该函数接收3个参数：变量地址、变量大小、变量名（可以为任何值）

#### 编译为LLVM位码

```sh
clang -I …/…/include -emit-llvm -c -g -O0 -Xclang -disable-O0-optnone get_sign.c
```

> - `-I<dir>`:添加目录到include搜索路径
> - `-emit-llvm`：对汇编程序和对象文件使用LLVM表示
> - `-c`：只运行预处理、编译和汇编步骤
> - `-g`：将调试信息添加到位代码文件中，以便之后使用该文件生成源代码行级别的统计信息。如果没有这个参数，将无法得到映射在源码层面的一些信息，不方便我们查看结果

#### 运行KLEE

```sh
klee get_sign.bc
```

<img src='%E7%AC%A6%E5%8F%B7%E6%89%A7%E8%A1%8C/2020-07-09-12-12-56.png' width='' height='350'/>

> - 共有33条指令，3条路径，并生成了3个测试用例
> - 输出结果报告放在了 klee-out-0(或者klee-last)目录下(这里说明下，klee每执行一次都会生成一个klee-out-N，其中N是表示第几次的执行，这里我们只执行了一次，因此是0。除此之外，会生成一个klee-last的符号链接，指向最新生成的这个k查看lee-out-N)

#### 利用KLEE生成测试用例

<img src='%E7%AC%A6%E5%8F%B7%E6%89%A7%E8%A1%8C/2020-07-09-13-21-21.png' width='' height='300'/>

```
args: 调用程序时使用的参数,在这只有程序名。
num objects: 符号对象在路径上的数量，在这只有1个。
object 0: name: 符号对面的名字
object 0 :int/uint: 符号对象实际的输入值（这里3个分别对应=0,>0,<0）
```

#### 重新使用测试用例运行程序

> 用klee提供的库来将得到的测试用例作为输入来运行我们的程序
> - `libkleeRuntest`

```sh
export LD_LIBRARY_PATH=/home/klee/klee_build/lib/:$LD_LIBRARY_PATH

gcc -I …/…/include -L /home/klee/klee_build/lib/ get_sign.c -lkleeRuntest

KTEST_FILE=klee-last/test000001.ktest ./a.out

echo $? 
```

<img src='%E7%AC%A6%E5%8F%B7%E6%89%A7%E8%A1%8C/2020-07-09-13-28-36.png' width='' height='120'/>

> 当我们用第一个用例测试，a=0，我们返回值也是0；第二个用例a=16843009,我们返回值为1；第三个用例a=-2147483648,返回值为-1（这里转换到了0-255范围，因此为255）

### 2. Testing a Simple Regular Expression Library

- `klee_src/examples/regexp/`

```sh
clang -I ../../include -emit-llvm -c -g -O0 -Xclang -disable-O0-optnone Regexp.c

klee --only-output-states-covering-new Regexp.bc
```
- `–only-output-states-covering-new`: 用来限制生成测试用例，只有在覆盖了新的代码时才生成

<img src='%E7%AC%A6%E5%8F%B7%E6%89%A7%E8%A1%8C/2020-07-09-13-35-52.png' width='' height='100'/>

报错原因

<img src='%E7%AC%A6%E5%8F%B7%E6%89%A7%E8%A1%8C/2020-07-09-13-39-39.png' width='' height='150'/>

> klee发现程序中2个内存错误不是因为正则函数中存在bug，而是我们的测试驱动中存在一个问题。我们将输入完全符号化了，但是匹配函数期待的是以null为结尾的字符串。

解决
- 强制指定'\0'
- `klee_assume`

```c
//1
re[SIZE - 1] = '\0';

//2
klee_assume(re[SIZE - 1] == '\0');
```

### 3. Solving a maze with KLEE

```sh
sudo git clone https://github.com/grese/klee-maze.git ~/maze

sudo docker cp ~/maze/maze.c <CONTAINER ID>:/home/klee/klee_src/examples/
```

```sh
gcc maze.c -o maze
# solution: ssssddddwwaawwddddssssddwwww

```
> 但是，这么多个报告，我们不可能每一个都去打开自己尝试（不然用KLEE有什么意义？），因此我们需要KLEE能够给出我们那些路径是正确的，即能找到出口

自动化
> 这里有一个类似于C语言里面的断言的函数klee_assert()，它强制条件为真，否则就会中止执行。我们可以用断言来标记我们感兴趣的部分代码，一旦KLEE执行到这个地方就会给出提醒。


```c
//并p添加头文件：
#include<klee/klee.h>

//read(0,program,ITERS);
//修改为
klee_make_symbolic(program,ITERS,"program");

//printf ("You win!\n");
//修改为
printf ("You win!\n");
klee_assert(0);
```

```sh
clang -c -I ../include/ -emit-llvm maze.c -o maze.bc

klee -emit-all-errors  maze.bc 

ls klee-last | grep err
```

<img src='%E7%AC%A6%E5%8F%B7%E6%89%A7%E8%A1%8C/2020-07-09-15-31-38.png' width='' height='200'/>

- 有几个可以穿墙..

## 常见问题

### Job for docker.service failed because the control process exited with error code

- `sudo dockerd --debug`

## 参考

- [Docker · KLEE](https://klee.github.io/docker/)
- [How to fix "Job for docker.service failed because the control process exited with error code" - Stack Overflow](https://stackoverflow.com/questions/55906503/how-to-fix-job-for-docker-service-failed-because-the-control-process-exited-wit)
- [KLEE学习——实例1_qq_26736193的博客-CSDN博客_klee测试多个c语言文件](https://blog.csdn.net/qq_26736193/article/details/107150957)

