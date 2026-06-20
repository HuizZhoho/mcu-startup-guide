# MCU 启动流程深度解读

> 从复位到 main() — 逐行解析启动文件、链接脚本与实际固件案例

---

## 目录

- [一、典型 MCU 启动流程概览](#一典型-mcu-启动流程概览)
- [二、链接脚本 (.ld) 逐行解析](#二链接脚本-ld-逐行解析)
- [三、启动文件 (.s) 逐行解析](#三启动文件-s-逐行解析)
- [四、SystemInit() 深度解读](#四systeminit-深度解读)
- [五、内存布局完整可视化](#五内存布局完整可视化)
- [六、启动流程图（指令级）](#六启动流程图指令级)
- [七、实际固件案例分析](#七实际固件案例分析)
- [八、对比：不同架构的启动差异](#八对比不同架构的启动差异)
- [九、常见问题与调试技巧](#九常见问题与调试技巧)
- [十、从启动到 RTOS 调度全链路时间线](#十从启动到-rtos-调度全链路时间线)
- [十一、MCU 以太网 PHY 控制完整过程](#十一切mcu-以太网-phy-控制完整过程)
  - [11.1 概述：MAC 与 PHY 的分工](#111-概述mac-与-phy-的分工)
  - [11.2 MII / RMII 接口详解](#112-mii--rmii-接口详解)
  - [11.3 SMI/MDIO 协议（寄存器读写机制）](#113-smimdio-协议寄存器读写机制)
  - [11.4 IEEE 802.3 PHY 标准寄存器详解](#114-ieee-8023-phy-标准寄存器详解)
  - [11.5 PHY 初始化全流程](#115-phy-初始化全流程)
  - [11.6 自动协商（Auto-Negotiation）深度解析](#116-自动协商auto-negotiation深度解析)
  - [11.7 链路状态检测与中断管理](#117-链路状态检测与中断管理)
  - [11.8 实战：STM32F407 + LAN8720 完整初始化代码](#118-实战stm32f407--lan8720-完整初始化代码)
  - [11.9 实战：STM32F4 + DP83848 初始化](#119-实战stm32f4--dp83848-初始化)
  - [11.10 RMII 时钟与 PCB 关键设计要点](#1110-rmii-时钟与-pcb-关键设计要点)
  - [11.11 常见问题与调试技巧](#1111-常见问题与调试技巧)
  - [11.12 LWIP 集成要点](#1112-lwip-集成要点)

---

## 一、典型 MCU 启动流程概览

MCU 上电后，从复位到执行 `main()` 函数，经历以下核心阶段：

```
上电复位 → 启动文件(Startup) → 系统初始化 → 分散加载 → 跳转 main()
```

### 1. 复位向量表 (Reset Vector Table)

MCU 上电后，CPU 从**复位向量地址**（通常为 `0x00000000` 或 `0x08000000`，取决于芯片）读取栈顶指针 SP 和复位入口 PC。

一个典型的 ARM Cortex-M 向量表示例（以 STM32 为例）：

| 偏移 | 内容 | 说明 |
|------|------|------|
| 0x00 | `__initial_sp` | 栈顶地址 |
| 0x04 | `Reset_Handler` | 复位中断入口 |
| 0x08 | `NMI_Handler` | NMI 中断 |
| 0x0C | `HardFault_Handler` | 硬错误中断 |
| ... | ... | 其他外设中断 |

### 2. 启动文件 (Startup File)

通常用汇编编写（如 `startup_stm32f103xe.s`），完成三件事：

```assembly
; 伪代码示意
Reset_Handler:
    ① 设置栈指针 SP（若硬件未自动完成）
    ② 初始化 .bss 段（清零）
    ③ 初始化 .data 段（从 Flash 拷贝到 RAM）
    ④ 调用 SystemInit() 配置时钟
    ⑤ 跳转 __main (C 运行时库入口) 或直接跳 main()
```

### 3. SystemInit() — 系统时钟配置

将 MCU 从默认低速时钟切换至目标主频，例如 STM32F103 的 **HSE → PLL → 72MHz**：

```c
void SystemInit(void) {
    /* ① 使能 HSE 外部高速晶振 */
    RCC->CR |= RCC_CR_HSEON;
    while (!(RCC->CR & RCC_CR_HSERDY));  // 等待就绪

    /* ② 配置 Flash 等待周期（72MHz 需 2 等待周期） */
    FLASH->ACR = FLASH_ACR_LATENCY_2;

    /* ③ 配置 PLL：HSE 8MHz * 9 = 72MHz */
    RCC->CFGR |= RCC_CFGR_PLLSRC_HSE | RCC_CFGR_PLLMULL9;

    /* ④ 使能 PLL */
    RCC->CR |= RCC_CR_PLLON;
    while (!(RCC->CR & RCC_CR_PLLRDY));

    /* ⑤ 切换系统时钟为 PLL */
    RCC->CFGR |= RCC_CFGR_SW_PLL;
    while ((RCC->CFGR & RCC_CFGR_SWS) != RCC_CFGR_SWS_PLL);
}
```

### 4. C 运行时初始化 (__main / __libc_init_array)

- 初始化 C 库（堆、全局变量构造）
- 调用**构造器**（C++ 全局对象构造函数）
- 最终跳转 `main()`

### 5. main() 中的 Board-Level 初始化

```c
int main(void) {
    HAL_Init();           // HAL 库初始化（配置 SysTick 等）
    SystemClock_Config(); // 进一步细化时钟树
    MX_GPIO_Init();       // GPIO 初始化
    MX_USART1_UART_Init();// 串口初始化
    // ... 其他外设

    while (1) {
        // 主循环
    }
}
```

---

## 二、链接脚本 (.ld) 逐行解析

文件：`STM32F407ZGTx_FLASH.ld`

```ld
/* 入口点声明 —— 告诉链接器程序入口地址 */
ENTRY(Reset_Handler)
```

> CPU 复位后第一条指令地址，必须是 Reset_Handler 的地址。

```ld
/* ═══════════ 内存区域定义 ═══════════ */
MEMORY
{
    /* FLASH:  只读+可执行, 起始 0x08000000, 大小 1MB */
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 1024K

    /* RAM:    读写,  起始 0x20000000, 大小 128KB  */
    RAM   (rw)  : ORIGIN = 0x20000000, LENGTH = 128K

    /* CCM RAM: 内核专用 RAM，不支持 DMA (STM32F4独有) */
    CCMRAM (rw) : ORIGIN = 0x10000000, LENGTH = 64K
}
```

> **为什么 Flash 起始是 0x08000000？**
> STM32F4 的 Flash **物理起始**是 0x08000000。但芯片复位后会做一个地址映射：`0x00000000 → 0x08000000`（别名映射），所以向量表既可以放在 0x08000000 也可以通过 `SYSCFG_MEMRMP` 重映射。

```ld
/* ═══════════ 栈定义 ═══════════ */
/* _estack = RAM 末尾 (栈向下增长，从高端地址开始) */
_estack = ORIGIN(RAM) + LENGTH(RAM);

/* 最小堆大小 (当使用 malloc / printf 时) */
_Min_Heap_Size  = 0x200;  /* 512 字节 */
_Min_Stack_Size = 0x400;  /* 1KB */
```

> **为什么栈放在 RAM 末尾？**
> 栈是向下增长（从高地址到低地址），所以初始 SP = RAM 最高地址。堆向上增长，栈向下增长，两者在 RAM 中间相遇——最大化利用 RAM。

```ld
/* ═══════════ 段定义 ═══════════ */
SECTIONS
{
    /* ──── ① 中断向量表 ──── */
    .isr_vector :
    {
        . = ALIGN(4);               /* 4 字节对齐 */
        KEEP(*(.isr_vector))        /* 强制保留，链接优化不会删除 */
        . = ALIGN(4);
    } > FLASH                       /* 放在 Flash 段 */
```

> `. = ALIGN(4)` 表示当前位置对齐到 4 字节边界。`KEEP()` 是 GNU ld 的关键——即使 `-gc-sections` 回收未引用段，`KEEP` 仍会保留。向量表是中断入口，必须保留。

```ld
    /* ──── ② 代码段 ──── */
    .text :
    {
        . = ALIGN(4);
        *(.text)                    /* 所有目标文件的 .text */
        *(.text*)                   /* 带后缀的变体 */
        *(.glue_7)                  /* ARM-Thumb 互操作胶水代码 */
        *(.glue_7t)
        *(.eh_frame)                /* 异常展开帧 (C++ 异常) */

        KEEP (*(.init))             /* C++ 构造函数表 */
        KEEP (*(.fini))

        . = ALIGN(4);
        _etext = .;                 /* 代码段结束地址 */
    } > FLASH
```

```ld
    /* ──── ③ 只读数据 ──── */
    .rodata :
    {
        . = ALIGN(4);
        *(.rodata)
        *(.rodata*)
        . = ALIGN(4);
    } > FLASH
```

```ld
    /* ──── ④ ARM 异常展开表 ──── */
    .ARM.extab   : { *(.ARM.extab* .gnu.linkonce.armextab.*) } > FLASH
    .ARM : {
        __exidx_start = .;
        *(.ARM.exidx*)             /* 异常索引表 */
        __exidx_end = .;
    } > FLASH
```

> `.ARM.exidx` 对于 **Cortex-M 不需要**（M 核不支持 ARM 模式），但链接器可能生成，保留它以避免链接错误。

```ld
    /* ──── ⑤ 初始化数据 (.data) ──── */
    /*   加载视图: 在 Flash 中 (AT > FLASH)                 */
    /*   运行视图: 在 RAM 中                                */
    . = ALIGN(4);
    _sidata = .;                    /* Flash 中初始值位置 */

    .data : AT ( _sidata )          /* AT 指定加载地址 */
    {
        . = ALIGN(4);
        _sdata = .;                 /* RAM 中起始地址 */
        *(.data)
        *(.data*)
        . = ALIGN(4);
        _edata = .;                 /* RAM 中结束地址 */
    } > RAM
```

> **这是整个启动流程中最关键的部分！**
>
> 有初始值的全局变量（如 `int x = 5;`），在 Flash 中占一份（`_sidata`），但运行在 RAM 中（`_sdata` → `_edata`）。
>
> 启动文件必须负责把 `_sidata` 处的内容拷贝到 `_sdata` → `_edata` 的区间。

```ld
    /* ──── ⑥ 零初始化数据 (.bss) ──── */
    . = ALIGN(4);
    .bss :
    {
        _sbss = .;                  /* BSS 起始 */
        *(.bss)
        *(.bss*)
        *(COMMON)                   /* 未初始化的全局变量也归入 BSS */
        . = ALIGN(4);
        _ebss = .;                  /* BSS 结束 */
    } > RAM
```

```ld
    /* ──── ⑦ 堆 ──── */
    ._user_heap_stack :
    {
        . = ALIGN(8);
        PROVIDE ( end = . );
        PROVIDE ( _end = . );
        . = . + _Min_Heap_Size;     /* 保留堆空间 */
        . = . + _Min_Stack_Size;    /* 保留栈空间 */
        . = ALIGN(8);
    } > RAM
```

> `PROVIDE(end = .)` 使得 `_sbrk()`（libc 的堆分配函数）知道堆从哪里开始。

```ld
    /* ──── ⑧ CCM RAM (可选) ──── */
    .ccmram :
    {
        . = ALIGN(4);
        *(.ccmram)
        *(.ccmram*)
        . = ALIGN(4);
    } > CCMRAM
```

```ld
    /* ──── ⑨ 符号检查 ──── */
    /* 栈顶不能超过 RAM 末尾 */
    ASSERT((_estack > .) || (_Min_Stack_Size == 0),
           "ERROR: RAM overflow with stack")
}
```

> **ASSERT**：如果链接时计算出栈顶位置已经超出 RAM 范围，直接报错，而不是在运行时神秘崩溃。

---

## 三、启动文件 (.s) 逐行解析

文件：`startup_stm32f407xx.s`

### 3.1 文件头与常量定义

```assembly
; 语法: GNU AS (GAS) for ARM, 兼容 ARM/Thumb
.syntax unified          ; 统一 ARM/Thumb 语法
.cpu cortex-m4           ; 指定 CPU 型号 (影响指令集选择)
.fpu softvfp             ; 软浮点 ABI (不使用 FPU 硬件)
.thumb                   ; Thumb-2 模式 (Cortex-M4 不支持 ARM 模式)

; ── 全局符号声明 ──
.globl  g_pfnVectors     ; 向量表为全局
.globl  Default_Handler  ; 默认中断处理

; 引入链接脚本中定义的符号 (外部符号)
.word   _sidata          ; .data 在 Flash 中的加载地址
.word   _sdata           ; .data 在 RAM 中的起始地址
.word   _edata           ; .data 在 RAM 中的结束地址
.word   _sbss            ; .bss 在 RAM 中的起始地址
.word   _ebss            ; .bss 在 RAM 中的结束地址
.word   _estack          ; 栈顶地址
```

### 3.2 中断向量表

```assembly
; ──────────────────────────────────
; 中断向量表
; 每个入口 4 字节，共约 90+ 个入口
; ──────────────────────────────────
.section .isr_vector      ; 放到 .isr_vector 段 (链接脚本会放在 Flash 开头)
.align  2                 ; 4 字节对齐

g_pfnVectors:
    .word   _estack                   ; 偏移 0x00: 栈顶指针 (MSP 初始值)
    .word   Reset_Handler             ; 偏移 0x04: 复位中断入口
    .word   NMI_Handler               ; 偏移 0x08: NMI
    .word   HardFault_Handler         ; 偏移 0x0C: Hard Fault
    .word   MemManage_Handler         ; 偏移 0x10: MPU 内存管理错误
    .word   BusFault_Handler          ; 偏移 0x14: 总线错误
    .word   UsageFault_Handler        ; 偏移 0x18: 用法错误
    .word   0                         ; 偏移 0x1C: 保留
    .word   0
    .word   0
    .word   0
    .word   SVC_Handler               ; 偏移 0x2C: SVC (系统调用)
    .word   DebugMon_Handler          ; 偏移 0x30: 调试监视器
    .word   0
    .word   PendSV_Handler            ; 偏移 0x38: PendSV (RTOS 上下文切换)
    .word   SysTick_Handler           ; 偏移 0x3C: SysTick (RTOS 系统滴答)

; ── 外设中断 (以 STM32F407 为例，共 82 个) ──
    .word   WWDG_IRQHandler           ; 窗口看门狗
    .word   PVD_IRQHandler            ; 电压监测
    .word   TAMP_STAMP_IRQHandler     ; 篡改检测/时间戳
    .word   RTC_WKUP_IRQHandler       ; RTC 唤醒
    .word   FLASH_IRQHandler          ; Flash 操作完成
    .word   RCC_IRQHandler            ; RCC 时钟中断
    .word   EXTI0_IRQHandler          ; 外部中断 0
    .word   EXTI1_IRQHandler          ; 外部中断 1
    ; ... 省略后续 ~70 个外设中断入口 ...
```

> **关键理解**：
> - 向量表的前两个入口（SP 和 Reset_Handler）由 CPU 硬件自动读取
> - Cortex-M 进入中断时，**硬件自动压栈**（R0-R3, R12, LR, PC, xPSR），无需软件干预
> - 空闲中断入口填 `Default_Handler`，防止未处理中断导致跑飞

### 3.3 Reset_Handler — 芯片上电第一段软件代码

```assembly
; ──────────────────────────────────
; Reset_Handler: CPU 复位后第一条软件指令
; ──────────────────────────────────
.section  .text.Reset_Handler
.weak     Reset_Handler               ; weak 允许用户覆盖
.type     Reset_Handler, %function
Reset_Handler:

    /* ── 第一步：设置栈指针 ── */
    ldr   sp, =_estack                ; 加载栈顶地址到 SP
```

> `ldr sp, =_estack`：虽然 CPU 硬件已经在取向量表时加载了 SP，但这里显式设置一次确保万无一失（尤其从低功耗唤醒时）。

```assembly
    /* ── 第二步：拷贝 .data 段 ── */
    ; 从 Flash 复制初始值到 RAM
    ldr   r0, =_sdata                 ; r0 = RAM 目标起始
    ldr   r1, =_edata                 ; r1 = RAM 目标结束
    ldr   r2, =_sidata                ; r2 = Flash 源起始
    b     LoopCopyDataInit            ; 先跳转检查是否需要拷贝

CopyDataInit:
    ldr   r3, [r2], #4               ; r3 = *r2, 然后 r2 += 4 (后增)
    str   r3, [r0], #4               ; *r0 = r3, 然后 r0 += 4
    ; 注: #4 = 32位 = 1 word

LoopCopyDataInit:
    cmp   r0, r1                      ; 比较当前地址和目标结束地址
    itt   lo                          ; 如果是小于 (lower)
    ldrlo r3, [r2], #4               ;   则继续加载 Flash 内容
    strlo r3, [r0], #4               ;   并写入 RAM
    blo   LoopCopyDataInit            ;   循环
```

> **Cortex-M4 特有的 IT 指令** (If-Then)：
> - `itt lo` = 如果小于，则执行接下来的**两条**指令
> - 这是 Thumb-2 的条件执行特性，减少分支跳转开销

```assembly
    /* ── 第三步：清零 .bss 段 ── */
    ldr   r0, =_sbss                  ; r0 = BSS 起始
    ldr   r1, =_ebss                  ; r1 = BSS 结束
    movs  r2, #0                      ; r2 = 0 (写入值)
    b     LoopFillZerobss

FillZerobss:
    str   r2, [r0], #4               ; *r0 = 0, r0 += 4

LoopFillZerobss:
    cmp   r0, r1
    bcc   FillZerobss                 ; 小于则继续 (branch if carry clear = unsigned <)

    ; ── 可选：额外清除 CCM RAM ──
    ; ... (STM32F4 的 64KB CCM RAM 也需要清零)
```

```assembly
    /* ── 第四步：调用 SystemInit ── */
    bl    SystemInit                  ; 配置时钟 (HSE → PLL → 168MHz)
```

```assembly
    /* ── 第五步：调用 C 运行时初始化 ── */
    bl    __libc_init_array           ; C++ 全局构造器 / C 库初始化
```

> `__libc_init_array` 是 ARM GCC 工具链提供的 C 运行时初始化函数：
> 1. 调用 `_init()`（如果有）
> 2. 遍历 `.init_array` 段，依次调用所有全局构造函数
> 3. 初始化 libc 内部的静态数据

```assembly
    /* ── 第六步：跳转到 main ── */
    bl    main                        ; 进入用户主函数

    ; ── 如果 main 返回（通常不应发生） ──
    b     .                           ; 无限循环 (相当于 while(1))
```

### 3.4 默认中断处理函数 (Weak Aliases)

```assembly
; ──────────────────────────────────
; Default_Handler: 所有未单独定义的中断入口
; ──────────────────────────────────
.section  .text.Default_Handler,"ax",%progbits
Default_Handler:
    b     .                           ; 死循环：如果看到此处的 PC，
                                      ; 说明有未处理的中断触发！

; ── 用 .weak 创建所有中断处理函数的弱符号 ──
; 如果用户在 C 代码中定义了同名函数，自动覆盖
.macro  def_irq_handler  handler_name
.weak   \handler_name
.set    \handler_name, Default_Handler
.endm

def_irq_handler  NMI_Handler
def_irq_handler  HardFault_Handler
def_irq_handler  MemManage_Handler
def_irq_handler  BusFault_Handler
def_irq_handler  UsageFault_Handler
def_irq_handler  SVC_Handler
def_irq_handler  DebugMon_Handler
def_irq_handler  PendSV_Handler
def_irq_handler  SysTick_Handler
; ... 所有外设中断 ...
```

> **Weak 机制是嵌入式 C 的神来之笔**：
> - 用户不需要实现所有中断，只有用到的才写处理函数
> - 如果用户写了 `void SysTick_Handler(void){...}`，链接器自动替换弱符号
> - 未实现的中断触发则会卡在 `Default_Handler` 的死循环中（便于调试）

---

## 四、SystemInit() 深度解读

文件：`system_stm32f4xx.c`

```c
void SystemInit(void)
{
    /* ── Step 1: 关闭 FPU (如果不用) ── */
    /* FPU 默认开启，这里按需关闭 */

    /* ── Step 2: 复位 RCC 寄存器到默认值 ── */
    RCC->CFGR = 0x00000000;     // 清空时钟配置: 系统时钟 = HSI (16MHz)
    RCC->CR   = 0x00004083;     // HSI 开启, PLL 关闭, CSS 关闭
    RCC->PLLCFGR = 0x24003010;  // PLL 配置复位
    RCC->CIR  = 0x00000000;     // 关闭所有时钟中断

    /* ── Step 3: 配置 Flash 预取缓冲 ── */
    /* 针对 168MHz 主频: 5 个等待周期 */
    #if (__SYSTEM_CLOCK == 168000000)
        FLASH->ACR = FLASH_ACR_ICEN |    // 指令缓存启用
                     FLASH_ACR_DCEN |    // 数据缓存启用
                     FLASH_ACR_LATENCY_5W;// 5 等待周期 (168MHz)
    #endif

    /* ── Step 4: 配置电源管理 ── */
    /* V1.8V 核心电压调节器 */
    PWR->CR |= PWR_CR_VOS_1 | PWR_CR_VOS_0;  // VOS = High performance

    /* ── Step 5: 配置预分频器 ── */
    RCC->CFGR |= RCC_CFGR_HPRE_DIV1;   // AHB = SYSCLK / 1 = 168MHz
    RCC->CFGR |= RCC_CFGR_PPRE1_DIV4;  // APB1 = AHB / 4 = 42MHz
    RCC->CFGR |= RCC_CFGR_PPRE2_DIV2;  // APB2 = AHB / 2 = 84MHz

    /* ── Step 6: 使能 HSE 并等待就绪 ── */
    RCC->CR |= RCC_CR_HSEON;            // 开启 HSE
    while(!(RCC->CR & RCC_CR_HSERDY));  // 等待 HSE 稳定

    /* ── Step 7: 配置 PLL ── */
    /*
     * PLL 公式: VCO = (HSE / PLL_M) * PLL_N
     * SYSCLK = VCO / PLL_P
     *
     * 给定: HSE = 8MHz
     * PLL_M = 8, PLL_N = 336, PLL_P = 2
     * VCO = (8MHz / 8) * 336 = 336MHz
     * SYSCLK = 336MHz / 2 = 168MHz
     *
     * PLL_Q 用于 USB/SDIO: 336MHz / 7 = 48MHz
     */
    RCC->PLLCFGR = (8 << 0)  |    // PLL_M  = 8
                   (336 << 6) |   // PLL_N  = 336
                   (0 << 16) |    // PLL_P  = 2 (编码: 00=2, 01=4, 10=6, 11=8)
                   (7 << 24);     // PLL_Q  = 7

    RCC->CR |= RCC_CR_PLLON;       // 使能 PLL
    while(!(RCC->CR & RCC_CR_PLLRDY));  // 等待 PLL 锁定

    /* ── Step 8: 切换系统时钟到 PLL ── */
    RCC->CFGR |= RCC_CFGR_SW_PLL;  // 系统时钟源 = PLL
    while((RCC->CFGR & RCC_CFGR_SWS) != RCC_CFGR_SWS_PLL);
    // 等待确认时钟已切换

    /* ── Step 9: 更新全局变量 ── */
    SystemCoreClock = 168000000;    // 供 HAL 库使用
}
```

> **为什么需要等待 HSE 和 PLL 就绪？**
> HSE 是外部晶振，起振需要几毫秒（晶体起振时间）。PLL 锁定需要几微秒到几百微秒。不等待直接切换会导致系统以错误的时钟运行或不运行。

---

## 五、内存布局完整可视化

启动完成后，内存分布为：

```
FLASH (0x08000000)
┌──────────────────────┐  ← 0x08000000
│   .isr_vector        │  向量表 (约 400 字节)
├──────────────────────┤  ← 0x08000190
│   .text              │  代码 & 常量字符串
│   main()             │
│   HAL_XXX()          │
│   printf() 等        │
├──────────────────────┤
│   .rodata            │  只读数据 (const 变量)
├──────────────────────┤
│   _sidata ← ─────┐   │  .data 原始值存放位置
│                  │   │
└──────────────────────┘  ← 0x08100000

RAM (0x20000000)
┌──────────────────────┐  ← 0x20000000
│   .data (已初始化)    │  全局变量: int x=5;
│              ↑       │  └─ 启动时从 _sidata 拷贝过来
├──────────────────────┤  ← _edata / _sbss
│   .bss (零初始化)    │  全局变量: int y;
├──────────────────────┤  ← _ebss
│   Heap (↑向上增长)   │  malloc / new 分配
├──────────────────────┤
│   Stack (↓向下增长)  │  局部变量 / 函数调用栈帧
│                      │
│   ← SP 初始指向这里  │  _estack
└──────────────────────┘  ← 0x20020000
```

---

## 六、启动流程图（指令级）

```
CPU 复位 (RST 引脚释放)
│
│  硬件自动完成:
├─ ① 从 0x00000000(别名→0x08000000) 读取 SP → MSP
│      _estack = 0x20020000
│
├─ ② 从 0x00000004(别名→0x08000004) 读取 PC → Reset_Handler
│
│  软件执行 (由 Reset_Handler 完成):
├─ ③ ldr sp, =_estack           SP = 0x20020000 (冗余保障)
│
├─ ④ .data 拷贝循环:
│      ldr   r0, =_sdata         r0 = 0x20000000
│      ldr   r1, =_edata         r1 = 0x20000200
│      ldr   r2, =_sidata        r2 = 0x0800XXXX
│  ┌─ cmp   r0, r1
│  ├─ itt   lo
│  ├─ ldrlo r3, [r2], #4        r3 = Flash[...]; r2 += 4
│  ├─ strlo r3, [r0], #4        RAM[...] = r3; r0 += 4
│  └─ blo   loop                if (r0 < r1) goto loop
│
├─ ⑤ .bss 清零循环:
│      ldr   r0, =_sbss         r0 = 0x20000200
│      ldr   r1, =_ebss         r1 = 0x20002000
│      movs  r2, #0
│  ┌─ str   r2, [r0], #4        RAM[r0] = 0; r0 += 4
│  └─ bcc   loop                if (r0 < r1) goto loop
│
├─ ⑥ bl   SystemInit            时钟: 8MHz → 168MHz
│
├─ ⑦ bl   __libc_init_array     C 运行时初始化
│
├─ ⑧ bl   main                  → 进入用户代码 (永不返回)
│
└─ ⑨ b    .                     (如果 main 返回, 死循环)
```

---

## 七、实际固件案例分析

### 案例 1：STM32F103 裸机 Blinky（最小启动）

- **芯片**：STM32F103C8T6, Cortex-M3, 72MHz
- **启动文件**：`startup_stm32f103xe.s`
- **链接脚本**：`STM32F103C8Tx_FLASH.ld`

**链接脚本关键段定义**：

```ld
MEMORY {
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 64K
    RAM   (rw)  : ORIGIN = 0x20000000, LENGTH = 20K
}

SECTIONS {
    .isr_vector  : { *(.isr_vector)  } > FLASH  ; 向量表 —— 必须从 0x08000000 开始
    .text        : { *(.text*)       } > FLASH  ; 代码段
    .rodata      : { *(.rodata*)     } > FLASH  ; 只读数据
    .data        : { *(.data*)       } > RAM AT > FLASH  ; 初始化数据（加载在 Flash，运行在 RAM）
    .bss         : { *(.bss*)        } > RAM    ; 零初始化数据
}
```

**启动文件核心汇编（简化）**：

```assembly
Reset_Handler:
    ldr   r0, =_estack      ; 设置栈顶
    msr   msp, r0

    ; 拷贝 .data 段
    ldr   r1, =_sidata      ; Flash 中初始值地址
    ldr   r2, =_sdata       ; RAM 中目标地址
    ldr   r3, =_edata       ; 结束地址
    bl    copy_data

    ; 清零 .bss 段
    ldr   r1, =_sbss
    ldr   r2, =_ebss
    bl    zero_bss

    bl    SystemInit        ; 时钟配置
    bl    main              ; 跳转主函数
    bx    lr
```

### 案例 2：FreeRTOS 任务化固件（智能传感器节点）

实际项目简况：

- **芯片**：STM32L476RG, Cortex-M4, 80MHz, 1MB Flash / 128KB RAM
- **RTOS**：FreeRTOS + CMSIS-RTOS v2
- **功能**：多传感器采集 → 数据处理 → LoRaWAN 上报

**启动流程差异**：

```c
int main(void) {
    HAL_Init();
    SystemClock_Config();      // MSI 16MHz → PLL 80MHz
    MX_GPIO_Init();
    MX_USART1_UART_Init();
    MX_LoRaWAN_Init();

    /* FreeRTOS 启动 —— 创建任务后不再返回 */
    osKernelInitialize();

    /* 创建任务 */
    osThreadNew(sensor_task,     NULL, &sensor_attr);     // 传感器采集
    osThreadNew(process_task,    NULL, &process_attr);    // 数据处理
    osThreadNew(lora_task,       NULL, &lora_attr);       // LoRa 发送
    osThreadNew(led_task,        NULL, &led_attr);        // 心跳 LED

    osKernelStart();  // 启动调度器，不再返回
    while (1);
}
```

**内存分布**（实际 map 文件中截取）：

```
Region      Start       End         Size      Description
──────      ─────       ───         ────      ───────────
Flash       0x08000000  0x08100000  1MB       Main Flash
  .isr_vector  0x08000000  0x08000188  392B     Vector table
  .text        0x08000188  0x0803A000  234KB    Code + const
  .rodata      0x0803A000  0x0804C000  72KB     String literals

RAM         0x20000000  0x20020000  128KB
  .data        0x20000000  0x20000200  512B     Global vars
  .bss         0x20000200  0x20003000  11.5KB   Zero-initialized
  Heap         0x20003000  0x20008000  20KB     malloc / FreeRTOS heap
  Stack        0x20008000  0x20008400  1KB      Main stack
```

### 案例 3：Bootloader + App 双区启动（OTA 升级）

- **芯片**：ESP32-S3, Xtensa LX7, 240MHz
- **特性**：支持远程固件升级（OTA）

**分区布局**：

```
Flash 0x0000  ┌─────────────────┐
              │   Bootloader    │  (ROM + 一级引导)
    0x8000    ├─────────────────┤
              │  Partition Table│
    0x9000    ├─────────────────┤
              │   App A (OTA_0) │  ← 当前运行
   0x100000   ├─────────────────┤
              │   App B (OTA_1) │  ← 下载备用区
   0x1A0000   ├─────────────────┤
              │   NVS / Storage │
   0x1D0000   └─────────────────┘
```

**启动流程（ESP-IDF）**：

```
ROM Bootloader (一级)
    └─ 读取 eFuse / GPIO 电平 → 进入下载模式或 Flash 启动
        └─ 二级 Bootloader (Flash)
            └─ 读取 Partition Table
                └─ 检查 ota_data 分区确认 "当前活跃" 分区
                    └─ 验证签名 & CRC
                        └─ 跳转 App Entry
                            └─ CallStartup()
                                └─ main()
```

**实际 OTA 切换逻辑**：

```c
/* 下载新固件到 OTA_1 分区后 */
esp_ota_set_boot_partition(ota_1_partition);

/* 重启后，bootloader 读取 ota_data 发现标记为 "ota_1" */
esp_restart();

/* 回滚机制：新固件需主动确认 */
void app_main(void) {
    /* 如果未调用 esp_ota_mark_app_valid(),
       下次重启自动回滚到旧分区 */
    if (first_boot) {
        esp_ota_mark_app_valid_cancel_rollback();
    }
}
```

**关键点**：Bootloader 本身不包含业务逻辑，只做**分区校验和跳转**，以保持最小化、高可靠性。

---

## 八、对比：不同架构的启动差异

### Cortex-M0+ (如 STM32G0)

与 M4 的差异：

| 项目 | M4 | M0+ |
|------|----|-----|
| 指令集 | Thumb-2 (含 IT 条件) | Thumb (无 IT) |
| 向量表 | 支持中断优先级可编程 | 固定优先级 |
| FPU | 可选 (单精度) | 无 |
| SysTick | 有 | 无 |
| 等待周期 | 需要配置 (高主频) | 通常不需要 |

M0+ 的启动文件更简单，`CopyDataInit` 循环无需 IT 指令：

```assembly
; M0+ 版本的 .data 拷贝 (无 IT)
LoopCopyDataInit:
    cmp   r0, r1
    bhs   EndCopyDataInit      ; >= 则退出
    ldr   r3, [r2], #4
    str   r3, [r0], #4
    b     LoopCopyDataInit
EndCopyDataInit:
```

### RISC-V (如 GD32VF103、CH32V307)

```assembly
; CH32V307 RISC-V 启动 (核心差异)
_start:
    lui   sp, %hi(_stack_top)
    addi  sp, sp, %lo(_stack_top)

    ; 设置陷阱向量 (mtvec)
    la    t0, __vector_table
    csrw  mtvec, t0            ; 写入 mtvec CSR 寄存器

    ; 使能全局中断
    csrsi mstatus, 0x8         ; MIE 位 = 1

    ; RISC-V 没有独立的 .data 拷贝指令
    ; 与 ARM 相同: 用循环实现
    la    a0, _sidata
    la    a1, _sdata
    la    a2, _edata
    bgeu  a1, a2, zero_bss     ; 如果 sdata >= edata, 跳过

copy_data:
    lw    t0, 0(a0)
    sw    t0, 0(a1)
    addi  a0, a0, 4
    addi  a1, a1, 4
    bltu  a1, a2, copy_data

zero_bss:
    ; ... 清零 .bss (同上模式) ...
```

**主要区别**：
1. 无硬件向量表自动加载 —— 需软件设置 `mtvec`
2. CSR 寄存器操作（`csrw`, `csrsi`）替代 ARM 的 MSR/MRS
3. 无 Thumb/ARM 模式切换（RISC-V 不需要）

---

## 九、常见问题与调试技巧

### 问题 1：上电后卡死，不进 main()

**排查步骤**：

```bash
# 1. 检查链接脚本中的入口点
arm-none-eabi-readelf -h firmware.elf | grep Entry

# 2. 查看向量表前 8 字节是否正确
arm-none-eabi-objdump -s -j .isr_vector firmware.elf
# 输出应为: 20020000 08000189 ...
# SP = 0x20020000, PC = 0x08000188 + 1 = 0x08000189 (Thumb LSB=1)

# 3. 反汇编 Reset_Handler 确认拷贝逻辑
arm-none-eabi-objdump -d firmware.elf | grep -A50 "Reset_Handler"
```

### 问题 2：全局变量值不对

```c
// 现象: int x = 5; 但实际 x = 0 或乱码

// 排查: 确认链接脚本中 _sdata / _sidata / _edata 地址
// map 文件中找:
// _sdata  = 0x20000000
// _edata  = 0x20000200
// _sidata = 0x08001A00  (Flash 地址，应在 .rodata 之后)

// 如果 _sidata 指向了错误的 Flash 地址，data 拷贝就拷错内容
// 检查链接脚本中的 AT 声明是否遗漏
```

### 问题 3：HardFault 定位

```c
// 方法：读取 LR 获取返回地址
// LR 在 HardFault 时 = 0xFFFFFFFx

// 如果 LR = 0xFFFFFFF9, 说明从线程模式 (main) 进入
// 查看 SP 指向的栈帧 (xPSR, PC, LR, R12, R3, R2, R1, R0)
// 其中 PC 就是出错的指令地址

// 实用技巧: 在 HardFault_Handler 中加:
void HardFault_Handler(void) {
    // 读取当前 SP
    uint32_t *sp;
    __asm volatile("mrs %0, psp" : "=r"(sp));
    // sp[6] = PC (出错地址)
    // sp[7] = xPSR
    while(1);
}
// 然后检查 sp[6] 反汇编定位
```

### 问题 4：RTOS 任务不调度

```c
// 确认 SysTick 已启动
void SysTick_Handler(void) {
    HAL_IncTick();           // HAL 滴答
    osSystickHandler();      // FreeRTOS 滴答 (CMSIS-RTOS)
}

// 确认 PendSV 优先级最低
// NVIC_SetPriority(PendSV_IRQn, 0xFF);
// PendSV 必须在所有可屏蔽中断之后执行

// 确认 SVC_Handler 未在 Default_Handler 死循环
// 否则 osKernelStart() 触发的 SVC 会卡死
```

---

## 十、从启动到 RTOS 调度全链路时间线

以 STM32F407 @ 168MHz 为例，实测时间：

| 阶段 | 周期数 | 时间 | 累计 |
|------|--------|------|------|
| 硬件复位 + 取向量表 | ~10 cycles | 60ns | 60ns |
| Pull RST high → HSE 起振 | 等待晶振稳定 | ~2-4ms | **~4ms** ⏱ |
| .data 拷贝 (1KB) | ~500 cycles | 3μs | ~4ms |
| .bss 清零 (10KB) | ~5000 cycles | 30μs | ~4.03ms |
| SystemInit (PLL 锁定) | ~200μs | 200μs | ~4.23ms |
| __libc_init_array | ~500 cycles | 3μs | ~4.23ms |
| **进入 main()** | | | **~4.23ms** |
| HAL_Init | ~100μs | | ~4.33ms |
| 外设初始化 (GPIO, UART, SPI...) | ~500μs | | ~4.83ms |
| RTOS 任务创建 | ~100μs | | ~4.93ms |
| osKernelStart → 第一个任务 | ~20μs | | ~4.95ms |

**结论**：从复位到 RTOS 第一个任务运行，大约 **5ms** 左右，其中 **4ms 在等晶振起振**。如果需要更快启动（如低功耗唤醒），可以直接用 HSI（内部 RC 振荡器，起振 < 1μs）。

---

## 十一、MCU 以太网 PHY 控制完整过程

> 从硬件接口到寄存器操作，从初始化到链路管理的完整指南

### 11.1 概述：MAC 与 PHY 的分工

以太网通信涉及两个关键芯片：

```
┌─────────────────────────────────────────────────┐
│                   MCU (SoC)                       │
│  ┌─────────────────────────────────────────────┐  │
│  │           Ethernet MAC (介质访问控制层)       │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐   │  │
│  │  │ TX FIFO  │  │ RX FIFO  │  │  DMA     │   │  │
│  │  └──────────┘  └──────────┘  └──────────┘   │  │
│  └──────────────────┬──────────────────────────┘  │
│                     │MII / RMII 总线               │
│  ┌──────────────────▼──────────────────────────┐  │
│  │         Ethernet PHY (物理层芯片)            │  │
│  │           LAN8720 / DP83848 / KSZ8081       │  │
│  └──────────────────┬──────────────────────────┘  │
└─────────────────────┼────────────────────────────┘
                      │
                  ═══ RJ45 (带网络变压器) ═══
```

| 层次 | MAC (MCU 内置) | PHY (外挂芯片) |
|------|---------------|---------------|
| 功能 | 数据帧封装/解封装、MAC 地址过滤、CRC 校验 | 信号编解码、时钟恢复、电平转换 |
| 速率控制 | 流量控制 (Pause Frame) | 自动协商 (Auto-Negotiation) |
| 接口 | 通过 DMA 与系统内存交换数据 | 通过 MII/RMII 与 MAC 连接 |
| 配置 | 通过 MMIO 寄存器控制 | 通过 MDIO/MDC 总线控制 |

**典型 PHY 芯片选型：**

| 芯片 | 接口 | 速率 | 特点 |
|------|------|------|------|
| LAN8720A | RMII | 10/100M | 最常用，25MHz 晶体，内置 1.2V LDO |
| DP83848 | MII/RMII | 10/100M | 工业级，稳定性好，TI 出品 |
| KSZ8081 | RMII | 10/100M | Microchip，超低功耗 |
| RTL8201F | RMII | 10/100M | Realtek，成本低 |
| KSZ9031 | RGMII | 10/100/1000M | 千兆 PHY，用于更高性能场景 |

---

### 11.2 MII / RMII 接口详解

#### 11.2.1 MII (Media Independent Interface) — 标准接口

| MCU 引脚 | PHY 引脚 | 方向 | 宽度 | 功能 |
|---------|---------|------|------|------|
| ETH_TX_CLK | TX_CLK | PHY → MAC | 1 | 发送参考时钟 (2.5/25MHz) |
| ETH_TX_EN | TX_EN | MAC → PHY | 1 | 发送使能 |
| ETH_TXD[3:0] | TXD[3:0] | MAC → PHY | 4 | 发送数据总线 |
| ETH_RX_CLK | RX_CLK | PHY → MAC | 1 | 接收参考时钟 (2.5/25MHz) |
| ETH_RX_DV | RX_DV | PHY → MAC | 1 | 接收数据有效 |
| ETH_RXD[3:0] | RXD[3:0] | PHY → MAC | 4 | 接收数据总线 |
| ETH_RX_ER | RX_ER | PHY → MAC | 1 | 接收错误 |
| ETH_CRS | CRS | PHY → MAC | 1 | 载波侦听 |
| ETH_COL | COL | PHY → MAC | 1 | 碰撞检测 |

**MII 信号时序（100Mbps 模式）：**
- TX_CLK / RX_CLK = 25MHz
- 每个时钟沿，4 位数据同时传输
- 数据速率 = 25M × 4bit = **100Mbps**

```
100M 模式 (时钟 25MHz):      ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐
TX_CLK / RX_CLK (25MHz)    ──┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─
                              │   │   │   │   │   │
TXD[3:0] / RXD[3:0]        ──<D0>──<D1>──<D2>──<D3>── ...
                              │   │   │   │   │   │

10M 模式 (时钟 2.5MHz):       ┌─┐ ┌─┐ ┌─┐ ┌─┐
TX_CLK / RX_CLK (2.5MHz)   ──┘ └─┘ └─┘ └─┘ └─┘
                              │   │   │   │
TXD[3:0] / RXD[3:0]        ──<D0>──<D1>──<D2>──<D3>── ...
```

> **MII 共需 18 个引脚**，对引线较多的小封装 MCU 不友好。因此 RMII 应运而生。

---

#### 11.2.2 RMII (Reduced MII) — 简化接口（实际项目最常用）

| MCU 引脚 | PHY 引脚 | 方向 | 宽度 | 功能 |
|---------|---------|------|------|------|
| ETH_REF_CLK | REF_CLK | PHY → MAC | 1 | 50MHz 参考时钟 |
| ETH_TX_EN | TX_EN | MAC → PHY | 1 | 发送使能 |
| ETH_TXD[1:0] | TXD[1:0] | MAC → PHY | 2 | 发送数据（仅 2 位） |
| ETH_RX_DV | CRS_DV | PHY → MAC | 1 | 载波侦听 + 数据有效（复合信号） |
| ETH_RXD[1:0] | RXD[1:0] | PHY → MAC | 2 | 接收数据（仅 2 位） |
| ETH_RX_ER | RX_ER | PHY → MAC | 1 | 接收错误（可选） |
| MDC | MDC | MAC → PHY | 1 | 管理时钟 |
| MDIO | MDIO | 双向 | 1 | 管理数据 |

**RMII 到 MII 的对应关系：**

```
MII:   TX_CLK + TX_EN + TXD[3:0] + RX_CLK + RX_DV + RXD[3:0] + CRS + COL = 14 信号线
RMII:  REF_CLK + TX_EN + TXD[1:0] + CRS_DV + RXD[1:0] + RX_ER    =  7 信号线 (减半!)
```

**关键转换细节：**

RMII 的 `CRS_DV` 是一个复合信号，由 PHY 内部将 MII 的 `CRS` 和 `RX_DV` 合并而成：

```
MII:   CRS  ─┐
              ├── OR ──→ RMII: CRS_DV
       RX_DV ─┘
```

> RMII 在 PHY 侧仍然需要完整的 MII 信号处理，芯片内部完成合并。对 MCU 来说，RMII 只需 **7 条信号线 + MDC/MDIO**，总计 9 条线。

**RMII 50MHz 时钟的特殊性：**

```
方案 A（推荐）：50MHz 由 PHY 产生，输出给 MCU
  25MHz 晶振 ─→ PHY (LAN8720) ──50MHz──→ MCU ETH_REF_CLK

方案 B（不推荐）：50MHz 由 MCU 产生，输出给 PHY
  MCU ETH_REF_CLK ──50MHz──→ PHY (部分 PHY 不支持)

方案 C（外部有源 OSC）：
  50MHz OSC ──→ 分到 PHY + MCU
```

> **为什么方案 A 最推荐？**
> PHY 内部有 PLL，从 25MHz 晶振倍频到 50MHz。MCU 的 REF_CLK 输入引脚（如 PA1）不需要自己产生高精度时钟，减少时钟抖动问题。

---

### 11.3 SMI/MDIO 协议（寄存器读写机制）

#### 11.3.1 物理层

SMI (Station Management Interface) = MDC (时钟) + MDIO (数据) 两条线：

```
┌──────┐          MDC (管理时钟)        ┌──────┐
│      │├─────────────────────────────→│      │
│ MAC  │          MDIO (管理数据)       │ PHY  │
│(MCU) │├─────────────────────────────→│(PHY1)│
│      │          (双向，需上拉 1.5kΩ)  │      │
└──────┘                               ├──────┤
                                       │ PHY  │
                                       │(PHY2)│  ← 最多 32 个 PHY
                                       └──────┘
```

- **MDC**：由 MAC 驱动，最大频率 2.5MHz（有些 PHY 支持 25MHz）
- **MDIO**：双向引脚，MAC 是主控，PHY 是从设备
- **PHY 地址**：每个 PHY 通过 `PHYAD[0:4]` 引脚配置独立地址（1-31），决定了 MDIO 帧中的目标地址

#### 11.3.2 MDIO 帧格式

标准 MDIO 协议（IEEE 802.3 Clause 22）的帧格式：

```
 读操作（MAC → PHY → MAC）:
 ┌────────┬───────┬──────┬──────┬───────┬─────────┬──────┐
 │ PRE    │ ST    │ OP   │PHYAD │REGAD  │ TA      │ DATA │
 │32×"1"  │"01"   │"10"  │5bit  │5bit   │"Z0"     │16bit │
 └────────┴───────┴──────┴──────┴───────┴─────────┴──────┘
   MAC 驱动  MAC 驱动  MAC    MAC   MAC    MAC→PHY   PHY→MAC

 写操作（MAC → PHY）:
 ┌────────┬───────┬──────┬──────┬───────┬─────────┬──────┐
 │ PRE    │ ST    │ OP   │PHYAD │REGAD  │ TA      │ DATA │
 │32×"1"  │"01"   │"01"  │5bit  │5bit   │"10"     │16bit │
 └────────┴───────┴──────┴──────┴───────┴─────────┴──────┘
   MAC 驱动  MAC     MAC   MAC    MAC     MAC       MAC
```

| 字段 | 说明 |
|------|------|
| PRE (Preamble) | 32 个连续的 "1"，用于同步 PHY 时钟 |
| ST (Start) | 起始码，"01" |
| OP (Operation) | "01" = 写, "10" = 读 |
| PHYAD | PHY 地址（0x00-0x1F），由 PHY 的 ADDR 引脚决定 |
| REGAD | 寄存器地址（0x00-0x1F），每个 PHY 有 32 个寄存器 |
| TA (Turnaround) | 读: MAC 释放总线(Z)，PHY 驱动"0"；写: MAC 驱动"10" |
| DATA | 16 位数据 |

#### 11.3.3 MCU 侧裸机实现（GPIO 模拟 MDIO）

若 MCU 没有内置 ETH 控制器（或内置 MAC 不支持 MDIO），可用 GPIO 模拟：

```c
/* GPIO 模拟 MDIO 读写 (Bit-Bang 实现) */
#define MDC_PIN    GPIO_PIN_12  /* PC12 */
#define MDIO_PIN   GPIO_PIN_11  /* PC11 */
#define MDIO_PORT  GPIOC

/* ── MDC 时钟脉冲 ── */
static void mdio_clock_pulse(void) {
    HAL_GPIO_WritePin(MDIO_PORT, MDC_PIN, GPIO_PIN_SET);
    delay_100ns();                        // 保持高电平 > 80ns
    HAL_GPIO_WritePin(MDIO_PORT, MDC_PIN, GPIO_PIN_RESET);
    delay_100ns();                        // 保持低电平 > 80ns
    // 完整周期 ≈ 400ns → MDC ≈ 2.5MHz
}

/* ── MDIO 引脚方向控制 ── */
static void mdio_set_output(void) {
    GPIO_InitTypeDef gpio = {0};
    gpio.Pin = MDIO_PIN;
    gpio.Mode = GPIO_MODE_OUTPUT_PP;
    gpio.Pull = GPIO_NOPULL;
    gpio.Speed = GPIO_SPEED_FREQ_HIGH;
    HAL_GPIO_Init(MDIO_PORT, &gpio);
}

static void mdio_set_input(void) {
    GPIO_InitTypeDef gpio = {0};
    gpio.Pin = MDIO_PIN;
    gpio.Mode = GPIO_MODE_INPUT;
    gpio.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(MDIO_PORT, &gpio);
}

/* ── MDIO 写 16 位数据 ── */
static void mdio_write_bits(uint16_t data, uint8_t bits) {
    for (uint8_t i = bits; i > 0; i--) {
        HAL_GPIO_WritePin(MDIO_PORT, MDIO_PIN,
            (data >> (i - 1)) & 1 ? GPIO_PIN_SET : GPIO_PIN_RESET);
        mdio_clock_pulse();
    }
}

/* ── MDIO 读 16 位数据 ── */
static uint16_t mdio_read_bits(uint8_t bits) {
    uint16_t data = 0;
    for (uint8_t i = 0; i < bits; i++) {
        data <<= 1;
        data |= (HAL_GPIO_ReadPin(MDIO_PORT, MDIO_PIN) == GPIO_PIN_SET) ? 1 : 0;
        mdio_clock_pulse();
    }
    return data;
}

/* ── PHY 寄存器写入 (SMI 写帧) ── */
void phy_reg_write(uint8_t phy_addr, uint8_t reg_addr, uint16_t data) {
    mdio_set_output();

    /* PRE: 32 个 "1" */
    mdio_write_bits(0xFFFF, 16);
    mdio_write_bits(0xFFFF, 16);

    /* ST(2) + OP(2) + PHYAD(5) + REGAD(5) + TA(2) + DATA(16) = 32 */
    mdio_write_bits(0x01, 2);        // ST = "01"
    mdio_write_bits(0x01, 2);        // OP = "01" (写)
    mdio_write_bits(phy_addr, 5);    // PHY 地址
    mdio_write_bits(reg_addr, 5);    // 寄存器地址
    mdio_write_bits(0x02, 2);        // TA = "10" (写操作)
    mdio_write_bits(data, 16);       // 16 位数据

    mdio_set_input();  // 释放 MDIO
}

/* ── PHY 寄存器读取 (SMI 读帧) ── */
uint16_t phy_reg_read(uint8_t phy_addr, uint8_t reg_addr) {
    uint16_t data;
    mdio_set_output();

    /* PRE: 32 个 "1" */
    mdio_write_bits(0xFFFF, 16);
    mdio_write_bits(0xFFFF, 16);

    /* ST(2) + OP(2) + PHYAD(5) + REGAD(5) */
    mdio_write_bits(0x01, 2);        // ST = "01"
    mdio_write_bits(0x02, 2);        // OP = "10" (读)
    mdio_write_bits(phy_addr, 5);    // PHY 地址
    mdio_write_bits(reg_addr, 5);    // 寄存器地址

    /* TA: MAC 释放 MDIO，等待 PHY 驱动 "0" */
    mdio_set_input();
    mdio_clock_pulse();              // TA 第 1 拍: PHY 拉低 (Z 状态)
    mdio_clock_pulse();              // TA 第 2 拍: PHY 输出 "0"

    /* DATA: 从 PHY 读 16 位 */
    data = mdio_read_bits(16);

    return data;
}
```

#### 11.3.4 HAL 库封装（使用 MCU 内置 MAC 的 MDIO）

如果 MCU 内置 MAC 控制器（如 STM32F4/F7/H7），可以直接用 HAL 的 MDIO 接口：

```c
/* ── 使用 STM32 HAL 库的 MDIO 读写 ── */

/* 读取 PHY 寄存器 */
uint32_t HAL_ETH_ReadPHYRegister(ETH_HandleTypeDef *heth,
                                  uint32_t PHYAddr, uint32_t PHYReg) {
    uint32_t tmpreg, timeout = 0;
    uint32_t phyreg = 0;

    /* 等待 MAC 的 MDIO 控制器空闲 */
    while ((heth->Instance->MACMIIAR & ETH_MACMIIAR_MB) && (timeout < 1000))
        timeout++;
    if (timeout >= 1000) return 0xFFFF;

    /* 写 MII 地址寄存器: 触发读操作 */
    tmpreg = (PHYAddr << 11) | (PHYReg << 6) | ETH_MACMIIAR_MW;
    heth->Instance->MACMIIAR = tmpreg & ~ETH_MACMIIAR_MW;  // MB=0, CR=读

    /* 等待操作完成 */
    timeout = 0;
    while ((heth->Instance->MACMIIAR & ETH_MACMIIAR_MB) && (timeout < 1000))
        timeout++;
    if (timeout >= 1000) return 0xFFFF;

    /* 读取数据 */
    phyreg = heth->Instance->MACMIIDR;
    return phyreg;
}

/* 写入 PHY 寄存器 */
void HAL_ETH_WritePHYRegister(ETH_HandleTypeDef *heth,
                               uint32_t PHYAddr, uint32_t PHYReg,
                               uint32_t Data) {
    uint32_t tmpreg, timeout = 0;

    /* 等待空闲 */
    while ((heth->Instance->MACMIIAR & ETH_MACMIIAR_MB) && (timeout < 1000))
        timeout++;
    if (timeout >= 1000) return;

    /* 写数据寄存器 */
    heth->Instance->MACMIIDR = Data;

    /* 写 MII 地址寄存器: 触发写操作 */
    tmpreg = (PHYAddr << 11) | (PHYReg << 6) | ETH_MACMIIAR_MW;
    heth->Instance->MACMIIAR = tmpreg | ETH_MACMIIAR_MB;  // MB=1, 启动传输

    /* 等待完成 */
    timeout = 0;
    while ((heth->Instance->MACMIIAR & ETH_MACMIIAR_MB) && (timeout < 1000))
        timeout++;
}
```

> **MACMIIAR 寄存器关键位：**
> - Bit[0] (MB): 忙标志，写 1 启动传输，硬件完成时清零
> - Bit[1] (MW): 1=读, 0=写
> - Bit[5:2] (CR): MDC 时钟分频 (STM32F4: 2=HCLK/42, 3=HCLK/62)
> - Bit[10:6] (PR): PHY 地址
> - Bit[15:11] (PA): 寄存器地址

---

### 11.4 IEEE 802.3 PHY 标准寄存器详解

IEEE 802.3 标准定义了 PHY 的基本寄存器集 (Clause 22)，地址范围 0x00 - 0x1F 共 32 个：

#### 寄存器 0x00 — BMCR (Basic Mode Control Register)

```
位      名称                    说明
────────────────────────────────────────────────────────
15      Reset                   1=软件复位 (自清除)
14      Loopback                1=回环模式 (用于测试)
13      Speed Select (LSB)      与位 6 组合:
                                    00=10M  01=100M
                                    10=1000M  11=Reserved
12      Auto-Negotiation Enable 1=使能自动协商
11      Power Down              1=省电模式
10      Isolate                 1=隔离模式 (MII 输出高阻)
9       Restart Auto-Neg        1=重启自动协商 (自清除)
8       Duplex Mode             1=全双工  0=半双工
7       Collision Test          1=碰撞测试使能
6       Speed Select (MSB)      与位 13 组合 (见上)
5:0     Reserved / 厂商自定义
```

**典型配置值：**

```c
/* 常见 BMCR 值 */
#define PHY_BMCR_RESET          0x8000   // Bit 15: 软件复位
#define PHY_BMCR_AUTONEG_EN     0x1200   // Bit 12 + Bit 9: 使能自动协商
#define PHY_BMCR_AUTONEG_RSTRT  0x1200   // 注意: 写入时包含 bit12
#define PHY_BMCR_100M_FULL      0x2100   // Bit 13 + Bit 8: 100M 全双工 (强制)
#define PHY_BMCR_100M_HALF      0x2000   // Bit 13: 100M 半双工 (强制)
#define PHY_BMCR_10M_FULL       0x0100   // Bit 8: 10M 全双工 (强制)
#define PHY_BMCR_10M_HALF       0x0000   // 10M 半双工 (强制)

/* 强制固定 100M 全双工 + 关闭自动协商 */
phy_reg_write(PHY_ADDR, 0x00, 0x2100);

/* 使能自动协商 (推荐) */
phy_reg_write(PHY_ADDR, 0x00, 0x1200);
```

#### 寄存器 0x01 — BMSR (Basic Mode Status Register)

```
位      名称                    说明
────────────────────────────────────────────────────────
15      100BASE-T4 Capable      100Base-T4 能力
14      100BASE-TX Full Duplex  100Base-TX 全双工能力
13      100BASE-TX Half Duplex  100Base-TX 半双工能力
12      10BASE-T Full Duplex    10Base-T 全双工能力
11      10BASE-T Half Duplex    10Base-T 半双工能力
10      100BASE-T2 Full Duplex  100Base-T2 (保留)
9       100BASE-T2 Half Duplex  100Base-T2 (保留)
8       Extended Status         扩展状态 (寄存器 0x0F 有更多信息)
7       Reserved                保留
6       MF Preamble Suppress    支持忽略 Preamble
5       Auto-Negotiation Complete   自动协商完成 (关键!)
4       Remote Fault            远端故障
3       Auto-Negotiation Capable    支持自动协商能力
2       Link Status             1=链路已建立 (只读, 延迟锁存!)
1       Jabber Detect           Jabber 检测
0       Extended Capability     扩展能力 (1=有扩展寄存器)
```

> **⚠️ 位 2 (Link Status) 是延迟锁存位**：链路断开时硬件清 0，但链路恢复时**不会自动置 1**，必须读两次或复位后第一次读才得到正确值。

```c
/* 正确读取链路状态的方式 */
uint16_t read_link_status(uint8_t phy_addr) {
    uint16_t status;

    /* 第一次读: 清除延迟锁存 */
    status = phy_reg_read(phy_addr, 0x01);

    /* 延时一小段时间 */
    delay_ms(10);

    /* 第二次读: 得到真实链路状态 */
    status = phy_reg_read(phy_addr, 0x01);

    return (status & 0x0004) ? 1 : 0;  // bit2 = Link Status
}
```

#### 寄存器 0x02-0x03 — PHY Identifier (芯片标识)

```
寄存器 0x02 (PHYIDR1):
  Bits[15:0] = OUI[21:6]           // OUI 高 16 位

寄存器 0x03 (PHYIDR2):
  Bits[15:10] = OUI[5:0]           // OUI 低 6 位
  Bits[9:4]   = Model Number       // 型号编号
  Bits[3:0]   = Revision Number    // 版本号
```

**常见 PHY 的 ID：**

| PHY 芯片 | ID (两个寄存器合并) | 区分方法 |
|---------|-------------------|---------|
| LAN8720A | 0x0007C0F1 | Model=0x1F, Rev=0x1 |
| DP83848 | 0x20005C90 | OUI=0x080028 |
| KSZ8081 | 0x00221560 | OUI=0x00110A |
| KSZ9031 | 0x00221622 | 千兆 RGMII |
| RTL8201F | 0x001CC... | 需查看 datasheet |

```c
/* 自动识别 PHY 型号 */
uint8_t detect_phy_model(uint8_t phy_addr) {
    uint16_t id1 = phy_reg_read(phy_addr, 0x02);
    uint16_t id2 = phy_reg_read(phy_addr, 0x03);
    uint32_t phy_id = (id1 << 16) | id2;

    printf("PHY ID = 0x%08X\r\n", phy_id);

    switch (phy_id) {
        case 0x0007C0F1:
            return PHY_MODEL_LAN8720;
        case 0x20005C90:
            return PHY_MODEL_DP83848;
        case 0x00221560:
            return PHY_MODEL_KSZ8081;
        default:
            return PHY_MODEL_UNKNOWN;
    }
}
```

#### 寄存器 0x04 — ANAR (Auto-Negotiation Advertisement Register)

```
位      名称                    说明
────────────────────────────────────────────────────────
15       Next Page              下一页面 (扩展协商)
14       Reserved
13       Remote Fault            远端故障
12       Technology Ability      (保留)
11       Asymmetric Pause       非对称暂停 (流控)
10       Symmetric Pause        对称暂停 (流控)
9       100BASE-T4              100Base-T4 支持
8       100BASE-TX Full Duplex  100Base-TX 全双工能力
7       100BASE-TX Half Duplex  100Base-TX 半双工能力
6       10BASE-T Full Duplex    10Base-T 全双工能力
5       10BASE-T Half Duplex    10Base-T 半双工能力
4:0     Selector                协议选择器 (00001=IEEE 802.3)
```

```c
/* 宣告本端能力: 100M 全双工 + 100M 半双工 + 10M 全/半双工 */
phy_reg_write(PHY_ADDR, 0x04,
    (1 << 9) |   // 100BASE-T4  (通常不设置)
    (1 << 8) |   // 100BASE-TX Full Duplex
    (1 << 7) |   // 100BASE-TX Half Duplex
    (1 << 6) |   // 10BASE-T Full Duplex
    (1 << 5) |   // 10BASE-T Half Duplex
    0x0001);     // Selector = 802.3
```

#### 寄存器 0x05 — ANLPAR (Auto-Negotiation Link Partner Ability)

> 对端设备宣告的能力——**只读**，自动协商结果由此决定。

```c
uint16_t lp_cap = phy_reg_read(PHY_ADDR, 0x05);

/* 检查对端支持的最高速率 */
if (lp_cap & (1 << 8))
    printf("Partner supports: 100BASE-TX Full Duplex\r\n");
if (lp_cap & (1 << 7))
    printf("Partner supports: 100BASE-TX Half Duplex\r\n");
if (lp_cap & (1 << 6))
    printf("Partner supports: 10BASE-T Full Duplex\r\n");
if (lp_cap & (1 << 5))
    printf("Partner supports: 10BASE-T Half Duplex\r\n");
```

#### 寄存器 0x06 — ANER (Auto-Negotiation Expansion Register)

```
位      名称                    说明
────────────────────────────────────────────────────────
0       Link Partner Auto-Negotiation Able    对端支持自动协商吗？
1       Link Partner Next Page Able          对端支持下一页面
2       Local Next Page Able                 本端支持下一页面
3       Parallel Detection Fault             并行检测故障
4       Link Partner Code Word Ack           对端确认了能力码字
```

#### 寄存器 0x1F — 特殊控制/状态（厂商自定义）

不同 PHY 的寄存器 0x1F 功能不同，**这是 PHY 移植工作中最关键的差异点**：

```c
/* ── LAN8720 的特殊寄存器 0x1F ── */
#define LAN8720_MCSR      0x1F    /* Mode Control Status Register */

/* LAN8720 MCSR 位定义 */
#define LAN8720_MCSR_100M    (1 << 14)   // 读: 1=100M, 0=10M
#define LAN8720_MCSR_FULLDUP (1 << 13)   // 读: 1=全双工, 0=半双工
#define LAN8720_MCSR_ALE     (1 << 12)   // 自动协商完成事件
#define LAN8720_MCSR_LS      (1 << 10)   // 链路状态 (1=up)

/* ── DP83848 的特殊寄存器 ── */
#define DP83848_STS         0x10    /* Status Register */
#define DP83848_STS_100M     (1 << 14)
#define DP83848_STS_FULLDUP  (1 << 13)
#define DP83848_STS_LINK     (1 << 1)

/* ── KSZ8081 的特殊寄存器 ── */
#define KSZ8081_PHYCR1      0x1F    /* PHY Control Register 1 */
#define KSZ8081_LNKSTS      0x1E    /* Link Status Register */
```

---

### 11.5 PHY 初始化全流程

完整的 PHY 初始化序列（从上电到可以收发数据）：

```
  时间轴
   │
   ├── ① 硬件上电与复位
   │      - PHY 供电 (3.3V + 1.2V core, 取决于芯片)
   │      - RST 引脚拉低 > 10ms 后释放
   │      - 等待内部 PLL 锁定 (约 5ms)
   │
   ├── ② MCU 侧 MAC 时钟使能
   │      - RCC 时钟: ETH_MAC, ETH_RX, ETH_TX
   │      - 配置 REF_CLK 引脚 (PA1 或其他)
   │
   ├── ③ GPIO 引脚配置
   │      - MII/RMII 引脚: 复用功能推挽输出
   │      - MDC/MDIO: 复用功能开漏 (外部上拉)
   │
   ├── ④ 读取 PHY ID (验证连接)
   │      - 读寄存器 0x02, 0x03
   │      - 匹配 PHY 型号
   │
   ├── ⑤ 软件复位
   │      - BMCR.15 = 1
   │      - 等待自清除 (轮询直到 bit15=0)
   │
   ├── ⑥ 配置 PHY 基本参数
   │      - 方案 A: 使能自动协商 (推荐)
   │      - 方案 B: 强制固定速率/双工
   │
   ├── ⑦ 等待自动协商完成 (如果使能)
   │      - 轮询 BMSR.5 (Auto-Negotiation Complete)
   │      - 超时处理 (通常 2-4 秒)
   │
   ├── ⑧ 读取协商结果
   │      - 速率: 100M / 10M
   │      - 双工: Full / Half
   │
   ├── ⑨ 配置 MAC 匹配 PHY 速率/双工
   │      - STM32: ETH_MACCR 寄存器
   │
   ├── ⑩ 使能 PHY 中断 (可选)
   │      - 链路状态变化通知
   │
   ├── ⑪ 使能 MAC DMA / 启动收发
   │      - ETH_DMAOMR 寄存器
   │      - 使能 DMA 发送/接收
   │
   └── ⑫ 开始收发数据
          - 链路已建立，可以 ping 通
```

---

### 11.6 自动协商（Auto-Negotiation）深度解析

#### 什么是自动协商？

自动协商（ANEG）是 IEEE 802.3 定义的协议，使链路上的两个设备能自动选择双方都支持的最高速率和双工模式。

```
  ┌──────┐                    ┌──────┐
  │ MAC1 │── PHY A ──── PHY B ──│ MAC2 │
  │100M-FD│  ════════════   │100M-FD│
  │100M-HD│   自动协商       │10M-FD │
  │10M-FD │     ↓           │10M-HD │
  │10M-HD │  100M FD ←── 共同最高能力
  └──────┘                    └──────┘
```

#### 自动协商过程（FLP Burst）

自动协商通过 **FLP (Fast Link Pulse)** 脉冲串交换能力信息：

```
正常链路脉冲 (NLP) —— 10M 模式下周期性脉冲:
          ┌┐                    ┌┐
  ────────┘└────────────────────┘└────────

快速链路脉冲 (FLP) —— 自动协商时的脉冲串:
  ┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐
  └┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘└┘
  ├─── 17 个脉冲位置 ───┤─── 16 个数据位 ──┤
  │  固定时钟位          │  FLP 码字 (Base Page 或 Next Page)
```

FLP Burst 以 **16ms** 为间隔连续发送，直到协商完成或超时。

#### 优先级裁决规则

当双方的能力码字交换完成后，按以下优先级选择共同能力：

```
优先级 (高→低)
 1.  1000BASE-T Full Duplex       (千兆全双工)
 2.  1000BASE-T Half Duplex       (千兆半双工)
 3.  100BASE-T2 Full Duplex       (百兆双绞线全双工)
 4.  100BASE-TX Full Duplex       (百兆全双工) ← 最常见
 5.  100BASE-T2 Half Duplex       (百兆双绞线半双工)
 6.  100BASE-TX Half Duplex       (百兆半双工)
 7.  100BASE-T4                   (百兆四对线)
 8.  10BASE-T Full Duplex         (十兆全双工)
 9.  10BASE-T Half Duplex         (十兆半双工) ← 最低
```

#### 并行检测 (Parallel Detection)

当对端设备**不支持**自动协商（例如只有 10BASE-T 的旧设备），PHY 使用并行检测：

```
  本端 (ANEG 使能)              对端 (无 ANEG)
  发送 FLP Burst ───────────→  不响应 FLP
       │
       └→ PHY 检测到链路脉冲 (NLP)
          或 100M 的 MLT3 编码
              │
              └→ 根据检测到的信号类型
                 锁定为 10M 半双工 (NLP)
                 或 100M 半双工 (MLT3)
```

> **并行检测的一个风险**：由于无法交换能力信息，并行检测的结果只能是**半双工**（除非对端也强制全双工并手动配置）。这就是为什么自动协商更推荐。

#### 自动协商超时处理

```c
#define AUTONEG_TIMEOUT_MS  3000   // 标准超时 3 秒

uint8_t phy_wait_autoneg_complete(uint8_t phy_addr) {
    uint32_t start = HAL_GetTick();
    uint16_t bmsr;

    do {
        /* 读两次 BMSR (清除延迟锁存) */
        bmsr = phy_reg_read(phy_addr, 0x01);
        bmsr = phy_reg_read(phy_addr, 0x01);

        if (bmsr & (1 << 5)) {
            printf("Auto-Negotiation COMPLETE\r\n");
            return 1;   // 成功
        }

        delay_ms(50);
    } while ((HAL_GetTick() - start) < AUTONEG_TIMEOUT_MS);

    printf("Auto-Negotiation TIMEOUT (%d ms)\r\n", AUTONEG_TIMEOUT_MS);
    return 0;   // 超时
}
```

---

### 11.7 链路状态检测与中断管理

#### 11.7.1 轮询方式

```c
/* ── PHY 链路监控任务 (在 RTOS 中每 500ms 执行一次) ── */
void ethernet_link_monitor_task(void *arg) {
    uint8_t last_link = 0;

    while (1) {
        uint8_t current_link = read_link_status(PHY_ADDR);

        if (current_link != last_link) {
            if (current_link) {
                /* 链路 UP：读取协商结果 */
                ETH_PHY_StatusTypeDef link_info;
                phy_get_link_info(PHY_ADDR, &link_info);

                printf("🎯 Link UP: %s %s Duplex\r\n",
                    link_info.speed == PHY_SPEED_100M ? "100M" : "10M",
                    link_info.duplex == PHY_DUPLEX_FULL ? "Full" : "Half");

                /* 通知 MAC 切换速率/双工 */
                eth_mac_config_speed_duplex(link_info.speed, link_info.duplex);
            } else {
                printf("🔌 Link DOWN\r\n");
            }
            last_link = current_link;
        }

        delay_ms(500);  // FreeRTOS: vTaskDelay(pdMS_TO_TICKS(500));
    }
}

/* ── 获取协商结果 (带 PHY 型号适配) ── */
typedef struct {
    uint8_t speed;   // 0=10M, 1=100M
    uint8_t duplex;  // 0=half, 1=full
} ETH_PHY_StatusTypeDef;

void phy_get_link_info(uint8_t phy_addr, ETH_PHY_StatusTypeDef *info) {
    uint16_t reg;

    /* 方法 1: 通过 BMSR + ANLPAR (标准方式) */
    uint16_t anlpar = phy_reg_read(phy_addr, 0x05);

    if (anlpar & (1 << 8)) {      // 100BASE-TX Full Duplex
        info->speed = PHY_SPEED_100M;
        info->duplex = PHY_DUPLEX_FULL;
    } else if (anlpar & (1 << 7)) { // 100BASE-TX Half Duplex
        info->speed = PHY_SPEED_100M;
        info->duplex = PHY_DUPLEX_HALF;
    } else if (anlpar & (1 << 6)) { // 10BASE-T Full Duplex
        info->speed = PHY_SPEED_10M;
        info->duplex = PHY_DUPLEX_FULL;
    } else {                        // 10BASE-T Half Duplex
        info->speed = PHY_SPEED_10M;
        info->duplex = PHY_DUPLEX_HALF;
    }

    /* 方法 2: 通过厂商特定寄存器 (更直接) */
    // LAN8720: 0x1F bit14=100M, bit13=Full
    // reg = phy_reg_read(phy_addr, 0x1F);
    // info->speed  = (reg & (1 << 14)) ? 100 : 10;
    // info->duplex = (reg & (1 << 13)) ? 1 : 0;
}
```

#### 11.7.2 中断方式

```c
/* ── 启用 PHY 链路状态变化中断 ── */
void phy_enable_link_interrupt(uint8_t phy_addr) {
    /* 步骤 1: 使能 PHY 的中断 (不同 PHY 寄存器不同) */
    // LAN8720: 寄存器 0x1B = 中断屏蔽寄存器
    //           bit14 (EnergyOn) + bit12 (AutoNegComplete) + bit10 (LinkDown)
    phy_reg_write(phy_addr, 0x1B, (1 << 14) | (1 << 12) | (1 << 10));

    // DP83848: 寄存器 0x11 = MISR1, 0x12 = MISR2
    // phy_reg_write(phy_addr, 0x11, (1 << 0));  // Link Status Change

    /* 步骤 2: 读取并清除初始中断状态 */
    // LAN8720: 读 0x1C = 中断源寄存器 (清除操作)
    phy_reg_read(phy_addr, 0x1C);
}

/* ── PHY 中断服务程序 ── */
void EXTI15_10_IRQHandler(void) {
    if (__HAL_GPIO_EXTI_GET_IT(PHY_INT_PIN) != RESET) {
        __HAL_GPIO_EXTI_CLEAR_IT(PHY_INT_PIN);

        /* 读取中断源 */
        uint16_t int_src = phy_reg_read(PHY_ADDR, 0x1C);

        if (int_src & (1 << 10)) {          // Link Down
            eth_notify_link_down();
        }
        if (int_src & (1 << 12)) {          // AutoNeg Complete
            /* 读取协商结果并配置 MAC */
            ETH_PHY_StatusTypeDef info;
            phy_get_link_info(PHY_ADDR, &info);
            eth_mac_config_speed_duplex(info.speed, info.duplex);
            eth_notify_link_up();
        }

        /* 通知 RTOS */
        // xSemaphoreGiveFromISR(xEthEventSem, NULL);
    }
}
```

---

### 11.8 实战：STM32F407 + LAN8720 完整初始化代码

这是实际项目中最常见的组合。STM32F4 Discovery 开发板内置 LAN8720A。

#### 硬件连接

```
STM32F407ZG                   LAN8720A
┌────────────┐               ┌──────────┐
│ PA1  ──────├───────────────┤ REF_CLK  │  ← 50MHz (由 PHY 提供)
│ PA2  ──────├───────────────┤ MDIO     │
│ PC1  ──────├───────────────┤ MDC      │
│ PG11 ──────├───────────────┤ TX_EN    │
│ PG13 ──────├───────────────┤ TXD0     │
│ PG14 ──────├───────────────┤ TXD1     │
│ PD3  ──────├───────────────┤ CRS_DV   │  ← RX_DV + CRS 复合
│ PC4  ──────├───────────────┤ RXD0     │
│ PC5  ──────├───────────────┤ RXD1     │
│ PB0  ──────├───────────────┤ RST      │  ← RESET (低电平复位)
│ PA8  ──────├───────────────┤ INT      │  ← PHY 中断输出 (可选)
│            │               │          │
│ 3.3V ──────┼───────────────┤ VDD(3.3) │
│            │               │ VDDCR(1.2)│  ← LAN8720 内置 LDO，悬空
│            │               │          │
│            │               │ TX+/TX-  ├──→ RJ45 变压器
│            │               │ RX+/RX-  ├──→
└────────────┘               └──────────┘

PHY 地址配置:
  LAN8720A 的 ADDR 引脚 (pin 9) 拉低 → PHY 地址 = 0x00
  LAN8720A 的 ADDR 引脚 (pin 9) 拉高 → PHY 地址 = 0x01
  (通常 Discovery 板默认 0x00)
```

#### 完整初始化代码

```c
/* ═══════════════════════════════════════════════════════
 * STM32F407 + LAN8720A 以太网初始化 (HAL 库 + LwIP)
 * ═══════════════════════════════════════════════════════ */

#include "stm32f4xx_hal.h"
#include "lwip/init.h"

/* ── PHY 地址 ── */
#define LAN8720_PHY_ADDR        0x00

/* ── 关键延时 (LAN8720 datasheet) ── */
#define PHY_RESET_PULSE_MS      20      // 复位脉冲宽度
#define PHY_PLL_LOCK_MS         5       // PLL 锁定时间
#define PHY_POST_RESET_WAIT_MS  30      // 复位后等待

/* ── LAN8720 寄存器 ── */
#define LAN8720_BMCR            0x00    // 基本模式控制
#define LAN8720_BMSR            0x01    // 基本模式状态
#define LAN8720_PHYIDR1         0x02    // PHY ID1
#define LAN8720_PHYIDR2         0x03    // PHY ID2
#define LAN8720_ANAR            0x04    // 自动协商通告
#define LAN8720_ANLPAR          0x05    // 对端能力
#define LAN8720_ANER            0x06    // 自动协商扩展
#define LAN8720_MCSR            0x1F    // 模式控制状态 (关键!)
#define LAN8720_ISR             0x1B    // 中断屏蔽
#define LAN8720_ISR_SRC         0x1C    // 中断源

/* ── BMCR 位定义 ── */
#define BMCR_RESET              (1 << 15)
#define BMCR_LOOPBACK           (1 << 14)
#define BMCR_SPEED_LSB          (1 << 13)
#define BMCR_AUTONEG            (1 << 12)
#define BMCR_POWERDOWN          (1 << 11)
#define BMCR_ISOLATE            (1 << 10)
#define BMCR_RESTART_AUTONEG    (1 << 9)
#define BMCR_FULLDUPLEX         (1 << 8)
#define BMCR_SPEED_MSB          (1 << 6)

/* ── MCSR (0x1F) 位定义 ── */
#define MCSR_100M               (1 << 14)
#define MCSR_FULLDUP            (1 << 13)
#define MCSR_ALE                (1 << 12)   // AutoNeg 完成事件
#define MCSR_LINKUP             (1 << 10)

/* ── 全局句柄 ── */
ETH_HandleTypeDef heth;

/* ════════════════════════════════════════════
 * Step 1: GPIO 引脚初始化 (RMII 模式)
 * ════════════════════════════════════════════ */
static void MX_ETH_GPIO_Init(void) {
    GPIO_InitTypeDef gpio = {0};

    /* 使能 GPIO 时钟 */
    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_GPIOB_CLK_ENABLE();
    __HAL_RCC_GPIOC_CLK_ENABLE();
    __HAL_RCC_GPIOD_CLK_ENABLE();
    __HAL_RCC_GPIOG_CLK_ENABLE();

    /* ── RMII 信号引脚配置表 ── */
    const struct {
        GPIO_TypeDef *port; uint16_t pin; uint8_t af;
    } rmii_pins[] = {
        /* PA1:  ETH_REF_CLK  (50MHz from PHY)  → AF11 */
        {GPIOA, GPIO_PIN_1,  GPIO_AF11_ETH},
        /* PA2:  ETH_MDIO                        → AF11 */
        {GPIOA, GPIO_PIN_2,  GPIO_AF11_ETH},
        /* PC1:  ETH_MDC                         → AF11 */
        {GPIOC, GPIO_PIN_1,  GPIO_AF11_ETH},
        /* PG11: ETH_TX_EN                       → AF11 */
        {GPIOG, GPIO_PIN_11, GPIO_AF11_ETH},
        /* PG13: ETH_TXD0                        → AF11 */
        {GPIOG, GPIO_PIN_13, GPIO_AF11_ETH},
        /* PG14: ETH_TXD1                        → AF11 */
        {GPIOG, GPIO_PIN_14, GPIO_AF11_ETH},
        /* PD3:  ETH_RX_DV / CRS_DV             → AF11 */
        {GPIOD, GPIO_PIN_3,  GPIO_AF11_ETH},
        /* PC4:  ETH_RXD0                        → AF11 */
        {GPIOC, GPIO_PIN_4,  GPIO_AF11_ETH},
        /* PC5:  ETH_RXD1                        → AF11 */
        {GPIOC, GPIO_PIN_5,  GPIO_AF11_ETH},
    };

    gpio.Mode  = GPIO_MODE_AF_PP;      // 复用功能推挽输出
    gpio.Pull  = GPIO_NOPULL;
    gpio.Speed = GPIO_SPEED_FREQ_100MHz; // 高速 (100Mbps 信号)

    for (int i = 0; i < sizeof(rmii_pins)/sizeof(rmii_pins[0]); i++) {
        gpio.Pin  = rmii_pins[i].pin;
        gpio.Alternate = rmii_pins[i].af;
        HAL_GPIO_Init(rmii_pins[i].port, &gpio);
    }

    /* ── PHY 复位引脚 (PB0) ── */
    gpio.Pin  = GPIO_PIN_0;
    gpio.Mode = GPIO_MODE_OUTPUT_PP;
    gpio.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOB, &gpio);

    /* ── PHY 中断引脚 (PA8) ── */
    gpio.Pin  = GPIO_PIN_8;
    gpio.Mode = GPIO_MODE_IT_FALLING;   // 下降沿触发
    gpio.Pull = GPIO_PULLUP;
    gpio.Speed = GPIO_SPEED_FREQ_LOW;
    HAL_GPIO_Init(GPIOA, &gpio);

    HAL_NVIC_SetPriority(EXTI9_5_IRQn, 5, 0);
    HAL_NVIC_EnableIRQ(EXTI9_5_IRQn);
}

/* ════════════════════════════════════════════
 * Step 2: PHY 硬件复位
 * ════════════════════════════════════════════ */
static void phy_hardware_reset(void) {
    /* 拉低 RST 引脚 */
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET);
    delay_ms(PHY_RESET_PULSE_MS);

    /* 释放复位 */
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);
    delay_ms(PHY_POST_RESET_WAIT_MS);
    /* PHY 完成内部初始化 + PLL 锁定需要 ~30ms */
}

/* ════════════════════════════════════════════
 * Step 3: ETH MAC & DMA 初始化 (HAL 库)
 * ════════════════════════════════════════════ */
static void MX_ETH_Init(void) {
    /* 使能 ETH 时钟 */
    __HAL_RCC_ETH_CLK_ENABLE();

    heth.Instance = ETH;
    heth.Init.MACAddr = (uint8_t *)"\x0C\x29\x12\x34\x56\x78";  // MAC 地址
    heth.Init.MediaInterface = HAL_ETH_RMII_MODE;
    heth.Init.TxDesc = NULL;          // 使用 DMA 描述符数组
    heth.Init.RxDesc = NULL;
    heth.Init.RxBuffLen = 1524;       // 以太网帧最大长度

    /* MAC 配置参数 (初始化时暂不设置速率, 留给 PHY 协商后配置) */
    heth.Init.AutoNegotiation = ETH_AUTONEGO_ENABLE;
    heth.Init.Speed = ETH_SPEED_100M;     // 初始值, 后续会更新
    heth.Init.DuplexMode = ETH_MODE_FULLDUPLEX;
    heth.Init.ChecksumMode = ETH_CHECKSUM_BY_HARDWARE;
    heth.Init.StoreForwardTx = ENABLE;
    heth.Init.StoreForwardRx = ENABLE;

    if (HAL_ETH_Init(&heth) != HAL_OK) {
        Error_Handler();
    }
}

/* ════════════════════════════════════════════
 * Step 4: PHY 软件复位 + 验证 ID
 * ════════════════════════════════════════════ */
static uint8_t phy_init_verify(uint8_t phy_addr) {
    uint32_t timeout;
    uint16_t reg;

    /* ── 4a. 读取 PHY ID 验证连接 ── */
    uint16_t id1 = HAL_ETH_ReadPHYRegister(&heth, phy_addr, 0x02);
    uint16_t id2 = HAL_ETH_ReadPHYRegister(&heth, phy_addr, 0x03);
    uint32_t phy_id = (id1 << 16) | id2;

    printf("🔍 PHY ID = 0x%08X\r\n", phy_id);

    if (phy_id != 0x0007C0F1) {
        printf("⚠️  Unexpected PHY ID! Expected LAN8720 (0x0007C0F1)\r\n");
        return 0;
    }

    /* ── 4b. 软件复位 ── */
    HAL_ETH_WritePHYRegister(&heth, phy_addr, 0x00, BMCR_RESET);

    timeout = 1000;
    while (timeout--) {
        reg = HAL_ETH_ReadPHYRegister(&heth, phy_addr, 0x00);
        if (!(reg & BMCR_RESET)) {
            printf("✅ PHY Software Reset Complete\r\n");
            break;
        }
        delay_ms(1);
    }
    if (timeout == 0) {
        printf("❌ PHY Reset Timeout!\r\n");
        return 0;
    }

    delay_ms(10);  // 复位后稳定时间
    return 1;
}

/* ════════════════════════════════════════════
 * Step 5: 配置自动协商
 * ════════════════════════════════════════════ */
static void phy_config_autoneg(uint8_t phy_addr) {
    /* ── 5a. 通告能力: 100M FD/HD + 10M FD/HD ── */
    HAL_ETH_WritePHYRegister(&heth, phy_addr, LAN8720_ANAR,
        (1 << 8) |   // 100BASE-TX Full Duplex
        (1 << 7) |   // 100BASE-TX Half Duplex
        (1 << 6) |   // 10BASE-T Full Duplex
        (1 << 5) |   // 10BASE-T Half Duplex
        0x0001);     // Protocol Selector = 802.3

    /* ── 5b. 使能自动协商 ── */
    uint16_t bmcr = HAL_ETH_ReadPHYRegister(&heth, phy_addr, 0x00);
    bmcr |= BMCR_AUTONEG | BMCR_RESTART_AUTONEG;
    HAL_ETH_WritePHYRegister(&heth, phy_addr, 0x00, bmcr);

    printf("⏳ Auto-Negotiation Started...\r\n");
}

/* ════════════════════════════════════════════
 * Step 6: 等待自动协商完成并读取结果
 * ════════════════════════════════════════════ */
static uint8_t phy_wait_link_up(uint8_t phy_addr) {
    uint32_t start = HAL_GetTick();
    uint16_t mcsr, bmsr;

    /* 自动协商最多等待 4 秒 */
    while ((HAL_GetTick() - start) < 4000) {
        /* 读取 LAN8720 的 MCSR (0x1F) 获取链路状态 */
        mcsr = HAL_ETH_ReadPHYRegister(&heth, phy_addr, LAN8720_MCSR);

        if (mcsr & MCSR_LINKUP) {
            uint8_t speed  = (mcsr & MCSR_100M)    ? 100 : 10;
            uint8_t duplex = (mcsr & MCSR_FULLDUP) ? 1 : 0;

            printf("✅ Link UP! %dMbps %s Duplex\r\n",
                   speed, duplex ? "Full" : "Half");

            /* 配置 MAC 匹配协商结果 */
            if (speed == 100) {
                heth.Init.Speed = ETH_SPEED_100M;
            } else {
                heth.Init.Speed = ETH_SPEED_10M;
            }
            heth.Init.DuplexMode = duplex ? ETH_MODE_FULLDUPLEX
                                          : ETH_MODE_HALFDUPLEX;
            HAL_ETH_ConfigMAC(&heth, NULL);  // 更新 MAC 配置

            return 1;
        }

        delay_ms(100);
    }

    printf("❌ Link Timeout (4s elapsed)\r\n");
    return 0;
}

/* ════════════════════════════════════════════
 * Step 7: 启动 ETH DMA
 * ════════════════════════════════════════════ */
static void eth_start_dma(void) {
    ETH_DMADescTypeDef dma_rx_descriptor[ETH_RXBUFNB];
    ETH_DMADescTypeDef dma_tx_descriptor[ETH_TXBUFNB];
    uint8_t rx_buffers[ETH_RXBUFNB][ETH_RX_BUF_SIZE];

    /* 初始化 DMA 描述符 */
    HAL_ETH_DescAssignMemory(&heth, &dma_rx_descriptor[0], rx_buffers[0], 0);
    // ... 循环初始化所有描述符 ...

    /* 启动 DMA 传输 */
    HAL_ETH_Start(&heth);
    printf("🚀 ETH DMA Started -- Ready to ping!\r\n");
}

/* ════════════════════════════════════════════
 * 主函数: 完整的启动序列
 * ════════════════════════════════════════════ */
int main(void) {
    HAL_Init();
    SystemClock_Config();

    /* ── 启动流程 ── */
    MX_ETH_GPIO_Init();              // 1. GPIO
    phy_hardware_reset();            // 2. PHY 硬件复位
    MX_ETH_Init();                   // 3. ETH MAC 初始化

    if (phy_init_verify(LAN8720_PHY_ADDR)) {       // 4. PHY 验证
        phy_config_autoneg(LAN8720_PHY_ADDR);      // 5. 自动协商
        if (phy_wait_link_up(LAN8720_PHY_ADDR)) {  // 6. 等待建链
            eth_start_dma();                       // 7. 启动 DMA
        }
    }

    /* ── LwIP 初始化 ── */
    lwip_init();
    netif_add(&gnetif, &ipaddr, &netmask, &gw, NULL, ðernetif_init, ðernet_input);
    netif_set_default(&gnetif);
    netif_set_up(&gnetif);

    while (1) {
        /* 轮询接收 */
        HAL_ETH_GetReceivedFrame(&heth);
        if (heth.RxFrameInfos.SegCount > 0) {
            ethernet_input(&gnetif);  // 送 LwIP 协议栈
        }
    }
}
```

---

### 11.9 实战：STM32F4 + DP83848 初始化

与 LAN8720 的主要区别在于寄存器映射不同：

```c
/* ── DP83848 特有的寄存器 ── */
#define DP83848_PHY_ADDR        0x01    // 通常 ADDR 引脚配置为 1

#define DP83848_PHYSTS          0x10    // PHY Status Register
#define DP83848_PHYSTS_LINK     (1 << 1)    // Link Status
#define DP83848_PHYSTS_SPEED    (1 << 7)    // 1=10M, 0=100M
#define DP83848_PHYSTS_DUPLEX   (1 << 8)    // 1=Full

void dp83848_init(uint8_t phy_addr) {
    /* 读取 PHY ID 验证 */
    uint16_t id1 = HAL_ETH_ReadPHYRegister(&heth, phy_addr, 0x02);
    uint16_t id2 = HAL_ETH_ReadPHYRegister(&heth, phy_addr, 0x03);
    uint32_t id = (id1 << 16) | id2;

    if (id != 0x20005C90) {
        printf("⚠️  DP83848 not detected (ID=0x%08X)\r\n", id);
        return;
    }
    printf("✅ DP83848 detected\r\n");

    /* 软件复位 */
    HAL_ETH_WritePHYRegister(&heth, phy_addr, 0x00, 0x8000);
    delay_ms(10);

    /* 使能自动协商 */
    HAL_ETH_WritePHYRegister(&heth, phy_addr, 0x00, 0x1200);
    delay_ms(100);

    /* 读取链路状态 (通过 PHYSTS 寄存器) */
    uint16_t sts = HAL_ETH_ReadPHYRegister(&heth, phy_addr, DP83848_PHYSTS);

    if (sts & DP83848_PHYSTS_LINK) {
        uint8_t speed  = (sts & DP83848_PHYSTS_SPEED) ? 10 : 100;
        uint8_t duplex = (sts & DP83848_PHYSTS_DUPLEX) ? 1 : 0;
        printf("DP83848: %dM %s\r\n", speed, duplex ? "Full" : "Half");
    }
}
```

**LAN8720 vs DP83848 主要差异：**

| 项目 | LAN8720 | DP83848 |
|------|---------|---------|
| PHY 地址 | 0x00 (ADDR 拉低) | 0x01 (常见配置) |
| 状态寄存器 | 0x1F (MCSR) | 0x10 (PHYSTS) |
| 内置 LDO | 有 (3.3V→1.2V) | 无 (需要独立的 1.2V/1.8V) |
| 典型封装 | QFN-24 | LQFP-48 |
| 功耗 | ~70mA | ~130mA |
| INT 引脚 | 有 (nINT/REGOFF) | 有 (INT) |

---

### 11.10 RMII 时钟与 PCB 关键设计要点

#### 时钟拓扑

RMII 要求所有信号与 REF_CLK 同步（50MHz），常见的三种方案：

```
方案 A: PHY 产生 50MHz (最常用)
  25MHz 晶振 ─→ LAN8720 XI
                LAN8720 CLKOUT ──50MHz──→ MCU PA1 (ETH_REF_CLK)
  优点: 时钟路径短, PHY 内部 PLL 抖动小
  缺点: MCU 必须支持 REF_CLK 输入

方案 B: MCU 产生 50MHz
  MCU MCO ──50MHz──→ LAN8720 XI + MCU ETH_REF_CLK
  优点: 少一个晶振
  缺点: MCO 输出抖动较大, 可能影响 PHY 接收性能

方案 C: 独立 50MHz 有源晶振
  50MHz OSC ──→ LAN8720 XI + MCU ETH_REF_CLK
  优点: 精度最高, 独立于 MCU/PHY
  缺点: 成本高
```

#### PCB 布局关键规则

```
┌─────────────────────────────────────────────────────────────┐
│                     PCB Layout Rules                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ① RMII 信号线等长:                                           │
│     TX_EN, TXD[1:0], CRS_DV, RXD[1:0] 长度差 < 20mm        │
│                                                              │
│  ② REF_CLK 信号线:                                           │
│     - 最短路径 (< 50mm)                                      │
│     - 远离其他高速信号 (至少 3W 间距)                           │
│     - 包地处理 (两侧 GND 走线)                                │
│                                                              │
│  ③ PHY 布局:                                                 │
│     - PHY 和 RJ45 间距 < 25mm                                │
│     - 网络变压器靠近 RJ45 连接器                                │
│     - 差分对 (TX±, RX±) 等长 + 差分阻抗 100Ω                  │
│                                                              │
│  ④ 电源去耦:                                                 │
│     - PHY 每个 VDD 引脚旁放置 100nF + 10μF 电容               │
│     - 电容距离 PHY 引脚 < 3mm                                 │
│                                                              │
│  ⑤ MDC/MDIO 走线:                                            │
│     - MDIO 需 1.5kΩ 上拉电阻到 3.3V                          │
│     - 如 MDC > 2.5MHz, 上拉可适当减小至 1kΩ                   │
│                                                              │
│  ⑥ PHY 晶振:                                                 │
│     - 25MHz 晶振距离 PHY < 10mm                               │
│     - 负载电容匹配: C1 = C2 = 18pF (典型值)                   │
│     - 晶振下方不走其他信号线                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

### 11.11 常见问题与调试技巧

#### 现象 1：MDIO 读回全是 0xFFFF

```c
/* 原因: MDIO 通信失败, PHY 无响应 */

/* 排查步骤: */
/* 1. 检查 PHY 供电电压 */
   用万用表量 PHY VDD = 3.3V ±5%
   LAN8720 内部 LDO 输出 1.2V, 量 VDDCR 引脚

/* 2. 检查 PHY 复位状态 */
   // 复位结束后确认 RST 引脚为高电平
   HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_0);  // 应为 1

/* 3. 检查 PHY 地址配置 */
   // LAN8720 ADDR 引脚 (pin 9):
   //   拉低 → 地址 0x00
   //   拉高 (通过 10kΩ 到 VDD) → 地址 0x01
   // 确认代码中的地址与实际硬件一致

/* 4. 检查 MDIO/MDC 上拉 */
   // MDIO 必须外接 1.5kΩ 上拉到 3.3V
   // 示波器量 MDIO: 空闲时为高电平

/* 5. 检查 MDC 时钟频率 */
   // 在 STM32 MACMIIAR 中配置 CR 字段:
   //   HCLK=168MHz: CR=3 (HCLK/62 = 2.7MHz) ✓
   //   HCLK=168MHz: CR=2 (HCLK/42 = 4MHz)   ⚠️ 可能超过 2.5MHz
```

#### 现象 2：MDIO 能读 PHY ID，但 Link 始终 Down

```c
/* 原因: 自动协商失败或硬件连接问题 */

/* 1. 检查 RJ45 连接 */
   // 示波器量 TX+/TX-
   // 正常时应有差分信号 (幅度 ~2V)
   // 对于 LAN8720, TX+/- 空闲时共模电压 = VDD/2 ≈ 1.65V

/* 2. 检查网络变压器 */
   // 中心抽头 (center tap) 是否连接到正确的电源
   // LAN8720: 中心抽头通过 0.1μF 电容接地 (电压驱动型)
   // DP83848: 中心抽头接 VDD (电流驱动型)

/* 3. 强制固定模式排除自动协商问题 */
   uint16_t bmcr = BMCR_SPEED_LSB | BMCR_FULLDUPLEX;  // 100M Full
   // bmcr &= ~BMCR_AUTONEG;  // 关闭自动协商
   HAL_ETH_WritePHYRegister(&heth, PHY_ADDR, 0x00, bmcr);
   // 如果强制模式下 Link 正常 → 自动协商配置有问题

/* 4. 检查 Link Partner (交换机/PC) */
   // 交换机端口是否被禁用? 换一个端口试试
   // PC 网卡是否被禁用?
```

#### 现象 3：能 Link Up 但 Ping 不通

```c
/* 原因: MAC 速率/双工与 PHY 不匹配 */

/* 1. 确认 MAC 配置与协商结果一致 */
   printf("MAC Speed: %d, Duplex: %d\r\n",
          heth.Init.Speed, heth.Init.DuplexMode);

/* 2. 检查 MAC 地址设置 */
   // 确保 MAC 地址在局域网内唯一
   // STM32F4: heth.Init.MACAddr = {0x0C, 0x29, ...};

/* 3. 检查 DMA 描述符配置 */
   // 描述符地址是否正确?
   // 缓冲区大小是否 >= 最大帧长 (1518 bytes)?

/* 4. 用逻辑分析仪抓 TX_EN 和 TXD */
   // 有数据发送时 TX_EN 应为高电平
   // 发送时 TXD 应有数据变化
   // 如果完全无数据 → MAC DMA 未启动
   // 如果 TX_EN 拉高但无法 Ping → 可能是 PHY 侧信号问题
```

#### 现象 4：PHY 中断无法触发

```c
/* 原因: 中断配置错误 */

/* 1. 确认 PHY 中断已使能 */
   // LAN8720: 写 0x1B 寄存器
   HAL_ETH_WritePHYRegister(&heth, PHY_ADDR, 0x1B,
       (1 << 14) | (1 << 12) | (1 << 10));

/* 2. 确认 MCU EXTI 配置 */
   // 中断引脚如 PA8: 配置下降沿触发
   // 使能 NVIC: EXTI9_5_IRQn

/* 3. 确认中断服务程序已清标志 */
   void EXTI9_5_IRQHandler(void) {
       if (__HAL_GPIO_EXTI_GET_IT(PA8_PIN)) {
           __HAL_GPIO_EXTI_CLEAR_IT(PA8_PIN);
           // ... 处理 PHY 中断
       }
   }

/* 4. 示波器量 PHY_INT 引脚 */
   // 链路变化时是否有脉冲?
   // 否 → PHY 没产生中断
   // 是 → MCU 没响应 → 检查 NVIC/时钟
```

#### 调试工具与技巧

```bash
# 1. 查看 PHY 全部寄存器 (Bash/串口工具)
for reg in 0x00 0x01 0x02 0x03 0x04 0x05 0x06 0x10 0x1F; do
    reg_val=$(phy_read $PHY_ADDR $reg)
    printf "Reg[0x%02X] = 0x%04X\n" $reg $reg_val
done

# 串口输出示例:
# Reg[0x00] = 0x3100   ← BMCR: AutoNeg=1, Speed=100M
# Reg[0x01] = 0x786D   ← BMSR: Link=1, ANEG_Complete=1
# Reg[0x02] = 0x0007   ← PHYID1: LAN8720
# Reg[0x03] = 0xC0F1   ← PHYID2: LAN8720 rev 1

# 2. 示波器关键测量点
#    - MDC 频率: 应约 2.5MHz
#    - MDIO 波形: 读写时应有数据变化
#    - REF_CLK: 50MHz ±50ppm
#    - TX+/TX-: 链路空闲时共模电压 1.65V (LAN8720)
```

#### LAN8720 寄存器快查表

| 地址 | 名称 | 复位值 | 说明 |
|------|------|--------|------|
| 0x00 | BMCR | 0x3100 | 基本控制 (ANEG=1, Speed=100M) |
| 0x01 | BMSR | 0x786D | 基本状态 (LinkUP, ANEG_COMPLETE) |
| 0x02 | PHYIDR1 | 0x0007 | PHY 标识高 16 位 |
| 0x03 | PHYIDR2 | 0xC0F1 | PHY 标识低 16 位 (Model=0x1F, Rev=0x1) |
| 0x04 | ANAR | 0x01E1 | 自动协商通告能力 |
| 0x05 | ANLPAR | 0xXXXX | 对端能力 (只读) |
| 0x06 | ANER | 0x0004 | 自动协商扩展 |
| 0x1B | ISR | 0x0000 | 中断屏蔽 |
| 0x1C | ISR_SRC | 0x0000 | 中断源 (读即清) |
| 0x1E | TCSR | 0x0000 | 发送控制/状态 |
| **0x1F** | **MCSR** | **0x1081** | **模式控制状态 (核心)** |

**MCSR (0x1F) 位图：**

```
位 15: Reserved
位 14: SPEED (1=100M, 0=10M)        ← 读此位获取协商速率
位 13: DUPLEX (1=Full, 0=Half)     ← 读此位获取双工
位 12: ALE (AutoNeg 完成事件)       ← 中断源, 读 0x1C 清除
位 11: Reserved
位 10: LINK (1=UP, 0=DOWN)          ← 读此位获取链路状态
位 9:  Reserved
位 8:  Reserved
位 7:  Reserved
位 6:  Energy On                    ← 节能模式检测
位 5:  Extended Status
位 4-0: Reserved
```

---

### 11.12 LWIP 集成要点

```c
/* ── LwIP 初始化时序 (必须与 PHY 初始化交错) ── */

int main(void) {
    HAL_Init();
    SystemClock_Config();

    /* ── 第 1 阶段: 硬件初始化 ── */
    MX_ETH_GPIO_Init();         // GPIO 引脚
    phy_hardware_reset();       // PHY 硬件复位 (如果 PHY_EN 可控)
    MX_ETH_Init();              // ETH MAC + DMA (HAL)

    if (!phy_init_verify(PHY_ADDR)) {
        Error_Handler();        // PHY 未找到或复位失败
    }

    /* ── 第 2 阶段: PHY 自动协商 + LwIP ── */
    /* LwIP 的 low_level_init 会读取 PHY 协商结果 */
    lwip_init();

    struct netif gnetif;
    ip4_addr_t ipaddr, netmask, gw;

    /* 静态 IP 配置 (或使用 DHCP) */
    IP4_ADDR(&ipaddr,  192, 168, 1, 100);
    IP4_ADDR(&netmask, 255, 255, 255, 0);
    IP4_ADDR(&gw,      192, 168, 1, 1);

    netif_add(&gnetif, &ipaddr, &netmask, &gw,
              NULL, ðernetif_init, ðernet_input);
    netif_set_default(&gnetif);

    /* ── 第 3 阶段: 启动 PHY 自动协商 ── */
    phy_config_autoneg(PHY_ADDR);

    /* ── 第 4 阶段: 等待链路并设置 netif ── */
    if (phy_wait_link_up(PHY_ADDR)) {
        netif_set_up(&gnetif);
        printf("✅ Network ready: %s\r\n", ip4addr_ntoa(&ipaddr));
    } else {
        printf("⚠️  Network NOT ready (no link)\r\n");
    }

    /* ── 第 5 阶段: 主循环 ── */
    while (1) {
        ethernetif_input(&gnetif);  // 轮询收包 (或中断通知)
        sys_check_timeouts();       // LwIP 定时器 (ARP, DHCP, TCP)
    }
}

/* ── LwIP 的 low_level_init 中读取 PHY 速率 ── */
static void low_level_init(struct netif *netif) {
    /* ... 初始化 DMA 描述符 ... */

    /* 获取 PHY 协商结果配置 MAC (重要!) */
    ETH_PHY_StatusTypeDef link;
    phy_get_link_info(PHY_ADDR, &link);

    heth.Init.Speed = (link.speed == 100) ?
                      ETH_SPEED_100M : ETH_SPEED_10M;
    heth.Init.DuplexMode = link.duplex ?
                          ETH_MODE_FULLDUPLEX : ETH_MODE_HALFDUPLEX;
    HAL_ETH_ConfigMAC(&heth, NULL);

    HAL_ETH_Start(&heth);
}
```

---

### 附录：PHY 寄存器调试转储函数

```c
/* ── 打印 PHY 所有寄存器 (调试用) ── */
void phy_dump_registers(uint8_t phy_addr) {
    static const char *reg_names[] = {
        "BMCR ", "BMSR ", "PHYID1", "PHYID2",
        "ANAR ", "ANLPAR", "ANER ", "NSR  "
    };

    printf("\r\n╔══════════════════════════════════╗\r\n");
    printf("║ PHY Register Dump (Addr=0x%02X) ║\r\n", phy_addr);
    printf("╚══════════════════════════════════╝\r\n");

    for (int i = 0; i < 32; i++) {
        uint16_t val = phy_reg_read(phy_addr, i);
        const char *name = (i < 8) ? reg_names[i] : "----";
        printf("  [0x%02X] %s = 0x%04X  ", i, name, val);

        /* 打印分隔符，每行 4 个寄存器 */
        if ((i + 1) % 4 == 0) printf("\r\n");
    }
    printf("\r\n");
}

/* 输出示例:
╔══════════════════════════════════╗
║ PHY Register Dump (Addr=0x00)  ║
╚══════════════════════════════════╝
  [0x00] BMCR  = 0x3100    [0x01] BMSR  = 0x786D
  [0x02] PHYID1= 0x0007    [0x03] PHYID2= 0xC0F1
  [0x04] ANAR  = 0x01E1    [0x05] ANLPAR= 0x01E1
  [0x06] ANER  = 0x0004    [0x07] NSR   = 0x0000
  ...
  [0x1F] MCSR  = 0x64C0   ← 100M Full Duplex, Link UP
*/
```

---

### 总结：PHY 控制流程图

```
上电复位
  │
  ├── 硬件初始化阶段
  │   ├─ GPIO 配置 (复用推挽, 高速)
  │   ├─ PHY RST 释放 (拉低 > 10ms → 拉高)
  │   └─ 等待 PHY PLL 锁定 (~5-30ms)
  │
  ├── PHY 验证阶段
  │   ├─ 读 PHYIDR1/2 (验证连接 + 识别型号)
  │   ├─ 软件复位 (BMCR.15=1, 轮询自清除)
  │   └─ 读取 BMSR (确认能力)
  │
  ├── 自动协商阶段
  │   ├─ 写 ANAR (通告本端能力)
  │   ├─ BMCR 使能 AutoNeg + 重启协商
  │   ├─ 轮询 BMSR.5 (等待完成, 超时 3-4s)
  │   ├─ 读取 ANLPAR (对端能力)
  │   └─ 裁决: 选择最高共同能力
  │
  ├── 链路配置阶段
  │   ├─ 读取速率 (100M/10M)
  │   ├─ 读取双工 (Full/Half)
  │   └─ 配置 MAC 寄存器匹配 PHY
  │
  └── 数据传输阶段
      ├─ MAC DMA 描述符初始化
      ├─ 使能 DMA 发送/接收
      ├─ 启动 LwIP 协议栈
      └─ 进入主循环: 轮询/中断收发
```

---



- [STM32F4 Reference Manual (RM0090)](https://www.st.com/resource/en/reference_manual/dm00031020.pdf)
- [ARM Cortex-M4 Technical Reference Manual](https://developer.arm.com/documentation/100166/0001/)
- [GNU LD Linker Script Documentation](https://sourceware.org/binutils/docs/ld/Scripts.html)
- [ESP-IDF OTA 文档](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/system/ota.html)
- [RISC-V Privileged Architecture Spec](https://riscv.org/technical/specifications/)
- [IEEE 802.3-2022 Standard](https://standards.ieee.org/ieee/802.3/10435/)
- [LAN8720A Datasheet (SMSC/Microchip)](https://www.microchip.com/en-us/product/LAN8720A)
- [DP83848 Datasheet (TI)](https://www.ti.com/product/DP83848)
- [KSZ8081 Datasheet (Microchip)](https://www.microchip.com/en-us/product/KSZ8081)
- [STM32 + LwIP + FreeRTOS 应用笔记 (AN4776)](https://www.st.com/resource/en/application_note/an4776.pdf)
- [LwIP 官方文档](https://savannah.nongnu.org/projects/lwip/)
- [MII/RMII 接口规范 (RM0401)](https://www.st.com/resource/en/reference_manual/rm0401.pdf)

---

> **作者**：HuizZhoho
> **许可证**：MIT
