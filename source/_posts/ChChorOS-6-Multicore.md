---
title: ChChorOS-6-Multicore
date: 2023-04-22 15:17:42
categories: OS
tags:
---

### 前言

> 本章ChChor多核相关内容。

<!--more-->

### 1 多核启动

所有的CPU核心在开机时被同时启动，在引导时则会被分为两种类型。

一个指定的CPI核心会引导整个操作系统和初始化自身，被称为**主CPU**。

其他CPU只初始化自身即可，被称为**副CPU**。

CPU核心仅在系统引导时有所区分，在其他阶段，每个CPU核心都是被相同对待的。

**选定主CPU：**

```asm
mrs x8, mpidr_el1
and x8, x8, #0xFF
cbz x8, primary
```

mpidr_el1 寄存器低8位为0，则表示主CPU。

**阻塞其他副CPU的执行：**

```asm
wait_for_bss_clear:
    adr x0, clear_bss_flag
    ldr x1, [x0]
    cmp     x1, #0
    bne wait_for_bss_clear

    /* Turn to el1 from other exception levels. */
    bl  arm64_elX_to_el1

    /* Prepare stack pointer and jump to C. */
    mov x1, #0x1000
    mul x1, x8, x1
    adr     x0, boot_cpu_stack
    add x0, x0, x1
    add x0, x0, #0x1000
        mov sp, x0

wait_until_smp_enabled:
    /* CPU ID should be stored in x8 from the first line */
    mov x1, #8
    mul x2, x8, x1
    ldr x1, =secondary_boot_flag
    add x1, x1, x2
    ldr x3, [x1]
    cbz x3, wait_until_smp_enabled

    /* Set CPU id */
    mov x0, x8
    bl  secondary_init_c
```

1. 等待主CPU清除`bss`段，循环检查`clear_bss_flag`变量
2. 转为`EL1`异常级别
3. 准备栈指针，这里的`boot_cpu_stack`是每个cpu都有自己单独的栈
4. 循环检查`secondary_boot_flag`变量
5. 设置`cpu id`后跳转到`secondary_init_c`执行，激活副cpu

这里引用boot一文中的流程图：

![startup](ChChorOS-6-Multicore/startup.png)

思考：这里所有的副核心是依次激活，而不是并行激活。如果并行激活，会有问题吗？

- 每个CPU核心不共享内核栈。

  ```c
  char kernel_stack[PLAT_CPU_NUM][KERNEL_STACK_SIZE];
  ```

  各个副CPU仅共享`clear_bss_flag`，但仅读取其中的数据，不会导致数据竞争。

### 2 大内核锁

现在，内核代码已经可以运行在多核上了。为了确保代码不会由于并行执行而引起错误，需要先解决并发问题。这里使用的是大内核锁，即内核的全局共享锁。

**在CPU核心以内核态访问任何数据之前，应该先获得大内核锁。在退出内核态之前释放大内核锁。以确保同时只有一个CPU核心执行内核代码、访问内核数据。**

kernel/common/lock.c中提供了一个简单的排号锁，以充当大内核锁。

**排号锁 （ticket lock）**

排号锁是sync一节的内容，这里用到了，简单介绍写实现：

数据结构：

```c
struct lock {
 volatile u32 owner;
 char pad0[pad_to_cache_line(sizeof(u32))];

 volatile u32 next;
 char pad1[pad_to_cache_line(sizeof(u32))];
} __attribute__ ((aligned(CACHELINE_SZ)));
```

主要是一个owner和一个next成员，owner == next 的时候，才是给自己的锁，否则是别人的锁。这样实现了先来后到。

```asm
struct lock big_kernel_lock;

int lock_init(struct lock *lock)
{
    BUG_ON(!lock);
    /* Initialize ticket lock */
    lock->owner = 0;
    lock->next = 0;
    return 0;
}

/* 持锁 */
void lock(struct lock *lock)
{
    u32 lockval = 0, newval = 0, ret = 0;

    BUG_ON(!lock);

    /* 原子操作 */
    lock->next = fetch_and_add(1);
    while(locl->next != lock->owner);
}

/* 释放锁：由于同时只有一个人可以持锁，因此释放锁不需要原子操作 */
void unlock(struct lock *lock)
{
    BUG_ON(!lock);
    asm volatile("dmb ish");

    lock->owner++;
}


int is_locked(struct lock *lock)
{
    return (lock->owner < lock->next);
}


void kernel_lock_init(void)
{
    lock_init(&big_kernel_lock);
}


void lock_kernel(void)
{
    lock(&big_kernel_lock);
}


void unlock_kernel(void)
{
    unlock(&big_kernel_lock);
}
```

### 3 锁、放锁时机

1. 在主CPU激活副CPU之前需要先获取大内核锁
2. 副CPU初始化完成之后且副CPU返回用户态之前获取大内核锁
3. exception_table.S中的el0_syscall：在跳转到syscall_table中相应的syscall条目之前获取大内核锁
4. handle_entry_c：在该异常处理函数第一行获取大内核锁。但是内核态也可能发生异常，如果异常是在内核中捕获的，则不应该获取大内核锁

```c
void handle_entry_c(int type, u64 esr, u64 address)
{
 /** 
  *
  * Acquire the big kernel lock, if the exception is not from kernel
  */
 if (type >= SYNC_EL0_64) { 
  lock_kernel();
 }
    ...
}
```

5. handle_irq：在中断处理函数第一行获取大内核锁（同样如果是内核异常，则不获取）

```c
void handle_irq(int type)
{
 // 如果是内核态捕获的异常，不需要持锁，因为进内核态的时候已经持过锁了。
 // 但是在idle线程（以内核态运行）中捕获了错误，还是需要获取大内核锁，原因是idle线程运行时并不会持锁
 if(type == IRQ_EL0_64 || type == IRQ_EL0_32 || current_thread->thread_ctx->type == TYPE_IDLE){ //see thread.h and sched.h
  lock_kernel();
 }

 plat_handle_irq();


 sched_handle_timer_irq();
 sched();
 eret_to_thread(switch_context());
}
```

6. exception_return：释放大内核锁。所有情况下，内核态返回用户态都使用exception_return，因此这是唯一需要调用 `unlock_kernel()`的位置。