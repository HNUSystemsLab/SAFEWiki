## 2 堆的操作

ptmalloc2 通过实现了 `malloc()`、`free()` 以及一些其它的函数，来提供动态内存管理的支持。分配器处在用户程序和内核之间，它响应用户的分配请求，向操作系统申请内存，然后将其返回给用户程序，为了保持高效的分配，分配器一般都会预先分配一块大于用户请求的内存，并通过某种算法管理这块内存。来满足用户的内存分配要求，用户释放掉的内存也并不是立即就返回给操作系统；相反，分配器会管理这些被释放掉的空闲空间，以应对用户以后的内存分配要求。也就是说，分配器不但要管理已分配的内存块，还需要管理空闲的内存块，当响应用户分配要求时，分配器会首先在空闲空间中寻找一块合适的内存给用户，在空闲空间中找不到的情况下才分配一块新的内存，为实现一个高效的分配器，需要考虑很多的因素。

本文将结合 glibc-2.24 版本的源代码，介绍了 ptmalloc2 响应用户内存分配和内存释放要求的具体步骤，以及一些堆内部经常使用的函数。

### 2.1 内存分配
ptmalloc2 主要使用的内存分配函数为 `malloc()`，`malloc()` 函数的定义位于 glibc 的 malloc.c 文件中。实际上在 glibc 内部 `malloc()` 只是 `__libc_malloc()` 函数的别名，而 `__libc_malloc()` 函数的主要工作由 `_int_malloc()` 完成，即 `__libc_malloc()` 是对 `_int_malloc()` 的封装。下面我们将通过介绍这两个函数的逻辑来分析 malloc 的实现。

#### 2.1.1 __libc_malloc()
`__libc_malloc()` 的源代码如下：
```c
void *__libc_malloc (size_t bytes)
{
  mstate ar_ptr;
  void *victim;

  void *(*hook) (size_t, const void *)
    = atomic_forced_read (__malloc_hook);
  if (__builtin_expect (hook != NULL, 0))
    return (*hook)(bytes, RETURN_ADDRESS (0));

  arena_get (ar_ptr, bytes);

  victim = _int_malloc (ar_ptr, bytes);
  /* Retry with another arena only if we were able to find a usable arena
     before.  */
  if (!victim && ar_ptr != NULL)
    {
      LIBC_PROBE (memory_malloc_retry, 1, bytes);
      ar_ptr = arena_get_retry (ar_ptr, bytes);
      victim = _int_malloc (ar_ptr, bytes);
    }

  if (ar_ptr != NULL)
    (void) mutex_unlock (&ar_ptr->mutex);

  assert (!victim || chunk_is_mmapped (mem2chunk (victim)) ||
          ar_ptr == arena_for_chunk (mem2chunk (victim)));
  return victim;
}
libc_hidden_def (__libc_malloc)
```
该函数的主要逻辑如下：
1. `__libc_malloc` 函数的参数`bytes`是用户申请分配的空间的大小，并且是用户提出申请的原始大小，比如说 malloc(12)，那么这里 bytes 就是 12，而非经过 `request2size` 宏计算得到的 chunk 对齐的大小；

2. 一开始，`__libc_malloc` 会检查是否有内存分配钩子函数 `__malloc_hook`，有的话则调用它；

3. 之后调用 `arena_get()` 来获取一个分配区，并上锁；

4. 然后调用 `_init_malloc()` 函数申请对应大小的内存；

5. 断言执行后可能会出现以下三种意外情况，这些情况将导致函数报错并退出；
   +  没有申请到内存
   +  mmap 得到的内存
   +  申请到的内存不在其所分配的 arena 中
  
6. 调用 `mutex_unlock` 对分配区解锁并返回申请到的空间 `victim`。

**__malloc_hook 全局钩子**
ptmalloc2 定义了一个全局钩子 `__malloc_hook`，这个钩子会由 `malloc_hook_ini` 函数来赋值。如果我们需要自定义堆分配函数，就可以把这个钩子函数重新设置为我们自定义的函数，即 ptmalloc 给我们提供了一个机会去使用自己定义的堆分配函数来完成对堆空间的申请；如果我们没有自定义堆分配函数，而是选择默认的ptmalloc 来帮我们完成申请，那么在用户第一次调用 malloc 函数时会首先转入 `malloc_hook_ini` 函数里面，这个函数的定义在 hook.c 文件中，源代码如下：
   ```c
   static void *
   malloc_hook_ini (size_t sz, const void *caller)
   {
      __malloc_hook = NULL;
      ptmalloc_init ();
      return __libc_malloc (sz);
   }
   ```
   可以看到在 `malloc_hook_ini` 里会把 `__malloc_hook` 设置为空，然后调用 `ptmalloc_init` 函数，这个函数将完成对 ptmalloc 的初始化，最后再一次调用 `__libc_malloc` 函数。
   综上可知，在我们第一次调用 malloc 申请堆空间的时候，首先会进入 `malloc_hook_ini` 里面进行对 ptmalloc 的初始化工作，然后再次进入 `__libc_malloc` 时会发现钩子已经被置空，从而继续执行下面的代码。

> 第一次调用 malloc 函数时函数的执行路径：
>   `malloc -> __libc_malloc -> __malloc_hook -> ptmalloc_init -> __libc_malloc -> _int_malloc`
> 之后再次调用时的执行路径：
>   `malloc -> __libc_malloc -> _int_malloc`



#### 2.1.2 _init_malloc()
通过前文我们已知 `_int_malloc()` 函数才是内存分配的核心，结合 [2.4.1 \_int\_malloc()源代码](#241-_int_malloc源代码) 进行分析，该函数的整体逻辑如下：
1. 检查是否有可用的 arena，如果没有就调用 `sysmalloc` 申请，有的话就继续下一步。

2. 计算 chunk 的大小，前面说到，`__libc_malloc` 的参数 `bytes` 是用户提交的最原始的空间大小，但是 ptmalloc 分配时是以 chunk 为单位分配的，由原始的空间大小计算得到 chunk 对齐后的大小由 `checked_request2size` 宏完成，`nb` 即为计算得到的 chunk 大小。
    ```c
      checked_request2size (bytes, nb);
    ```
3. 如果 `nb` 落在 fastbins 范围，首先根据大小计算 index 获取对应的 fastbin 链表，然后获取链表上第一个 chunk：
>+ 如果该 chunk 不为空，则更改链表头指向第二个 chunk，并通过 `chunk2mem` 宏返回 chunk 中用户数据区的地址即可。注意这里无需设置当前空闲 chunk 的使用标志位，因为 fastbins 中的空闲 chunk 为了避免被合并，它的使用标志位是 1，即表示在“使用中”。
>+ 如果该 chunk 为空，说明当前 fastbin 中没有恰好匹配 `nb` 大小的空闲 chunk。注意在 fastbin 中分配 chunk 时是精确匹配而非最佳匹配。

4. 如果 `nb` 落在 small bins 范围内，则前往 small bin 中查找。根据大小计算 index 获取对应的 small bin 链表，然后判断该链表是否为空：
>+ 如果为空，则说明获取失败，没有合适的空闲 chunk，直接跳到后面的步骤。
>+ 如果不为空，则判断获取到的链表的尾部的 chunk 是否为空：
>   + 如果为空，说明当前分配区还没有初始化，small bin 也没有被初始化，此时会转至 `malloc_consolidate` 函数进行初始化。（如果 small bin 已经初始化但没有找到合适的空闲 chunk，是不会调用该函数来清空 fastbins 的）
>   + 如果不为空，则获取该 chunk 并返回。这里需要设置当前返回 chunk 的使用标志位以及非主分配区标志位。

5. 如果 `nb` 不属于 small bins，说明申请的空间为 largebins，这时候不会立即检查 largebins，而是会调用 `malloc_consolidate` 函数将 fastbins 里面的空闲 chunk 合并整理到 unsorted bin 中。


6. 如果经过上述步骤都没有找到合适的空闲 chunk，此时将开始检查并整理 unsorted bin，这个过程是一边检查一边整理的，即如果有合适的就返回退出，如果没有就把 unsorted bin 中的每一个空闲 chunk 整理到 small bins 或 large bins 中。
>+ 按照 FIFO 的方式逐个将 unsorted bin 中的 chunk 取出来判断，判断 chunk 本身信息是否合法：
>     + size 在合法范围内
>     + 下一个 chunk 的 size 是否合法
>     + 下一个 chunk 的`prev_size`是不是与当前 size 相等
>     + `fd` 指针是不是正确
>     + 下一个 chunk 的 `prev_inuse` 位是否正确
>
>+ 如果 `nb` 落在 small bins 范围内，而 unsorted bin 中只有 last_remainder chunk，并且它的 size 足够大，就将这个唯一的空闲 chunk 分割后返回，剩余的空间作为新的 last_remainder chunk 并插入到 unsorted bin 链表的头部。
>
>+ 如果可以精确匹配则返回。unsorted bin 本来就是作为 small bins 和 large bins 的缓冲区存在，它里面存放着许多 free 时插入的空闲 chunks 或整理 fastbins 之后插入的空闲空间，很有可能这些空间里就有一个刚好能够精确匹配用户需要申请的空间大小。如果找到这样的空闲 chunk，直接返回即可。
>
>+ 如果当前空闲 chunk 不能精确匹配并且属于 small bins 大小范围，那么就将其插入到 small bins 中相应的 small bins 链表的头部。
>
>+ 如果当前空闲 chunk 落在 large bins 大小范围，那么就将其插入到 large bins 链表中。
>   + 这种插入更加复杂。因为 large bins 每一条链表上的 chunk 大小并不是相等的，而是处于一个范围内即可，而且每一个 large bin 并不是一个组双向循环链表，而是两组：`fd` 和 `bk` 构成的一组，`fd_nextsize` 和 `bk_nextsize` 构成的另一组。因此在插入时不仅需要合适的插入位置，还需要对链表做更多的工作。
>   + 如果 large bin 为空，则直接插入到链表头即可。
>   + 查找当前 large bin 的最后一个 chunk，因为在 large bin 链表中 chunk 按大小从大到小排列，因此最后一个 chunk 也是该链表中最小的一个 chunk。如果待插入的 chunk 小于该最小的空闲 chunk，则将其插入到链表尾部。
>   + 插入到链表中间。首先通过一个 while 循环找到合适的插入位置，找到位置之后进行判断，判断链中是否存在大小相同的一组chunks。
>      + 如果存在一组大小相同的 chunks，则插入到该组 chunks 的第二位，这样就无需修改第二组双向循环链表。
>+ 中断整理 unsorted bin 有两种情况。
>   + 一种是已经遍历整理完 unsorted bin 中所有空闲 chunk，在 while 判断处即可自然终止。
>   + 一种情况是，当整理 unsorted bin 中的空闲的 chunks 总数达到10000个时，就会 break 中断循环，避免因为处理 unsorted bin 而耗费太多时间。

7. 到该步骤，unsorted bin 已经变空，继续从 small bins 或者 large bins 分配。两种 bins 有一点不同的是，此时small bins 已经确定无法精确匹配了（对 unsorted bin 处理之前已经扫描了一轮 small bins），只能使用最佳匹配，而  large bins 还存在精确匹配的可能：
>+ 首先判断 `nb` 落在 large bins 范围内，如果属于则检查 `nb` 对应的那一个 large bin 链表上是否有满足要求的空闲 chunk。注意仅仅在这个 large bin 上还存在着精确匹配的可能。
>   + 首先判断当前 large bin 链表是否非空，并且最大的一个空闲 chunk 是否比 `nb` 大，如果都满足，则通过一个 while 循环找到满足分配要求的那一组相同大小的空闲 chunks：
>      + 如果这一组 chunks 的数量超过两个且大小刚好精确匹配，则返回这一组中的第二个空闲 chunk（返回第一个需要重新调整第二组双向循环链表），否则就直接返回找到的第一个 chunk。此处使用了 `unlink` 将选择的chunk脱链。
>      + 如果找到的 chunk 不满足精确匹配的要求，就会进行分割并产生一个剩余的空间，如果这个空间不足以成为一个新的 chunk，即比最小的 chunk 的大小还要小，那么就直接把整个 chunk 返回给用户，否则将分割之后的剩余空间作为一个新 chunk 插入到 unsorted bin 中。

8. 如果 `nb` 属于 small bins 范围，或者不属于 large bins 但经过以上步骤仍然没有找到合适的 chunk，此时就只能从 small bin 或者 large bin 中进行最佳匹配了。
>+ 最佳匹配的意思：从 `nb` 对应的那一个 small bin 或者 large bin 链表开始往上搜索离它最近并且非空的一个 small bin 或者 large bin。从中取出最小的一个空闲 chunk 返回，这个空闲 chunk 必定是不会精确匹配并且会产生剩余空间的，但它已经是能找到的离 `nb` 最接近的一个空闲 chunk 了。
>
>+ ptmalloc 使用 binmap 来加快查找的速度。binmap 是 bins 的位图，标记每一个 bin 是否为空，通过前面的介绍我们知道，bins 数组长度为128，而标记每一个 bin 是否非空只需要用 1 bit 即可，因此对于所有 bins，ptmalloc 使用四个 `unsigned int`，一共32个字节128位即可标记所有 bins 的非空状态了。
>+ 另外需要注意的是，如果分割之后剩余的 chunk 大小落在 small bins 范围内，那么除了把这个剩余 chunk 插入到 unsorted bin 中之外，它还同时成为当前分配区的 last_remainder。(这也是仅有的更改 last_remainder 的两处地方之一，一处是在整理 unsorted bin 切割 last_remainder 时剩余的空间将继承成为新的 last_remainder，另一处即是这里。)

9. 如果以上步骤仍然没有找到合适的空闲 chunk 可以满足分配要求，此时 ptmalloc 将从 top chunk 中分割空间来进行分配。
>+ 首先获取到 top chunk 的大小，然后和 `nb` 做比较，如果 top chunk 在满足分配后还能够形成一个 chunk，即剩余空间大小大于最小的 chunk 的大小，则开始进行分割，并把剩余的 chunk 作为新的 top chunk。

10. 如果至此还没有找到满足要求的 chunk，则说明当前 ptmalloc 缓存的空闲 chunks 都不能满足分配要求，只能调用 `sysmalloc` 直接向系统申请内存了。

#### 2.1.3 总结
响应用户内存分配请求的流程如下：
![](https://gitlab.com/Lelouny/heap-image/-/raw/main/malloc2.png)


### 2.2 内存释放
用户通过 `malloc()` 或类似的如 `realloc()` 等函数申请的堆空间不会被系统自动回收，必须由用户手动通过 `free()` 函数回收。`free()` 函数的定义位于 glibc 的 malloc.c 文件中，实际上在 glibc 内部 `free()` 函数只是 `__libc_free()` 函数的别名，而 `__libc_free()` 函数的工作又主要由 `_int_free()` 完成。因此，分析 `free()` 函数，即是分析 `__libc_free()` 以及 `_int_free()` 这两个函数。
在一开始，`free()` 函数接受一个指向分配区域的指针作为参数，释放该指针所指向的 chunk，具体的释放方法则看该 chunk 所处的位置和该 chunk 的大小。

#### 2.2.1 __libc_free()
`strong alias` 表示给函数一个强符号的别名，`free()` 为 `__libc_free()` 的别名：
> `strong_alias (__libc_free, __free) strong_alias (__libc_free, free)`

`__libc_free()` 的源码如下：
```C
void __libc_free (void *mem)
{
  mstate ar_ptr;
  mchunkptr p;                          /* chunk corresponding to mem */
  /*检查是否有钩子函数 __free_hook */
  void (*hook) (void *, const void *)
    = atomic_forced_read (__free_hook);
  if (__builtin_expect (hook != NULL, 0))
    {
      (*hook)(mem, RETURN_ADDRESS (0));
      return;
    }
  /*释放一个空指针可能会产生错误，因此直接返回*/
  if (mem == 0)                              /* free(0) has no effect */
    return;
  /*将mem指针转换为chunk指针*/
  p = mem2chunk (mem);
  /*内存是mmap分配的，在此释放掉*/
  if (chunk_is_mmapped (p))                       /* release mmapped memory. */
    {
      /* See if the dynamic brk/mmap threshold needs adjusting.
	 Dumped fake mmapped chunks do not affect the threshold.  */
      if (!mp_.no_dyn_threshold
          && chunksize_nomask (p) > mp_.mmap_threshold
          && chunksize_nomask (p) <= DEFAULT_MMAP_THRESHOLD_MAX
	  && !DUMPED_MAIN_ARENA_CHUNK (p))
        {
          mp_.mmap_threshold = chunksize (p);
          mp_.trim_threshold = 2 * mp_.mmap_threshold;
          LIBC_PROBE (memory_mallopt_free_dyn_thresholds, 2,
                      mp_.mmap_threshold, mp_.trim_threshold);
        }
      munmap_chunk (p);
      return;
    }

  /*根据chunk获得arena的指针*/
  ar_ptr = arena_for_chunk (p);
  /*调用_int_free进行释放*/
  _int_free(ar_ptr, p, 0);
}
```
`__libc_free(void *mem)` 的参数 `void *mem` 实际上就是 `free(p)` 中的 `p` ，该函数主要做了以下几件事：
+ 首先检查释放的指针 `p` 是否是空指针，如果是，那么 `__libc_free()` 不会有任何动作，直接返回；
+ 检查是否有内存分配钩子函数，有的话调用，这一步操作与 `malloc` 类似；
+ 如果 `p` 是通过 `mmap()` 配得到的，则由 `munmap_chunk()` 负责释放，调用 `munmap_chunk()` 进行内存释放；
+ 根据 chunk 获得其所在分配区的指针；
+ 非 `mmap()` 分配的内存，由 `_int_free()` 负责释放，调用 `_int_free` 进行内存释放。

#### 2.2.2 _init_free()
在 `__libc_free()` 函数中，如果释放的堆空间不是通过 `mmap()` 分配的，那么剩下的工作就会交给 `_int_free()` 完成。结合 [2.4.2 \_int\_free()源代码](#242-_int_free源代码) 进行分析，`_int_free (mstate av, mchunkptr p, int have_lock)` 中的 `p` 是需要释放的 `chunk`，该函数的整体逻辑如下：

**1）通过 `chunksize()` 宏得到 `p` 的大小，存放在 `size` 中，然后进行 chunk 检查：**
   + 判断对齐；
   + 大小需比最小的 chunk 大，且需要是 `MSLLOC_ALIGNMENT` 的整数倍，还需小于 `-size` ；
   + 检查 `p` 是否正在使用中。

**2）满足条件的`p`被插入到fastbin中**
  判断 `p` 的尺寸，如果小于或等于 `get_max_fast ()` 并且不和 top chunk 相邻，则说明 `p` 应当被回收插入到 fastbin 中：
  ```C
  if ((unsigned long)(size) <= (unsigned long)(get_max_fast ())

  #if TRIM_FASTBINS
        /*
      If TRIM_FASTBINS set, don't place chunks
      bordering top into fastbins
        */
        && (chunk_at_offset(p, size) != av->top)
  #endif
        )
  ```
  fastbin 是一个单向链表，它的工作方式被定义为插入和分配都是在链表头，即 LIFO。所以主要的工作其实就是 `P->FD = *FB; *FB = P;`，具体的插入操作：
  ```C
  /* Atomically link P to its fastbin: P->FD = *FB; *FB = P;  */
      mchunkptr old = *fb, old2;
      unsigned int old_idx = ~0u;
      do
        {
      /* Check that the top of the bin is not the record we are going to add
        (i.e., double free).  */
      if (__builtin_expect (old == p, 0))
        {
          errstr = "double free or corruption (fasttop)";
          goto errout;
        }
      /* Check that size of fastbin chunk at the top is the same as
        size of the chunk that we are adding.  We can dereference OLD
        only if we have the lock, otherwise it might have already been
        deallocated.  See use of OLD_IDX below for the actual check.  */
      if (have_lock && old != NULL)
        old_idx = fastbin_index(chunksize(old));
      p->fd = old2 = old;
        }
      while ((old = catomic_compare_and_exchange_val_rel (fb, p, old2)) != old2);
  ```

**3）如果 p 是通过 mmap() 分配得到，由`munmap_chunk()`负责释放。**

和 `__libc_free()` 中一样，如果是通过 `mmap()` 分配的，那么剩下的释放工作都将交由 `munmap_chunk()` 负责。但是相较于 `__libc_free()` ，这里少了对 mmap 的分配阈值以及系统的收缩阈值的修改操作。
```C
else if (!chunk_is_mmapped(p))
{
    ...
}
/*
  If the chunk was allocated via mmap, release via munmap().
*/
else {
  munmap_chunk (p);
}
```
> 注意，其实之前 `__libc_free()` 中判断过，只有当 `p` 不是通过 `mmap()` 分配的情况下才会进入 `_int_free()`，再次进行判断是因为 `_int_free()` 在 `sysmalloc()` 、`__libc_realloc()` 等函数中也被调用了，这时传递给 `_int_free()` 的 `p` 不一定能通过 `chunk_is_mmapped(p)` 的检查。

**4）如果 p 不是通过 mmap() 分配得到，有两种和空闲块合并的方式。**
如果不是通过 `mmap()` 分配的，那么将进入 if 语句块，将 `p` 合并后插入到 unsorted bin 或者直接合并到 top chunk。因为频繁地进行释放操作，系统可能会产生大量的堆内存碎片。因而为了避免产生大量碎片，系统在释放堆空间的时候会尝试进行合并操作，即如果相邻的 chunk 空闲，则合并成一个大的空闲 chunk，包括向前合并以及向后合并。
+ 向后合并，是合并相邻的低地址 chunk，通过 `prev_inuse(p)` 检查后方 chunk 是否在使用中，如果没有则进行合并，
  - 相应代码：
    ```C
      /* consolidate backward */
      if (!prev_inuse(p)) {
        prevsize = p->prev_size;
        size += prevsize;
        p = chunk_at_offset(p, -((long) prevsize));
        unlink(av, p, bck, fwd);
      }

      ```
+ 向前合并，是合并相邻的高地址的chunk，
  - 如果向前相邻的是 top chunk，那么直接合并到 top chunk，不再理会 unsorted bin；
  - 如果向前相邻的不是 top chunk，那么尝试合并后，插入到 unsorted bin 中。
  - 相应代码：
    ```C
      if (nextchunk != av->top) {
            /* get and clear inuse bit */
            nextinuse = inuse_bit_at_offset(nextchunk, nextsize);

            /* consolidate forward */
            if (!nextinuse) {
          unlink(av, nextchunk, bck, fwd);
          size += nextsize;
            } else
          clear_inuse_bit_at_offset(nextchunk, 0);

            /*
          Place the chunk in unsorted chunk list. Chunks are
          not placed into regular bins until after they have
          been given one chance to be used in malloc.
            */

            bck = unsorted_chunks(av);
            fwd = bck->fd;
            if (__glibc_unlikely (fwd->bk != bck))
          {
            errstr = "free(): corrupted unsorted chunks";
            goto errout;
          }
            p->fd = fwd;
            p->bk = bck;
            if (!in_smallbin_range(size))
          {
            p->fd_nextsize = NULL;
            p->bk_nextsize = NULL;
          }
            bck->fd = p;
            fwd->bk = p;

            set_head(p, size | PREV_INUSE);
            set_foot(p, size);

            check_free_chunk(av, p);
          }

          /*
            If the chunk borders the current high end of memory,
            consolidate into top
          */

          else {
            size += nextsize;
            set_head(p, size | PREV_INUSE);
            av->top = p;
            check_chunk(av, p);
          }


      ```

**5）释放空间足够大，触发 fastbin 的整理**
接下来还将判断，`p` 的大小（包含合并后的空间）是否大于或等于 `FASTBIN_CONSOLIDATION_THRESHOLD`，如果大于或等于，则调用 `malloc_consolidate()` 函数，这个函数会将 fastbin 中的每一个 chunk 合并后插入整理到 unsorted bin 中。
```C
if ((unsigned long)(size) >= FASTBIN_CONSOLIDATION_THRESHOLD) {
      if (have_fastchunks(av))
    malloc_consolidate(av);
    ...
}
```

#### 2.2.3 总结
`free()` 的整体操作流程图如下：
![](https://gitlab.com/Lelouny/heap-image/-/raw/main/free2.jpg)
<br>

### 2.3 其他内部函数
这是内部使用的一些常用函数的列表。要注意的是，某些函数实际上是使用 `#define` 指令定义的，因此，对调用参数的更改实际上在调用操作之后会保留。

#### 2.3.1 malloc_consolidate()
`malloc_consolidate()` 函数是定义在 malloc.c 中的一个函数，用于将 fastbin 中的空闲 chunk 合并整理到 unsorted bin 中以及进行初始化堆的工作，在 `malloc()` 以及 `free()` 中均有可能调用 `malloc_consolidate()` 函数。在该函数源代码的注释里说到，`malloc_consolidate()` 是 `free()` 的一个小的变体，专门用于处理 fastbin 中的空闲 chunk，同时还负责堆管理的初始化工作。

从 [2.4.3 malloc\_consolidate()源代码](#243-malloc_consolidate源代码) 的角度分析，该函数的整体逻辑如下：
  1. 若 `get_max_fast()` 返回 0，则进行堆的初始化工作，然后进入第 7 步；
  2. 从 fastbin 中获取一个空闲 chunk；
  3. 尝试向后合并；
  4. 若向前相邻 top chunk，则直接合并到 top chunk，然后进入第 6 步；
  5. 否则尝试向前合并后，插入到 unsorted bin 中；
  6. 获取下一个空闲 chunk，回到第 2 步，直到所有 fastbin 清空后进入第 7 步；
  7. 退出函数。


#### 2.3.2 unlink()
`unlink()` 是一个宏，用于将某一个空闲 chunk 从其所处的 bin 中脱链。在 `malloc_consolidate()` 函数中 `unlink()`会将 fastbin 中的空闲 chunk 整理到 unsorted bin，在 `malloc()` 函数中用于将 unsorted bin 中的空闲 chunk 整理到 smallbin 或者 largebin，以及在 `malloc()` 中获得堆空间时，均有可能调用 `unlink()` 宏。
该函数源代码如下：
```C
/* Take a chunk off a bin list */
#define unlink(AV, P, BK, FD) {                                            
    FD = P->fd;                                   
    BK = P->bk;                                   
    if (__builtin_expect (FD->bk != P || BK->fd != P, 0))             
      malloc_printerr (check_action, "corrupted double-linked list", P, AV);  
    else {                                    
        FD->bk = BK;                                  
        BK->fd = FD;                                  
        if (!in_smallbin_range (P->size)                      
            && __builtin_expect (P->fd_nextsize != NULL, 0)) {            
        if (__builtin_expect (P->fd_nextsize->bk_nextsize != P, 0)        
        || __builtin_expect (P->bk_nextsize->fd_nextsize != P, 0))    
          malloc_printerr (check_action,                      
                   "corrupted double-linked list (not small)",    
                   P, AV);                        
            if (FD->fd_nextsize == NULL) {                    
                if (P->fd_nextsize == P)                      
                  FD->fd_nextsize = FD->bk_nextsize = FD;             
                else {                                
                    FD->fd_nextsize = P->fd_nextsize;                 
                    FD->bk_nextsize = P->bk_nextsize;                 
                    P->fd_nextsize->bk_nextsize = FD;                 
                    P->bk_nextsize->fd_nextsize = FD;                 
                  }                               
              } else {                                
                P->fd_nextsize->bk_nextsize = P->bk_nextsize;             
                P->bk_nextsize->fd_nextsize = P->fd_nextsize;             
              }                                   
          }                                   
      }                                       
}
```
从源代码的角度分析，该函数的整体逻辑如下：
   1. 获取 `FD` 以及 `BK`；
   2. 检查 `FD->bk != P || BK->fd != P`；
   3. 如果 `P` 不是 smallbin 且 `P->fd_nextsize != null` 成立，进入第 4 步，否则进入第 5 步；
   4. 根据具体情况适当设置 `fd_nextsize` 和 `bk_nextsize`；
   5. 结束。

#### 2.3.3 malloc_init_state()
`malloc_init_state()`负责初始化`malloc_state`结构，源码如下：
```C
void malloc_init_state (mstate av){
  int     i;
  mbinptr bin;

  /* Establish circular links for normal bins */
  for (i = 1; i < NBINS; ++i) {
    bin = bin_at(av,i);
    bin->fd = bin->bk = bin;
  }

  #if MORECORE_CONTIGUOUS
    if (av != &main_arena)
  #endif
    set_noncontiguous(av);

    set_max_fast(av, DEFAULT_MXFAST);

    av->top = initial_top(av);
}

#define bin_at(m,i) ((mbinptr)((char*)&((m)->bins[(i)<<1]) - (SIZE_SZ<<1)))
```

从源代码的角度分析，该函数的整体逻辑如下：
   1. 对于不是 fastbin 的 bins 来说，需要为每个 bin 创建一个空循环链表；
   2. 为 `av` 设置 `FASTCHUNKS_BIT` 的 flag 参数；
   3. 把 `av->top` 初始化为第一个未经过排序的 chunk。

#### 2.3.4 arena_get()
`arena_get` 不仅负责arena的获取，还负责arena的加锁操作，因为ptmalloc是支持多线程的，但是一个arena在同一时间只能被一个线程操作，所以需要加锁。如果当前没有arena链上没有可用的arena，那么它还会负责创建一个新的arena并返回，bytes参数就是在创建新的arena时用于作为新的arena的空间大小的参考。

#### 2.3.5 alloc_perturb()
`void alloc_perturb (char *p, size_t n)`：如果 `perturb_byte`（使用 `M_PERTURB` 的 `malloc` 的可调参数）不为零（默认情况下为 0），则修改 `p` 指向的 n 个字节的内容，使其与 `perturb_byte ^ 0xff` 相等。

#### 2.3.6 free_perturb()
`void free_perturb (char *p, size_t n)`：如果 `perturb_byte`（使用 `M_PERTURB` 的 `malloc` 的可调参数）不为零（默认情况下为 0），则修改 `p` 指向的 n 个字节的内容，使其与 `perturb_byte` 相等。

#### 2.3.6 Sysmalloc 
Sysmalloc 需要系统用 malloc 函数分配更多的内存, 在使用的时候假设 `av->top` 的空间并不够请求 `nb` 字节，因此需要延长或者替换 `av->top` 。

### 2.4 源代码
#### 2.4.1 _int_malloc()源代码
```C
/*
   ------------------------------ malloc ------------------------------
 */

static void *
_int_malloc (mstate av, size_t bytes)
{
  INTERNAL_SIZE_T nb;               /* normalized request size */
  unsigned int idx;                 /* associated bin index */
  mbinptr bin;                      /* associated bin */

  mchunkptr victim;                 /* inspected/selected chunk */
  INTERNAL_SIZE_T size;             /* its size */
  int victim_index;                 /* its bin index */

  mchunkptr remainder;              /* remainder from a split */
  unsigned long remainder_size;     /* its size */

  unsigned int block;               /* bit map traverser */
  unsigned int bit;                 /* bit map traverser */
  unsigned int map;                 /* current word of binmap */

  mchunkptr fwd;                    /* misc temp for linking */
  mchunkptr bck;                    /* misc temp for linking */

  const char *errstr = NULL;

  /*
     Convert request size to internal form by adding SIZE_SZ bytes
     overhead plus possibly more to obtain necessary alignment and/or
     to obtain a size of at least MINSIZE, the smallest allocatable
     size. Also, checked_request2size traps (returning 0) request sizes
     that are so large that they wrap around zero when padded and
     aligned.
   */

  checked_request2size (bytes, nb);

  /* There are no usable arenas.  Fall back to sysmalloc to get a chunk from
     mmap.  */
  if (__glibc_unlikely (av == NULL))
    {
      void *p = sysmalloc (nb, av);
      if (p != NULL)
    alloc_perturb (p, bytes);
      return p;
    }

  /*
     If the size qualifies as a fastbin, first check corresponding bin.
     This code is safe to execute even if av is not yet initialized, so we
     can try it without checking, which saves some time on this fast path.
   */

  if ((unsigned long) (nb) <= (unsigned long) (get_max_fast ()))
    {
      idx = fastbin_index (nb);
      mfastbinptr *fb = &fastbin (av, idx);
      mchunkptr pp = *fb;
      do
        {
          victim = pp;
          if (victim == NULL)
            break;
        }
      while ((pp = catomic_compare_and_exchange_val_acq (fb, victim->fd, victim))
             != victim);
      if (victim != 0)
        {
          if (__builtin_expect (fastbin_index (chunksize (victim)) != idx, 0))
            {
              errstr = "malloc(): memory corruption (fast)";
            errout:
              malloc_printerr (check_action, errstr, chunk2mem (victim), av);
              return NULL;
            }
          check_remalloced_chunk (av, victim, nb);
          void *p = chunk2mem (victim);
          alloc_perturb (p, bytes);
          return p;
        }
    }

  /*
     If a small request, check regular bin.  Since these "smallbins"
     hold one size each, no searching within bins is necessary.
     (For a large request, we need to wait until unsorted chunks are
     processed to find best fit. But for small ones, fits are exact
     anyway, so we can check now, which is faster.)
   */

  if (in_smallbin_range (nb))
    {
      idx = smallbin_index (nb);
      bin = bin_at (av, idx);

      if ((victim = last (bin)) != bin)
        {
          if (victim == 0) /* initialization check */
            malloc_consolidate (av);
          else
            {
              bck = victim->bk;
    if (__glibc_unlikely (bck->fd != victim))
                {
                  errstr = "malloc(): smallbin double linked list corrupted";
                  goto errout;
                }
              set_inuse_bit_at_offset (victim, nb);
              bin->bk = bck;
              bck->fd = bin;

              if (av != &main_arena)
                victim->size |= NON_MAIN_ARENA;
              check_malloced_chunk (av, victim, nb);
              void *p = chunk2mem (victim);
              alloc_perturb (p, bytes);
              return p;
            }
        }
    }

  /*
     If this is a large request, consolidate fastbins before continuing.
     While it might look excessive to kill all fastbins before
     even seeing if there is space available, this avoids
     fragmentation problems normally associated with fastbins.
     Also, in practice, programs tend to have runs of either small or
     large requests, but less often mixtures, so consolidation is not
     invoked all that often in most programs. And the programs that
     it is called frequently in otherwise tend to fragment.
   */

  else
    {
      idx = largebin_index (nb);
      if (have_fastchunks (av))
        malloc_consolidate (av);
    }

  /*
     Process recently freed or remaindered chunks, taking one only if
     it is exact fit, or, if this a small request, the chunk is remainder from
     the most recent non-exact fit.  Place other traversed chunks in
     bins.  Note that this step is the only place in any routine where
     chunks are placed in bins.

     The outer loop here is needed because we might not realize until
     near the end of malloc that we should have consolidated, so must
     do so and retry. This happens at most once, and only when we would
     otherwise need to expand memory to service a "small" request.
   */

  for (;; )
    {
      int iters = 0;
      while ((victim = unsorted_chunks (av)->bk) != unsorted_chunks (av))
        {
          bck = victim->bk;
          if (__builtin_expect (victim->size <= 2 * SIZE_SZ, 0)
              || __builtin_expect (victim->size > av->system_mem, 0))
            malloc_printerr (check_action, "malloc(): memory corruption",
                             chunk2mem (victim), av);
          size = chunksize (victim);

          /*
             If a small request, try to use last remainder if it is the
             only chunk in unsorted bin.  This helps promote locality for
             runs of consecutive small requests. This is the only
             exception to best-fit, and applies only when there is
             no exact fit for a small chunk.
           */

          if (in_smallbin_range (nb) &&
              bck == unsorted_chunks (av) &&
              victim == av->last_remainder &&
              (unsigned long) (size) > (unsigned long) (nb + MINSIZE))
            {
              /* split and reattach remainder */
              remainder_size = size - nb;
              remainder = chunk_at_offset (victim, nb);
              unsorted_chunks (av)->bk = unsorted_chunks (av)->fd = remainder;
              av->last_remainder = remainder;
              remainder->bk = remainder->fd = unsorted_chunks (av);
              if (!in_smallbin_range (remainder_size))
                {
                  remainder->fd_nextsize = NULL;
                  remainder->bk_nextsize = NULL;
                }

              set_head (victim, nb | PREV_INUSE |
                        (av != &main_arena ? NON_MAIN_ARENA : 0));
              set_head (remainder, remainder_size | PREV_INUSE);
              set_foot (remainder, remainder_size);

              check_malloced_chunk (av, victim, nb);
              void *p = chunk2mem (victim);
              alloc_perturb (p, bytes);
              return p;
            }

          /* remove from unsorted list */
          unsorted_chunks (av)->bk = bck;
          bck->fd = unsorted_chunks (av);

          /* Take now instead of binning if exact fit */

          if (size == nb)
            {
              set_inuse_bit_at_offset (victim, size);
              if (av != &main_arena)
                victim->size |= NON_MAIN_ARENA;
              check_malloced_chunk (av, victim, nb);
              void *p = chunk2mem (victim);
              alloc_perturb (p, bytes);
              return p;
            }

          /* place chunk in bin */

          if (in_smallbin_range (size))
            {
              victim_index = smallbin_index (size);
              bck = bin_at (av, victim_index);
              fwd = bck->fd;
            }
          else
            {
              victim_index = largebin_index (size);
              bck = bin_at (av, victim_index);
              fwd = bck->fd;

              /* maintain large bins in sorted order */
              if (fwd != bck)
                {
                  /* Or with inuse bit to speed comparisons */
                  size |= PREV_INUSE;
                  /* if smaller than smallest, bypass loop below */
                  assert ((bck->bk->size & NON_MAIN_ARENA) == 0);
                  if ((unsigned long) (size) < (unsigned long) (bck->bk->size))
                    {
                      fwd = bck;
                      bck = bck->bk;

                      victim->fd_nextsize = fwd->fd;
                      victim->bk_nextsize = fwd->fd->bk_nextsize;
                      fwd->fd->bk_nextsize = victim->bk_nextsize->fd_nextsize = victim;
                    }
                  else
                    {
                      assert ((fwd->size & NON_MAIN_ARENA) == 0);
                      while ((unsigned long) size < fwd->size)
                        {
                          fwd = fwd->fd_nextsize;
                          assert ((fwd->size & NON_MAIN_ARENA) == 0);
                        }

                      if ((unsigned long) size == (unsigned long) fwd->size)
                        /* Always insert in the second position.  */
                        fwd = fwd->fd;
                      else
                        {
                          victim->fd_nextsize = fwd;
                          victim->bk_nextsize = fwd->bk_nextsize;
                          fwd->bk_nextsize = victim;
                          victim->bk_nextsize->fd_nextsize = victim;
                        }
                      bck = fwd->bk;
                    }
                }
              else
                victim->fd_nextsize = victim->bk_nextsize = victim;
            }

          mark_bin (av, victim_index);
          victim->bk = bck;
          victim->fd = fwd;
          fwd->bk = victim;
          bck->fd = victim;

#define MAX_ITERS       10000
          if (++iters >= MAX_ITERS)
            break;
        }

      /*
         If a large request, scan through the chunks of current bin in
         sorted order to find smallest that fits.  Use the skip list for this.
       */

      if (!in_smallbin_range (nb))
        {
          bin = bin_at (av, idx);

          /* skip scan if empty or largest chunk is too small */
          if ((victim = first (bin)) != bin &&
              (unsigned long) (victim->size) >= (unsigned long) (nb))
            {
              victim = victim->bk_nextsize;
              while (((unsigned long) (size = chunksize (victim)) <
                      (unsigned long) (nb)))
                victim = victim->bk_nextsize;

              /* Avoid removing the first entry for a size so that the skip
                 list does not have to be rerouted.  */
              if (victim != last (bin) && victim->size == victim->fd->size)
                victim = victim->fd;

              remainder_size = size - nb;
              unlink (av, victim, bck, fwd);

              /* Exhaust */
              if (remainder_size < MINSIZE)
                {
                  set_inuse_bit_at_offset (victim, size);
                  if (av != &main_arena)
                    victim->size |= NON_MAIN_ARENA;
                }
              /* Split */
              else
                {
                  remainder = chunk_at_offset (victim, nb);
                  /* We cannot assume the unsorted list is empty and therefore
                     have to perform a complete insert here.  */
                  bck = unsorted_chunks (av);
                  fwd = bck->fd;
      if (__glibc_unlikely (fwd->bk != bck))
                    {
                      errstr = "malloc(): corrupted unsorted chunks";
                      goto errout;
                    }
                  remainder->bk = bck;
                  remainder->fd = fwd;
                  bck->fd = remainder;
                  fwd->bk = remainder;
                  if (!in_smallbin_range (remainder_size))
                    {
                      remainder->fd_nextsize = NULL;
                      remainder->bk_nextsize = NULL;
                    }
                  set_head (victim, nb | PREV_INUSE |
                            (av != &main_arena ? NON_MAIN_ARENA : 0));
                  set_head (remainder, remainder_size | PREV_INUSE);
                  set_foot (remainder, remainder_size);
                }
              check_malloced_chunk (av, victim, nb);
              void *p = chunk2mem (victim);
              alloc_perturb (p, bytes);
              return p;
            }
        }

      /*
         Search for a chunk by scanning bins, starting with next largest
         bin. This search is strictly by best-fit; i.e., the smallest
         (with ties going to approximately the least recently used) chunk
         that fits is selected.

         The bitmap avoids needing to check that most blocks are nonempty.
         The particular case of skipping all bins during warm-up phases
         when no chunks have been returned yet is faster than it might look.
       */

      ++idx;
      bin = bin_at (av, idx);
      block = idx2block (idx);
      map = av->binmap[block];
      bit = idx2bit (idx);

      for (;; )
        {
          /* Skip rest of block if there are no more set bits in this block.  */
          if (bit > map || bit == 0)
            {
              do
                {
                  if (++block >= BINMAPSIZE) /* out of bins */
                    goto use_top;
                }
              while ((map = av->binmap[block]) == 0);

              bin = bin_at (av, (block << BINMAPSHIFT));
              bit = 1;
            }

          /* Advance to bin with set bit. There must be one. */
          while ((bit & map) == 0)
            {
              bin = next_bin (bin);
              bit <<= 1;
              assert (bit != 0);
            }

          /* Inspect the bin. It is likely to be non-empty */
          victim = last (bin);

          /*  If a false alarm (empty bin), clear the bit. */
          if (victim == bin)
            {
              av->binmap[block] = map &= ~bit; /* Write through */
              bin = next_bin (bin);
              bit <<= 1;
            }

          else
            {
              size = chunksize (victim);

              /*  We know the first chunk in this bin is big enough to use. */
              assert ((unsigned long) (size) >= (unsigned long) (nb));

              remainder_size = size - nb;

              /* unlink */
              unlink (av, victim, bck, fwd);

              /* Exhaust */
              if (remainder_size < MINSIZE)
                {
                  set_inuse_bit_at_offset (victim, size);
                  if (av != &main_arena)
                    victim->size |= NON_MAIN_ARENA;
                }

              /* Split */
              else
                {
                  remainder = chunk_at_offset (victim, nb);

                  /* We cannot assume the unsorted list is empty and therefore
                     have to perform a complete insert here.  */
                  bck = unsorted_chunks (av);
                  fwd = bck->fd;
      if (__glibc_unlikely (fwd->bk != bck))
                    {
                      errstr = "malloc(): corrupted unsorted chunks 2";
                      goto errout;
                    }
                  remainder->bk = bck;
                  remainder->fd = fwd;
                  bck->fd = remainder;
                  fwd->bk = remainder;

                  /* advertise as last remainder */
                  if (in_smallbin_range (nb))
                    av->last_remainder = remainder;
                  if (!in_smallbin_range (remainder_size))
                    {
                      remainder->fd_nextsize = NULL;
                      remainder->bk_nextsize = NULL;
                    }
                  set_head (victim, nb | PREV_INUSE |
                            (av != &main_arena ? NON_MAIN_ARENA : 0));
                  set_head (remainder, remainder_size | PREV_INUSE);
                  set_foot (remainder, remainder_size);
                }
              check_malloced_chunk (av, victim, nb);
              void *p = chunk2mem (victim);
              alloc_perturb (p, bytes);
              return p;
            }
        }

    use_top:
      /*
         If large enough, split off the chunk bordering the end of memory
         (held in av->top). Note that this is in accord with the best-fit
         search rule.  In effect, av->top is treated as larger (and thus
         less well fitting) than any other available chunk since it can
         be extended to be as large as necessary (up to system
         limitations).

         We require that av->top always exists (i.e., has size >=
         MINSIZE) after initialization, so if it would otherwise be
         exhausted by current request, it is replenished. (The main
         reason for ensuring it exists is that we may need MINSIZE space
         to put in fenceposts in sysmalloc.)
       */

      victim = av->top;
      size = chunksize (victim);

      if ((unsigned long) (size) >= (unsigned long) (nb + MINSIZE))
        {
          remainder_size = size - nb;
          remainder = chunk_at_offset (victim, nb);
          av->top = remainder;
          set_head (victim, nb | PREV_INUSE |
                    (av != &main_arena ? NON_MAIN_ARENA : 0));
          set_head (remainder, remainder_size | PREV_INUSE);

          check_malloced_chunk (av, victim, nb);
          void *p = chunk2mem (victim);
          alloc_perturb (p, bytes);
          return p;
        }

      /* When we are using atomic ops to free fast chunks we can get
         here for all block sizes.  */
      else if (have_fastchunks (av))
        {
          malloc_consolidate (av);
          /* restore original bin index */
          if (in_smallbin_range (nb))
            idx = smallbin_index (nb);
          else
            idx = largebin_index (nb);
        }

      /*
         Otherwise, relay to handle system-dependent cases
       */
      else
        {
          void *p = sysmalloc (nb, av);
          if (p != NULL)
            alloc_perturb (p, bytes);
          return p;
        }
    }
}
```


#### 2.4.2 _int_free()源代码
```C

/*
   ------------------------------ free ------------------------------
 */

static void
_int_free (mstate av, mchunkptr p, int have_lock)
{
  INTERNAL_SIZE_T size;        /* its size */
  mfastbinptr *fb;             /* associated fastbin */
  mchunkptr nextchunk;         /* next contiguous chunk */
  INTERNAL_SIZE_T nextsize;    /* its size */
  int nextinuse;               /* true if nextchunk is used */
  INTERNAL_SIZE_T prevsize;    /* size of previous contiguous chunk */
  mchunkptr bck;               /* misc temp for linking */
  mchunkptr fwd;               /* misc temp for linking */

  const char *errstr = NULL;
  int locked = 0;

  size = chunksize (p);

  /* Little security check which won't hurt performance: the
     allocator never wrapps around at the end of the address space.
     Therefore we can exclude some size values which might appear
     here by accident or by "design" from some intruder.  */
  if (__builtin_expect ((uintptr_t) p > (uintptr_t) -size, 0)
      || __builtin_expect (misaligned_chunk (p), 0))
    {
      errstr = "free(): invalid pointer";
    errout:
      if (!have_lock && locked)
        (void) mutex_unlock (&av->mutex);
      malloc_printerr (check_action, errstr, chunk2mem (p), av);
      return;
    }
  /* We know that each chunk is at least MINSIZE bytes in size or a
     multiple of MALLOC_ALIGNMENT.  */
  if (__glibc_unlikely (size < MINSIZE || !aligned_OK (size)))
    {
      errstr = "free(): invalid size";
      goto errout;
    }

  check_inuse_chunk(av, p);

  /*
    If eligible, place chunk on a fastbin so it can be found
    and used quickly in malloc.
  */

  if ((unsigned long)(size) <= (unsigned long)(get_max_fast ())

#if TRIM_FASTBINS
      /*
    If TRIM_FASTBINS set, don't place chunks
    bordering top into fastbins
      */
      && (chunk_at_offset(p, size) != av->top)
#endif
      ) {

    if (__builtin_expect (chunk_at_offset (p, size)->size <= 2 * SIZE_SZ, 0)
    || __builtin_expect (chunksize (chunk_at_offset (p, size))
                 >= av->system_mem, 0))
      {
    /* We might not have a lock at this point and concurrent modifications
       of system_mem might have let to a false positive.  Redo the test
       after getting the lock.  */
    if (have_lock
        || ({ assert (locked == 0);
          mutex_lock(&av->mutex);
          locked = 1;
          chunk_at_offset (p, size)->size <= 2 * SIZE_SZ
            || chunksize (chunk_at_offset (p, size)) >= av->system_mem;
          }))
      {
        errstr = "free(): invalid next size (fast)";
        goto errout;
      }
    if (! have_lock)
      {
        (void)mutex_unlock(&av->mutex);
        locked = 0;
      }
      }

    free_perturb (chunk2mem(p), size - 2 * SIZE_SZ);

    set_fastchunks(av);
    unsigned int idx = fastbin_index(size);
    fb = &fastbin (av, idx);

    /* Atomically link P to its fastbin: P->FD = *FB; *FB = P;  */
    mchunkptr old = *fb, old2;
    unsigned int old_idx = ~0u;
    do
      {
    /* Check that the top of the bin is not the record we are going to add
       (i.e., double free).  */
    if (__builtin_expect (old == p, 0))
      {
        errstr = "double free or corruption (fasttop)";
        goto errout;
      }
    /* Check that size of fastbin chunk at the top is the same as
       size of the chunk that we are adding.  We can dereference OLD
       only if we have the lock, otherwise it might have already been
       deallocated.  See use of OLD_IDX below for the actual check.  */
    if (have_lock && old != NULL)
      old_idx = fastbin_index(chunksize(old));
    p->fd = old2 = old;
      }
    while ((old = catomic_compare_and_exchange_val_rel (fb, p, old2)) != old2);

    if (have_lock && old != NULL && __builtin_expect (old_idx != idx, 0))
      {
    errstr = "invalid fastbin entry (free)";
    goto errout;
      }
  }

  /*
    Consolidate other non-mmapped chunks as they arrive.
  */

  else if (!chunk_is_mmapped(p)) {
    if (! have_lock) {
      (void)mutex_lock(&av->mutex);
      locked = 1;
    }

    nextchunk = chunk_at_offset(p, size);

    /* Lightweight tests: check whether the block is already the
       top block.  */
    if (__glibc_unlikely (p == av->top))
      {
    errstr = "double free or corruption (top)";
    goto errout;
      }
    /* Or whether the next chunk is beyond the boundaries of the arena.  */
    if (__builtin_expect (contiguous (av)
              && (char *) nextchunk
              >= ((char *) av->top + chunksize(av->top)), 0))
      {
    errstr = "double free or corruption (out)";
    goto errout;
      }
    /* Or whether the block is actually not marked used.  */
    if (__glibc_unlikely (!prev_inuse(nextchunk)))
      {
    errstr = "double free or corruption (!prev)";
    goto errout;
      }

    nextsize = chunksize(nextchunk);
    if (__builtin_expect (nextchunk->size <= 2 * SIZE_SZ, 0)
    || __builtin_expect (nextsize >= av->system_mem, 0))
      {
    errstr = "free(): invalid next size (normal)";
    goto errout;
      }

    free_perturb (chunk2mem(p), size - 2 * SIZE_SZ);

    /* consolidate backward */
    if (!prev_inuse(p)) {
      prevsize = p->prev_size;
      size += prevsize;
      p = chunk_at_offset(p, -((long) prevsize));
      unlink(av, p, bck, fwd);
    }

    if (nextchunk != av->top) {
      /* get and clear inuse bit */
      nextinuse = inuse_bit_at_offset(nextchunk, nextsize);

      /* consolidate forward */
      if (!nextinuse) {
    unlink(av, nextchunk, bck, fwd);
    size += nextsize;
      } else
    clear_inuse_bit_at_offset(nextchunk, 0);

      /*
    Place the chunk in unsorted chunk list. Chunks are
    not placed into regular bins until after they have
    been given one chance to be used in malloc.
      */

      bck = unsorted_chunks(av);
      fwd = bck->fd;
      if (__glibc_unlikely (fwd->bk != bck))
    {
      errstr = "free(): corrupted unsorted chunks";
      goto errout;
    }
      p->fd = fwd;
      p->bk = bck;
      if (!in_smallbin_range(size))
    {
      p->fd_nextsize = NULL;
      p->bk_nextsize = NULL;
    }
      bck->fd = p;
      fwd->bk = p;

      set_head(p, size | PREV_INUSE);
      set_foot(p, size);

      check_free_chunk(av, p);
    }

    /*
      If the chunk borders the current high end of memory,
      consolidate into top
    */

    else {
      size += nextsize;
      set_head(p, size | PREV_INUSE);
      av->top = p;
      check_chunk(av, p);
    }

    /*
      If freeing a large space, consolidate possibly-surrounding
      chunks. Then, if the total unused topmost memory exceeds trim
      threshold, ask malloc_trim to reduce top.

      Unless max_fast is 0, we don't know if there are fastbins
      bordering top, so we cannot tell for sure whether threshold
      has been reached unless fastbins are consolidated.  But we
      don't want to consolidate on each free.  As a compromise,
      consolidation is performed if FASTBIN_CONSOLIDATION_THRESHOLD
      is reached.
    */

    if ((unsigned long)(size) >= FASTBIN_CONSOLIDATION_THRESHOLD) {
      if (have_fastchunks(av))
    malloc_consolidate(av);

      if (av == &main_arena) {
#ifndef MORECORE_CANNOT_TRIM
    if ((unsigned long)(chunksize(av->top)) >=
        (unsigned long)(mp_.trim_threshold))
      systrim(mp_.top_pad, av);
#endif
      } else {
    /* Always try heap_trim(), even if the top chunk is not
       large, because the corresponding heap might go away.  */
    heap_info *heap = heap_for_ptr(top(av));

    assert(heap->ar_ptr == av);
    heap_trim(heap, mp_.top_pad);
      }
    }

    if (! have_lock) {
      assert (locked);
      (void)mutex_unlock(&av->mutex);
    }
  }
  /*
    If the chunk was allocated via mmap, release via munmap().
  */

  else {
    munmap_chunk (p);
  }
}
```

#### 2.4.3 malloc_consolidate()源代码
```C
/*
  ------------------------- malloc_consolidate -------------------------

  malloc_consolidate is a specialized version of free() that tears
  down chunks held in fastbins.  Free itself cannot be used for this
  purpose since, among other things, it might place chunks back onto
  fastbins.  So, instead, we need to use a minor variant of the same
  code.

  Also, because this routine needs to be called the first time through
  malloc anyway, it turns out to be the perfect place to trigger
  initialization code.
*/

static void malloc_consolidate(mstate av)
{
  mfastbinptr*    fb;                 /* current fastbin being consolidated */
  mfastbinptr*    maxfb;              /* last fastbin (for loop control) */
  mchunkptr       p;                  /* current chunk being consolidated */
  mchunkptr       nextp;              /* next chunk to consolidate */
  mchunkptr       unsorted bin;       /* bin header */
  mchunkptr       first_unsorted;     /* chunk to link to */

  /* These have same use as in free() */
  mchunkptr       nextchunk;
  INTERNAL_SIZE_T size;
  INTERNAL_SIZE_T nextsize;
  INTERNAL_SIZE_T prevsize;
  int             nextinuse;
  mchunkptr       bck;
  mchunkptr       fwd;

  /*
    If max_fast is 0, we know that av hasn't
    yet been initialized, in which case do so below
  */

  if (get_max_fast () != 0) {
    clear_fastchunks(av);

    unsorted bin = unsorted_chunks(av);

    /*
      Remove each chunk from fast bin and consolidate it, placing it
      then in unsorted bin. Among other reasons for doing this,
      placing in unsorted bin avoids needing to calculate actual bins
      until malloc is sure that chunks aren't immediately going to be
      reused anyway.
    */

    maxfb = &fastbin (av, NFASTBINS - 1);
    fb = &fastbin (av, 0);
    do {
      p = atomic_exchange_acq (fb, NULL);
      if (p != 0) {
    do {
      check_inuse_chunk(av, p);
      nextp = p->fd;

      /* Slightly streamlined version of consolidation code in free() */
      size = p->size & ~(PREV_INUSE|NON_MAIN_ARENA);
      nextchunk = chunk_at_offset(p, size);
      nextsize = chunksize(nextchunk);

      if (!prev_inuse(p)) {
        prevsize = p->prev_size;
        size += prevsize;
        p = chunk_at_offset(p, -((long) prevsize));
        unlink(av, p, bck, fwd);
      }

      if (nextchunk != av->top) {
        nextinuse = inuse_bit_at_offset(nextchunk, nextsize);

        if (!nextinuse) {
          size += nextsize;
          unlink(av, nextchunk, bck, fwd);
        } else
          clear_inuse_bit_at_offset(nextchunk, 0);

        first_unsorted = unsorted bin->fd;
        unsorted bin->fd = p;
        first_unsorted->bk = p;

        if (!in_smallbin_range (size)) {
          p->fd_nextsize = NULL;
          p->bk_nextsize = NULL;
        }

        set_head(p, size | PREV_INUSE);
        p->bk = unsorted bin;
        p->fd = first_unsorted;
        set_foot(p, size);
      }

      else {
        size += nextsize;
        set_head(p, size | PREV_INUSE);
        av->top = p;
      }

    } while ( (p = nextp) != 0);

      }
    } while (fb++ != maxfb);
  }
  else {
    malloc_init_state(av);
    check_malloc_state(av);
  }
}
```

### 2.5 参考文章
+ [Glibc：浅谈 malloc() 函数具体实现](https://blog.csdn.net/Plus_RE/article/details/80211488)
+ [Glibc：浅谈 free() 函数具体实现](https://blog.csdn.net/Plus_RE/article/details/79258536)
+ [堆溢出(2)-堆内存分配操作流程](https://www.sec4.fun/2020/05/25/heapOverflow2/)
+ [Understanding the heap by breaking it](https://www.blackhat.com/presentations/bh-usa-07/Ferguson/Whitepaper/bh-usa-07-ferguson-WP.pdf)