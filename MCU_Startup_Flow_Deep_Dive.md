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

## 参考资源

- [STM32F4 Reference Manual (RM0090)](https://www.st.com/resource/en/reference_manual/dm00031020.pdf)
- [ARM Cortex-M4 Technical Reference Manual](https://developer.arm.com/documentation/100166/0001/)
- [GNU LD Linker Script Documentation](https://sourceware.org/binutils/docs/ld/Scripts.html)
- [ESP-IDF OTA 文档](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/system/ota.html)
- [RISC-V Privileged Architecture Spec](https://riscv.org/technical/specifications/)

---

> **作者**：HuizZhoho
> **许可证**：MIT
