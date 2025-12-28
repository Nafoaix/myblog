---
title: STM32使用DWT实现微秒延时
tags:
  - 学习笔记
  - STM32
categories:
  - 学习笔记
  - STM32
toc: true
top_img: 'https://cdn.nafx.top/post_cover/20240322222147.png'
cover: 'https://cdn.nafx.top/post_cover/20240322222147.png'
abbrlink: 96aa65ec
date: 2023-04-22 22:25:41
---

## STM32使用DWT实现微秒延时

### 前言

&emsp;&emsp;在使用Cortex-M内核单片机开发驱动时总是需要延时函数做超时判断，这时想实现微秒级延时，不想让CPU干等时间，又不想浪费硬件定时器资源，这时候可以使用`DWT(Data Watchpoint and Trace )数据观察点与跟踪组件`来实现延时功能。

### DWT数据观察点与跟踪组件介绍

&emsp;&emsp;DWT挂载在AHB私有外设总线上，是一个处理数据观察点功能的模块。通过 DWT，可以设置数据观察点。当一个数据地址或数据的值匹配了观察点时，就说产生了一次匹配命中事件。匹配命中事件可以用于产生一个观察点事件，后者能激活调试器以产生数据跟踪信息，或者让 ETM 联动以跟踪在哪条指令上发生了匹配命中事件。作为调试功能这里就不过多展开了，可以去查看Cortex-M3 权威指南中内容，下面讲解以下DWT有关的寄存器。

### 相关的DWT寄存器

&emsp;&emsp; 要实现延时的功能，总共涉及到三个寄存器：DEMCR、DWT_CTRL、DWT_CYCCNT，分别用于开启DWT功能、开启CYCCNT及获得系统时钟计数值。

#### DEM_CR寄存器

![DEMCR寄存器](https://cdn.nafx.top/post_cover/20240322230442.png)

&emsp;&emsp;DEM_CR寄存器`地址:0xE000EDFC`不仅包含了调试监视器的控制位，还包含了跟踪系统的使能位（TRCENA）以及若干向量抓捕（Vector Catch, VC）控制位。这里用到的是DEMCR寄存器的第24位TRCENA`DEMCR  |=  1 << 24 `来使能DWT，这里需要注意的是DEMCR中的控制位是在上电复位时得到复位的。系统复位（例如，往 NVIC 应用程序中断及复位寄存器中写命令）不会影响到它们，在调试相关功能时推荐关掉下载器调试器的自动复位运行功能手动复位。

#### DWT_CR寄存器

![DWT_CR寄存器1](https://cdn.nafx.top/post_cover/20240323000833.png)
![DWT_CR寄存器2](https://cdn.nafx.top/post_cover/20240323000952.png)

&emsp;&emsp;DWT_CR寄存器`地址:0xE0001000`的第1位CYCCNTENA`DWTCR |= 1 << 0`是CYCCNT计数器的使能开关，写1使能否则CYCCNT计数器将不会工作。

#### DWT_CYCCNT寄存器

![DWT_CYCCNT寄存器](https://cdn.nafx.top/post_cover/DWT_CYCCNT.png)

&emsp;&emsp;DWT_CYCCNT寄存器`地址:0xE0001004` DWT周期计数器寄存器是一个32位向上的计数器，，用于记录处理器时钟的周期数，内核时钟跳动一次，该计数器就加1，当CYCCNT溢出之后，会清0重新开始向上计数，通过读取该寄存器的值，可以实现精确的计时功能。
&emsp;&emsp;例如F103系列，内核时钟是72M，那精度就是1/72M = 14ns，而程序的运行时间都是微秒级别的，所以14ns的精度是远远够的。 最长能记录的时间为：60s=2的32次方/72000000， 而如果是H7这种400M主频的芯片，那它的计时精度高达2.5ns（1/400000000 = 2.5），而如果是 i.MX RT1052这种高速的处理器， 最长能记录的时间为： 8.13s=2的32次方/528000000 (假设内核频率为528M，内核跳一次的时间大概为1/528M=1.9ns) 。

### 函数实现

#### 实现过程

1. 先使能DWT外设，这个由另外内核调试寄存器DEM_CR的位24控制，写1使能。
2. 使能CYCCNT寄存器之前，先清0。
3. 使能DWT_CYCCNT寄存器，这个由DWT_CR的CYCCNTENA控制，也就是DWT控制寄存器的位0控制，写1使能。

#### 头文件 dwt_stm32_delay.h

```c
#ifndef DWT_STM32_DELAY_H
#define DWT_STM32_DELAY_H

#include "stm32f10x.h"

// 使能DEMCR寄存器的DWT使能位 
#define  DEM_CR      *(volatile u32 *)0xE000EDFC
#define  DEM_CR_TRCENA                   (0x01 << 24) //0x01000000
 
// 使能DWT_CR寄存器的CYCCNT使能位 CYCCNT计数器开始计数
#define  DWT_CR      *(volatile u32 *)0xE0001000
#define  DWT_CR_CYCCNTENA                (0x01 <<  0) //0x00000001

//0xE0001004 DWT_CYCCNT RW Cycle Count register,   
#define  DWT_CYCCNT  *(volatile u32 *)0xE0001004

/**
 * @brief  Initializes DWT_Cycle_Count for DWT_Delay_us function
 * @return Error DWT counter
 *         1: DWT counter Error
 *         0: DWT counter works
 */
uint32_t DWT_Delay_Init(void);
void DWT_Delay_us(uint32_t _ulDelayTime);
void DWT_Delay_ms(uint32_t _ulDelayTime);

#endif
```

### dwt_stm32_delay.c文件

```c
#include "dwt_stm32_delay.h"
 
/**
 * @brief  Initializes DWT_Clock_Cycle_Count for DWT_Delay_us function
 * @retval Error DWT counter
 *         1: clock cycle counter not started
 *         0: clock cycle counter works
 */
uint32_t DWT_Delay_Init()
{
    DEM_CR  &= ~DEM_CR_TRCENA;  /* Disable TRCENA ~0x01000000 */ 
    DEM_CR  |=  DEM_CR_TRCENA; /*对DEMCR寄存器的位24控制，写1使能DWT外设。0x01000000 */
    DWT_CR  &= ~DWT_CR_CYCCNTENA; /* Disable clock cycle counter ~0x00000001 */
    DWT_CR  |=  DWT_CR_CYCCNTENA;/*对DWT控制寄存器的位0控制，写1使能CYCCNT寄存器。0x00000001 */

    /* Reset the clock cycle counter value */
    DWT_CYCCNT = 0;/*对于DWT的CYCCNT计数寄存器清0。*/

    /* 3 NO OPERATION instructions */
    __ASM volatile ("NOP");
    __ASM volatile ("NOP");
    __ASM volatile ("NOP");
    /* Check if clock cycle counter has started */
     if(DWT_CYCCNT)
      return 0; /*clock cycle counter started*/
    else
      return 1; /*clock cycle counter not started*/
  
}

void DWT_Delay_us(uint32_t _ulDelayTime)
{
    uint32_t tCnt, tDelayCnt;
    uint32_t tStart;
           
    tStart = DWT_CYCCNT; /* 刚进入时的计数器值 */
    tCnt = 0;
    tDelayCnt = _ulDelayTime * (SystemCoreClock / 1000000);
    /* 需要的节拍数 */    /*SystemCoreClock :系统时钟频率*/                 

    while(tCnt < tDelayCnt)
      {
        tCnt = DWT_CYCCNT - tStart; 
        /* 求减过程中，如果发生第一次32位计数器重新计数，依然可以正确计算 */       
      }
}

void DWT_Delay_ms(uint32_t _ulDelayTime)
{
  DWT_Delay_us(1000*_ulDelayTime);
}

```

### 说在最后

&emsp;&emsp; 在计算延时过程中并没有对溢出做太多特殊处理，因为对于延时时间超过最大延时时间的溢出在使用中基本不会遇到，另外是在使用了RTOS的系统中对于DWT_CYCCNT值的不确定性，DWT_CYCCNT < tStart溢出的情况由于变量都是uint32_t类型，tCnt会变成一个巨大的无符号数直接结束延时；DWT_CYCCNT > tStart 溢出的情况可能会增加延时时间，对延时准确性，安全性等有考量不建议使用。
