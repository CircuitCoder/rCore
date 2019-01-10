# 2018操作系统专题训练

# 最终报告

计53 王润基 2015011279

2018.12.27

## 本学期工作总结

* 实现各平台多核启动和调度
* 完善文件系统相关功能
* 实现NoMMU，RISCV M-Mode，RV64，K210支持

## 后半学期任务报告

### NoMMU

#### 原理

* 去掉内存分页映射机制，将用户程序加载到任意地址运行
* 内核为剩余物理内存空间建立另一个堆分配器（原来是用作物理帧）
* 当加载用户程序时，分配一段连续内存空间，并计算新的入口地址

#### 位置无关代码

- 用户程序需使用**相对寻址**：
  - 代码段使用相对跳转
  - 数据段使用相对寻址，或去GOT（全局偏移表）查询地址
- 为此需添加编译选项：
  - 编译时`-fPIC`（位置无关代码）
  - 链接时`-pie`（位置无关可执行程序）
- 原来ucore编译出的用户程序：x86绝对寻址，RISCV相对寻址

有待完善的地方：

* 实现GOT表的填写，即可支持代码段和数据段分别分配地址

### RISCV M-Mode

将现有的运行于S态的kernel移植到M态。

BBL为kernel提供了很多底层功能，因此需要保留。

#### 具体任务

* 去掉分页机制（NoMMU）
* 让BBL保持M态进入kernel
* 将kernel中操作的S-CSR改为M-CSR
* 融合kernel和BBL的运行环境

#### BBL提供的功能

* SBI：getchar，putchar，IPI
* 时间与时钟中断：模拟rdtime，提供set_timer()
* CPU不支持的指令模拟：乘除、浮点等

#### 融合kernel和BBL

* 中断：原来kernel和BBL各有一套中断处理流程，融合后以kernel的为主体
* SBI：BBL将相关函数符号传给kernel（BBL的角色变为静态库）
* 委托中断处理：当发生`IllegalInst`和`MachineEcall`异常时，调用BBL的函数处理

#### 问题

* QEMU中`mie.MEIE`（M态外设中断）不可开启，可能是个Bug。导致串口中断不工作，只能使用轮询方式getchar。
* M态BBL是否还有存在的意义？
  * RV下boot十分简单
  * BBL依赖FDT（设备树）初始化外设，但一般只有M态的开发版都不提供FDT。

### RV64 & K210

* 基于dzy同学的两阶段编译流程：Rust=>LLVM-IR=>Machine

* 12月17日LLVM master已经完善地支持rv64，RustOS完整编译通过
* 此后配合dzy完成rv64移植

#### 地址空间处理

* llc不支持code-model=medium，即位于任意地址4G范围内
* 对此张蔚同学做了一个patch，但编译RustOS仍有问题，因此没用上
* 于是目前只能将代码和数据放在+-2G区域，即[0,0x7fffffff], [0xffffffff_80000000, 0xffffffff_ffffffff]
* 在QEMU/HiFiveU上，支持S态、有页机制。因此将地址放在0xffffffff_80000000。需在BBL中设置初始页表并建立好映射，然后再进入S态Kernel
* K210上RAM位于0x80000000，但也可以通过0x40000000访问（无cache）。因此将地址放在0x40000000。

#### K210适配

* 主要参考官方SDK
* 串口：
  * 使用UARTHS，和普通UART规范不一致
  * 在boot阶段就初始化好，方便调试
  * TODO：串口事件中断
* 原子操作：
  - 原子指令对0x40000000地址区无效（？？？）
  - 解决方法：在自己实现的`__atomic_*`函数中将地址移到0x80000000处。
* 多核：
  * reset后同时开始运行，不需额外boot
  * 目前只用了单核，因为还有Bug……
* 时钟中断：
  * SDK中的处理较为复杂，懒得重新造轮子
  * 将SDK编译为静态库，链接到kernel，kernel导入符号使用
* 修复前期M-Mode的Bug

#### 搞K210的困难

* 没有完整的模拟器（其实部分也够了）
* 烧程序慢，导致实验验证周期长（提高波特率：2min->0.5min）

### 其它

* 适配 Rust 2018 edition
* 实现文件内存映射，用户程序内存均延迟分配，以后可实现mmap
