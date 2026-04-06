# Freertos移植

## ARM移植

## RISC v移植

## 核心移植文件架构

```plain
your_project/
├── FreeRTOS-Kernel/
│   ├── include/                    # 核心头文件
│   ├── tasks.c queue.c list.c ...  # 核心源码
│   └── portable/
│       ├── GCC/
│       │   └── RISC-V/
│       │       ├── portASM.S       # 上下文切换汇编（关键）
│       │       ├── port.c          # C 层移植接口
│       │       └── portmacro.h     # 移植层宏定义
│       └── MemMang/
│           └── heap_4.c            # 内存管理（推荐）
└── your_chip_specific/
    ├── freertos_risc_v_chip_specific_extensions.h  # 芯片扩展
    └── riscv_hal.c                 # 你的 HAL 封装
```

### 创建 `freertos_risc_v_chip_specific_extensions.h`（配置文件）

```c
#ifndef __FREERTOS_RISC_V_CHIP_SPECIFIC_EXTENSIONS_H__
#define __FREERTOS_RISC_V_CHIP_SPECIFIC_EXTENSIONS_H__

/* 你的 RISC-V 核配置 - 以 SiFive E31 为例 */
#define portasmHAS_CLINT              1   // 是否有 Core Local Interruptor (CLINT)
#define portasmHAS_MTIME              1   // 是否有机器模式定时器
#define portasmADDITIONAL_CONTEXT_SIZE 0  // 额外保存的寄存器数（浮点等）

/* 中断控制器基地址 - 根据你的芯片手册修改 */
#define portasmCLINT_BASE_ADDR        ( 0x02000000UL )
#define portasmMTIME_OFFSET           ( 0xBFF8UL )     // mtime 寄存器偏移
#define portasmMTIMECMP_OFFSET        ( 0x4000UL )     // mtimecmp 寄存器偏移

/* 如果支持浮点，添加 FPU 上下文保存 */
#ifdef __riscv_flen
    #define portasmHAS_FPU 1
    #define portasmFPU_CONTEXT_SIZE (32)  // 32个浮点寄存器
#else
    #define portasmHAS_FPU 0
    #define portasmFPU_CONTEXT_SIZE (0)
#endif

/* 你的芯片可能有额外的 CSR 需要保存 */
#define portasmADDITIONAL_CONTEXT_SIZE (portasmFPU_CONTEXT_SIZE)

#endif /* __FREERTOS_RISC_V_CHIP_SPECIFIC_EXTENSIONS_H__ */
```

### 中断处理适配（`portASM.S` 关键修改点）

```c
/* 在 portASM.S 中，确保你的中断向量入口调用 xPortPendSVHandler */
.section .text.handlers
.global freertos_risc_v_trap_handler
.align 4

freertos_risc_v_trap_handler:
    /* 保存当前上下文到栈 */
    portasmSAVE_CONTEXT

    /* 判断中断类型 */
    csrr t0, mcause
    bge t0, zero, synchronous_exception    # mcause >= 0 是异常

    /* 异步中断（定时器/外部） */
    li t1, 0x80000007                      # 机器模式定时器中断
    beq t0, t1, handle_timer_interrupt

    /* 你的芯片特定中断处理 */
    call your_chip_external_irq_handler

    j context_restore

handle_timer_interrupt:
    /* 清除定时器中断（你的芯片特定） */
    li t0, portasmCLINT_BASE_ADDR
    li t1, -1
    sw t1, portasmMTIMECMP_OFFSET(t0)      # 写 mtimecmp 清除

    /* 调用 FreeRTOS tick 处理 */
    call xPortSysTickHandler
    j context_restore

synchronous_exception:
    /* 处理 ecall/ebreak 等 */
    call handle_sync_exception

context_restore:
    portasmRESTORE_CONTEXT
    mret
```

### 关键 C 文件适配 (`port.c`)

#### 定时器配置（SysTick 替代）

```c
#include "FreeRTOS.h"
#include "task.h"
#include "portmacro.h"

/* 你的芯片 CLINT 寄存器定义 */
#define CLINT_BASE    ( 0x02000000UL )
#define CLINT_MTIME   ( *( volatile uint64_t * ) ( CLINT_BASE + 0xBFF8 ) )
#define CLINT_MTIMECMP( hart ) ( *( volatile uint64_t * ) ( CLINT_BASE + 0x4000 + ( hart ) * 8 ) )

static uint64_t ulTimerReloadValue = 0;

void vPortSetupTimerInterrupt( void )
{
    uint32_t ulHartId;
    __asm volatile ( "csrr %0, mhartid" : "=r" ( ulHartId ) );

    /* 计算 reload 值：假设 10MHz CLINT 时钟，1ms tick */
    ulTimerReloadValue = configCPU_CLOCK_HZ / configTICK_RATE_HZ;

    /* 设置首次中断 */
    CLINT_MTIMECMP( ulHartId ) = CLINT_MTIME + ulTimerReloadValue;

    /* 使能机器模式定时器中断 */
    __asm volatile ( "csrs mie, %0" :: "r" ( 0x80 ) );  // 置位 MTIE
}

/* 在 xPortSysTickHandler 中重新加载 */
void xPortSysTickHandler( void )
{
    uint32_t ulHartId;
    __asm volatile ( "csrr %0, mhartid" : "=r" ( ulHartId ) );

    /* 原子地更新 mtimecmp，避免错过中断 */
    CLINT_MTIMECMP( ulHartId ) = CLINT_MTIME + ulTimerReloadValue;

    /* 调用 FreeRTOS 核心 tick 处理 */
    if( xTaskIncrementTick() != pdFALSE )
    {
        vTaskSwitchContext();
    }
}
```

#### 上下文切换触发

```c
void vPortYield( void )
{
    /* 触发机器模式软件中断（如果 CLINT 支持）或直接上下文切换 */
    __asm volatile ( "ecall" );  // 或使用特定 CSR 触发
}

/* 任务启动时的初始栈帧设置 */
StackType_t *pxPortInitialiseStack( StackType_t *pxTopOfStack, 
                                    TaskFunction_t pxCode, 
                                    void *pvParameters )
{
    /* RISC-V 栈向下增长，pxTopOfStack 已是栈顶 */

    /* 预留上下文保存空间（按 portASM.S 的 portasmCONTEXT_SIZE） */
    pxTopOfStack -= ( portasmCONTEXT_SIZE / sizeof( StackType_t ) );

    /* 初始化 mstatus：使能中断，设置 MPP=0（用户模式）或 3（机器模式） */
    pxTopOfStack[ 0 ] = ( 0x188 );  // mstatus: MPIE=1, MPP=M-mode

    /* 返回地址 = 任务入口 */
    pxTopOfStack[ 1 ] = ( StackType_t ) pxCode;

    /* 参数寄存器 a0 */
    pxTopOfStack[ 2 ] = ( StackType_t ) pvParameters;

    /* 其余寄存器初始化为 0 或特定模式值 */

    return pxTopOfStack;
}
```

| STM32 概念 | RISC-V 等价物     | 注意点                                  |
| -------- | -------------- | ------------------------------------ |
| NVIC     | CLINT + PLIC   | CLINT 处理软件/定时器，PLIC 处理外部中断           |
| SysTick  | mtime/mtimecmp | 64位计数器，需原子访问                         |
| PendSV   | 软件中断或 ecall    | RISC-V 没有 PendSV，通常用 msoft 中断或 ecall |
| PRIMASK  | mstatus.MIE    | 全局中断使能位                              |
| BASEPRI  | 无直接等价          | RISC-V 中断无优先级嵌套（除非 PLIC 支持）          |
| SVC      | ecall          | 系统调用指令                               |

reeRTOS 使用的是**固定优先级抢占式调度（Fixed-Priority Preemptive Scheduling）**，配合时间片轮转（Time Slicing）作为可选扩展


freertos使用位图管理

FreeRTOS 使用**优先级位图算法**（`uxTopReadyPriority`）来快速找到最高优先级就绪任务

搞懂freertos是怎么管理数据结构的！！！  列表和列表项 位示图
