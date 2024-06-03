[TOC]

# 0.前言

## 0.1 变量及函数命名规范

- 变量名
  - 'c' for `char`
  - 's' for `int16_t`(short)
  - 'l' for `int32_t` (long)
  - 'x' for `BaseType_t` and any other non-standard types (structures, task handles, queue handles, etc.).
  - If a variable is unsigned, it is also prefixed with a 'u'.
  - If a variable is a pointer, it is also prefixed with a 'p'.

> For example, a variable of type `uint8_t` will be prefixed with 'uc', and a variable of type pointer to char (`char *`) will be prefixed with 'pc'.

- 函数名
  - 函数的前缀是它们返回的类型和定义它们的文件

  - v**Task**PrioritySet() 返回值是 *v*oid 存在与 **tasks**.c.
  - x**Queue**Receive() 返回值是变量 of type *BaseType_t* 存在与 **queue**.c.
  - pv**Timer**GetTimerID() 返回值是*v*oid指针存在与 **timers**.c.

  - 文件作用域(私有)函数以`prv`.

- 宏定义

大多数宏都是用大写字母写成的，并以小写字母作为前缀，表示宏的定义位置。表3提供了一个前缀列表。

<a name="tbl3" title="Table 3 Macro prefixes"></a>

* * *
| Prefix                                       | Location of macro definition   |
|----------------------------------------------|--------------------------------|
| port (for example, `portMAX_DELAY`)          | `portable.h` or `portmacro.h`  |
| task (for example, `taskENTER_CRITICAL()`)   | `task.h`                       |
| pd (for example, `pdTRUE`)                   | `projdefs.h`                   |
| config (for example, `configUSE_PREEMPTION`) | `FreeRTOSConfig.h`             |
| err (for example, `errQUEUE_FULL`)           | `projdefs.h`                   |

***Table 3*** *Macro prefixes*
* * *

请注意，信号量API几乎完全是作为一组宏编写的，但遵循函数命名约定，而不是宏命名约定。

## 0.2 宏编写函数和内联函数的区别

```c
/* STATIC FUNCTIONS ARE DEFINED AS MACROS TO MINIMIZE THE FUNCTION CALL DEPTH. */
#define prvInsertBlockIntoFreeList( pxBlockToInsert )
{

}
/*-----------------------------------------------------------*/
```

1. 编译器决定：内联函数是否真正内联展开最终由编译器决定，因此不能保证一定会内联展开。
2. 所以使用宏编写函数，可以保证一定会内联展开。

## 0.3 `mtCOVERAGE_TEST_MARKER`跟踪宏的作用

默认情况下，跟踪宏为空，因此跟踪宏不会生成任何代码。因此，使用虚拟宏定义执行 MISRA 合规性检查。

## 0.4 计算机安全领域中的`Canary`

`canary`值是在栈中插入的随机值，然后在执行函数之前检查该值是否被修改。如果值被修改，就表明栈可能遭到缓冲区溢出攻击。这种机制有助于保护计算机系统免受恶意代码的攻击。它是一种安全机制，用于防止缓冲区溢出攻击。

## 0.5 taskENTER_CRITICAL 和 vTaskSuspendAll

- vTaskSuspendAll

将`uxSchedulerSuspended`+1, 在上下文切换时, 任务调度器会检查`uxSchedulerSuspended`是否为0, 如果不为0, 则不会进行任务调度.

- taskENTER_CRITICAL 执行中断屏蔽操作 

# 1.内存管理

heap_1 —— 最简单，不允许释放内存。

heap_2—— 允许释放内存，但不会合并相邻的空闲块。

heap_3 —— 简单包装了标准 malloc() 和 free()，以保证线程安全。

heap_4 —— 合并相邻的空闲块以避免碎片化。 包含绝对地址放置选项。

heap_5 —— 如同 heap_4，能够跨越多个不相邻内存区域的堆。

注意：
heap_1 不太有用，因为 FreeRTOS 添加了静态分配支持。heap_2 现在被视为旧版，因为较新的 heap_4 实现是首选。

## 1.1 HEAP_1

Heap\_1.c 实现了基础版本的 `pvPortMalloc()`, 没有实现 `vPortFree()`.  从不删除任务或其他内核对象的应用程序可能会使用 heap\_1. 关键系统通常会禁止动态内存分配，因为存在与非确定性、内存碎片和分配失败相关的不确定性。Heap_1 始终是确定性的，不会产生内存碎片。

Heap_1 的 pvPortMalloc() 实现只是在每次调用时将一个简单的 uint8_t 数组（称为 FreeRTOS 堆）细分为更小的块。FreeRTOSConfig.h 常量 configTOTAL_HEAP_SIZE 设置数组的大小（以字节为单位）。将堆实现为静态分配的数组会使 FreeRTOS 看起来消耗大量 RAM，因为堆成为 FreeRTOS 数据的一部分。

每个动态分配的任务都会调用两次 pvPortMalloc()。第一次分配任务控制块 (TCB)，第二次分配任务堆栈.

## 1.2 HEAP_2

Heap_2 被 heap_4 取代，后者包含增强的功能。Heap_2 保留在 FreeRTOS 发行版中以实现向后兼容，不建议用于新设计。

最佳匹配算法可确保 `pvPortMalloc()` 使用大小最接近请求的字节数的空闲内存块。例如，考虑以下场景：堆包含三个空闲内存块，分别为 5 字节、25 字节和 100 字节。 `pvPortMalloc()` 请求 20 字节 RAM。 可容纳请求的字节数的最小空闲 RAM 块是 25 字节块，因此 `pvPortMalloc()` 将 25 字节块拆分为一个 20 字节块和一个 5 字节块，然后返回指向 20 字节块的指针。新的 5 字节块仍可用于将来对 pvPortMalloc() 的调用。 与 heap_4 不同，heap_2 不会将相邻的空闲块合并为一个更大的块，因此它比 heap_4 更容易出现碎片。但是，如果分配和随后释放的块始终大小相同，则碎片不是问题.

## 1.3 HEAP_3

Heap_3.c 使用标准库 malloc() 和 free() 函数，因此链接器配置定义堆大小，并且不使用 configTOTAL_HEAP_SIZE 常量。 Heap_3 通过在 malloc() 和 free() 执行期间暂时挂起 FreeRTOS 调度程序，使它们线程安全。

## 1.4 HEAP_4

- configENABLE_HEAP_PROTECTOR

`configENABLE_HEAP_PROTECTOR` 设置为 1 以启用混淆堆中的内部堆块指针,防止恶意软件通过修改堆块指针来破坏堆管理器的内部数据结构。这是一种安全功能，用于防止堆管理器被恶意软件破坏。

## 1.5 HEAP_5

Heap_5 使用与 heap_4 相同的分配算法。与 heap_4 不同，heap_4 只能从单个数组分配内存，而 heap_5 可以将来自多个独立内存空间的内存组合成一个堆。当运行 FreeRTOS 的系统提供的 RAM 在系统的内存映射中不是单个连续（无空间）的块时，Heap_5 非常有用。

# 2.任务管理

- 任务不能直接删除自己，因为任务删除自己会导致任务调度器的数据结构被破坏。任务可以调用 vTaskDelete(NULL) 以删除自己。如果任务使用 vTaskDelete() API 函数删除自身，则空闲任务必须有足够的处理时间。这是因为空闲任务负责清理删除自身的任务所使用的内核资源。

- 每个任务需要两个 RAM 块：一个用于保存其任务控制块 (TCB)，另一个用于存储其堆栈。 

- 一些 FreeRTOS 移植支持在“受限”或“非特权”模式下运行的任务。名称中带有“Restricted”的 FreeRTOS API 函数会创建在执行时对系统内存具有有限访问权限的任务。名称中不带“Restricted”的 API 函数会创建在“特权模式”下执行并可以访问系统整个内存映射的任务。

- 支持对称多处理 (SMP) 的 FreeRTOS 移植允许不同的任务在同一 CPU 的多个核心上同时运行。对于这些移植，您可以使用名称中带有“Affinity”的函数来指定任务将在哪个核心上运行

- 注意，该值指定堆栈可以容纳的字数，而不是字节数。例如，如果堆栈宽度为 32 位且 usStackDepth 为 128，则 xTaskCreate() 分配 512 字节的堆栈空间（128 * 4 字节）

- 0 为最低优先级，(configMAX_PRIORITIES - 1) 为最高优先级。

- configUSE_PORT_optimized_TASK_SELECTION = 1,设置为 1 以使用架构优化实现。架构优化的实现是用架构特定的汇编代码编写的，比通用的 c 实现性能更高，并且所有 configMAX_PRIORITIES 值的最坏情况执行时间都是相同的。

查看M3内核的优化实现，仅针对位图查找做了内核架构优化,优化了查找速度.

## 2.1 任务创建

1. 将任务栈初始化;并且将栈顶指针指向的位置初始化为任务入口函数的地址.

```c
//GCC M3
StackType_t * pxPortInitialiseStack( StackType_t * pxTopOfStack,
                                     TaskFunction_t pxCode,
                                     void * pvParameters )
{
    /* Simulate the stack frame as it would be created by a context switch
     * interrupt. */
    pxTopOfStack--;                                                      /* Offset added to account for the way the MCU uses the stack on entry/exit of interrupts. */
    *pxTopOfStack = portINITIAL_XPSR;                                    /* xPSR */
    pxTopOfStack--;
    *pxTopOfStack = ( ( StackType_t ) pxCode ) & portSTART_ADDRESS_MASK; /* PC */
    pxTopOfStack--;
    *pxTopOfStack = ( StackType_t ) portTASK_RETURN_ADDRESS;             /* LR */
    pxTopOfStack -= 5;                                                   /* R12, R3, R2 and R1. */
    *pxTopOfStack = ( StackType_t ) pvParameters;                        /* R0 */
    pxTopOfStack -= 8;                                                   /* R11, R10, R9, R8, R7, R6, R5 and R4. */

    return pxTopOfStack;
}
```

- 其中LR退出时,跳转到`prvTaskExitError`,里面执行空循环

```c
static void prvTaskExitError( void )
{
    volatile uint32_t ulDummy = 0UL;
    configASSERT( uxCriticalNesting == ~0UL );
    portDISABLE_INTERRUPTS();

    while( ulDummy == 0 )
    {
    }
}
```

2. 创建所需的链表:`xDelayedTaskList1`,`xDelayedTaskList2`,`xPendingReadyList`,`xTasksWaitingTermination`,`xSuspendedTaskList`.

3. 将pxTCB代表的任务放入适当的就绪列表中任务。它被插入到列表的末尾。

  - 将uxTopReadyPriority每个bit代表一个优先级,进行置位,快速查找最高优先级任务.

## 2.2 任务删除

1. 从就绪或延迟列表中删除任务

2. 任务删除时,没有正在等待的事件,则直接从事件链表中移除

3. 任务删除时,**线程已经开始执行调度**  ->  正在运行（或产生）
- 将任务插入删除任务链表末尾

4. 任务删除时,任务处于挂起状态
- 重置下一个预期的解锁时间，以防它引用刚刚删除的任务

5. 如果调度没有开始,直接删除TCB

6. 调度开始,当前执行任务就是删除任务,执行一次`Yield`操作,即去产生一次调度

## 2.3 内核任务挂起

1. uxSchedulerSuspended+1
> https://www.freertos.org/FreeRTOS_Support_Forum_Archive/September_2013/freertos_Concerns_about_the_atomicity_of_vTaskSuspendAll_d165e9c3j.html

> 基本上，关键在于每个任务都维护自己的上下文，如果变量非零，则不会发生上下文切换。因此，只要从寄存器写入内存是原子的，就没有问题。

```c
/*此加载将变量的值存储在寄存器中。*/
将 uxSchedulerSuspendeded 加载到寄存器中

...现在上下文切换导致另一个任务运行，并且另一个任务使用相同的变量。另一个任务将看到变量为零，因为变量有
尚未被原始任务更新……最终原始任务再次运行。**这只能发生当 uxSchedulerSuspended 再次为零时**，并且当原来的任务再次运行CPU的内容寄存器被恢复到原来的样子被切换出去——因此它读入的值寄存器仍然匹配的值
xuSchedulerSuspended 变量...

/* 该值将增加至等于 1。 */
增量寄存器

/* 恢复到 uxSchedulerSuspended 的值将是正确值为 1，即使使用了变量同时完成其他任务。*/
将寄存器存储到 uxSchedulerSuspendeded 中
```

## 2.4 内核任务恢复

1. uxSchedulerSuspended必须为0,才能进行任务调度

2. 从就绪列表中找到最高优先级任务,进行调度

3. 移除所用于的事件链表和状态链表

4. 就绪链表任务 > 当前任务优先级,执行`Yield`调度

## 2.5 任务延时挂起

1. 将任务从就绪列表中删除，然后再将其添加到阻止列表中,因为两个列表使用相同的列表项.
2. 如果延时时间是永久,将任务添加到挂起任务列表中，而不是延迟任务列表以确保它不会被定时事件唤醒。 它会无限期阻塞。
3. 否则,根据当前tick计算下一次唤醒时间,插入就绪链表中;
4. 如果溢出,则插入溢出延时链表中

## 2.6 空闲线程

1. 检查并清除已删除的任务
2. 低功耗 TICKLESS 功能

- 获取预期空闲时间
  1. 当前任务优先级 > 空闲线程,即还需要退出空闲线程 -> 不需要空闲时间
  2. 处于就绪状态的任务的优先级高于空闲优先 -> 不需要空闲时间
  3. 其余情况 空闲时间 = 下一个解锁时间 - 当前tick时间

- 预期空闲时间
  1. 设定的最小进入低功耗时间
  2. 进入低功耗
  3. 设置SYSTICK下一次唤醒时间
  4. 唤醒后计算任务补偿时间
- 完成低功耗进退过程	

## 2.7 任务启动

1. 创建空闲线程
2. 创建定时器线程

3. 关闭中断，以确保不会发生SYStcik中断.
4. 硬件调度启动`xPortStartScheduler`

- 使PendSV和SysTick成为最低优先级中断，并使SVCall最高优先级
- 设置SYSTICK定时器和所需的频率

- 启动第一个任务

# 3.队列管理

- FreeRTOS 消息缓冲区（在 TBD 章节中描述）为保存可变长度消息的队列提供了一种更轻量的替代方案
- 队列可以有多个读取，因此单个队列上可能有多个任务被阻塞以等待数据。在这种情况下，当数据可用时，只有一个任务被解除阻塞。被解除阻塞的任务始终是等待数据的**最高优先级**任务。如果两个或多个被阻塞的任务具有相同的优先级，那么被解除阻塞的任务是**等待时间最长**的任务.
- 就像从队列读取时一样，任务在写入队列时也可以选择指定阻塞时间。在这种情况下，如果队列已满，阻塞时间就是任务处于阻塞状态以等待队列上**有可用空间的最长时间**