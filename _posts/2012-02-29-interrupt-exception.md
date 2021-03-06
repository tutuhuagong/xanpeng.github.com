---
title: Linux 中断和异常
layout: post
tags: linux interrupt exception
category: linux
---

#概念
同步(synchronous)中断: 指令执行时由 CPU 控制单元产生, 只有在一条指令终止执行后 CPU 才会发出中断.  
异步(asynchronous)中断: 由其他硬件设备依照 CPU 时钟信号随机产生的.  
在 Intel 微处理手册中, 把同步和异步中断分别称为**异常(exception)**和**中断(interrupt)**, 有时术语 "中断信号" 指这两种类型(同步和异步).

中断:

* 可屏蔽中断(maskable interrupt): 可被控制单元忽略.
* 非可屏蔽中断(nonmaskable interrupt): 只被几个危急事件(如硬件故障)引起. 非可屏蔽中断总是由 CPU 辨认.

异常:

* 处理器探测异常(processor-detected exception)
    * 故障(fault): 通常可以纠正, 从而不失连贯性地继续执行. 保存在内核态堆栈 eip 寄存器中的值是引起故障的指令地址.
    * 陷阱(trap): 保存在 eip 中的是一个随后要执行的指令地址. 只有当没有必要重新执行已终止的指令时, 才触发陷阱. 陷阱的主要用途是为了调试程序, 在这种情况下, 中断信号的作用是通知调试程序一条特殊指令已被执行(例如到了一个程序内的断点).
    * 异常终止(abort): 发生一个严重的错误, CPU 控制单元出了问题, 不能在 eip 寄存器中保存引起异常的指令的确切位置.
* 编程异常(programmed exception): 在编程者发出请求时发生. 由 int 或 int3 指令触发. 当 int(检查溢出)和 bound(检查地址出界)指令检查的条件不为真时, 也引起编程异常. 控制单元把编程异常当成陷阱来处理. 编程异常通常也叫**软中断(software interrupt)**, 这样的异常有两种常用的用途: **执行系统调用**, **给调试程序通报一个特定的事件**.

**中断处理**是内核执行的最敏感任务之一, 需满足约束:

* 中断要尽可能快地处理完. 内核响应中断后的操作需分两部分: 关键而紧急的部分立即执行; 其余部分推迟执行.
* 中断随时会来. 中断处理程序需考虑这种情况, 处理前一个中断时可以接受一个新的中断.
* 内核代码中还是存在一些临界区, 在临界区中, 中断必须被禁止. 但这种临界区的数目必须尽可能地被限制.

每个中断和异常是由0~255之间的一个数来标志, Intel 称这个8位无符号整数为**向量(vector)**. 非可屏蔽中断的向量和异常的向量是固定的, 可屏蔽中断的向量可以通过对中断控制器的编程来改变.

#中断和异常处理程序的嵌套执行
![alt kernel control path recursively executing](http://www.oreilly.de/catalog/9780596005658/figs/d1e16825-web.png)

# ULK "中断和异常" TOC:
* 中断信号的作用
* 中断和异常
    * IRQ 和中断
    * 高级可编程中断控制器
    * 异常
    * 中断描述符表
    * 中断和异常的硬件处理
* 中断和异常处理程序的嵌套执行
* 初始化中断描述符表
    * 中断门, 陷阱门及系统门
    * IDT 的初步初始化
* 异常处理
    * 为异常处理程序保存寄存器的值
    * 进入和离开异常处理程序
* 中断处理
    * IO 中断处理
    * 处理器间中断处理
* 软中断及 tasklet
* 工作队列
* 从中断和异常返回

#参考资料
* ULK 第三版, 第四章"中断和异常"
