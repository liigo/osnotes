# Tock

# 一些有趣的术语

sandbox：用来形容各个基于 Rust 类型系统编写的 capsule 之间的某种类型安全

pointer integrity：应该也是形容从某种程度上保持指针类型的语义而不是仅仅是一个地址

# 论文阅读

## 1 Introduction

通用计算平台的硬件特征：特权级（阻止直接访问硬件）、MMU（虚拟地址空间，隔离）、较大的 RAM（可以动态分配内核数据结构，更有效利用内存）。然而在嵌入式平台成本和功耗受限，只能实现上述的部分特征：更简单的特权级、简化版的内存保护 MPU、以及很小（甚至只有 64KiB ）的 RAM。于是虽然**内存隔离**和**动态内存管理**在软件工程和性能（？）的角度更好，在嵌入式平台上也只能使用比较简单的编程模型。

比如：嵌入式平台上经常把应用和内核合成为一个单独的程序。优点：更容易共享数据，系统调用变为开销更小的函数调用。但是也有缺点：难以支持多任务。而且缺少了内存隔离难以保证整个系统的稳定性和应用的安全性。

嵌入式平台为了**长时运行**和**避免错误**经常使用静态内存分配。优点：避免了堆内存耗尽，尤其是在内存紧缺的情况极大提升了稳定性。缺点：需要预留最大并发情况下的内存，内存利用效率较低。而且为每种资源预留多少内存不好估计，还会随着具体应用的特征而发生变化。

而 Tock 的目标是在**嵌入式平台上更好支持多任务**，优点如下：

* 提供错误隔离和内核数据结构的动态分配。
* 使用类型安全的 Rust 编写，绝大部分代码通过编译器的检验。
* 为很多实际硬件提供了进程抽象。
* 为了避免内存利用效率和并发之间的 trade-off，使用了 **grant**。暂时看不懂这是个什么东西，后面再看。

## 2 Background & Motivation

曾经的嵌入式程序用途单一，应用本身决定一整套硬件、OS 配置和软件编写。但现在出现了新型的嵌入式软件平台支持多个独立且可以动态加载的应用。（比如说鸿蒙？大概就是需要有灵活可配置、相互隔离以保证安全的多个组件。）

### 2.1 Microcontrollers

MCU 资源受限：频率数十MHz/RAM数十KiB/1MiB甚至更少的闪存。受限于功耗，摩尔定律并不会突破这些资源限制。文中给出了相隔 13 年的两款 MCU 的配置参数，可以发现变化不大，比如 RAM 只是从 10KiB 增长到 64KiB。

32 位 Cortex-M CPU 提供了早期 MCU 没有提供的 MPU 内存管理，它既不是分页（虚拟地址）也不是分段（线性地址），直接访问物理地址，但是会在应用/应用和应用/内核之间提供内存保护。但是在 OS 中使用 MPU 进行内存保护好处有限。比如在 FreeRTOS 中使用 MPU 禁止应用写入内核的内存区域。但是 FreeRTOS 的 syscall 需要内核**信任应用传过来的任何指针**（糟糕的安全性），这样应用还是能通过 syscall 读写内存的任意位置。

### 2.2 Embedded Operating Systems

嵌入式操作系统主要需要提供五种关键特性：并发、资源耗尽的可靠性、错误隔离、内存有效利用、运行时应用更新。

**可靠性**：嵌入式应用非常注意稳定性，即使以性能、吞吐量、手动维护性为代价。比如 RAM 就经常静态分配或是只允许在启动的时候动态分配。

**并发**：嵌入式平台受功耗限制，经常需要提高并发，让很多操作在同一时间进行，因为 I/O 的时候处理器就可以低频休眠，更加**省电**。Ti nyOS 和 SOS 使用协作调度简化了栈管理（类似协程）。但是这样也有 CPU 资源饥饿的风险，所以 TOSThreads/FreeRTOS/RIOTOS 都有采用抢占式调度。

**效率**：特指更有效利用珍贵的 RAM 资源。无论是静态分配（TinyOS/Arduino）或是动态分配（SOS/TOSThreads）都需要考虑到这一点。

**错误隔离**：在各组件之间隔离错误是支持多任务的基础。但是直到最近嵌入式硬件的保护机制（如内存保护）都很不完善，因此只能依靠小心的 API 设计或者是功能受限的 MPU。甚至一些已有系统将应用运行在类似 JVM 的字节码解释器上来通过软件的方式进行错误隔离。

强大的 Tock 基于新的硬件特性和编程语言支持实现了所有特性！

## 3 Tock Architecture

Tock 有两类代码：capsule 和 process。其中 capsule 相当于内核中的组件，它们在编译期由一个语言沙盒（language sandbox）所限制，协作式调度（优点是能够减小上下文开销）； process 则类似其他 OS 中的进程，抢占式调度，并提供地址空间隔离，使用系统调用来获取服务。长时间运行的进程可以降低优先级来节能。

无论是 capsule 还是 process 都是在危机模型（threat model）的指导下设计的，优点：可以构造**相互不信任**的模块。

### 3.1 Threat Model

Tock 中的危机模型（threat model）提供了一种设计范式，而根据实际的应用场景有具体的设计。这是一种分层设计：四层软件，从下而上为 BSP 开发者、内核组件开发者、应用开发者、用户各自负责完整的系统的一部分功能，而且它们对其他层的信任度是不同的。

* BSP 开发者：将内核组件（如通用驱动、一些通信协议栈的实现）用平台相关的代码粘起来。它能够完全控制硬件并将一部分能力转交给内核组件。在危机模型中，它负责确定端对端的危机模型并基于它组织内核组件。
* 内核组件开发者：在 capsule 里面实现内核的大部分功能。BSP 开发者可以获取内核组件的源代码用于编译，同时它也并不会完全确认内核组件的安全性，需要某些额外机制对这些内核组件进行保护。比如，一个 capsule 可能饥饿 CPU 或者强制系统重启，但是它不允许破坏一些共享资源限制。
* 应用开发者：在 process 里面利用内核提供的功能开发应用程序。应用程序可能在出厂的时候板载，*也有可能在安装之后更新*。BSP 开发者不会审计应用代码，即使是应用开发者本身在实际部署之前也没办法确认安全性（存疑，如果也是用比较安全的编程语言？）。因此在 Tock 中认为应用程序是完全带有恶意的，需要防范它的攻击。
* 用户：用户可以在已部署的嵌入式系统上*安装、替换、更新*应用程序。用户被认为没有相关专业经验，且不一定能够理解运行时出现的错误。用户所能够带来的唯一一种风险是替换掉设备上的内核。

### 3.2 Capsules

capsule 是用 Rust 编写的内核组件。每个 capsule 是一个 Rust 结构体的实例，包含字段、方法、以及可以访问的全局变量。内核协作式调度 capsule，所有 capsule **共享一个栈**（类比协程），这样可以消除一些 capsule 边界（？）。但是，这也意味着 capsule 被信任不会破坏系统存活性和时序限制。

capsule 抽象提供了高内存利用率、可靠性、并发能力和内存隔离。但是它**不能动态加载**，同时也有**耗尽 CPU 资源的隐患**。

#### 3.2.1 Capsule 类型

capsule **不被信任**也不允许**破坏 Rust 类型系统**。Rust 类型和模块系统保证每个 capsule 不被允许访问**其他 capsule 的私有数据或者 process 地址空间**。

复用 capsule：多个由 OS 开发者和社区开发的 capsule 可以复用板上资源，它们是不被信任的，比如一些外设驱动或者是通信协议栈。它们是硬件相关的。

syscall capsule：接通应用 process 和内核接口，也是不被信任的。

另外一小部分 capsule 必须直接和硬件打交道，它使用大量的 unsafe，但是是被信任的。比如 MCU 的底层外设抽象将 mmio 转化为内存安全的结构体的过程。也包括内核的核心 capsule，比如 process scheduler。基于这些 capsule 会有更多功能复杂的 capsule，它们不必 unsafe，因此安全性是可以保证的。

#### 3.2.2 Capsule 隔离

capsule 主要靠 Rust 的类型和模块系统进行隔离。由于它在内核中使用非常广泛，需要说明的是它对于内存的高利用率和零成本抽象。举得例子是*无需对指针进行运行时检查*，感觉哪里不太对劲。

Rust 的完备的类型安全系统使得外部模块只能调用模块暴露出来的接口来访问对应的资源。比如，DMA 是一个经常破坏内核内存的家伙，因为 DMA 硬件能够跟内存中的任何位置打交道。在 Tock 中，用一个平台特定的 capsule 将 DMA 的设备寄存器放在一个结构体中进行管理。

```rust
struct DMAChannel {
    ...
    enabled: bool,
    buffer: &'static [u8],
}
```

这里将缓冲区表示为一个 Rust slice，在运行时可以进行缓冲区溢出检查。重要的是，传递 `&'static [u8]` 参数的调用者必须拥有所有权或者在编译器知晓的情况进行借用。

#### 3.2.3 Capsule 并发

内核的 capsule scheduler 是一个事件驱动的协作式调度，因此所有的 capsule 共享一个栈。事件的来源有两种：异步外部中断或者是应用 process 发起的 syscall。

**capsule 不能发起新的事件。**它们与内核剩下的部分通过普通控制流进行交互。这有两个优点：

1. capsule 之间的交互不需要通过 event scheduler，更加简单，可执行文件的容量更小；
2. 可以静态分配 event queue，减小运行时出错的风险。

### 3.3 processes

process 就像其他 OS 一样，独立的逻辑地址空间，独立的栈允许抢占式调度。**所有内核事件都比应用的优先级更高。**process scheduler 是最简单的 RR。process 之间打交道使用 IPC 机制，通过 syscall 与内核打交道。

与 Linux 等 OS 不同的是：首先 process **不支持虚拟内存**（硬件只支持物理地址），也不会使用共享库。其次是**所有 syscall 都是非阻塞的**。

process 相比 capsule 有两个优点：首先它们依赖硬件机制隔离而不是 Rust 类型系统提供的沙盒，所以它们可以**使用任何编程语言编写**，也更容易使用已有的各种库；其次它们是**抢占式调度，能够避免饥饿**。

这里需要简单介绍一下目前 MCU 的内存保护单元，它可以为 8 块下至 32 字节大小的内存区域设置 R/W/X 权限控制。*这可以用来进行 IPC*。对于设备寄存器直接设置权限会更容易，但对于需要间接访问的外设并不能直接这样做，而是需要良好设计的接口。

process 提供了所有五种特性：它可以被独立加载/替换；它支持并发；它在硬件支持下完成隔离；它们避免耗尽系统资源，因此它有隔离的地址空间和抢占式调度；而且事实上它们也能够高效利用内存。

### 3.4 syscall interface

tock 使用一套专为事件驱动设计的 syscall 接口，process 通过 5 个 syscall 的拓展来和内核打	交道。

* command 允许 process 向 capsule 传入整数参数并请求服务。比如配置 timer 或者开始访问总线。*参数不需要经过 kernel 的检查*
* allow 允许 process 向 capsule 传入一个数据缓冲区来保存更加复杂的数据。*kernel 检查缓冲区的开头和长度在 process 地址空间之内*，然后对应创建一个 Rust 的类型安全的结构体。**该结构体会在每次被使用之前检查源 process 是否还存在。**于是它常用来在 **process 和 capsule 之间共享内存**。
* subscribe 的参数是一个函数指针和一个用户数据指针，kernel 将它们包装成一个透明的 callback 结构体并*绑定到对应的 process*，然后传给 capsule。**capsule 可以调用在 process callback 调度队列中的一个 callback**。
* memop 和内核交互而不是 capsule，它的功能类似于 sbrk，可以调整 process 的堆空间。
* yield 和内核交互而不是 capsule，它**阻塞当前 process 直到它的 callback 队列不为空**。而且，只有在调用 yield 之后，callback 才会被处理。callback 类似于传统 Unix 中的 signal。

## 4 Grants

内核只有静态内存分配，应用需要动态内存的时候如何处理？现有的方法都有缺陷：伪动态，全局堆会造成应用互相影响等等

`Grant<T>` 是分配在堆上的数据结构，每个进程能够使用的总数有限。

一个进程与若干 capsules 相关，每个 capsule 里面初始都有 grant，但只有进程真正调用 enter 的时候才会从进程的限额内分配内存。

具体的数据结构必须被包裹 `Owned` 里面并在闭包里面使用。

`Grant` 提供了 each 操作来遍历已经被实际分配的数据结构。

1. 使用 MPU 保证进程无法访问它的限额部分内的内核数据结构
2. 要保证 capsule 只有在进程存活的时候才访问限额内的数据结构。首先就是确认 capsule 不会和进程并行，再就是依靠 Rust 的类型系统和和 Grant/Owned API 设计的好；
3. 进程退出后内核要正确回收内存。

内存分配：进程创建时内核分配一块内存，其中包括栈、用户堆还有用来存 Grants 的区域。后两者都可以增长，如果两头碰上了就说明内存用完了，可能杀死进程。

Grant region 相关回收则有两种可能：第一是 Owned 退出作用域；第二则是进程退出后，此时直接把相应内存块刷掉就行，后面的 capsule 访问里面的 grant 会失败的。

这导致内核对于每种状态都要对每个进程单独维护，一些操作就需要遍历所有进程，但是开销可以接受。

## 5 Implementation





