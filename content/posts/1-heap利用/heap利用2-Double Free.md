---
title: "heap利用 2/9: Double Free"
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

double free指的是多次释放资源可能会导致内存泄漏，造成分配器数据结构的损坏，进而被攻击者利用。在下面的示例程序中，一个fastbin chunk被释放了两次。为了避免glibc关于 'double free' 或 'corruption（fasttop）'的安全检查，攻击者通常会在两次释放相同chunk空间的操作之间释放另一个chunk的空间。然后，如果再次使用malloc申请内存空间，malloc表面上会返回两个不同的指针，但这两个指针将指向相同的内存地址。如果其中一个在攻击者的控制之下，那么他就可以修改另一个指针的内存，从而导致各种攻击（包括代码执行攻击）。

如下方代码所示就存在一个double free漏洞：

```C
a = malloc(10);     // 0xa04010
b = malloc(10);     // 0xa04030
c = malloc(10);     // 0xa04050

free(a);
free(b);  // To bypass "double free or corruption (fasttop)" check
free(a);  // Double Free !!

d = malloc(10);     // 0xa04010
e = malloc(10);     // 0xa04030
f = malloc(10);     // 0xa04010   - Same as 'd' !
```

<br>
特定fastbin的状态变化情况如下：

1. `a` 被释放后。
  > fastbin链表：head -> a -> tail
2. `b` 被释放后。
  > fastbin链表：head -> b -> a -> tail
3. `a` 再一次被释放后。
  > fastbin链表：head -> a -> b -> a -> tail
4. 调用 `malloc` 分配获取 `d` 的内存空间后。
  > fastbin链表：head -> b -> a -> tail [ `a` 已被返回 ]
5. 调用 `malloc` 分配获取 `e` 的内存空间后。
  > fastbin链表：head -> a -> tail [ `b` 已被返回 ]
6. 调用 `malloc` 分配获取 `f` 的内存空间后。
  > fastbin链表：head -> tail [ `a` 已被返回 ]


在该示例中，`d` 和 `f` 指针指向了相同的内存地址，其中一个指针的任何更改都会影响另一个。需要注意的是，如果实例中chunk的大小更改为 smallbin 范围内的大小，则该示例将不起作用。在第一次释放的时候，`a` 的下一个块会将前一个正在使用的位设置为 '0'；在第二次释放的时候，由于此位为 '0'，系统将抛出 'double free' 或 'corruption(!prev)' 错误。

