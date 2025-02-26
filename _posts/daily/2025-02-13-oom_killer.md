---
layout: post
title: "[每日一文]Linux OOM Killer机制"
date: 2025-02-13
tags: [Linux,每日一文]
comments: false
author: 小辣条
toc : false
---
当系统内存不足以分配时，Linux内核会使用一种OOM Killer（Out-Of-Memory Killer）机制释放内存，该机制通过一系列比较选择出最适合的进程并将其kill掉，从而达到保障系统稳定运行的目的。那么在内核中，OOM Killer具体是怎么运转的呢？
<!-- more -->
## 1、触发过程

在申请内存时，必然会调用alloc_page()，如果定义了oom_killer_disabled，就会直接goto到nopage，不会触发OOM机制（此值默认为0)

## 2、OOM Killer的选择

当内核检测到内存不足，执行到out_of_memory时，OOM Killer会选择一个进程并把他kill掉:
1. 首先，内核会遍历所有进程，计算出每个进程的OOM分数，然后选择OOM分数最高的进程kill掉。
2. OOM分数的考虑方面: 
   - 进程的内存使用量
   - 进程的运行时间
   - 进程的nice值
   - 进程消耗的内存大小
   - 进是否为超级用户（root）启动的进程
   - 是否有活跃的子进程
   - 进程的oom_score_adj值是用户通过/proc/pid/oom_score_adj文件设置的，可以用来调整进程的OOM分数。
3. OOM Killer会选择OOM分数最高的进程kill掉，释放内存。
4. 如果OOM Killer选择了一个进程，那么内核会向该进程发送一个SIGKILL信号，进程会被kill掉。

## 3、到底为什么会发生Out Of Memory？

因为物理内存页的分配发生在使用的瞬间而非分配的瞬间。若某个进程申请了200MB内存，但实际上只使用了100MB，未使用到的100MB根本没有分配物理内存页。当进程需要内存时，进程从内核得到的只是虚拟地址的使用权，而不是实际的物理地址，实际的物理内存只有当进程真的去访问新获取的虚拟地址时，产生缺页异常，从而进入分配实际物理地址的过程，之后系统返回产生异常的地址，重新执行内存访问。虚拟内存需要物理内存作为支撑，当分配了太多虚拟内存，导致物理内存不够时，就发生了Out Of Memory。这种允许超额commit的机制就是overcommit。

overcommit即操作系统在应用申请内存空间时不去检查是否超出当前可用量，随意满足申请要求，应用也不管实际是否有足够多的内存可使用，认为我申请了2G，OS肯定就给我2G使用。最后，随着内存越用越多，OS发现内存不够用了，必须要收回一些内存才行，就触发了上述的OOM Killer机制回收内存。

Linux根据参数 vm.overcommit_memory设置overcommit：

0 ——默认值，启发式overcommit，它允许overcommit，但太明显的overcommit会被拒绝，比如malloc一次性申请的内存大小就超过了系统总内存。

1 ——Always overcommit. 允许overcommit，对内存申请来者不拒。

2 ——不允许overcommit，提交给系统的总地址空间大小不允许超过CommitLimit。（CommitLimit 就是overcommit的阈值，申请的内存总数超过CommitLimit的话就算是overcommit）

## 4、总结

由于物理内存的分配机制，以及overcommit的存在，导致了在物理内存不够时的OOM Killer。OOM Killer机制很有意思，它为了保护整个系统的安全稳定运行，需要找出一个最合适的进程kill掉。这是不得已而为之，内核必须在kill掉进程和系统崩溃之间选择其中一个。内核代码中out_of_memory注释中也体现了这种无奈。

在选择合适的进程时，OOM Killer会挑选一个占用内存最大的进程，这也很好理解，毕竟kill掉一个大的可以获得更多的物理内存，并且损失也比较小。如果kill掉多个小的，损失会比较大。Linux内核总是去选择更高效的方法。


## 参考
https://mp.weixin.qq.com/s/c1z7rqAtg3tFwWvelCNHoA