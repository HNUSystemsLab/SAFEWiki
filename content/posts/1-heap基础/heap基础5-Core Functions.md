---
title: "heap基础 5/6: Core Functions核心函数 "
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

# <center>核心函数<center>

说明：

## void *_init_malloc(mstate av,size_t bytes)
1.更新bytes以处理对齐等。
\
2.检查`av`是否为NULL。
\
3.在没有可用分区的情况下（`av`为NULL），调用`sysmalloc`来使用mmap获取chunk。如果成功，将调用`alloc_perturb`，然后返回指针。
\
4.
* A. 如果要申请的chunk大小在fast bin范围内，则：
\
① 获取fast bin数组的索引，从而根据请求大小访问合适的bin。
\
② 删除该bin中的第一个chunk并让`victim`指向它。
\
③ 如果`victim`为NULL，则转到情况B.
\
④ 如果`victim`不为NULL，检查chunk的大小以确保它属于该特定的bin。否则会抛出错误（`"malloc():memory corruption(fast)"`）
\
⑤ 调用`alloc_perturb`然后返回指针。

* B. 如果大小落在small bin范围内：
\
① 获取small bin数组的索引，以根据请求大小访问合适的bin。
\
② 如果这个bin中没有chunk（通过比较指针`bin`和`bin->bk`来检查），则继续执行情况C。
\
③ `victim`等于`bin->bk`(`bin`中的最后一个chunk)。如果它为NULL（在此期间发生初始化），则调用`malloc_consolidate`并跳过检查不同bin的完整步骤。
\
④ 否则，当`victim`不为NULL时，检查`victim->bk->fd`和`victim`是否相等。如果它们不相等，则抛出错误`"malloc(): smallbin double linked list corrupted"`.
\
⑤ 为`victim`的下一个chunk（在内存中，而不是在双向链表中）设置`PREV_INSUE`位。
\
⑥ 从bin列表中删除这个chunk。
\
⑦ 根据`av`为这个chunk设置适当的分区位。
\
⑧ 调用`alloc_perturb`然后返回指针。

* C. 如果大小不在smallbin范围内：
\
① 获取largebin数组的索引以根据请求大小访问合适的bin。
\
② 看`av`有没有fastchunks。这是通过检查`av->flags`中的`FASTCHUNKS_BIT`来完成的。如果是这样，调用`av`上的`malloc_consolidate`.

* 如果还没有返回任何指针，则意味着以下一种或多种情况：
\
① 大小落在fast bin范围但没有可用的fastchunk。
\
② 大小落在small bin范围但没有可用的smallchunk（在初始化期间调用malloc_consolidate）。
\
③ 大小落在large bin范围内。

* 接下来，检查unsorted chunks并遍历chunks将其放入bins中。这是唯一将chunks放入bins中的地方。从'TAIL'中迭代unsorted bin。
\
① `victim`指向正在处理的当前chunk。
\
② 检查`victim`的chunk大小是否在最小(`2*SIZE SZ`)和最大(`av->system_mem`)范围内。否则抛出错误（`"malloc(): memory corruption"`）
\
③ 如果请求的chunk大小在small bin范围内且`victim`是最后一个剩余chunk和unsorted bin中的唯一一个chunk，以及此chunks大小 >= 请求的chunk，则将chunk分为两个chunks：
a. 第一个chunk匹配请求的大小并返回。
b. 剩下的chunk成为新的最后一个剩余chunk，并被插回到unsorted bin中。
Ⅰ 为两个chunks设置合适的`chunk_size`和`chunk_prev_size`字段.
Ⅱ 调用`alloc_perturb`后返回第一个chunk。
\
④ 如果上述条件为假，则控制到达此处。从unsorted bin中移除`victim`。如果`victim`的大小与请求的大小完全匹配，则在调用`alloc_perturb`后返回此chunk。
\
⑤ 如果`victim`的大小落在smallbin范围内，则将chunk添加到相应small bin的HEAD中。
\
⑥ 否则，在保持排序顺序的同时，插入到合适的large bin：
Ⅰ 首先检查最后一个chunk（最小）。如果`victim`的大小小于最后一个chunk，将其插入到最后一个chunk。
Ⅱ 否则，循环查找大小 >= `victim`的大小的一个chunk。如果大小完全相同，则始终插入第二个位置。
\
⑦ 重复整个步骤最多10000次，或者直到unsorted bin中的所有chunks都用完为止。

* 在检查完unsorted chunks之后，检查请求的大小是否落在small bin范围内，如果不在，则检查large bin。
\
① 获取large bin数组的索引，以根据请求大小访问适当的bin。
\
② 如果最大的chunk（bin中的第一个chunk）的大小大于请求的大小：
Ⅰ 从"TAIL"中迭代以找到最小大小的>=请求大小的chunk(`victim`)。
Ⅱ 调用`unlink`从bin中删除`victim`chunk。
Ⅲ 计算`victim`chunk的`remainder_size`（即`victim`chunk大小-请求的大小）。
Ⅳ 如果此`remainder_size` >= MINSIZE（包括headers在内的最小chunk大小），则将chunk分为两个chunks。否则，将返回整个`victim`chunk。将剩余的chunk插入unsorted bin（在"TAIL"端）。检查在unsorted bin中是否存在`unsorted_chunks(av)->fd->bk == unsorted_chunks(av)`。否则抛出错误(`"malloc(): corrupted unsorted chunks"`)
Ⅴ 调用`alloc_perturb`后返回`victim`chunk。

* 到目前为止，我们已经检查了unsorted bins以及相应的fast bins，small bins或large bins。注意：使用所请求chunk的确切大小检查了单个bin（fast bins或small bins）。重复以下步骤，直到所有bins都用完为止：
\
① 递增bin数组的索引，以检查下一个bin。
\
② 使用`av-binmap`的map跳过空的bins。
\
③ `victim`指向当前bin的'TAIL'。
\
④ 使用binmap可以确保如果跳过了一个bin（在②中），它肯定是空的。然而，binmap不能确保跳过了所有空的bins。检查`victim`是否为空。如果为空，再次跳过bin并重复上述过程，直到我们到达非空bin。
\
⑤ 将chunk（`victim`指向非空bin的最后一个chunk）分成两个chunks。将剩余的chunk插入到unsorted bin（'TAIL'端）。在unsorted bin中检查`unsorted_chunks(av)->fd->bk == unsorted_chunks(av)`是否成立。如果不成立会抛出错误(`"malloc(): corrupted unsorted chunks 2"`)
\
⑥ 调用`alloc_perturb`后返回`victim`chunk。

* 如果仍未找到空的bin，将使用top chunk来为请求提供服务：
\
① `victim`指向`av->top`。
\
② 如果top chunk的大小 >= '请求的大小'+MINSIZE，将它分为两个chunks。在这种情况下，剩余chunk将成为新的top chunk，另一个chunk在调用`alloc_perturb`后返回给用户。
\
③ 看`av`有没有fastchunks。这是通过检查`av->flags`中的`FASTCHUNKS_BIT`来完成的。如果有，调用`av`上的`malloc_consolidate`。返回到**步骤1.6**（检查unsorted bin）
\
④ 如果`av`没有fastchunk，则调用`sysmalloc`，并返回调用`alloc_perturb`后得到的指针。

## _libc_malloc(size_t bytes)
1.调用`arena_get`获取mastate指针。
\
2.使用分区指针和大小调用`_init_malloc`.
\
3.解锁分区。
\
4.在返回指向chunk的指针之前，以下条件之一应该为真：
Ⅰ 返回指针为NULL
Ⅱ chunk是MMAPPED
Ⅲ chunk的分区与①中找到的相同。

## _init_free(mstate av,mchunkptr p,int have_lock)
* 检查在内存中p是否在p + chunksize(p) 之前（以避免wrapping）。否则会抛出错误(`"free():invalid pointer"`).
* 检查chunk的大小是否至少为MINSIZE或MALLOC_ALIGNMENT的倍数。否则会抛出错误（`"free(): invalid size"`）
* 如果chunk的大小在fast bin列表中：
Ⅰ 检查下一个chunk的大小是否在最小和最大大小（`av->system_mem`）之间，否则抛出错误（`"free(): invalid next size (fast)"`）.
Ⅱ 调用`free_perturb`处理chunk.
Ⅲ 为`av`设置FASTCHUNKS_BIT.
Ⅳ 根据chunk的大小获取fast bin数组的索引。
Ⅴ 检查bin的顶部是否不是我们要添加的chunk。否则，抛出错误（`"double free or corruption (fasttop)"`）
Ⅵ 检查顶部的fast bin chunk的大小是否与我们添加的chunk相同。否则，抛出错误（`"invalid fastbin entry (free)"`）
Ⅶ 将chunk插入fast bin列表的顶部并返回。

* 如果chunk没有mmapped：
Ⅰ 检查该chunk是否是top chunk。如果是，则抛出错误(`"double free or corruption (top)"`)。
Ⅱ 检查下一个chunk（按内存）是否在分区的边界内。如果不是，则抛出错误(`"double free or corruption (out)"`)。
Ⅲ 检查下一个chunk（按内存）的先前使用位是否被标记。如果不是，则抛出错误(`"double free or corruption (!prev)"`).
Ⅳ 检查下一个chunk的大小在最小和最大大小之间（`av->system_mem`）.如果不在，则抛出错误(``).
Ⅴ 调用`free_perturb`处理chunk。
Ⅵ 如果前一个chunk（按内存）未被使用，就调用`unlink`处理前一个chunk.
Ⅶ 如果下一个chunk（按内存）不是top chunk：
如果下一个chunk未被使用，则调用`unlink`处理下一个chunk。将chunk和上一个chunk、下一个chunk（按内存）（如果有被释放的chunk也一起）合并，并将其插入到unsorted bin的头部。在插入之前，检查是否存在`unsorted_chunks(av)->fd->bk == unsorted_chunks(av)`。如果否，则抛出错误(`"free(): corrupted unsorted chunks"`)
Ⅷ 如果下一个chunk（按内存）是top chunk，则将这些chunks适当地合并到单个top chunk中。

* 如果chunk是mmapped，则调用`munmap_chunk`.

## _libc_free(void *mem)
1. 如果`mem`为NULL则直接返回。
2. 如果相应的chunk是mmapped，在brk/mmap阈值需要动态调整时调用`munmap_chunk`.
3. 获取相应chunk的分区指针。
4. 调用`_init_free`.


