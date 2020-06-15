---
layout:     post
title:      Huge core dump 分析
subtitle:   复盘一些cases，争取更快更准的定位问题
date:       2020-06-11
author:     Dongyang
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - Kernel
---

## 对Huge core dump file 产生原因的分析

### 1. Background
客户提出修改Cgroup中OOM killer的机制，由直接干掉某task，改为生成core dump，这样便于分析为何OOM了。
在某次生成core dump后，客户发现core dump为上百G，而物理内存只有10几G。
同时客户对core进行了压缩，发现压缩程序存在报错情况。
So begin to investigate this issue~.

### 2. Analysis 
- 找客户要复现步骤 （客户环境很复杂，出现一次OOM要几个月，也不会把他们的APP给我们，其步骤参考意义有限）
- 找客户要Core文件及出现core时的/proc等信息 
- 自己尝试复现

#### 2.1 GDB读core文件及分析/proc
由于客户不提供他们的APP，所以GDB意义有限。加载后，提示文件长度与预期不一致（由此说明该core生成时出了问题）。
客户有一套完整的dump流程，除了core外还有proc, kernel config 等等信息会一并dump出来，然鹅，此次dump出来的东西，只有一个不完整的core，其他均无。。。（由此说明发生core时其系统环境已经不稳定了，可能存在磁盘或物理内存不足的情况）

#### 2.2 自己复现
测试代码如下
```
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

#define SIZE 500

int main(int argc, char **argv)
{
    int max = -1;
    int mb = 0;
    char *buffer;
    int i;
    unsigned int *p = malloc(1024 * 1024 * SIZE);

    printf("malloc buffer: %p\n", p);

    for (i = 0; i < 1024 * 1024 * (SIZE/sizeof(int)); i++) {
        p[i] = 123;
        if ((i & 0xFFFFF) == 0) {
            printf("%dMB written\n", i >> 18);
            sleep(5);
        }
    }
    return 0;
}
```
在Cgroup中设置memory limit为20MB，观察core dump文件大小。
同时在非cgroup环境中，用'kill -6 PID' 的方式生成core dump 做对比观察
通过du -h 观察core dump size。 发现core dump的大小，稳定的为 实际获得的物理内存的大小+1M(左右).
不存在huge core dump 的情况。 
** 至此，复现失败。**

### 3. Analysis again
- Read source code
从原理上分析分析，为什么会huge。 但generate core dump，这部分是kernel代码(未改动)，先不怀疑这部分代码有问题。
- 查看core dump文件本身
 Elf文件的内容应该是按一定格式组织的，虽然GDB可能读不了，但硬刚 core dump 本身，会不会有什么发现？

其实目前只有一个core文件，没有APP，没有/proc信息，无法知道出现core时，系统的内存容量剩余多少，磁盘容量剩余多少，以及APP期望获得和实际获得的物理内存是多少，在很多信息缺少的情况下，也只有读源码和读core文件这两条路了。

#### 边读代码边硬刚core dump文件
客户的core dump 太大， 先分析自己的core.
**该core生成条件为，malloc 50MB，在向内存实际写入8M数据后，收到Abort信号。**
在使用readelf -l core.file 读取了core的头部信息后
```
Elf file type is CORE (Core file)
Entry point 0x0
There are 17 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  NOTE           0x00000000000003f8 0x0000000000000000 0x0000000000000000
                 0x0000000000000a3c 0x0000000000000000         0
  LOAD           0x0000000000001000 0x0000000000400000 0x0000000000000000
                 0x0000000000001000 0x0000000000001000  R E    1000
  LOAD           0x0000000000002000 0x0000000000410000 0x0000000000000000
                 0x0000000000001000 0x0000000000001000  RW     1000
  LOAD           0x0000000000003000 0x0000000000411000 0x0000000000000000
                 0x0000000000021000 0x0000000000021000  RW     1000
```
通过累加各个LOAD segment的 FileSize，惊奇的发现，其大小合计为**51MB**, 而不是8MB. 遂开始仔细观察core dump文件。 
在使用了'ls -al'命令后，发现其显示的file size 是51MB。 同时使用 du -h 其大小是8MB。

这里就引入了`文件大小`和`实际占用磁盘大小`这两个概念。

Google了下，发现即存在 ‘文件大小' > ’实际占用磁盘大小’ ，也存在 ‘文件大小’ < '实际占用磁盘大小’ 的情况。

对于'文件大小‘ > '实际占用磁盘大小’的情况，是因为文件中存在 **黑洞**。

>首先要理解什么是黑洞，怎么才能产生黑洞？（以下来自《UNIX 环境高级编程》）
在向一个文件中写数据的时候，文件偏移量可以大于文件的当前长度，在这种情况下，对该文件的下一次写将加长该文件，
并在文件中构成一个空洞，这一定是允许的。位于文件中但没有写过的字节都被读为0.
文件中的空洞并不要求在磁盘上占用存储区。具体处理方式与文件系统的实现有关，当定位超出文件尾端之后写时，对于新写的数据需要分配磁盘块，但是对于原文件尾端和新开始写位置之间的部分则不需要分配磁盘块。
 
>例如：用dd if=/dev/zero of=a.out seek=1023 bs=1M count=1创建a.out文件后，用ls查看a.out的文件大小为1G，用du查看a.out文件大小为1M。

更准确的描述为，Linux支持稀疏文件系统Sparse filesystem，所以允许这样的文件存在。
通过阅读源码，证明了这点（core dump代码会在下文分析）。
而在使用不同的工具操作稀疏文件时，可能会导致其“黑洞”被填充的情况。 
比如，scp命令不支持稀疏文件， 如果你将该稀疏文件scp到另一机器，其大小为ls -al的大小，即所有空洞都被真实填充为0了。
**至此，怀疑core本身生成的没问题，客户在APP中申请了大量内存（几百G），在生成core后因为客户的某种操作使得黑洞被填充了，进而出现了Huge core** 
接下来就是调查客户到底对生成的core都干了啥，好不容易拿到了客户的dump相关代码，发现其仅对core进行了压缩操作，并无额外动作。 
模仿客户的流程对我自己的测试core操作了几遍，发现黑洞都没有被填充。
> lz4, tar 等压缩命令支持稀疏文件，在板子上生成core后，最好压缩一下，再scp到Host，这样就不会出现黑洞被填充的问题  

**至此，路又有些走不通了。。**

### 4. Analysis again and again
在发现core是稀疏文件，可能会被填充后，一度以为找到了Root Cause.  既然走不通，只能再次分析了。
手里只有一个core，无法知道出现core时的系统各项资源的情况，看来只能在这个core身上再dig dig了。

简单学习了Core文件的ELF格式，通过readelf -l core命令，打出了其header信息，发现其segment有几万个之多。
再观察这些segment的memory和file size部分，发现其中绝大部分是 **成对出现的0x7fff00和0x1000**。
结合该APP运行在server上，直觉告诉我，这些segment可能是一个个的threads。

但0x1000的Flags为空，没太明白意思。 计算了 0x7fff000 * 其出现数量， 发现大小与GDB提示的预计文件大小基本相同。
查看了板子上的stack info，发现其大小是0x800000。这与0x7fff000感觉有些关联。
```
Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
                 0x0000000000ca3000 0x0000000000ca3000  RW     1000
  LOAD           0x0000000001028000 0x00003fbfae000000 0x0000000000000000
                 0x0000000000001000 0x0000000000001000         1000
  LOAD           0x0000000001029000 0x00003fbfae001000 0x0000000000000000
                 0x00000000007ff000 0x00000000007ff000  RW     1000
  LOAD           0x0000000001828000 0x00003fbfae800000 0x0000000000000000
                 0x0000000000001000 0x0000000000001000         1000
```
光怀疑没用，还得实践。 自己创建了10个thread，生成core后，readelf一看，证明了自己的猜想是正确的，core中有10个0x7fff000 以及10个以上的0x1000. 

**即，客户APP出现core时，其APP  create了几万个Threads.  **
虽然这在大型服务上，可能是小case，不过在嵌入式领域，在一个10几G内存的板卡上...这个情况就不一定合适了。
模拟客户情况，创建了与客户一样多的Threads, 在创建到3W多个时，提示了创建失败，并且系统出现了无法reboot，无法ps等异常问题， 说明创建这么多Threads，在当前环境下不太合理。

把该信息告知给客户后，FAE反馈说该信息对客户很有意义，但其出现该core时，其系统并没有打文章开头提到的OOM时主动产生core的patch，即该APP为何会收到ABORT信号？
**至此，问题其实已经变为，协助客户分析APP为何会收到ABORT**

这个问题比较简单，Google结果如下：
>`abort()`  is usually called by library functions which detect an internal error or some seriously broken constraint. For example  `malloc()`  will call  `abort()`  if its internal structures are damaged by a heap overflow.
[https://stackoverflow.com/questions/3413166/when-does-a-process-get-sigabrt-signal-6](https://stackoverflow.com/questions/3413166/when-does-a-process-get-sigabrt-signal-6)

这个情况与客户的情况很相似，上万个Threads很可能是导致了malloc分配失败，进而触发abort的。
到此，客户主动关闭了这个case（应该是以此为线索找到了他们自己的root cause），算是该问题得到解决。

于我个人而言，引出了一个新的思考问题：
如何monitor glibc里标准函数的一些行为动作？ 比如如何监控 malloc函数 是否触发了abort?

### 待记录 core dump的 soure code，core dump elf格式 相关的学习
### 待记录如何监控库函数
<!--stackedit_data:
eyJoaXN0b3J5IjpbNDE5ODU1NjY2LDEwMzc1NzMwODVdfQ==
-->
