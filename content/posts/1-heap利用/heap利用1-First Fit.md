---
title: "heap利用 1/9: First Fit"
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



first-fit 是一个glibc堆管理机制。任何一个chunk（fast chunk除外）被free后都会被插入到unsorted bin链表的头部。当请求一个新的chunk时（非fast chunk)，`malloc`会先从unsorted bin链表的尾部开始查找合适的chunk。如果查找到unsorted bin链表中存在一个满足要求的chunk，就不再继续查找其他chunk，如果该chunk的大小大于等于请求的大小，则将其拆分成两个chunk并返回。在unsorted bin链表的头部进行的插入，从尾部开始的查找，保证了先进先出的特性。

观察以下代码:
```c
char *a = malloc(300);    // 0x***010
char *b = malloc(250);    // 0x***150

free(a);

a = malloc(250);          // 0x***010
```
chunk 'a'被分成a1，a2 两个chunk，因为请求的大小为250个字节，小于'a'的大小（300个字节）。这在`_int_malloc`中有对应的部分。
对应unsorted bin链表的状态变化如下：
1. 'a'被free后
>unsorted bin链表：head->a->tail
2. 'malloc'请求后，返回a1，bin中留下a2
>unsorted bin链表：head->a2->tail

---

在fast chunk的情况下也是如此。fast chunk被free之后会被放入到fast bins中。如前所述，fast bin维护一个单链接的列表，并从头部进行插入和删除操作，这种操作会"颠倒"获得的chunk的顺序。
观察以下代码：
```c
char *a = malloc(20);     // 0xe4b010
char *b = malloc(20);     // 0xe4b030
char *c = malloc(20);     // 0xe4b050
char *d = malloc(20);     // 0xe4b070

free(a);
free(b);
free(c);
free(d);

a = malloc(20);           // 0xe4b070
b = malloc(20);           // 0xe4b050
c = malloc(20);           // 0xe4b030
d = malloc(20);           // 0xe4b010
```
程序中使用较小的大小（20个字节），确保了在free时，chunk进入fast bin而不是unsorted bin。
对应fast bin的状态如下：
1. `a`被free后
>fast bin链表：head->a->tail
2. `b`被free后
>fast bin链表：head->b->a->tail
3. `c`被free后
>fast bin链表：head->c->b->a->tail
4. `d`被free后
>fast bin链表：head->d->c->b->a->tail
5. `malloc`请求后，返回`d`
>fast bin链表：head->c->b->a->tail
6. `malloc`请求后，返回`c`
>fast bin链表：head->b->a->tail
7. `malloc`请求后，返回`b`
>fast bin链表：head->a->tail
8.`malloc`请求后，返回`a`
>fast bin链表：head->tail

## Use after Free漏洞
在上面的示例中，我们看到`malloc`可能会返回先前使用以及被free的chunk，而使用被free的内存chunk很容易受到攻击。chunk一旦被free，应该假设攻击者现在可以通过各种手段控制此chunk内的数据。为了安全起见，该chunk不应该再次使用，而应总是分配一个新的chunk。
参考以下攻击示例：
```c
char *ch = malloc(20);

// Some operations
//  ..
//  ..

free(ch);

// Some operations
//  ..
//  ..

// Attacker can control 'ch'
// This is vulnerable code
// Freed variables should not be used again
if (*ch=='a') {
  // do this
}
```