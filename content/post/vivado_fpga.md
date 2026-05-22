---
title: Xilinx Vivado FPGA的配置笔记
cover: zynq_processing.png
description: 自用整理，若有错误请指正
draft: false
date: 2026-05-10T20:12:52+08:00
lastmod: 2026-05-17T20:12:52+08:00
weight: 0

---
## Vivado FPGA 的配置笔记

### ZYNQ Processing system

根据项目需求和开发板设置：

#### PS-PL Configuration

PS与PL端间的通信和控制

设置时钟重置和和AXI交互的使能

- UART baud rate: 115200

为保证PL工作默认使能的（不使用PL则关闭）：

- Enable Clock Reset -> FCLK_ RESET0 _N （时钟rst）

AXI接口：

- AXI Non Secure Enablement -> GP Master AXI Interface -> M AXI GP0 interface（PL作为从机与PS进行交互 - 通用目的）
- AXI Non Secure Enablement -> HP Slave AXI Interface -> S AXI HP0 interface（PL作为主机与PS进行交互- 高性能）

#### Peripheral I/O Pins

按照电原理图上的端口和需要使能各外设和IO（UART、MIO、CAN、I2C等），以及复用IO口

#### MIO Configuration

各复用IO的信号使能，需要什么启用什么

- SD卡使用LVCMOS18，推挽输出GPIO及UART使用LVCMOS33
- SD端口速度为fast，UART及GPIO为slow

#### Clock Configuration

配置时钟频率及使能

为保证PL工作默认使能的（不使用PL则关闭）：

- PL Fabric Clocks -> FCLK_CLK0

#### DDR Configuration

根据板子上的内存选择内存种类、**内存型号**、**DRAM总线宽度**等

#### Interrupts

PL向PS所发送的中断信号、使用的中断端口的使能

接收来自 PL的中断信号：Fabric Inturrupts -> PL-PS Inutrrupt Ports -> IRQ_F2P[15:0]
