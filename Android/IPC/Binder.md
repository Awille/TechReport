[TOC]



# Binder实现解析

## 1、背景

以往在抽象的基础上理解Binder通信，比如Binder的C/S架构，内存映射、单次复制等等， 但一直没有在实现的层面去看Binder的实现细节。在当前背景下，鸿蒙的出现，苹果 ios系统的AI能力集成，如果一直停留在Android的层面去理解IPC通信的上层实现方式，有些局限，希望通过本次的细节分析，整理出大致的框架，理解在计算机底层是怎么完成一次跨进程通信的，以及Android Binder实现方式的优劣是什么，这种设计的考量是什么。



## 2、Binder架构简介

![upgit_20240727_1722058417.png](https://raw.githubusercontent.com/Awille/MyBlog/main/img/2024/07/upgit_20240727_1722058417.png)





系统Process创建过程：
![upgit_20240728_1722157941.png](https://raw.githubusercontent.com/Awille/MyBlog/main/img/2024/07/upgit_20240728_1722157941.png)

http://codemx.cn/2017/09/13/AndroidOS005-Process/index.html







Binder驱动：https://juejin.cn/post/7062654742329032740

https://zenki2001cn.github.io/Wiki/Android/Binder%E5%86%85%E6%A0%B8%E9%A9%B1%E5%8A%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.html



上：https://juejin.cn/post/7062654742329032740
中：https://juejin.cn/post/7069675794028560391

下：https://juejin.cn/post/7073783503791325214



https://juejin.cn/post/7062654742329032740



http://www.wowotech.net/linux_kenrel/binder1.html



https://gityuan.com/2015/11/01/binder-driver/



wsl：

https://blog.csdn.net/feinifi/article/details/129572652



## 3、Binder初始化

### 3.1、数据结构

binder_alloc.c中持有了一个struct list_lru binder_freelist数据结构

类型为list_lru 

```c
struct list_lru {
	struct list_lru_node	*node;
#ifdef CONFIG_MEMCG
	struct list_head	list;
	int			shrinker_id;
	bool			memcg_aware;
	struct xarray		xa;
#endif
};

struct list_head {
	struct list_head *next, *prev;
};

struct list_lru_node {
	/* 自旋锁 protects all lists on the node, including per cgroup */
	spinlock_t		lock;
	/* global list, used for the root cgroup in cgroup aware lrus */
	struct list_lru_one	lru;
	long			nr_items;
} ____cacheline_aligned_in_smp;
```

初始化binder内存回收

drivers\android\binder_alloc.c

```c
int binder_alloc_shrinker_init(void)
{
	int ret;

	ret = list_lru_init(&binder_freelist);
	if (ret)
		return ret;

	binder_shrinker = shrinker_alloc(0, "android-binder");
	if (!binder_shrinker) {
		list_lru_destroy(&binder_freelist);
		return -ENOMEM;
	}

	binder_shrinker->count_objects = binder_shrink_count;
	binder_shrinker->scan_objects = binder_shrink_scan;

	shrinker_register(binder_shrinker);

	return 0;
}
```

include\linux\list_lru.h

```c
#define list_lru_init(lru)				\
	__list_lru_init((lru), false, NULL, NULL)

int __list_lru_init(struct list_lru *lru, bool memcg_aware,
		    struct lock_class_key *key, struct shrinker *shrinker)
{
	int i;

#ifdef CONFIG_MEMCG
	if (shrinker)
		lru->shrinker_id = shrinker->id;
	else
		lru->shrinker_id = -1;

	if (mem_cgroup_kmem_disabled())
		memcg_aware = false;
#endif

	lru->node = kcalloc(nr_node_ids, sizeof(*lru->node), GFP_KERNEL);
	if (!lru->node)
		return -ENOMEM;

	for_each_node(i) {
		spin_lock_init(&lru->node[i].lock);
		if (key)
			lockdep_set_class(&lru->node[i].lock, key);
		init_one_lru(&lru->node[i].lru);
	}

	memcg_init_list_lru(lru, memcg_aware);
	list_lru_register(lru);

	return 0;
}
```



# 附录

这些概念是与操作系统相关的常见操作和机制，下面是它们的简要介绍：

1. **poll**:
   - **描述**：`poll` 是一个系统调用，用于查询多个文件描述符（文件句柄）是否准备好进行 I/O 操作而不会被阻塞。
   - **工作原理**：通过 `poll` 系统调用，进程可以监视一组文件描述符，确定它们是否已准备好进行读取、写入或异常处理，而无需阻塞等待。
2. **mmap**:
   - **描述**：`mmap`（memory map）是一种通过将文件或设备映射到内存的方式来实现文件 I/O 的机制。
   - **工作原理**：通过 `mmap` 系统调用，进程可以将一个文件或设备映射到其地址空间中的一块内存区域，允许直接在内存中对文件进行读写操作，而不需要通过传统的 `read` 和 `write` 系统调用。
3. **flush**:
   - **描述**：`flush` 是指将缓冲区中的数据刷新（写入）到其对应的实际存储介质（如磁盘）中。
   - **工作原理**：在操作系统中，写入数据通常会首先存储在内存中的缓冲区中，以提高写入效率。`flush` 操作强制将缓冲区中的数据写入到实际的存储介质中，以确保数据的持久性。
4. **ioctl**:
   - **描述**：`ioctl`（input/output control）是一个通用的 I/O 控制系统调用，用于在设备驱动程序和用户空间程序之间进行交互。
   - **工作原理**：`ioctl` 允许用户空间程序向设备驱动程序发送命令以执行各种操作，如配置设备参数、控制设备行为等。它提供了一种灵活的机制，用于处理设备特定的操作，而不需要通过标准的读写操作进行。







`pmap` 命令用于显示进程的内存映射情况，包括虚拟内存地址与物理内存的映射关系。在 `pmap` 输出中，`RSS` 表示 "Resident Set Size"，即进程当前驻留在物理内存中的部分，也就是实际正在使用的物理内存量。

`RSS` 反映了进程当前实际使用的物理内存，包括代码段、数据段、堆栈等部分。`RSS` 不包括进程的虚拟内存（例如未分配的部分）或共享的内存（例如共享的动态链接库）。

通过观察 `RSS` 可以了解进程当前实际占用的物理内存量，有助于监视进程的内存使用情况，诊断内存泄漏或性能问题。



以下是一个 `pmap` 命令的示例输出：

basic

复制

```
00400000   2008K r-x--  /usr/bin/example_binary
00602000    108K rw---  /usr/bin/example_binary
00734000     12K rw---  [ anon ]
40000000   1024K rw---  [ anon ]
40300000    128K rw---  [ anon ]
40400000    768K rw---  [ anon ]
```

现在让我们来解释如何确认哪些是数据段（Data Segment）、文本段（Text Segment）、栈（Stack）、堆（Heap）：

1. **数据段（Data Segment）**:
   - 数据段包含程序中已初始化的全局变量和静态变量。
   - 通常会被标记为 `rw`（可读写）或 `r--`（只读）。
   - 在示例中，`rw` 标记的部分可能是数据段。
2. **文本段（Text Segment）**:
   - 文本段包含程序的机器代码（指令）。
   - 通常会被标记为 `r-x`（可执行，可读）。
   - 在示例中，`r-x` 标记的部分是文本段。
3. **栈（Stack）**:
   - 栈用于存储函数调用、本地变量等数据。
   - 通常不会被具体指明，通常表现为 `[ anon ]`，在示例中可能是一些没有具体标识的部分。
4. **堆（Heap）**:
   - 堆用于动态分配内存，比如通过 `malloc` 或 `new` 分配的内存。
   - 堆通常也表现为 `[ anon ]`，在示例中也可能是一些没有具体标识的部分。

要区分这些段，可以根据标记（`r-x`、`rw`、`r--`）和映射的文件路径来推断哪些部分是代码、数据、堆或栈。根据常见的标记和内存段的典型用途，可以初步确定不同部分的作用。然而，这仅是一种初步的推断方法，实际上确定内存段的确切用途需要更深入的分析和了解进程的内部结构。





这三个寄存器指针的全称如下：

1. **RBP**：`RBP` 的全称是 "Base Pointer Register"，它通常用作帧指针，指向当前函数的栈帧（stack frame）的基址。
2. **RSP**：`RSP` 的全称是 "Stack Pointer Register"，它指向当前栈顶的位置，用于管理函数调用时的栈操作。
3. **RIP**：`RIP` 的全称是 "Instruction Pointer Register"，它包含了当前正在执行指令的地址，即下一条要执行的指令的地址。

这些寄存器在 x86 架构的处理器中非常常见，对于函数调用、栈操作和指令执行起着关键作用。`RBP` 和 `RSP` 是用于管理函数调用和栈操作的寄存器，而 `RIP` 则用于跟踪当前执行的指令地址。





在使用 mmap 进行文件映射时，的确会涉及磁盘 I/O 操作。

当使用 mmap 将文件映射到内存时，操作系统会将文件的一部分或整个文件映射到进程的虚拟地址空间中，这样进程就可以直接访问文件内容，而不需要通过传统的 read 和 write 系统调用。这种直接内存映射的方式带来了一些好处，比如避免了频繁的系统调用，简化了文件访问的逻辑等。

然而，尽管 mmap 提供了一种高效的文件访问方式，但在实际使用过程中仍然会触发磁盘 I/O 操作。具体来说，当通过 mmap 修改内存中映射的文件内容时，操作系统会将这些修改缓存在内存中，并在适当的时机将这些修改写回到文件中。这个过程涉及到了内存到磁盘的数据传输，即磁盘 I/O 操作。

因此，虽然 mmap 提供了一种高效的文件访问方式，但在底层仍然会涉及磁盘 I/O 操作，以确保文件的持久性和一致性。



Linux内存管理API

https://deepinout.com/linux-kernel-api/linux-kernel-api-memory-management/linux-kernel-api-kmalloc.html



WSL源码版本：
https://github.com/microsoft/WSL2-Linux-Kernel/releases/tag/linux-msft-wsl-5.15.153.1



bison

dwarves



**Bison**：
Bison是一个自由的生成器，用于解析器的开发。它是GNU Project的一部分，用于生成LALR(1)或LR(1)解析器。Bison读取一个描述语法的文件，然后生成一个对应的解析器程序。在编译器和解释器的开发中，Bison经常与Lex（词法分析器生成器）一起使用，用于构建语法分析器。

Bison的主要作用是根据给定的文法规则生成解析器代码。它将输入的语法规则转换为解析器程序，这个解析器程序可以识别和解释符合该语法规则的输入。

**Dwarves**：
Dwarves是一个工具集合，用于处理和分析程序的调试信息（debugging information）。这些调试信息包含了关于程序中变量、函数、数据结构等信息的元数据。Dwarves提供了一组工具，用于提取、显示和处理这些调试信息，以帮助开发人员调试程序、分析内存映像等。

Dwarves工具集包括一些常用的命令行工具，如`dwarfdump`（用于显示调试信息的内容）、`dwarfls`（用于列出调试信息文件中的内容）等。这些工具有助于开发人员在调试和分析程序时更好地理解程序的结构和运行时信息。

总的来说，Bison用于生成解析器代码，而Dwarves用于处理和分析程序的调试信息，这两个工具在软件开发中扮演着不同但重要的角色。



在Linux中安装Bison和Dwarves这两个工具，您可以通过包管理器来进行安装。以下是在常见的Debian/Ubuntu和CentOS/RHEL系统上安装这两个工具的方法：

### 安装Bison

#### Debian/Ubuntu系统

bash

复制

```
sudo apt update
sudo apt install bison
```

#### CentOS/RHEL系统

bash

复制

```
sudo yum install bison
```

### 安装Dwarves

#### Debian/Ubuntu系统

Dwarves工具可能包含在`elfutils`包中，您可以通过安装`elfutils`来获取Dwarves工具。

bash

复制

```
sudo apt update
sudo apt install elfutils
```

#### CentOS/RHEL系统

bash

复制

```
sudo yum install elfutils
```

安装完成后，您就可以在系统上使用Bison和Dwarves这两个工具了。如果您需要更详细的信息或遇到任何问题，请随时告诉我。





WSL 头文件编译内核模块 头文件缺失问题

https://blog.csdn.net/helaisun/article/details/128426641

https://roderickchan.github.io/zh-cn/2022-11-28-wsl%E4%BD%BF%E7%94%A8%E9%97%AE%E9%A2%98%E8%AE%B0%E5%BD%95/

替换linux内核：
https://blog.csdn.net/m0_59339540/article/details/138974581



  OBJCOPY arch/x86/boot/setup.bin
  BUILD   arch/x86/boot/bzImage
Kernel: arch/x86/boot/bzImage is ready  (#4)

/home/linuxkernel/WSL2-Linux-Kernel-linux-msft-wsl-5.15.153.1/arch/x86/boot/bzImage  /mnt/e/UbuntuWSL

/mnt/e/OsTestCode/kernalHello/hello.c   /kernelstudy

/mnt/e/OsTestCode/kernalHello/Makefile  /kernelstudy



