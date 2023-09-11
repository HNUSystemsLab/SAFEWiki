---
title: "heap基础 3/6: Bins 和 Chunks "
date: 2023-02-05T15:36:08+08:00
draft: false

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

## Bins和Chunks

Bin 是由free chunk（未分配）组成的一个列表（双向或单向链表）。bins 根据其包含的块的大小被分为以下4类：
1. Fast bin
2. Unsorted bin
3. Small bin
4. Large bin


Fast bins 使用以下方法声明： 
 
```C
typedef struct malloc_chunk *mfastbinptr;

mfastbinptr fastbinsY[]; // Array of pointers to chunks
```
 
Unsorted、small和large bins使用以下方式声明：
```C
typedef struct malloc_chunk *mchunkptr; 

mchunkptr bins[]; // Array of pointers to chunks
```
    
最初，在初始化过程中，small和large bin为空。每个bin由 bin 数组中的两个值表示，第一个值是指向“HEAD”的指针，第二个值是指向“TAIL”的指针。对于fast bin（单向链表），第二个值会设为 `NULL`。 

### Fast bins
共有10个fast bin，每一个bin都是一个单向链表，对链表的添加和删除操作都是从链表头进行，遵循LIFO算法。

每个bin中的chunk大小都是相同的。10 个bin中包含的chunk的大小有：16、24、32、40、48、56、64、72、80 和 88字节，此处提到的大小也包括元数据。在指针大小为 4 个字节的平台上，将会提供不超过4个字节的空间来存储chunk。该chunk的`prev_size`和`size`字段将保存已分配区块的元数据。下一个连续区块`prev_size`将保存用户数据。

两个连续的空闲fast chunk不会合并在一起。

### Unsorted bin
只有1个unsorted bin。在大型chunk或者小型chunk释放后，会进入这个bin当中。此 bin 的主要用途是充当缓存层（某种）以加快分配和取消分配请求的速度。

### Small bins
共有62个small bin，small bins的块就叫做small chunks。就内存的分配和释放速度而言，small bins的速度要比large bins快，但比fast bins慢。每一个small bin都是双向链表，采用FIFO（先入先出）算法，插入操作发生在“头部”，而删除发生在“尾部”（以FIFO方式）。

与fast bins一样，每个small bin都有相同大小的chunk。62 个箱的大小为：16、24、...、504 字节，以8字节一次递进。

在释放时，空闲的small chunk可能会合并在一起，然后最终进入unsorted bin。

### Large bins
共有63个large bin，每一个large bin都是双向链表。特定large bin中chunk的大小各不相同，其中chunk根据其大小降序排序（即最大的块在“HEAD”和最小的块在“TAIL”）。插入和删除操作可以发生在列表中的任何位置。  

前 32 个bin中包含的chunk相距64个字节：  

第一个bin：512-568字节  
第二个bin：576-632字节  
...  


总得来说就是：

```C
No. of Bins Spacing between bins

64 bins of size       8  [ Small bins ]
32 bins of size      64  [ Large bins ]
16 bins of size     512  [ Large bins ]
8 bins of size     4096  [ ...        ]
4 bins of size    32768
2 bins of size   262144
1 bin  of size  what's left
```

和small chunk一样，在释放时large chunk可能会合并在一起，然后进入unsorted bin。

> **注意**，有两种特殊类型的块不属于任何bin，分别是顶部块（top chunk）和最后剩余块（last remainder chunk）

### Top chunk

Top chunk是位于Arena顶部的块。在为“malloc”请求提供服务时，它被用作最后的手段。可以使用 sbrk 系统调用指令来增大Top chunk的大小。Top chunk会被设置有 PREV_INUSE 标志。

### Last remainder chunk
Last remainder chunk是从最后一次拆分中获得的块。有时，当较小的块不可用时，较大的块会分成两个部分，一部分返回给用户，另一部分会成为last remainder chunk。