---
title: "heap基础 0/6-new: malloc_chunk "
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

---
title: "heap基础 6/6: Security Checks "
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

## Security Checks

这里总结了 glibc 实现中引入的安全检查，用于检测和防止与堆相关的攻击。


函数 | 安全检查 | 错误
---------|----------|---------
 unlink | 区块大小是否等于下一个区块（在内存中）中设置的先前大小 | corrupted size vs. prev_size
 unlink | 验证`P->fd->bk == P`和`P->bk->fd == P*`是否成立 | 双链表损坏
 _int_malloc | 从 fastbin 中删除第一个区块（为 malloc 请求提供服务）时，检查区块的大小是否在快速区块大小范围内 | malloc(): memory corruption(fast)
 _int_malloc | 从小垃圾箱中删除最后一个块（`victim`）时（为 malloc 请求提供服务），检查`victim->bk->fd` 和`victim`是否相等 | malloc(): 小箱双链表已损坏
 _int_malloc | 在未排序的 bin 中迭代时，检查当前块的大小是否在最小 （`2*SIZE_SZ`） 和最大 （`av->system_mem`） 范围内 | malloc(): memory corruption
 _int_malloc | 将最后一个剩余块插入未排序的 bin 时（拆分大块后），检查`unsorted_chunks(av)->fd->bk == unsorted_chunks(av)` 是否成立 | malloc(): unsorted chunks 损坏
 _int_malloc | 将最后一个剩余块插入未排序的 bin 时（拆分快速或小块后），检查`unsorted_chunks(av)->fd->bk == unsorted_chunks(av)` 是否成立| malloc(): unsorted chunks 2 损坏
 _int_free | 检查内存中 p** 是否在 p + chunksize（p） 之前（以避免包装） | free(): 无效指针
 _int_free | 检查块的大小是否为 `MINSIZE` 或 `MALLOC_ALIGNMENT` 的倍数 | free(): 无效size
 _int_free | 对于大小在 fastbin 范围内的区块，检查下一个区块的大小是否介于最小和最大大小 （`av->system_mem`） 之间 | free(): 下一个size无效(fast)
 _int_free | 将快速块插入快速箱（在 `HEAD` 处）时，检查已经在 `HEAD` 上的块是否不同 | double free 或者是 corruption (fasttop)
 _int_free | 将快速块插入 fastbin 时（在 `HEAD` 处），检查 `HEAD` 处的块大小是否与要插入的块相同 | 无效 fastbin 入口 (free)
 _int_free | 如果块不在 fastbin 的大小范围内，也不是映射的块，请检查它是否与顶部块不同 | double free 或者是 corruption (top)
 _int_free | 检查下一个块（按内存）是否在 Arena 的边界内 | double free 或者是 corruption (out)
 _int_free | 检查下一个块（按内存）的先前使用位是否被标记 | double free 或者是 corruption (!prev)
 _int_free | 检查下一个区块的大小是否在最小和最大大小 （`av->system_mem`） 内 | free(): 下一个size无效(normal)
 _int_free | 将合并的块插入未排序的箱中时，检查`unsorted_chunks->(av)->fd->bk == unsorted_chunks(av)`是否成立 | free(): unsorted chunks 损坏



 *：“P”是指未链接的块
 **：“p”是指被释放的块