---
layout:     post
title:      Huge core dump 分析
subtitle:   未完待续
date:       2020-06-11
author:     Dongyang
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - 
---

## 对Huge core dump file 产生原因的分析

### 1. Background
客户提出修改Cgroup中，OOM killer的机制，由直接干掉某task，改为生成core dump，这样便于分析为何OOM了。
在某次生成core dump后，客户发现core dump为上百G，而物理内存只有10几G。
So begin to investiage this issue~.


### 2. Analysis 

- 找客户要复现步骤 （客户环境很复杂，出现一次OOM要几个月，也不会把他们的APP给我们，所以基本不指望这个）
- 自己尝试复现


#### 2.1 自己复现

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
通过du -h 观察core dump size。
发现core dump的大小，稳定的为 实际获得的物理内存的大小+1M(左右).
不存在huge core dump 的情况。 

至此，复现失败。

### 3. Analysis again

1. GDB 分析huge core dump 
可以向客户要他们的core dump, 但听说是残缺的,而且几百G, 估计用GDB分析不出什么（主要GDB不熟.....）

2. Read source code
从原理上分析分析，为什么会huge。 但generate core dump，
这部分是kernel代码(未改动)，先不怀疑这部分代码有问题。

3. 查看core dump文件本身，Elf文件的内容应该是按一定格式组织的，
虽然GDB可能读不了，但硬刚 core dump 本身，会不会有什么发现？


#### 3.1 边读代码边硬刚core dump文件

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

在使用了'ls -al'命令后，发现其显示的file size 是51MB。
同时使用 du -h 其大小是8MB。

这里就引入了`文件大小`和`实际占用磁盘大小`这两个概念。

Google了下，发现即存在 ‘文件大小' > ’实际占用磁盘大小’ 
也存在 ‘文件大小’ < '实际占用磁盘大小’ 的情况。

对于'文件大小‘ > '实际占用磁盘大小’的情况，是因为文件中存在 **黑洞**。

首先要理解什么是黑洞，怎么才能产生黑洞？（以下来自《UNIX 环境高级编程》）
在向一个文件中写数据的时候，文件偏移量可以大于文件的当前长度，在这种情况下，对该文件的下一次写将加长该文件，
并在文件中构成一个空洞，这一定是允许的。位于文件中但没有写过的字节都被读为0.
文件中的空洞并不要求在磁盘上占用存储区。具体处理方式与文件系统的实现有关，当定位超出文件尾端之后写时，对于新写的数据需要分配磁盘块，但是对于原文件尾端和新开始写位置之间的部分则不需要分配磁盘块。
 
例如：用dd if=/dev/zero of=a.out seek=1023 bs=1M count=1创建a.out文件后，用ls查看a.out的文件大小为1G，用du查看a.out文件大小为1M。

### 4. root cause
- 因为Linux的‘延时分配’策略，malloc 50MB，实际写入8MB数据的情况下，虚拟内存vma中记录是50M, 物理内存其实只分配8MB.
- core dump 会遍历vma，完整记录下所有vma信息，对于存在物理page的情况，会将实际分配到的内存信息，写入core dump文件。
对于没有分配物理page的情况，会在该位置写入0填充。
- 写0的位置会成为黑洞，不占用磁盘空间（这里跟文件系统有关系，具体为何是黑洞，需要进一步看）。 
这也是为何'ls -al' 为 50MB,  'du -h'为8MB的原因。  
- 至于客户那里为何变为huge core file. 是因为文件被重新写(比如scp)，在写的过程中这些hole都被填充了。



## 记录下source code和core dump的学习






