---
title: "模糊测试 fuzzing "
date: 2023-02-05T15:36:08+08:00           

tags : [                                    
"模糊测试 fuzzing",
]
categories : [                              
"模糊测试 fuzzing",
]
keywords : [                                
"模糊测试 fuzzing",
]
---

## <center>认识malloc_chunck</center>


此结构用于表示特定的内存块。在程序执行中，我们称`malloc`申请的内存为chunk，用`malloc_chunk`结构来表示，当程序申请的chunk被free后，会被加入到对应的空闲chunk管理队列。无论chunk的大小、状态如何，他们都是使用同一数据结构`malloc_chunk`，只不过是各个字段对于不同类型的chunk具有不同的含义。


<!-- TOC -->

- [认识malloc\_chunck](#认识malloc_chunck)
  - [1 malloc chunk结构定义](#1-malloc-chunk结构定义)
    - [1.1 字段说明](#11-字段说明)
      - [1.1.1 `mchunk_prev_size`](#111-mchunk_prev_size)
      - [1.1.2 `mchunk_size`](#112-mchunk_size)
      - [1.1.3 `fd`,`bk`](#113-fdbk)
      - [1.1.4 `fd_nextsize`,`bk_nextsize`](#114-fd_nextsizebk_nextsize)
  - [2 chunk类型](#2-chunk类型)
    - [2.1 allocated chunk](#21-allocated-chunk)
    - [2.2 free chunk](#22-free-chunk)
  - [3 参考资料](#3-参考资料)

<!-- /TOC -->

---

## 1 malloc chunk结构定义
```python
struct malloc_chunk {
  INTERNAL_SIZE_T      mchunk_prev_size;  /* Size of previous chunk, if it is free. */
  INTERNAL_SIZE_T      mchunk_size;       /* Size in bytes, including overhead. */
  struct malloc_chunk* fd;                /* double links -- used only if this chunk is free. */
  struct malloc_chunk* bk;
  /* Only used for large blocks: pointer to next larger size.  */
  struct malloc_chunk* fd_nextsize; /* double links -- used only if this chunk is free. */
  struct malloc_chunk* bk_nextsize;
};
typedef struct malloc_chunk* mchunkptr;
```
### 1.1 字段说明
#### 1.1.1 `mchunk_prev_size`
>若上一个chunk（即物理相邻的前一地址chunk，物理相邻的意思是两个指针的地址差为上一个chunk大小）是空闲的，则此字段记录上一个chunk的大小，否则此字段被用来存储上一个chunk的用户数据。
#### 1.1.2 `mchunk_size`
>**大小规范**：此字段为该chunk的大小，大小必须是2*`SIZE_SZ`的整数倍。如果申请的内存不是2*`SIZE_SZ`的整数倍，会被转换为满足大小的最小的2*`SIZE_SZ`的倍数。32位系统中，`SIZE_SZ`是4；64位系统中，`SIZE_SZ`是8。

>**低三个比特位**：该字段的低三个比特位对chunk的大小没有影响，由高到低分别是 `NON_MAIN_ARENA`,`IS_MAPPED`,`PREV_INUSE`。

|名称|简写|含义|
|:---:|:---:|:---|
|`NON_MAIN_ARENA`|**`A`**|记录当前chunk是否是`main_arena`管理的堆块，1表示是，0表示不是|
|`IS_MAPPED`|**`M`**|记录当前的chunk是否由mmap分配的|
|`PREV_INUSE`|**`P`**|记录上一个chunk是否被分配，一般来说，堆中第一个被分配的内存块的`mchunk_size`字段的P位都会被设置为1，以便于防止访问前面的非法内存。当一个chunk的`mchunk_size`字段的P位为0时，我们能通过`mchunk_prev_size`获取上一个chunk的大小及地址，便于进行空闲堆块的合并。|

#### 1.1.3 `fd`,`bk`
当chunk是被分配状态时，从`fd`字段开始是用户的数据。
当chunk空闲时，空闲chunk会被添加到对应的空闲chunk管理链表中，这两个字段含义如下。
|字段名称|含义|
|:---:|:---|
|`fd`|指向下一个（非物理相邻）空闲的chunk|
|`bk`|指向上一个（非物理相邻）空闲的chunk|
通过`fd`和`bk`可以将空闲的chunk加入到空闲chunk链表进行统一管理。
#### 1.1.4 `fd_nextsize`,`bk_nextsize`
只有当chunk是空闲时才使用，且用于较大的chunk.
|字段名称|含义|
|:---:|:---|
|`fd_nextsize`|指向上一个与当前chunk大小不同的第一个空闲块，不包含bin的头指针|
|`bk_nextsize`|指向下一个与当前chunk大小不同的第一个空闲块，不包含bin的头指针|

一般空闲的较大的chunk在`fd`的遍历顺序中，按照由大到小的顺序排列。这样做可以避免在寻找合适chunk时挨个遍历。

## 2 chunk类型
由上可知，根据不同的chunk类型，`malloc_chunk`会有部分内容选择性“表示”。堆栈中存在的chunk类型如下。
### 2.1 allocated chunk
分配给用户的chunk结构如下图。我们称前两个字段为`chunk header`，后面的部分称为`user data`。每次malloc申请得到的内存指针，其实指向`user data`的起始处。当一个chunk处于被分配状态时，它的下一个chunk的`mchunk_prev_size`域无效，所以下一个chunk的该部分也可以被当前chunk使用，即chunk中的**空间复用**。

![](C:/Users\w2393\Desktop\研究生\实验室\安全资料\heap\image\1-1.png)
上图中的箭头分别表示：
|箭头名称|含义|
|:---:|:---|
|`chunk`|该allocated chunk的起始地址|
|`mem`|该allocated chunk中用户可用区域的起始地址，也是返回给用户的指针。|
|`nextchunk`|下一个chunk的起始地址|

### 2.2 free chunk
free chunk就是用户free后释放的chunk，被释放的chunk会被记录在链表中（可能是循环双向链表，也可能是单向链表），具体结构如下图。
![](C:/Users\w2393\Desktop\研究生\实验室\安全资料\heap\image\1-2.png)

一般情况下，物理相邻的两个空闲chunk会被合并为一个chunk。堆管理器会通过`mchunk_prev_size`字段及`mchunk_size`字段合并两个物理相邻的空闲的chunk。
## 3 参考资料
[1] https://heap-exploitation.dhavalkapil.com/heap_memory
[2] https://www.jianshu.com/p/484926468136
[3] https://blog.csdn.net/qq_41661777/article/details/119703517
