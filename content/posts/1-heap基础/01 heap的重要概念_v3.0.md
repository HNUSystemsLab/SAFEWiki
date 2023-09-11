---
title: "heap基础 new "
date: 2023-02-05T15:36:08+08:00           

tags : [                                    
"heap基础1",
]
categories : [                              
"heap基础1",
]
keywords : [                                
"heap基础1",
]
---


## 1 重要概念-arena、chunks、bins

当开发人员使用各种基于栈的漏洞缓解措施时，攻击者通常使用堆相关漏洞（如use-after-free、double-free和heap-overflows等）来构建漏洞。与基于栈的漏洞相比，这些基于堆的漏洞更难理解，因为针对基于堆漏洞的攻击技术可能非常依赖于堆分配器内部实现的实际工作方式。不同平台的堆内存管理机制不相同，下面是几个常见平台的堆内存管理机制：

<div align=center><img src="https://gitlab.com/wanikiki/blog-pic-rep/-/raw/main/2023-heap/heap-manage.jpg"></div>

本文主要学习介绍在 Linux 的 glibc 使用的 ptmalloc2 实现原理，首先介绍了glibc堆实现的重要概念和相关数据结构。

### 1.1 heap_info

在c/c++中，系统分配内存是在堆上进行的，程序刚开始执行时，每个线程并没有heap区域，只有当其申请内存时，会使用一个结构`heap_info`来记录其申请的heap空间的信息。当申请的heap的资源被使用完后，必须再次申请内存。（此处的 heap 并非广义上的进程的虚拟内存空间中的堆，而是线程通过系统调用`mmap`从操作系统申请的一块内存空间。）

在glibc中，`heap_info`结构被用来表示堆空间的具体信息，例如指定堆的大小、提供堆对应的arena的地址，也称为 heap header。此外，一般申请的heap空间是不连续的，因此`heap_info`还记录了不同heap之间的链接结构。

`heap_info`结构的声明如下：
```C
typedef struct _heap_info
{
  mstate ar_ptr;            /* Arena for this heap. */
  struct _heap_info *prev;  /* Previous heap. */
  size_t size;              /* Current size in bytes. */
  size_t mprotect_size;     /* Size in bytes that has been mprotected
                             PROT_READ|PROT_WRITE.  */
  /* Make sure the following data is properly aligned, particularly
     that sizeof (heap_info) + 2 * SIZE_SZ is a multiple of
     MALLOC_ALIGNMENT. */
  char pad[-6 * SIZE_SZ & MALLOC_ALIGN_MASK];
}
```
具体说明：
+ `mstate ar_ptr`：指向arena的指针，一个`heap_info`只对应一个arena。
+ `struct _heap_info *ptr`：指向前一个`heap_info`结构的指针，分配的所有`heap_info`结构构成一个循环链表。
+ `size_t size`：当前堆的大小
+ `char pad[...]`：为了确保正确对齐而用于填充

该数据结构是专门为从 Memory Mapping Segment 处申请的内存准备的，即为非主线程准备的。而主线程可以通过`sbrk()`函数扩展`program break location`获得（直到触及 Memory Mapping Segment），但是其只有一个 heap，没有`heap_info`数据结构。


<br>

### 1.2 arena（分配区）
ptmalloc对进程内存的管理是通过arena来进行管理的。每个线程都有一个自己的 arena 用于堆内存的分配，这个区域是调用 malloc 的时候从操作系统获得的，一般情况下比实际 malloc 要大一些，当下次再次调用 malloc ，可以直接从 arena 中进行堆内存分配。
一个线程只有一个arnea，主线程或者第一个线程的arnea称为`main_arena`，即为主arena；子线程的arnea称为`thread_arena`，即为非主arena。主arena和非主arena的区别是主arena可以使用`sbrk`和`mmap`向os申请内存，而非主arena只能通过`mmap`向os申请内存。

> **thread_arena与main_arena**：
> 不同于`thread_arena`，主线程的`main_arena`的arena header（即`malloc_state`结构）并不在堆区中，而是一个全局变量，因此它属于`libc.so`的Data Segment区域。

#### 1.2.1 malloc_state
arena用数据结构`malloc_state`来表示，`malloc_state`记录了每个arena当前申请内存的具体状态，比如说是否有空闲chunk，有什么大小的空闲chunk等等，也被称为arena header。

`malloc_state`结构的声明如下：
```C

struct malloc_state {
 mutex_t mutex;                 /* Serialize access. */
 int flags;                       /* Flags (formerly in max_fast). */
 #if THREAD_STATS
 /* Statistics for locking. Only used if THREAD_STATS is defined. */
 long stat_lock_direct, stat_lock_loop, stat_lock_wait;
 #endif
 mfastbinptr fastbins[NFASTBINS];    /* Fastbins */
 mchunkptr top;
 mchunkptr last_remainder;
 mchunkptr bins[NBINS * 2];
 unsigned int binmap[BINMAPSIZE];   /* Bitmap of bins */
 struct malloc_state *next;           /* Linked list */
 INTERNAL_SIZE_T system_mem;
 INTERNAL_SIZE_T max_system_mem;
 };
```

具体说明：
+ **`mutex_t mutex`**：用于确保对 ptmalloc 实现采用的各种数据结构的同步访问，通常在调用`_int_XXX()`函数之前获得，例如，在调用`free()`时，实际上是在调用`public_free()`，它锁定互斥锁并调用`_int_free()`；
+ **`int flags`**：用于表示当前arena的各种特征，例如是否有fastbin chunks，内存是否非连续等等。
+ **`long stat_lock_direct`**：
  **`long stat_lock_loop`**：
  **`long stat_lock_wait`**：用于提供各种锁信息。
+ **`mfastbinptr fastbins[...]`**：fastbin 的数组，用作容纳分配和`free()`的chunk，该数组操作速度更快，这在很大程度上是由于对它们执行的操作较少。
+ **`mchunkptr top`**：代表特殊的chunk，top chunk；稍后做详解。
+ **`mchunkptr last_remainder`**：代表特殊的chunk，last remainder chunk；稍后做详解。
+ **`mchunkptr bins[...]`**：bin数组，类似于 fastbin 数组，用于存放内存空闲的chunk，常被用于较大的“正常” chunk ；bin的概念将在下面详细描述。
+ **`unsigned int binmap[...]`**：用作一级索引，以帮助加快确定给定bin是否绝对为空的过程。
+ **`struct malloc_state *next`**：指向下一个区域，循环单链表。
+ **`INTERNAL_SIZE_T system_mem`**：
  **`INTERNAL_SIZE_T max_system_mem`**：用于跟踪系统当前分配的内存量；`INTERNAL_SIZE_T`宏在大多数平台上定义为`size_t`。



#### 1.2.2 arena数量
需要注意的是，非主arena是通过`mmap`向os申请内存，一次申请64MB，一旦申请了，该arena就不会被释放，为了避免资源浪费，ptmalloc对arena是有个数限制的。对于不同系统，arena 数量的约束如下：
```C
    For 32 bit systems:
        Number of arena = 2 * number of cores.
    For 64 bit systems:
        Number of arena = 8 * number of cores.
```
即，arena对于32位系统，数量最多为核心数量2倍，64位则最多为核心数量8倍，可以用来保证多线程的堆空间分配的高效性。

#### 1.2.3 arena的分配原则
每一个进程只有一个主arena和若干个非主arena，**主arena和非主arena用环形链表连接起来**。每个线程一定对应一个arena，但是因为每个程序中arnea的数量是有限制的。当arena满了之后就不再创建，子线程需要与其他线程共享一个arena（不包含主arena），也就是说，`malloc`会通过系统调用`mmap`为该线程申请新的heap（这部分空间本来是位于内存映射区区域），新的heap会被添加到当前`thread_arena`中，**同一个arena下的堆以链表方式进行连接**。分配原则如下:
+ 当一个线程调用malloc申请内存时，该线程先查看线程私有变量中是否已经存在一个arena。
  - 如果存在，则对该arena加锁，加锁成功的话就用该arena进行内存分配；
  - 如果失败，则搜索环形链表找一个未加锁的arena。
  - 如果所有arena都已经加锁，那么`malloc`会开辟一个新的arena加入环形链表并加锁，用它来分配内存。
+ 释放操作同样需要获得锁才能进行。


#### 1.2.4 heap和arena

> **回顾malloc_state与heap_info**：
> + `heap_info`记录内存中一个堆段的总体信息，并提供了指向对应arena的指针；
> + `malloc_state`用来存储并管理各种堆块的数据信息。

在这里，我们通过内存分布图理清`malloc_state`与`heap_info`之间的组织关系。下图是只有一个 heap 的`main_arena`和`thread_arena`的内存分布图。
![](https://murphypei.github.io/images/posts/cplusplus/heap/arena-single-segment.png)

下图是一个`thread_arena`中含有多个heap的情况：
![](https://murphypei.github.io/images/posts/cplusplus/heap/arena-multi-segments.png)

从上图可以看出，`thread_arena`只含有一个`malloc_state`（即 arena header），却有两个`heap_info`（即 heap header）。由于两个 heap 是通过`mmap`从操作系统申请的内存，两者在内存布局上并不相邻而是分属于不同的内存区间，所以为了便于管理，glibc 的`malloc`将第二个`heap_info`结构体的`prev`成员指向了第一个`heap_info`结构体的起始位置（即 `ar_ptr`成员），而第一个`heap_info`结构体的`ar_ptr`成员指向了`malloc_state`，这样就构成了一个单链表，方便后续管理。

<br>

### 1.3 chunks
chunk 在内存中可以理解为是用来表示一块内存。为了便于管理和更加高效的利用内存，一个 heap 被分为多个chunk，每个 chunk 的大小不是固定，具体由用户的请求决定的，可以大概理解为，用户调用`malloc(size_t size)`传递的`size`参数就是 chunk 的大小(只是便于理解，这种表述并不准确)。用户调用`free()`函数释放的内存，也并不是立即归还给操作系统，它们也会被表示为一个 chunk。

#### 1.3.1 malloc_chunk
chunk的结构用`malloc_chunk`来表示，`malloc_chunk`结构的声明如下：
```C
struct malloc_chunk {  
  INTERNAL_SIZE_T      prev_size;    /* Size of previous chunk (if free).  */  
  INTERNAL_SIZE_T      size;         /* Size in bytes, including overhead. */  
  
  struct malloc_chunk* fd;           /* double links -- used only if free. */  
  struct malloc_chunk* bk;  
  
  /* Only used for large blocks: pointer to next larger size.  */  
  struct malloc_chunk* fd_nextsize;      /* double links -- used only if free. */  
  struct malloc_chunk* bk_nextsize; 
};  
```
具体说明：
+ **`INTERNAL_SIZE_T prev_size`**：代表前一个chunk的大小，具体细节见1.3.2。
+ **`INTERNAL_SIZE_T size:`**：当前区块的大小，用于遍历分配的区块。
+ **`struct malloc_chunk *fd`**：
  **`struct malloc_chunk *bk`**：当前的chunk空闲时，该类指针才存在。两者分别指向循环双链接空闲列表中的下/上一个块的指针，用于将对应的空闲chunk加入到空闲chunk块链表中统一管理。
+ **`struct malloc_chunk* fd_nextsize`**：
  **`struct malloc_chunk* bk_nextsize`**：当前的chunk空闲并存在于large bins中时，该类指针才存在，分别指向下一个比当前chunk大小 **大/小** 的第一个空闲chunk。同一大小的chunk可能有多块，在总体大小有序的情况下，要想找到下一个比自己大或小的chunk，需要遍历所有相同的chunk，因此该指针被用于加快遍历空闲chunk，并查找满足需要的空闲chunk。为避免浪费，在被分配的chunk中，该指针的空间被当作应用程序的使用空间。

#### 1.3.2 chunk类别
在 glibc 的`malloc`中将整个堆内存空间分成了连续的、大小不一的 chunk，即对于堆内存管理而言 chunk 就是最小操作单位，chunk可以根据其使用状态被分为2类：
+ **allocated chunk**
+ **free chunk**

从本质上来说，所有类型的 chunk 都是内存中一块连续的区域，只是通过该区域中特定位置的某些标识符加以区分。首先介绍allocated chunk和free chunk，前者表示已经分配给用户使用的chunk，后者表示未使用的 chunk。在ptmalloc中，为了尽可能的节省内存，使用中的chunk和未使用的chunk在具体结构上是不一样的。

##### 1）allocated chunk
![](https://murphypei.github.io/images/posts/cplusplus/heap/allocated-chunk.png)

+ **`prev_size`**: 如果前一个chunk是free chunk，则这个内容保存的是前一个chunk的大小。如果前一个chunk是allocated chunk，则这个区域保存的是前一个chunk的用户数据（一部分而已，主要是为了能够充分利用这块内存空间）。
+ **`size`**: 保存的是当前这个 chunk 的大小。总共是 32 位，并且最后的 3 位作为标志位：2
  - **`PREV_INUSE (P)`**: 表示前一个 chunk 是否为 allocated chunk，而当前是不是 chunk allocated 可以通过查询下一个 chunk 的这个标志位来得知）。
  - **`IS_MMAPPED (M)`**: 表示当前 chunk 是否是通过 mmap 系统调用产生的，子线程是 mmap，主线程则是通过 brk。
  - **`NON_MAIN_arena (N)`**: 表示当前 chunk 是否属于 main arena，也就是主线程的 arena。（主线程和子线程的堆区不一样，前文已经做了详细说明）。

##### 2）free chunk
与非空闲chunk相比，空闲chunk在用户区域多了2（4）个指针，分别为`fd`,`bk`（`fd_nextsize`,`bk_nextsize`）。
![](https://murphypei.github.io/images/posts/cplusplus/heap/free-chunk.png)

+ **`prev_size`**：为了防止碎片化，堆中不存在两个相邻的free chunk（如果存在，则被堆管理器合并了）。因此对于一个free chunk，这个`prev_size`区域中一定包含的上一个chunk的部分有效数据或者为了地址对齐所做的填充对齐。
+ **`size`**：同 allocated chunk ，表示当前 chunk 的大小，其标志位N，M，P 也同 allocated chunk 一样。

<br>

### 1.4 bins

**堆管理器回收内存的基本策略**
1.  如果chunk在metadata中设置了`M`位，则表示其是在堆外分配的，应该使用munmaped进行内存回收。
2.  如果chunk的上一个chunk是空闲的，则该chunk会向后合并为一个更大的空闲chunk。
3.  类似地，如果chunk的下一个chunk是空闲的，则该chunk会向前合并为一个更大的空闲chunk。
4.  如果chunk（或合并之后更大的chunk）靠近堆的顶部，那么整个chunk将被吸收到堆的尾部，而不会进入bins中。
5.  如果不满足上述的情况，chunk会被放进合适的bins中。
   
bins表示未分配的chunk链表。由上可知，用户调用`free`函数释放内存的时候，ptmalloc不会立即将其归还操作系统，而是可能将它们放入bins中。下次再调用`malloc`函数申请内存的时候，会从bins中取出一块返回。
根据chunk的大小，ptmalloc将bin分为fast bin, unsorted bin, small bin, large bin四类;从前面`malloc_state`结构定义，可将bin分为两类，一类是fast bin，另一类是bins，bins包括unsorted bin, small bin以及large bin。在glibc中，四类bin的个数各不相等，下文会进行具体介绍。

#### 1.4.1 fast bin
fast bin是在其他三种bins基础上的优化体现。程序在运行时会经常需要申请和释放一些较小的内存空间。当分配器合并了相邻的几个小的chunk之后，也许会马上收到另一个对小chunk的请求，这样分配器又需要从大的空闲内存中切分出一个小chunk，这样无疑是低效率的，因此，引入了fast bin。fast bin的本质是将最近释放的小chunk存储在bin中，而不将这些chunk与其邻居合并，以便之后收到对小chunk的请求时，能够立即进行分配。
在前面`malloc_state`中，可以看到fast bin是这样维护的：
```c
typedef struct malloc_chunk *mfastbinptr;
mfastbinptr fastbinsY[NFASTBINS]; // Array of pointers to chunks
```
1. fast bin的默认个数是10个，即fastbinsY的长度为10，`NFASTBINS=10`。
2. bin中的每一个元素维护一个单链表，添加、删除chunk遵循LIFO（后进先出），即操作都是在链表的头部进行的。
3. 每个bin链表中chunk大小相同。10个fast bin链表中chunk的大小以8字节递增：即第一个fast bin链表中的chunk大小为16字节，第二个fast bin链表中的chunk大小为24字节，依此类推，最后一个fasta bin链表中的chunk大小是88字节。（大小包括元数据）
4. 不会对空闲的chunk进行合并操作。fast bin设计的初衷就是小chunk的快速分配和释放，因此系统将属于fast bin的chunk的`p`位总是设置为1，这样即使当fast bin中有某个chunk同一个空闲的chunk相邻的时候，系统也不会进行自动合并操作，而是保留两者。
5. fast bin的缺点是fast bin中的chunk没有被真正释放（没有进行合并操作），最终可能会导致进程的内存随着时间的推移而碎片化严重。为了解决这个问题，堆管理器定期对堆进行合并操作，即将chunk与相邻的空闲chunk进行合并，并将生成的空闲chunk放置到unsorted bin中以供`malloc`使用。上述情况一般发生在发出的`malloc`请求大于fast bin可以提供的服务时。

#### 1.4.2 unsorted bin
某些情况下，释放操作通常会聚集在一起，而且释放之后通常会立即分配大小相似的块，例如更新列表中条目的程序可能会在为其更新项分配之前释放的空间之和。在这些情况下，在将释放的较大的chunk放在合适的bin中之前合并这些chunk会避免一些开销，并且能够快速返回最近释放的空间，可以加快分配过程。unsorted bin由此出现。
1. unsorted bin是一个循环双链表，且只有一个，其使用bins数组的第一个，是bins的一个缓冲区，用于加速内存分配和释放请求。当用户释放的内存大于`max_fast`或者fast bin合并后的chunk都会首先进入unsorted bin中。
2. 在unsorted bin链表中，chunk的大小没有限制，也就是说任何大小的chunk都可以放进unsorted bin中。收到用户的malloc请求时，malloc如果在fast bins中没有找到合适的chunk，就会现在unsorted bin中查找合适的空闲chunk，如果有，malloc可以立即返回它，如果没有，malloc会将unsorted bin中的chunk放入small bin或large bin中，然后再到bins上查找合适的空闲chunk。
3. 与fast bin所不同的是，unsorted bin遵循FIFO（先进先出）。

#### 1.4.3 small bin
small bin共62个，使用bins数组的第2个到第63个元素。每个元素维护一个循环双链表，每个small bin链表中存储大小相同的chunk，每个链表中chunk的大小以8字节递增，依次为16,24,……,254。
1. 对于32位系统，大小小于512字节的chunk被称为small chunk；对于64位系统，大小小于1024字节的chunk被称为small chunk。而存储small chunk的bin链表就被称为small bin。
2. small bin遵循FIFO（先进先出）。 内存释放操作后将新释放的chunk添加到链表的头部，分配操作就从链表的尾部获取chunk。

#### 1.4.4 large bin
small bin对于小chunk的分配是好的，但是无法对每个大小的chunk都设置一个专门的bin。对于大于512字节的chunk（64位上位大于1024字节的chunk），堆管理器使用large bin。
1. large bin位于small bins后面，个数为63。
   
2. 每个large bin都以与small bin相同的方式进行操作，但不同的是，large bin不是存储固定大小的chunk，而是存储一个大小范围内的chunk。每个large bin中的chunk大小范围都被设计为不与small bin中的chunk大小或其他large bin中的chunk大小范围重叠。换句话说，给定一个chunk的大小，这个大小对应的正好是一个small bin或large bin。
   
3. 在这63个large bins中，第一组的32个large bin链表依次以63字节步长为间隔，即第一个large bin链表中chunk的大小范围为1024~1087字节，第二个large bin链表中chunk的大小范围为1088~1151字节。第二组的16个large bin链表依次以512字节步长为间隔；第三组的8个large bin链表依次以4096字节步长为间隔；第四组的4个large bin链表以32768字节为步长间隔；第五组的2个large bin链表以262144字节为步长间隔；最后一组larg bin链表中的chunk大小无限制。

4. 在进行malloc操作时，首先确定用户请求的大小属于哪一个large bin链表，然后判断该链表中最大的chunk的大小是否大于用户请求的size。
   
>+ ① 如果大于，就从尾开始遍历该链表，找到第一个大小相等或接近的chunk，分配给用户。如果该chunk大于用户请求的size的话，就将该chunk拆分为两个chunk：前者返回给用户，且大小等同于用户请求的size；后者作为一个新的chunk添加到unsorted bin中。
>
>+ ② 如果小于，那么就依次查看后续的large bin链表中是否有满足需求的chunk，不过需要注意的是鉴于bin的个数较多，而且large bin的使用频率较低，按照①介绍的遍历每个bin中的chunk的方法可能会发生多次内存页中断操作，进而严重影响检索速度。因而，glibc malloc设计了Binmap结构体来帮助提供bin-by-bin检索的速度。Binmap记录了各个bin是否为空，荣光bitmap可以避免检索一些空的bin。如果通过Binmap找到了下一个非空的large bin，就按照①的方法分配chunk；如果找不到合适的chunk，就使用top chunk来分配合适的内存。

#### 1.4.5 tcache
tcache全称是per-thread cache，是glibc 2.26版本后引入的新机制。前文主要针对glibc2.26之前的版本进行介绍，所以并没有把tcache加入bin的分类中。tcache在默认情况下是启用的。目的是提高堆管理性能，缓解不同线程之间在堆分配时的资源竞争。

##### 1）tcache的设计背景
给定计算机系统上的每个进程都有一个或多个线程同时运行，拥有多个线程允许进程执行多个并发操作。例如，高容量web服务器可能有多个同时传入的请求，web服务器会在自己的线程上为每个传入的请求提供服务，而不是让每个请求排队等待服务。

给定进程中的每个线程共享相同的地址空间，也就是说，每个线程都可以在内存中看到相同的代码和数据。每个线程都有自己的寄存器和堆栈来存储临时本地变量，但是像全局变量和堆这样的资源在所有线程之间共享。协调对堆等全局资源的访问是一个复杂的问题，如果处理不当，可能会导致“race condition”的问题。一种常用的方法是通过使用锁强制这些同时访问全局资源的请求进入顺序队列。这种方法确保了全局资源一次只能由一个线程使用，但也有代价：等待资源的线程必须停止等待，造成时间的浪费。

glibc中为了防止线程之间共享堆而出现问题，在每一个线程调用malloc时都会创建一个新的arena，但是如[1.2.2](#122-arena数量)和[1.2.3](#123-arena的分配原则)所述，arena的数量是有限制的，如果超过了阈值，多出的线程就需要共享堆。为了解决这些线程的同步问题，堆管理器增加了锁的机制，这就导致一部分的线程在运行时可能会被阻塞，需要等待其他的线程使用完堆之后才能使用。这会占用程序执行的大量时间，为了对此进行优化，tcache被引入。

##### 2）tcache bin
**相关数据结构**：
```c
/* We overlay this structure on the user-data portion of a chunk when
   the chunk is stored in the per-thread cache.  */
typedef struct tcache_entry
{
  struct tcache_entry *next;
} tcache_entry;

/* There is one of these for each thread, which contains the
   per-thread cache (hence "tcache_perthread_struct").  Keeping
   overall size low is mildly important.  Note that COUNTS and ENTRIES
   are redundant (we could have just counted the linked list each
   time), this is for performance reasons.  */
typedef struct tcache_perthread_struct
{
  char counts[TCACHE_MAX_BINS];
  tcache_entry *entries[TCACHE_MAX_BINS];
} tcache_perthread_struct;
```
`tcache_entry`用于链接空闲的chunk，存放chunk的user data部分，并指向下一个`tcache_entry`，因此`next`指向的就直接是user data的开始，也对应chunk的`fd`指针。
`tcache_perthread_struct`是整个tcache机制的管理结构，每个线程都会维护一个`tcache_perthread_struct`，其中包含`TCACHE_MAX_BINS`个`tcache_entry`链表，链表中的每个chunk大小相同，所以也称其为tcahce bin。其特性如下：
   1. 每个tcache bin是一个单链表结构，最多包含7个chunk。
   2. tcache bin中的chunk的`in use`位不会置零，也就是说不会对它进行合并。
   3. 遵循“后进先出”的规则。
   4. 默认情况下，每个线程最多有64个tcache bins。对于32系统而言，bin中的chunk大小的范围是12-516字节，对于64位系统而言，大小范围是24-1032字节。

综上可以看到tcache bin的特性和fast bin是非常类似的（需要注意的是，这里的`next`指向chunk的user data，而fast bin的`fd`指向chunk的开头）。free时会优先将chunk放入tcache bin中，分配时也先从tcache bin中搜索。

**基本工作方式**：
1. 第一次malloc时，会先分配一块内存用来存放`tcache_perthread_struct`。
2. 对chunk进行free时，如果chunk的大小不满足放入small bin的条件：
   > 在tcache被引入前，该chunk会被放到fast bin或者unsorted bin中。
   > 在tcache被引入后，该chunk先放到对应的tcache bin中，直到tcache bin被填满。
3. malloc请求时，如果请求的chunk的大小在tcache范围内，则先从tcache bin中取chunk，直到tcache bin为空。这样当线程请求一个chunk时，如果tcache bin中有可用的chunk，那么就可以不等待堆锁进行分配。如果tcache bin为空，则：
   > 从其他类型的bin中寻找，如果fast bin/ small bin/ unsorted bin中有大小符合要求的chunk，会先把fast bin/small bin/unsorted bin中的chunk放到tcache bin中，直到填满。之后再从tcache bin中取出chunk。


<br>

### 1.5 特殊chunks
上节内容讲述了几种bin以及各种bin内存的分配和释放特点，但是，仅仅上面几种bin还不能够满足。为此，glibc提出了另外几种特殊的chunk供分配和释放，分别为top chunk，mmaped chunk和last remainder chunk。

#### 1.5.1 top chunk
当一个chunk处于一个arena的最顶部（即最高内存地址处）的时候，就称之为top chunk。该chunk并不属于任何bin，而是在当前的heap所有的free chunk（无论那种bin）都无法满足用户请求的内存大小的时候，将此chunk当做一个应急消防员，分配给用户使用。如果top chunk的大小比用户请求的大小要大的话，就将该top chunk分作两部分：用户请求的chunk和剩余的部分（成为新的top chunk）。否则，就需要扩展heap或分配新的heap了，在`main arena`中通过`sbrk`扩展 heap，而在`thread arena`中通过`mmap`分配新的heap。注意，至此我们已经多次强调，主线程和子线程的堆管理方式的差异。

#### 1.5.2 mmaped chunk
当分配的内存非常大（大于分配阀值，默认128K）的时候，需要被`mmap`映射，则会放到mmaped chunk上，当释放mmaped chunk上的内存的时候，该chunk会直接交还给操作系统。（chunk中的M标志位置1）

#### 1.5.3 last remainder chunk
Last remainder chunk是从最后一次拆分中获得的块。当用户请求一个small chunk，且该请求无法被small bin满足，那么就转而交由unsorted bin处理。同时，假设当前unsorted bin中只有一个chunk的话，也就是last remainder chunk，那么就将该chunk分成两部分：前者分配给用户，剩下的部分放到unsorted bin中，并成为新的last remainder chunk。这样就保证了连续`malloc`产生的各个small chunk在内存分布中是相邻的，提高了内存分配的局部性。

<br>

### 参考资料：
+ [Understanding the heap by breaking it](https://www.blackhat.com/presentations/bh-usa-07/Ferguson/Whitepaper/bh-usa-07-ferguson-WP.pdf)
+ [Understanding glibc malloc](https://sploitfun.wordpress.com/2015/02/10/understanding-glibc-malloc/comment-page-1/)
+ [UNDERSTANDING THE GLIBC HEAP IMPLEMENTATION](https://azeria-labs.com/heap-exploitation-part-2-glibc-heap-free-bins/)
+ [heap-exploitation-bins and chunks](https://heap-exploitation.dhavalkapil.com/diving_into_glibc_heap/bins_chunks)
+ [tcache](https://ctf-wiki.org/pwn/linux/user-mode/heap/ptmalloc2/implementation/tcache/)

