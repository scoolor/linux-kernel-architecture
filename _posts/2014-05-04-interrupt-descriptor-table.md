---
layout:    post
title:     中断描述符表
category:  中断和异常
description: 中断描述符表...
tags: IDT 中断描述符表
---
中断描述符表（*Interrupt Descriptor Table，IDT*）是一个系统表，它与每一个中断或异常向量相联系，每一个向量在表中有相应地中断或异常处理程序地入口地址，内核在允许中断发生前，必须适当地初始化IDT。

GDT和IDT地格式非常相似，表中地每一项对应一个中断或异常向量，每个向量由8个字节组成，因此，最多需要256x8=2048字节来存放IDT。*idtr*CPU寄存器使IDT可以位于内存地任何地方，它指定IDT地线性地址及其限制地最大长度。在允许中断之前，必须用*lidt*汇编指令初始化*idtr*

IDT包含三种类型地描述符：

{:.center}
![IDT](/linux-kernel-architecture/images/idt.png){:style="max-width:700px"}

{:.center}
中断描述符表

这些描述符是：

1. 任务门（*task gate*），当中断信号发生时，必须取代当前进程地那个进程地TSS选择符存放在任务门中。
2. 中断门（*interrupt gate*），包含段选择符和中断或异常处理程序地段内偏移量，当控制权转移到一个适当地段时，处理器清IF标志，从而关闭将来会发生的可屏蔽中断。
3. 陷阱门（*trap gate*），和中断门类似，只是控制权传递到一个适当地段处理器不修改IF标志。

### 中断和异常的硬件处理 ###

假定内核已经被初始化，因此，CPU在保护模式下运行，Linux只有在刚刚启动的时候是在实模式，之后便进入保护模式。

在执行了一条指令之后，*cs*和*eip*这对寄存器包含下一条将要执行的指令的逻辑地址，在处理了那条指令之后，控制单元会检查在运行前一条指令时是否已经发生了一个中断异或异常[^1]。如果发生了一个中断或者异常，那么控制单元执行下列操作：

[^1]: 汇编里通常一条指令执行结束后才会产生一个异常。

1. 确定与中断或异常的关联向量i。
2. 读由*idtr*寄存器指向的IDT表中的第i项门描述符。
3. 从*gdtr*寄存器获得GDT的基地址，并在GDT中查找，以读取IDT表项中的选择符所标识的段描述符，这个描述符指定只哦你果断或异常处理程序所在的段的基地址。
4. 确定中断是由授权的中断发生源发出的。
5. 检查是否发生了特权等级变化。
6. 如果故障已经发生，用引起异常的指令地址装载*cs*和*eip*寄存器[^2]，从而使这条指令能够再次被执行。
7. 在栈中保存*eflags*、*cs*以及*eip*的内容。
8. 如果异常产生了一个硬件出错码，则保存在栈中。
9. 装载*cs*和*eip*寄存器，其值分别是IDT表中的第i项门描述符的段选择符和偏移量，这些值给出了中断或者异常处理程序的第一条指令的逻辑地址。

[^2]: CPU执行的指令跟*cs*和*eip*有关，CPU会读取这eip寄存器的内容并找到cs寄存器里相应的地址，然后执行指令。

控制单元所执行的最后一步就是跳转到中断或者异常处理程序，也就是说，醋栗完中断信号后，控制单元所执行的指令就是被选中处理程序的第一条指令。

中断或异常被处理完毕后，相应的处理程序必须产生一条*iret*指令，把控制权转交给被中断的进程，这样控制单元就会产生以下操作：

1. 用保存在栈中的值装载*cs*、*eip*或*eflags*寄存器，如果一个硬件出错码曾被押入栈中，并且在*eip*内容上面，那么执行*iret*指令必须先弹出这个硬件出错码。
2. 检查处理程序的CPL是否等于*cs*中的最低两位的值，如果是，说明在同一特权级，*iret*中止执行，否则转入下一步。
3. 从栈中装载*ss*和*esp*寄存器，返回到与旧特权级相关的栈。
4. 检查*ds*、*es*、*fs*以及*gs*段寄存器的内容，如果其中一个寄存器包含的选择符是个段描述符，并且其DPL值小于CPL，那么就清除相应的段寄存器。

### 初始化中断描述符表 ###

内核启用中断之前，必须把IDT表的初始地址装到*idtr*寄存器，并初始化表中的每一个项，这项工作是在初始化系统时完成的。

*int*指令允许用户态进程发出一个中断信号，其值可以时0～255的任意一个向量。因此，为了防止用户通过*int*指令模拟非法的中断和异常，IDT的初始化必须非常小心，因此可以通过把中断和陷进门描述符的DPL字段设置为0来实现。

如果进程试图发出其中的一个中断信号，控制单元将检查出CPL的值与DPL字段有冲突，并且产生一个『General protection』异常。

在一些少数的情况下，用户态进程必须能够发出一个编程异常，为此，只要把中断或陷阱门描述符的DPL字段设置为3，即特权级尽可能高一点就可以了。