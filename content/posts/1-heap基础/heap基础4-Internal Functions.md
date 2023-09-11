---
title: "heap基础 4/6: Internal Functions内部函数 "
date: 2023-02-05T15:36:08+08:00

tags : [                                    
"heap基础",
]
categories : [                              
"heap基础",
]
keywords : [                                
"heap基础",
]
---

## Internal Function内部函数
这是内部使用的一些常用函数的列表。要注意的是，某些函数实际上是使用 #define 指令定义的。因此，对调用参数的更改实际上在调用操作之后会保留。同时，假定没有设置MALLOC_DEBUG。

### arena_get (ar_ptr, size)
首先获取一块区域然后锁住相对应的互斥锁，ar_ptr指针指向相应的区域，size参数只是表明要占用多少内存。

### sysmalloc [TODO]
sysmalloc需要系统用malloc函数分配更多的内存。在使用的时候假设av->top的空间并不够请求nb字节，因此需要延长或者替换av->top.

### void alloc_perturb (char *p, size_t n)
如果 perturb_byte（使用 M_PERTURB 的 malloc 的可调参数）不为零（默认情况下为 0），则修改 p 指向的 n 个字节的内容，使其与perturb_byte ^ 0xff相等。

### void free_perturb (char *p, size_t n)

如果 perturb_byte（使用 M_PERTURB 的 malloc 的可调参数）不为零（默认情况下为 0），则修改 p 指向的 n 个字节的内容，使其与 perturb_byte相等。

### void malloc_init_state (mstate av)

初始化malloc_state结构。这仅从内部调用malloc_consolidate函数（malloc_consolidate需要在相同环境被调用）它从不在 malloc _ merge之外直接调用，因为一些编译器试图在所有调用点对他进行内联操作，反而造成了负优化的后果。（虽然将它内联在malloc_consolidate中很棒就是了）
1.对于不是fastbin的bins来说，需要为每个bin创建一个空循环链表
2.为av 设置FASTCHUNKS_BIT的flag参数
3.把av->top初始化为第一个未经过排序的chunk


### unlink(AV, P, BK, FD)
定义一个宏指令从而从bin中删除一个chunk
1.检查chunk的大小是否和下一个chunk里之前设置的大小相等，否则的话将显示"corrupted size vs. prev_size"错误
2.检查P->fd->bk是否和P相等和P->bk->fd是否和P相等，否则的话将显示"corrupted double-linked list"错误
3.调整列表中的相邻chunk的前后指针以便删除
  1.设置P->fd->bk = P->bk
  2.设置P->bk->fd = P->fd

### void malloc_consolidate(mstate av)

free函数的特殊版本。
1.在av未被初始化的条件下检查global_max_fast是否为0，如果为 0，则以 av 作为参数调用 malloc_init_state 并返回。
2. 如果global_max_fast不为0，清除 av 的FASTCHUNKS_BIT flag。
3. 从第一个索引迭代 fastbin 数组到最后一个索引：
   1）锁住当前的fastbin chunk 并且如果它不是空的话便继续。
   2）如果未使用上一个内存块，调用上一个块上的unlink函数
   3）如果下一个内存块不是在最顶部：
     1））如果下一个内存块没有被使用，则为下一个内存块调用unlink函数
     2））如果其中任意一个内存块是空的话，则将上下内存块合并，然后将合并之后的内存块放到为排序的bin的头部
   4）如果下一个内存块在顶部，把所有在顶部空的内存块合并为一个内存块
注意：使用PREV_IN_USE来检查该内存块是否被使用，这样的话其他的fastbin chunk就不会被标志为空闲。
