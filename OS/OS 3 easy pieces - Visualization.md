# Visualization

[TOC]

## Process

#### 分离策略和机制

即将**高层策略**与**底层实现**分开设计。

#### 进程创建

- 加载所有对应的代码和静态数据到进程对应的地址空间中。（lazily执行）
- 创建运行时栈，为运行时栈分配内存
- 必要的话，分配一定的堆内存
- 其他初始化任务（如I/O）

#### 进程状态

- 运行
- 堵塞
- 就绪

#### PCB

> 以xv6为例

- 上下文`context`（各种寄存器）
- 进程地址空间（初始位置与大小）
- 内核栈的栈底地址
- 进程状态
- pid
- 父进程
- chan, killed
- Open file
- 当前路径
- Trap frame

> 除了三大状态（三大状态的特点是会多次出现）还有**初始状态**（正在建立），**僵尸状态**（进程退出但未清理）。

#### API

- `fork`：创建子进程，执行当前父进程fork后的代码。

> 子进程并不是完整的拷贝父进程，其从fork返回的值是不同的。

```c++
#include <cstdio>
#include <unistd.h>

// rc == 0 => 当前进程就是子进程
// rc <  0 => 失败

int main()
{
    printf("Now --> pid:%d\n", (int)getpid());
    int rc = fork();
    if( rc < 0 ) // fork failed
        printf("fork failed\n");
    else if( rc == 0 ) // child process
        printf("--> Child: %d\n", (int) getpid());
    else
        printf("--> Father( %d ) of %d\n", (int) getpid(), rc);
}
```

- `wait`

```c++
#include <cstdio>
#include <unistd.h>
#include <sys/wait.h>

// rc == 0 => 当前进程就是子进程
// rc <  0 => 失败

int main()
{
    printf("Now --> pid:%d\n", (int)getpid());
    int rc = fork();
    if( rc < 0 ) // fork failed
        printf("fork failed\n");
    else if( rc == 0 ) // child process
        printf("--> Child: %d\n", (int) getpid());
    else
    {
        int wc = wait(NULL); // 等待子进程执行完毕。
        printf("--> Father( %d ) of %d\n", (int) getpid(), rc);
    }
}
```

- `exec`

> fork后，可调用exec去执行别的程序。这样不会产生新的子进程。
>
> 调用exec后，当前子进程的程序，静态数据，堆，栈等都会被覆盖（重新初始化），然后执行exec对应的程序。（使用exec之后，exec之后的程序都没了，因为当前进程被刷新了，其也会有返回值，但是成功调用的话不会返回）
>
> <u>对exec的成功调用永远都不会返回。</u>

```c++
#include <unistd.h>
#include <cstdio>
#include <sys/wait.h>

int main()
{
    auto fuck = fork();
    char* f[] =  {"echo", "balabala", nullptr};
    if(fuck == 0)
        execvp(f[0], f);
    else
        wait(NULL);
    printf("hello\n"); // 并不会输出2次
}
```

```c++
#include <unistd.h>
#include <cstdio>
#include <sys/wait.h>

int main()
{
    auto fuck = fork();
    char* f[] =  {"echo1", "balabala", nullptr}; // 错误
    if(fuck == 0)
        execvp(f[0], f);
    else
        wait(NULL);
    printf("hello\n"); // 输出2次
}
```

```c++
#include <cstdio>
#include <unistd.h>
#include <sys/wait.h>
#include <cstring>

// rc == 0 => 当前进程就是子进程
// rc <  0 => 失败

int main()
{
    printf("Now --> pid:%d\n", (int)getpid());
    int rc = fork();
    if( rc < 0 ) // fork failed
        printf("fork failed\n");
    else if( rc == 0 ) // child process
    {
        printf("--> Child: %d\n", (int) getpid());
        char* args[3] = {
                strdup("wc"),
                strdup("../util.hpp"),
                nullptr
        };
        execvp(args[0], args);
        printf("child is doing sth\n");
    }
    else
    {
        int wc = wait(NULL); // 等待子进程执行完毕。
        printf("--> Father( %d ) of %d\n", (int) getpid(), rc);
    }
}
```

#### Shell

- shell也是一个程序，我们通过传入shell命令，shell收到命令后fork再exec（的某个变体），再调用wait等待进程结束。
- **重定向的实现**：调用exec前关闭stdio，打开目标文件，然后输出到该文件上。

```c++
#include <fcntl.h>

// ...

close(STDOUT_FILENO);
open(output_file, O_CREAT | O_WRONLY | O_TRUNC, S_IRWXU);

// ...
```

- UNIX管道，使用了pipe()系统调用。一个**进程的输出**被链接到了一个内核管道上（队列），另外一个**进程的输入**也被链接到了一个内核管道上。然后相互match，以实现管道。`grep -r main ./* | wc -l`。

#### 受限直接执行

虚拟化机制的2大重点：

- 控制权（保证操作系统自生的控制权）
- 高性能（尽量接近zero-overhead）

> 直接运行协议（无限制）

|                              OS                              |          Program          |
| :----------------------------------------------------------: | :-----------------------: |
| 在进程列表上创建条目<br />为程序分配内存<br />加载程序和静态数据<br />根据argc/argv设置运行时栈<br />清空寄存器<br />call main |                           |
|                                                              | 执行main<br />return main |
|               释放进程内存，并从进程列表中移除               |                           |

> 直接运行的优点是快速，但这样无法进行受限操作（因为操作系统代码无法控制）。

> **受限操作**：
>
> 区分用户模式和内核模式：用户模式不能完全访问资源，而内核模式可。内核通过“系统调用”向用户代码提供一些特殊的受限功能。
>
> > 要执行系统调用，程序必须执行特殊的**<u>陷阱指令</u>**。
> >
> > - 调用系统调用 -> **陷入内核(trap)**
> > - 将特权级别提升为内核模式
> > - 返回 -> **陷阱返回(return-from-trap)**
> > - 降低特权级别
>
> > x86架构在进行系统调用的时候会把寄存器push到**内核栈**上，返回时再pop。
>
> > trap后，操作系统根据不同的异常码，找到对应的内核代码并执行。这个找的过程，靠的是陷阱表(trap table)，陷阱表由操作系统在开机初始化的时候对硬件进行配置，而跳转的过程是通过CPU的一些特殊指令完成的。

> **受限执行协议（LDE）**

![](https://i.loli.net/2019/09/21/MFwcDOXHepyGgfi.png)

> https://www.kernel.org/doc/html/latest/x86/kernel-stacks.html
>
> https://stackoverflow.com/questions/12911841/kernel-stack-and-user-space-stack
>
> > In a Linux system, every user process has 2 stacks, a user stack and a dedicated kernel stack for the process. The user stack resides in the user address space (first 3GB in 32-bit x86 arch.) and the kernel stack resides in the kernel address space (3GB-4GB in 32bit-x86) of the process.
> >
> > When a user process needs to execute some privileged instruction (a system call) it traps to kernel mode and the kernel executes it on behalf of the user process. **This execution takes place on the process' kernel stack.**
> >
> > The [**TSS (Task State Segment)**](https://en.wikipedia.org/wiki/Task_state_segment) is used to store the segment selector and stack pointer of the process' kernel stack.
> >
> > Upon a system call, the user process pushes all caller save registers in the process' user stack and executes [**int $0x80**](https://en.wikipedia.org/wiki/INT_(x86_instruction)) (0x80 is for system call**)** instruction. Then the **hardware** (not software) finds the process' kernel stack address from the TSS, loads those values to %**ss** (stack segment selector ) and pushes the old stack stack-pointer (**%esp**), old program-counter (**%eip**), old stack segment (**%ss**), code segment(**%cs**), and EFLAGS registers in the **process' kernel stack.** Hence, any operation requiring stack access uses this stack (not the user stack). After execution is over, the hardware pops the saved values from the kernel stack to their respective registers and resumes in the user stack.
> >
> > For a trap execution, the user stack can't be used because it might be corrupted by malicious user process. The kernel stack is used by kernel code only, so it is safe.
> >
> > X86有个叫做TaskStateSegment的硬件，保存了当前进程的Kstack的位置。在进入内核态的时候，硬件通过TSS找到内核栈（%ss），然后将ss & esp & eip & cs & eflags push到内核栈。

#### User stack & kernel stack

地址不一样，权限不一样。

- User stack在用户态
- Kernel stack在内核态
- 每个线程都有一个kernel stack
- syscall的时候，通过硬件，寄存器data将进行暂时的存储和恢复

#### 等待系统调用的协作方式

> 即OS相信程序会定期give up。
>
> - yield
> - I/O
> - 发生错误，陷入trap
> - 其他系统调用

#### 操作系统干预的非协作方式

依靠时钟中断，即运行一段时间（一般是几毫秒就中断一次）。中断的时候，OS就可以获得操控权了。（当然操作系统也可以关中断->特权操作）

#### 上下文切换

内核栈push&pop。

![](https://i.loli.net/2019/09/21/OZchYsWv8CnGKf3.png)

> 注意，寄存器被保存和恢复有2种方式：
>
> - 硬件隐式保存（目标地址是内核栈，由硬件实现）；
> - 操作系统显式保存（目标地址是PCB的内存，linux3.6内核代码`switch_to`中的注释表示）；
>
> > And，context是用于用户进程上下文切换，k-stack是用与内核态与用户态的切换。

```c
/*
 * Context-switching clobbers all registers, so we clobber      \
 * them explicitly, via unused output variables.                \
 * (EAX and EBP is not listed because EBP is saved/restored     \
 * explicitly for wchan access and EAX is the return value of   \
 * __switch_to())           
 */
```

> > 再次注意，每个进程有2种类型的stack，user-mode和kernel-mode，kernel-mode stack就是我们说内核栈。
> >
> > Each user thread has both a **user-mode** stack and a **kernel-mode** stack. When a thread enters the kernel, the current value of the <u>user-mode stack</u> (`SS:ESP`) and <u>instruction pointer</u> (`CS:EIP`) are saved to the thread's <u>kernel-mode stack,</u> and the CPU switches to the kernel-mode stack - with the `int $80` syscall mechanism, this is done by the CPU itself. <u>The remaining register values and flags are then also saved to the kernel stack.</u>
> >
> > When a thread context-switches, it calls into the scheduler (the scheduler does not run as a separate thread - <u>it always runs in the context of the current thread</u>). The scheduler code selects a process to run next, and calls the `switch_to()` function. This function essentially just switches the kernel stacks - it saves <u>the current value of the stack pointer into the TCB</u> for the current thread (called `struct task_struct` in Linux), and loads a previously-saved stack pointer from the TCB for the next thread. At this point it also saves and restores some other thread state that isn't usually used by the kernel - things like floating point/SSE registers. <u>If the threads being switched don't share the same virtual memory space (ie. they're in different processes), the page tables are also switched.</u>
> >
> > So you can see that the core user-mode state of a thread isn't saved and restored at context-switch time - it's saved and restored to the thread's kernel stack <u>when you enter and leave the kernel.</u> The context-switch code doesn't have to worry about clobbering the user-mode register values - those are already safely saved away in the kernel stack by that point.

#### 上下文切换开销

- 200-MHz Linux 1.3.37 ~ 系统调用 3μs 上下文切换6μs
- 内核不能被抢占式调度（抢占式的调度都是一个要返回用户态的进程在抢）

> 理论分析：
>
> - 寄存器的保存与恢复
> - Cache刷新
> - 流水线，分支预测结果刷新
> - TLB（线程级别可不考虑）

#### x86硬件支持

- IDT
- 将寄存器保存到内核栈
  - user-mode->kernel-mode: `EIP,CS; ESP,SS; EFLAGS`
  - Kernel-mode->user-mode: `EIP, CS; EFLAGS`

## 进程调度

在进行上下文切换的时候，如何选择即将要切换到的进程，涉及的就是进程调度的问题了。

> 调度问题2大aim：公平+性能；

#### 指标

- 希望尽早完成（性能）：周转时间=完成时间-到达时间
- 希望尽早被响应（公平+性能）：响应时间=首次运行时间-到达时间（适用于对时间敏感的程序）

#### 策略

- FIFO
- SJF（shortest job first）：最优，平均周转时间最短（同时到达进程由开始到被调度完的时间总合/被调度进程数量）（$T=Nt_1+(N-1)t_2+\cdots+t_N$）

> 缺点：非抢占式，若有新的短时间进程来了，还是会让老进程先执行。

- STCF（最短完成时间优先Shortest Time-to-Completion First）：抢占式。同样是最优的。
- RR(Round-Robin 轮转)：解决的是响应时间的问题。（对周转时间不友好，但符合公平性）
- Overlap I/O

#### 问题

上面很多策略是假设我们知道整个运转时间的长度，但实际上我们并无法知道…所以我们应该利用最近的策略去预测未来…（多级反馈队列）

## MLFQ 多级反馈队列

#### 设计初衷

- naive使用分时策略会导致**周转时间**不佳
- 使用其他的策略**响应时间**

> 嗯，就是拿来权衡周转时间和响应时间的。

#### 规则

1. 🌟 按优先级执行
2. 🌟 同优先级则轮转
3. 🌟 新任务从最高优先级开始
4. 🌟 使用完一个时间片则降低一次优先级
5. 🌟 因为其他原因（比如IO）而释放CPU则不改变优先级（对于交互型的进程，由于我们希望它被即使响应，所以我们希望它优先级高）

> 只要调度不是特别频繁，那么相对来说调度的开销就可以忽略不计。

#### 问题

- 恶意程序可以通过频繁IO来抢占优先级（愚弄问题）
- 如果“非交互”型进程突然变成“交互型”进程，那么响应的问题还是没有被解决

#### 改进：提升优先级

- 思路一：周期性提升所有工作的优先级（针对问题2）
- 🌟 经过一段时间（John Ousterhout称之为“巫毒常量(voo-doo constant)”），系统将所有工作重新加入最高优先队列。
- 思路二：更好的计时方式（解决愚弄问题）

> 要解决愚弄问题，我们首先得解除“发生IO不降低等级”的魔咒，然后再保证公平，那么久重新规定计时方式。

- 之前的计时：【调度时重新计时】给定时间片，时间片结束或发生中断重新计时。+中断不降级
- 新的计时：【每个任务对应一个时间】维护时间，只有当一个进程彻彻底底的用完一个时间片才降级。
- 🌟 一旦一个工作完成了其在**<u>某一层</u>**中的**时间配额**，那么就降低优先级。（针对问题一）

> 之前的时间片是包括IO等其他操作的，而新的时间配额是某一级指用于“running”的时间。

#### 配置

- 提升优先级的时间周期的设置
- 优先队列的层数
- （每一层）时间片大小

> 一般来说MLFQ越高优先级，时间片越短（响应快），越低则时间片越长。

#### Ousterhout定律

尽量避免巫毒常量（voo-doo constant）。

#### 实现

- Solaris: 表（配置），默认60层…，20ms~几百ms，每一秒刷新一次优先级。
- 其他：数学公式调整优先级。——FreeBSD，基于当前进程使用了多少CPU，然后有个计算优先级的公式。

#### 系统设计的big idea

- 使用hint，通过对hint的处理，来选择更优的策略；

#### 总结

- 初始都在最高优先级
- 同优先级使用时间片轮转
- 时间配额——记录**<u>某一级</u>**优先队列的**<u>真正的CPU run time</u>**
- 通过一段时间就刷新工作队列

> 没错他是反馈（自适应）的！！！

## 基于彩票机制的调度

> 这种机制可以用于“资源分配”的各个问题上。
>
> 主题：**<u>一切为了公平。</u>**

#### 目的

换一个目标函数：**<u>认为调度程序的最终目标，是确保每个工作获得一定比例的CPU时间。</u>**

#### How

每个进程分配一定量的彩票，按某一进程彩票占所有进程彩票的比例作为 **被调度的概率**。

需要的东西：

> - 随机数生成器
> - 进程列表

> **彩票抽奖系统的高效实现**：对于通过

#### 随机性是个好东西

既然聊到概率，那么就有随机数。随机数是个好东西，能够帮助系统从难以遇到的“麻烦的边角情况”中解救出来。

#### 更多特性

- 彩票转让（在server-client模型中可以通过转让提升两端性能）
- 彩票通胀（一个进程可以临时增加或减少自己的彩票数量）

#### Metric

> 对于两个同等级的任务：其完成**时刻**之比为我们说的不公平指标。（越靠近1越公平）

$$
U=\frac{T_1}{T_2}
$$

## 步长调度：更加公平

记录每个进程的行程，每个进程执行一次：`行程 += 步长`。

- 进程步长与彩票数成反比；
- 每次调度时，选择**<u>行程短</u>**的进程，将其加入调度队列的队首；

## 进程调度总结

- 彩票调度：不需要修改全局状态，只需要在加入进程的时候给彩票就好，然后就能做决定了；
- MLFQ：符合低周转时间，低响应时间的要求；
- 步长调度：公平；

> 彩票调度和步长调度都是试图保证比例符合预期。

## 地址空间

#### 上古时代

- 操作系统是个库，一台PC一个程序，直接用物理地址；

#### 因为昂贵所以共享

- 多道程序
- 时分共享（提高交互性）

> 那么内存呢？
>
> 当时是直接让一个进程独占一段时间的内存，其他的东西先保存到磁盘。问题就是效率太低了（使用的内存越多，切换效率越低）。
>
> 打破这点需要让 ***多个进程同时使用内存***。新的问题也来了，如何保护各个进程。

#### 地址空间

对物理内存的地址做一层抽象，得到虚拟化后的地址空间。

- 静态数据
- 代码
- 栈
- 堆

#### 空间虚拟化的目标

- 私有
- 尽可能大

> 其实现也是需要依托硬件；

#### 隔离

隔离是可靠system的一个关键原则。目的是减少个体之前的影响（进程A炸了，不会影响进程B）。

#### 空间虚拟化的其他目标

- 透明性

因为是透明的，所以进程要做到“相信相信操作系统的骗局”。（写一段打印地址的程序打印的是虚拟地址）

- 高性能（时间和空间）
- 保护隔离

#### GLIBC Malloc

为字符串分配内存的时候应该多分配一个`byte`。

## 地址转换

- 程序每次访问内存，硬件和操作系统都需要将虚拟地址转物理地址，反之亦然。

#### 动态重定位

> 通过MMU(Memory Management Unit)的**基址寄存器**（虚拟化）+**界限寄存器**（保护），来实现地址虚拟化。
>
> `physical addr = virtual addr + %base`

> 静态重定位，即在程序跑之前就定位，他的缺点是不提供保护（硬件层面的保护）。程序还是可以随意访问其他程序。而且静态定位很难“重定位”到其他位置。

> 界限寄存器的检查机制：
>
> - 算出物理地址之**前**就完成了检查；
> - 算出地址之**后**，拿着物理地址去compare检查。

#### 硬件支持

- **特权操作**：防止用户执行这一系列操作；
- **基址/界限寄存器**：per CPU
- **能够检查访问地址是否合法**
- **修改基址/界限寄存器的特权指令**：在用户程序开始之前，我们要能运行OS对其进行设置；
- **注册异常的特权指令**：这个指令告诉硬件，相关异常发生后，执行什么代码（Trap table）
- **越界/用户尝试使用特权指令时能直接触发异常**

#### 操作系统支持

- 创建进程的时候，要帮进程找到对应的地址空间（从free list里面找）
- 上下文切换的时候要恢复和存储基址寄存器和界限寄存器
- 当进程没有运行的时候，操作系统可以改变进程的地址空间copy them）

## 分段

#### 不分段的坏处

- 一个程序一坨地址空间会造成大块的“空闲”空间，不利于内存的利用

#### 分段咋分

首先有3个段：

- 代码段
- 栈
- 堆

每个段对应一对基址/界限寄存器（`*begin`&`size`）；

#### 段错误

简单来说，即当MMU将虚拟地址转换为物理地址的时候，发现不在自己对应的段中。

注意这些段在虚拟地址空间上是连续的，但在物理地址上不一定是连续的。

#### 段查找

> 即给我一个虚拟地址我该如何算出其对应哪个段；

- 用前几个bit来标识（VAX/VMS就这么干的）（比如我有3个段，我用2个bit，有的操作系统直接把栈和堆放一个段，这样只要1bit）
- 除了前几个bit，其他都是段偏移量；
- 非法情况：如果段的偏移地址大于对应的段长，那么就触发异常中断；

> 硬件也可以隐式的确定段：
>
> - 如果是取指，那么肯定是代码段；
> - 如果用到了基址指针和栈顶指针，那就是栈；

#### 增长方向标记

对于各个段的增长方向会有标记的（比如有的系统的栈是反向增长，Linux的堆是反向增长，会有个bit来记录的）

#### 段共享

比如有的程序可以共享代码段；

操作系统在额外的硬件支持的情况下，可以给每个段增加保护位，比如代码段的保护位表达的意思可以是：“读-执行”，其他段是“读-写”。

#### 段的粒度

早期的段，为了节省内存，都比较细粒度(fine-grained)，即分成百上千个段。现代操作系统一般都是粗粒度(coarse-grained)。

#### 操作系统支持

- 上下文切换要保存段寄存器；
- 管理空闲内存空间，若不管理，会造成很多的碎片（就是有很多小内存，因为他小所以他不能被用来分配一个段，也不方便扩充已有段，但它很多，就是说如果我管理的好的话，他可以汇聚成big chunk of memory.）
- 外部碎片解决方案：
  - compact（紧凑重排）
  - 最优匹配
  - 最坏匹配
  - 首次匹配
  - 下次匹配(在我们学校课上将其称之为**循环首次适应算法**)
  - 伙伴算法

> 如果一个问题有很多种解决方案，说明这个问题没有特别好的解决方案。

## 内存管理（*）

- 主存空间管理
- 地址变换（基址寄存器和界限寄存器管理段，同时负责内存保护）
- 内存扩充
  - 交换：暂存到外部存储器中；
  - 覆盖：共享主存段；
- 内存共享与保护

## <u>连续</u>内存分配（*）

- 单一连续分配：单用户、单任务naive
- 固定分区分配：对每个进程分**固定**大小的空间（不变）
- 可变分区分配：
  - 最佳匹配
  - 最坏匹配
  - 首次适应
  - 下次适应
- 可重定位分区分配：整个虚拟地址空间对应的物理地址发生了改变

## 空闲空间管理

空闲内存空间的管理可以分为两个部分，一个部分是程序内部对自身堆内存的管理，另外一个部分是操作系统自身对于段的管理。这里主要讨论程序内部的堆管理。

#### 目标

- 满足内存申请的请求
- 节省空间
- 降低时间开销

> 管理堆上内存空间的数据结构一般被称之为“空闲链表”(Free List)。当程序的堆内存不足的时候，用户程序可以通过`sbrk`这样的系统调用来向内核申请增加堆内存，操作系统会对其分配一个新的物理内存页，并将其映射到进程的地址空间。

#### 底层机制

- **Splitting分割**：对于一个内存node，其`node_size`大于我想要申请的`alloc_size`，那么就将其分割成`alloc_size`和`node_size-alloc_size`，前者返回给用户，后者继续留在`free_list`中。
- **Coalescing合并**：调用`free`归还内存的时候，检查一下地址空间上连续的node是否也是free的，如果是的则合并（得到更大的node）。

#### 追踪已分配的内存

我们在free的时候不需要告诉系统要free的大小，因为在我们调用malloc的时候，已经额外增加了一个`header_t{ int size; int magic; }`，首先检查一下magic是不是某个事先约定好的值（来检查这块地方是不是`header_t`，比如`assert(hptr->magic == 1234567)`），如果是那么就在当前位置free size大小的内存。

#### 空闲链表的表结构

> 切记，空闲链表就是个链表，上面存着**“不连续”**（假设连续的空闲内存我们已经合并好了）的空闲内存记载块。那么每个节点要记录的东西很简单：
>
> - 内存大小`size`
> - 下一个节点位置`node_t* next`

那么对于不空闲的节点我们也给他加上一个表头`header_t`（上面提到的）。当一个节点被归还的时候，那么就先尝试合并，然后将`header_t`改成`node_t`。对于一个申请（要加上表头的大小），那么就是找到足够大的node，然后split。

## 匹配策略

#### 一般但通用的策略

- 最优匹配：找到满足大小的最小块
  - 需要遍历整个list
- 最差匹配：找到满足大小的最大块
  - 其初衷是为了让块尽量都比较大，但实践效果不好
  - 需要遍历整个list，效率不好
- 首次匹配：
  - 效率好
  - 但容易造成list开头一堆碎片
  - 可以通过对空闲块地址排序，让内存的申请和归还集中在一块区域，也利于合并操作
- 下次匹配：在首次匹配的基础上，保存上次匹配到的指针
  - 避免造成list头部一堆碎片而使整个遍历变得低效的情况
  - 同时这样做本身也能减少内存碎片

#### 分离空闲链表

其实就对象池，slab allocator就这么干的。（因为之前的问题主要是有内存block大小不一致导致的，如果block大小一致就没这个问题了）。然后slab的回收通过引用计数计数来实现。

#### 伙伴系统

> 用于更加方便的进行merge。

以二分伙伴系统为例子，对于一个大的内存块（大小为$2^N$bytes），不断二分直至恰好能cover请求（给出去的块也是$2^K$bytes，所以有可能会有内部碎片）。然后其释放的时候，尝试去merge隔壁的伙伴块。

## 其他

#### 优化

可以用其他数据结构来优化查找块的开销：

- 平衡二叉树
- 偏展树
- 偏序树

除此之外还需要考虑多线程情况下的内存分配问题。

#### Malloc&Free

这俩并不是系统调用，而是GLIBC这样的库里的函数。但其可能会去call系统调用，如`sbrk`，`mmap`等。

## 分页

- 页号 ~ 虚拟地址（32位和64位不同）
- 页框号 ~ 物理地址（看硬件）

#### PTE

- **32位**：`(20 | 12)` 4KB的page
- **有效位**：是否有效，如果无效则不会对其分配物理页；
- **存在位**：看其是否在内存/交换区；
- **保护位**： 即不允许访问的PTE，若访问则会陷入操作系统；
- **脏位**：淘汰之前要写回内存；

## TLB

- CISC: 主要是Hardware
- RISC: 可以Software
- **重点区分**：TLB有效位 `!=` 页表有效位：页表（index表示位置）因为代表的是整个表的内容，each address counts，如果页表的有效位是0，那么说明这个页没有被申请。而TLB是cache，只有被用到的页表才会被装入TLB，所以其有效位表示当前地址是否有效（若无效，则说明那个地址不具有实际意义，不能保证访问的正确性）

> 主要看看上下文切换时TLB的行为：
>
> // TODO.