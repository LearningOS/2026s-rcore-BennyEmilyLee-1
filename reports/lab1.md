## sys_trace 实现总结

sys_trace（系统调用号41e）有三个功能：
1.trace_request=0：读取用户虚拟地址处的1字节数据，id参数作为只读指针使用
2.trace_request=1：写入数据到用户虚拟地址，只写入数据的最低字节，id参数作为可写指针使用
3.trace_request=2：查询指定系统调用的调用次数。返回调用次数后将该计数+1，实现"本次调用也计入统计"实现要点：在TaskControlBlock中维护syscall_counts512数组记录每个系统调用的调用次数。非trace系统调用在syscall()入口处计数，trace系统调用本身在sys_trace内部处理计数。特别地，trace_request=2
会先返回旧值再+1，使测试可验证"本次调用也计入"的语义同时返回正确的旧值。

## 简答作业

1. 正确进入 U 态后，程序的特征还应有：使用 S 态特权指令，访问 S 态寄存器后会报错。 请同学们可以自行测试这些内容（运行 三个 bad 测例 (ch2b_bad_*.rs) ）， 描述程序出错行为，同时注意注明你使用的 sbi 及其版本。
回答：
使用的sbi版本如下：
```bash
[rustsbi] RustSBI version 0.3.0-alpha.2, adapting to RISC-V SBI v1.0.0
```

测例1 
ch2b_bad_address.rs的出错信息：

```bash
[kernel] PageFault in application, bad addr = 0x0, bad instruction = 0x804003a4, kernel killed it.
```

这个出错信息显示有2个错误：
a. ch2b_bad_address程序产生PageFault的错误，要访问内存地址是0x0，该地址是S态特权的地址范围，而不是程序能访问的地址范围，当CPU检测到这种非法内存访问时，会触发页访问异常，操作系统内核随后会终止该应用程序。
b. 同时，PageFault误发生在指令地址0x804003a4，这对应于代码中的write_volatile(0)操作，尝试写入一个不在该程序内存范围（或者是受保护的内存范围）的值时，就会产生这种异常。

测例2
ch2b_bad_instructions.rs的出错信息：
```bash
[kernel] IllegalInstruction in application, kernel killed it.
```

这个出错信息显示有如下错误：
ch2b_bad_instructions程序产生了IllegalInstruction错误，表明程序做了一个非法的指令，对应于代码中是执行了一个`sret`汇编指令，该指令是从S态返回到U态，因为该程序的执行完全是在U态下，无法从S态返回到U态，所以程序出现了这个错误。


测例3
ch2b_bad_register.rs的出错信息：
```bash
[kernel] IllegalInstruction in application, kernel killed it.
```

这个出错信息显示有如下错误：
ch2b_bad_instructions程序产生了IllegalInstruction错误，表明程序做了一个非法的指令。应用程序试图在U态下读取特权寄存器`sstatus`。`sstatus`属于RISC-V架构中的‌特权寄存器‌，只能在M模式(Machine Mode)‌或S模式(Supervisor Mode)‌下访问。当CPU在U模式下执行csrr指令读取该寄存器时，会触发非法指令异常。

2. 深入理解 trap.S 中两个函数 __alltraps 和 __restore 的作用，并回答如下问题:

2.1 L40：刚进入 __restore 时，sp 代表了什么值。请指出 __restore 的两种使用情景。
回答： sp 指向内核栈上的 TrapContext
__restore 的两种使用情景
a1. 首次启动用户程序
a2. 中断处理完毕后返回用户态

2.2 L43-L48：这几行汇编代码特殊处理了哪些寄存器？这些寄存器的的值对于进入用户态有何意义？请分别解释。
回答：

```s
ld t0, 32*8(sp)
ld t1, 33*8(sp)
ld t2, 2*8(sp)
csrw sstatus, t0
csrw sepc, t1
csrw sscratch, t2
```

这几行汇编代码特殊处理了sstatus、sepc、sscratch寄存器，其中三个`ld`的作用是从 TrapContext 恢复 sstatus、sepc 和用户栈指针。三个`csrw`的作用是将恢复的值写回 CSR 寄存器。

3. L50-L56：为何跳过了 x2 和 x4？
回答：
因为 x2 对应的是sp（stack pointer），x4对应的是tp（thread pointer），这两个pointer不用保存，在后续的程序中使用。

4. L60：该指令之后，sp 和 sscratch 中的值分别有什么意义？
回答：
sp 表示用户栈指针；sscratch 表示内核栈指针（为下次 trap 准备）。

5. __restore：中发生状态切换在哪一条指令？为何该指令执行之后会进入用户态？
回答：
__restore：中发生状态切换在 `sret` 指令。因为 `sret` 跳转到 sepc 指向的用户地址，恢复用户态执行。

6. L13：该指令之后，sp 和 sscratch 中的值分别有什么意义？
回答：
L13：该指令之后，sp 表示内核栈指针，sscratch 表示用户栈指针。

7. 从 U 态进入 S 态是哪一条指令发生的？
回答：
从 U 态进入 S 态是以下指令：

```s
csrrw sp, sscratch, sp
```

## 荣誉准则

1. 在完成本次实验的过程（含此前学习的过程）中，我曾分别与 以下各位 就（与本次实验相关的）以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：

《你交流的对象说明》

2. 此外，我也参考了 以下资料 ，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：

《你参考的资料说明》

3. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。 我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

4. 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。 我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。