---
title: "DWC3 等时传输的事件与中断体系：DEPEVT、DEVT、Bus Expiry"
date: 2026-08-16
tags: [usb, dwc3, isochronous, interrupt, depcmd]
status: notes
---

## 背景

做 udc_dwc3.c 时需要决定开哪些中断、每个事件处理什么。梳理 DWC3 的两层中断
（端点级 DEPEVT + 设备级 DEVT），并对照 Linux 驱动的实际使能组合。

## 提问

1. 硬件提供了哪些中断供可靠等时传输使用？各自带什么信息？
2. 每个事件的使能位在哪？Linux 实际开了哪些？
3. Bus Expiry 从哪来、怎么处理？

## 理解

### 端点级事件（DEPEVT，等时传输的主战场）

使能位在 **DEPCFG 参数 1**（Linux 定义见 gadget.h:24-26，三个全开）：

| 事件 | 使能 | 触发时机 | 携带信息 | 用途 |
|---|---|---|---|---|
| **XferNotReady** | Param1 BIT(10) | 主机 poll 但传输未激活（IN 无数据 / OUT 无缓冲）；每轮只一次，END_TRANSFER 后重武装 | 高 16 位 = 微帧号 | **起始同步唯一入口** |
| **XferInProgress** | Param1 BIT(9) | 激活传输的每个间隔：TRB 完成（IOC）/ 错过（IMI） | 子状态 IOC / Short / **MissedIsoc(BIT15)** / BusErr；参数=帧号 | 回收 BD、补环、错过上报、刷新帧号 |
| **XferComplete** | Param1 BIT(8) | 传输终结 | — | End Transfer 后确认 |
| **EPCMDCMPLT** | 命令带 CMDIOC | Start/Update/End Transfer 命令完成 | 命令类型 + DEPCMD.CmdStatus | End Transfer 状态机推进；Bus Expiry 重试 |
| RXTXFIFOEVT | — | FIFO 型端点数据需求 | — | Linux 未实现（空 case，gadget.c:3929）；本控制器无 FIFO 型，忽略 |

**事件生成矩阵（表 4-11，尾 TRB 的 IOC/IMI 组合）**：

| 场景 | IOC | IMI | 事件 |
|---|---|---|---|
| 正常完成 | 1 | x | XferInProgress(IOC=1) |
| 短包完成 | 1 | x | XferInProgress(Short=1, IOC=1) |
| 错过 | 0 | 0 | **无中断（静默！只能轮询 HWO 发现）** |
| 错过 | 0 | 1 | XferInProgress(MissedIsoc=1, IOC=0) |
| 错过 | 1 | x | XferInProgress(MissedIsoc=1, IOC=1) |

IOC 单独就能带出 MissedIsoc（错过时尾 TRB 被回收、IOC 随回收触发）；IMI 的价值是
"只错过才打断"。**尾 TRB 至少设其一，错过不能静默。**

### 设备级中断（DEVT，DEVTEN 使能）

Linux 实际使能组合（gadget.c:2846-2861）：EVNTOVERFLOW + CMDCMPLT + ERRTICERR +
WKUP + CONNECT_DONE + USBRST + DISCINT，有条件时加 ULStChg / U3L2L1SUSP。

| 中断 | 对等时传输的意义 |
|---|---|
| USBRST / DISCINT / CONNECT_DONE | 帧号体系重置 → 全端点 End Transfer、清环、重配 |
| ULStChg | U0/U1/U2/U3 变化：低功耗返回后保守做法是 isoc 端点重启重对齐 |
| **EVNTOVERFLOW** | **事件环溢出 = 事件丢了**：清环后必须轮询 TRB 状态重建认知——可靠性兜底 |
| SOF | **不开**（Linux 也没开）：节拍由队列深度承担，同步由 XferNotReady 承担；每 125 µs 一次太频繁 |

### Bus Expiry（命令状态，不是中断）

- 来源：发完 DEPCMD 后轮询 **DEPCMD.CmdStatus[15:12]**（core.h:571）：0=成功、1=No Resource、**2=Bus Expiry**。
- 含义：START TRANSFER 指定的起始帧号已过（或超出约 4s 调度窗口）→ 命令被拒、传输未启动。
- Linux 映射为 **-EAGAIN**（gadget.c:408-420），处理：推后一个间隔重试（最多 5 次，
  core.h:51）；5 次全败 → END_TRANSFER + 等下一个 XferNotReady 重来。

## 要点

- 可靠性最小集合：**XferNotReady + XferInProgress + XferComplete**（DEPCFG Param1 三位）
  + **USBRST/DISCINT/CONNECT_DONE**（DEVTEN）；建议加 ULStChg、EVNTOVERFLOW。
- XferNotReady 解决"何时开始"，XferInProgress 解决"每个间隔发生了什么"——缺一不可。
- Bus Expiry 是"再试一次"性质，不是硬件故障；连撞说明帧号信息已过期，整体重启重对齐。

## 参考

- 翻译 PDF 表 4-11（TRB 事件生成）、4.3.7（检查间隔状态）
- `qemu-linux/linux/drivers/usb/dwc3/core.h`（DEVTEN 位定义 494-505、CmdStatus 571）
- `qemu-linux/linux/drivers/usb/dwc3/gadget.c`（使能组合 2846-2861、-EAGAIN 映射 408-420）
