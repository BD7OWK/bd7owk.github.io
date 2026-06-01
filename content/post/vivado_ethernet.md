---
title: 关于Vivado ZYNQ FPGA Eternet 报错：Unconstrained Logical Port的解决方案
cover: zynq_processing.png
description: 自用整理，若有错误请指正
draft: false
date: 2026-05-10T20:12:52+08:00
lastmod: 2026-05-17T20:12:52+08:00
weight: 0

---
## 关于Vivado ZYNQ FPGA Ethernet 报错：Unconstrained Logical Port的解决尝试

使用Vivado 2018.3

AXI 1G/2.5G Ethernet Subsystem IP核，MII模式下

Ethernet芯片：RTL8201F

开发板：微相科技ZYNQ7010 CLG400

### 起因

由于微相科技开发板使用了RTL8201F，没有提供`mii_rx_er`和`mii_tx_er`的连接：

![schematic](schematic.png "images")

*RXER网络并未发现其他连接*

我便打算不为`mii_rx_er`和`mii_tx_er` make external：

![block](block.png "images")

### 报错

在implementation时报错如下：

![error](error.png "images")

发现是由`mii_rx_er_0`和`mii_tx_er_0`引起的。

结合报错信息，发现是vivado自动为这两个信号make external，而引脚约束文件中又没有这两个信号的引脚约束：

```verilog
set_property IOSTANDARD LVCMOS33 [get_ports {reset_rtl_0[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports sy_clk]
set_property IOSTANDARD LVCMOS33 [get_ports mdio_rtl_0_mdc]
set_property IOSTANDARD LVCMOS33 [get_ports mdio_rtl_0_mdio_io]

set_property IOSTANDARD LVCMOS33 [get_ports mii_tx_en_0]
set_property IOSTANDARD LVCMOS33 [get_ports {mii_txd_0[3]}]
set_property IOSTANDARD LVCMOS33 [get_ports {mii_txd_0[2]}]
set_property IOSTANDARD LVCMOS33 [get_ports {mii_txd_0[1]}]
set_property IOSTANDARD LVCMOS33 [get_ports {mii_txd_0[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports {mii_rxd_0[3]}]
set_property IOSTANDARD LVCMOS33 [get_ports {mii_rxd_0[2]}]
set_property IOSTANDARD LVCMOS33 [get_ports {mii_rxd_0[1]}]
set_property IOSTANDARD LVCMOS33 [get_ports {mii_rxd_0[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports mii_rx_dv_0]
set_property IOSTANDARD LVCMOS33 [get_ports mii_rx_clk_0]
set_property IOSTANDARD LVCMOS33 [get_ports mii_tx_clk_0]
set_property PACKAGE_PIN N16 [get_ports mii_tx_en_0]
set_property PACKAGE_PIN L14 [get_ports mii_tx_clk_0]
set_property PACKAGE_PIN K18 [get_ports mii_rx_dv_0]
set_property PACKAGE_PIN K17 [get_ports mii_rx_clk_0]
set_property PACKAGE_PIN J14 [get_ports {mii_rxd_0[0]}]
set_property PACKAGE_PIN K14 [get_ports {mii_rxd_0[1]}]
set_property PACKAGE_PIN M18 [get_ports {mii_rxd_0[2]}]
set_property PACKAGE_PIN M17 [get_ports {mii_rxd_0[3]}]
set_property PACKAGE_PIN M14 [get_ports {mii_txd_0[0]}]
set_property PACKAGE_PIN L15 [get_ports {mii_txd_0[1]}]
set_property PACKAGE_PIN M15 [get_ports {mii_txd_0[2]}]
set_property PACKAGE_PIN N15 [get_ports {mii_txd_0[3]}]
set_property PACKAGE_PIN G14 [get_ports mdio_rtl_0_mdc]
set_property PACKAGE_PIN J15 [get_ports mdio_rtl_0_mdio_io]
set_property PACKAGE_PIN H20 [get_ports {reset_rtl_0[0]}]
set_property PACKAGE_PIN N18 [get_ports sy_clk]

set_property SLEW FAST [get_ports mdio_rtl_0_mdc]
set_property SLEW FAST [get_ports  mdio_rtl_0_mdio_io]
set_property SLEW SLOW [get_ports {reset_rtl_0[0]}]
set_property SLEW FAST [get_ports mii_tx_en_0]
set_property SLEW FAST [get_ports {mii_txd_0[3]}]
set_property SLEW FAST [get_ports {mii_txd_0[2]}]
set_property SLEW FAST [get_ports {mii_txd_0[1]}]
set_property SLEW FAST [get_ports {mii_txd_0[0]}]
```

原本尝试了将这两个信号绑定至同一bank的空闲gpio，虽能进行implementation和bitstream generation却发现进系统网络测试时ping不通。用wireshark显示主机可以收到来自zynq的ping并回复，但zynq没有收到。

重新阅读手册发现这两个信号控制着错误指示和错误使能：

![er](er.png "images")

让这两个信号悬空可能会导致ethernet误判接收到的包是错误的（`mii_rx_er`为高电平），并将其直接丢弃。

### 解决方案

手动修改HDL wrapper，将模块定义和端口类型声明中关于`mii_tx_er_0`和`mii_rx_er_0`的内容删去，并在端口数据类型声明后加上：

```verilog
assign mii_rx_er_0 = 1'b0;
assign mii_tx_er_0 = 1'b0;
```

接着，来到约束文件，添加：

```verilog
set_property IOSTANDARD LVCMOS33 [get_ports mii_rx_er_0]
set_property IOSTANDARD LVCMOS33 [get_ports mii_tx_er_0]
```

将这两个信号拉地，成功通过implementation和bitstream generation，编译下载到开发板成功ping通。
