---
title: "heap基础 2/6: malloc_state"
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


## malloc_state



`malloc_state`展示了Arena标头的详细信息。主线程的Arena是一个全局变量，而不是heap段的一部分。其他线程的 Arena 标头（`malloc_state`结构）本身存储在heap段中。非主要的Arena可以有多个与它们关联的heap，这里的“heap”是指使用的内部结构而不是heap段。

```C
struct malloc_state
{
  /* Serialize access.  */
  __libc_lock_define (, mutex);
  /* Flags (formerly in max_fast).  */
  int flags;

  /* Fastbins */
  mfastbinptr fastbinsY[NFASTBINS];
  /* Base of the topmost chunk -- not otherwise kept in a bin */
  mchunkptr top;
  /* The remainder from the most recent split of a small request */
  mchunkptr last_remainder;
  /* Normal bins packed as described above */
  mchunkptr bins[NBINS * 2 - 2];

  /* Bitmap of bins */
  unsigned int binmap[BINMAPSIZE];

  /* Linked list */
  struct malloc_state *next;
  /* Linked list for free arenas.  Access to this field is serialized
     by free_list_lock in arena.c.  */
  struct malloc_state *next_free;
  /* Number of threads attached to this arena.  0 if the arena is on
     the free list.  Access to this field is serialized by
     free_list_lock in arena.c.  */

  INTERNAL_SIZE_T attached_threads;
  /* Memory allocated from the system in this arena.  */
  INTERNAL_SIZE_T system_mem;
  INTERNAL_SIZE_T max_system_mem;
};

typedef struct malloc_state *mstate;


```