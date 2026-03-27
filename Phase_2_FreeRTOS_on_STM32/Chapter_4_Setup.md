# Chapter 4: FreeRTOS on STM32 — Setup

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Install and configure STM32CubeIDE for RTOS development
2. Create a new STM32 project with FreeRTOS enabled
3. Understand and configure `FreeRTOSConfig.h`
4. Configure SysTick for FreeRTOS operation
5. Build, flash, and debug your first FreeRTOS project
6. Resolve common setup issues

---

## 4.1 STM32CubeIDE Setup

### 4.1.1 What You Need

| Item | Details |
|---|---|
| **IDE** | STM32CubeIDE (free from ST) |
| **Board** | STM32 Nucleo/Discovery (e.g., NUCLEO-F446RE, NUCLEO-F411RE, STM32F407G-DISC1) |
| **USB cable** | Micro-USB or USB-C (depending on board) |
| **Serial terminal** | PuTTY, Tera Term, or CubeIDE built-in terminal |

### 4.1.2 Installation Steps

1. **Download STM32CubeIDE** from [st.com/stm32cubeide](https://www.st.com/en/development-tools/stm32cubeide.html)
2. **Install** with default settings (includes ST-LINK drivers)
3. **Launch** and select a workspace directory
4. **Install board support** (STM32CubeIDE downloads firmware packages automatically when you create a project)

### 4.1.3 Creating a New Project

**Step-by-step with board selector:**

```
1. File → New → STM32 Project
2. Board Selector tab:
   - Type your board name (e.g., "NUCLEO-F446RE")
   - Select the board → Next
3. Project Setup:
   - Project Name: "RTOS_FirstProject"
   - Targeted Language: C
   - Targeted Binary Type: Executable
   - Targeted Project Type: STM32Cube
   - Click Finish
4. Initialize Peripherals with Default Mode? → Yes
5. Wait for code generation
```

**Your project structure should look like:**

```
RTOS_FirstProject/
├── Core/
│   ├── Inc/
│   │   ├── main.h
│   │   ├── stm32f4xx_hal_conf.h
│   │   ├── stm32f4xx_it.h
│   │   └── FreeRTOSConfig.h         ◄── Generated when FreeRTOS enabled
│   └── Src/
│       ├── main.c
│       ├── stm32f4xx_hal_msp.c
│       ├── stm32f4xx_it.c
│       └── system_stm32f4xx.c
├── Drivers/
│   ├── CMSIS/
│   └── STM32F4xx_HAL_Driver/
├── Middlewares/
│   └── Third_Party/
│       └── FreeRTOS/                 ◄── FreeRTOS source code
│           ├── Source/
│           │   ├── tasks.c
│           │   ├── queue.c
│           │   ├── list.c
│           │   ├── timers.c
│           │   ├── event_groups.c
│           │   ├── stream_buffer.c
│           │   ├── portable/
│           │   │   ├── GCC/ARM_CM4F/
│           │   │   │   ├── port.c    ◄── Cortex-M4F port layer
│           │   │   │   └── portmacro.h
│           │   │   └── MemMang/
│           │   │       └── heap_4.c  ◄── Default heap implementation
│           │   └── include/
│           │       ├── FreeRTOS.h
│           │       ├── task.h
│           │       ├── queue.h
│           │       └── ...
│           └── License/
└── STM32F446RETX_FLASH.ld          ◄── Linker script
```

---

## 4.2 FreeRTOS Integration

### 4.2.1 Enabling FreeRTOS via CubeMX

```
1. Open the .ioc file (double-click in Project Explorer)
2. In the Pinout & Configuration tab:
   - Categories → Middleware → FREERTOS
   - Interface: CMSIS_V2 (recommended) or CMSIS_V1
3. Configuration:
   - Config parameters tab → Set configTOTAL_HEAP_SIZE (e.g., 15360 = 15 KB)
   - Tasks and Queues tab → Add your tasks
4. Project → Generate Code (or Ctrl+Shift+G)
```

### 4.2.2 Manual FreeRTOS Integration (For Understanding)

If you want to understand what CubeMX does behind the scenes:

```
1. Download FreeRTOS from freertos.org
2. Copy these files into your project:
   Source/
   ├── tasks.c           ◄── Core: task management
   ├── queue.c           ◄── Core: queues (needed even if you don't use queues)
   ├── list.c            ◄── Core: linked list (required)
   ├── timers.c          ◄── Optional: software timers
   ├── event_groups.c    ◄── Optional: event groups
   ├── stream_buffer.c   ◄── Optional: stream/message buffers
   ├── portable/
   │   ├── GCC/ARM_CM4F/ ◄── Port for your specific Cortex-M + compiler
   │   │   ├── port.c
   │   │   └── portmacro.h
   │   └── MemMang/
   │       └── heap_4.c  ◄── Choose one heap implementation
   └── include/          ◄── All header files

3. Add include paths to your compiler settings:
   - FreeRTOS/Source/include
   - FreeRTOS/Source/portable/GCC/ARM_CM4F

4. Create FreeRTOSConfig.h in your project's Inc/ directory
```

### 4.2.3 Choosing the Right Port

| Cortex-M Core | FPU? | Port Directory |
|---|---|---|
| Cortex-M0 | No | `portable/GCC/ARM_CM0` |
| Cortex-M3 | No | `portable/GCC/ARM_CM3` |
| Cortex-M4 | No | `portable/GCC/ARM_CM4_MPU` |
| Cortex-M4F | Yes | `portable/GCC/ARM_CM4F` |
| Cortex-M7 | Yes | `portable/GCC/ARM_CM7/r0p1` |
| Cortex-M33 | Yes | `portable/GCC/ARM_CM33_NTZ` |

> **📝 Beginner Note:** If using STM32CubeIDE with CubeMX, the correct port is selected automatically. Manual selection is only needed for hand-configured projects.

---

## 4.3 FreeRTOSConfig.h — The Master Configuration

This is the **most important file** in any FreeRTOS project. It controls every aspect of the kernel.

### 4.3.1 Essential Configuration

```c
/* FreeRTOSConfig.h — Annotated for STM32F4 at 168 MHz */

#ifndef FREERTOS_CONFIG_H
#define FREERTOS_CONFIG_H

/*──────────── Scheduling ────────────*/

/* 1 = Preemptive, 0 = Cooperative */
#define configUSE_PREEMPTION                     1

/* 1 = Round-robin for equal-priority tasks */
#define configUSE_TIME_SLICING                   1

/* CPU clock frequency — MUST match SystemCoreClock */
#define configCPU_CLOCK_HZ                       (SystemCoreClock)

/* RTOS tick frequency: 1000 = 1 ms tick */
#define configTICK_RATE_HZ                       ((TickType_t)1000)

/* Maximum number of priority levels (each uses RAM) */
#define configMAX_PRIORITIES                      7

/* Idle task should yield to other priority-0 tasks */
#define configIDLE_SHOULD_YIELD                  1


/*──────────── Memory ────────────*/

/* Minimum stack for any task (in WORDS, not bytes!) */
#define configMINIMAL_STACK_SIZE                 ((uint16_t)128)

/* Total heap available for FreeRTOS objects */
#define configTOTAL_HEAP_SIZE                    ((size_t)15360)  /* 15 KB */

/* Maximum task name length (including null terminator) */
#define configMAX_TASK_NAME_LEN                  16


/*──────────── Features ────────────*/

/* Enable/disable RTOS features (saves code space if disabled) */
#define configUSE_MUTEXES                        1
#define configUSE_RECURSIVE_MUTEXES              1
#define configUSE_COUNTING_SEMAPHORES            1
#define configUSE_QUEUE_SETS                     0
#define configUSE_TASK_NOTIFICATIONS             1
#define configUSE_TRACE_FACILITY                 1
#define configUSE_16_BIT_TICKS                   0   /* 32-bit tick counter */


/*──────────── Hook Functions ────────────*/

/* 1 = provide vApplicationIdleHook() */
#define configUSE_IDLE_HOOK                      0

/* 1 = provide vApplicationTickHook() */
#define configUSE_TICK_HOOK                      0

/* 1 or 2 = provide vApplicationStackOverflowHook() */
#define configCHECK_FOR_STACK_OVERFLOW           2

/* 1 = provide vApplicationMallocFailedHook() */
#define configUSE_MALLOC_FAILED_HOOK             1


/*──────────── Software Timers ────────────*/

#define configUSE_TIMERS                         1
#define configTIMER_TASK_PRIORITY                 (configMAX_PRIORITIES - 1)
#define configTIMER_QUEUE_LENGTH                  10
#define configTIMER_TASK_STACK_DEPTH              (configMINIMAL_STACK_SIZE * 2)


/*──────────── Debug / Statistics ────────────*/

/* Enable runtime stats (CPU usage per task) */
#define configGENERATE_RUN_TIME_STATS            0

/* Enable vTaskList() and vTaskGetRunTimeStats() */
#define configUSE_STATS_FORMATTING_FUNCTIONS     1

/* Assert macro — ESSENTIAL for development */
#define configASSERT(x) if ((x) == 0) { taskDISABLE_INTERRUPTS(); for(;;); }


/*──────────── Interrupt Configuration (CRITICAL) ────────────*/

/*
 * ARM Cortex-M priority bits used by this MCU.
 * STM32F4 uses 4 priority bits (16 priority levels: 0-15)
 *
 * IMPORTANT: These MUST match your hardware!
 */

/* Number of priority bits implemented in the NVIC */
#ifdef __NVIC_PRIO_BITS
    #define configPRIO_BITS    __NVIC_PRIO_BITS
#else
    #define configPRIO_BITS    4    /* STM32F4 = 4 bits */
#endif

/* Lowest interrupt priority (used for PendSV and SysTick) */
#define configLIBRARY_LOWEST_INTERRUPT_PRIORITY         15

/* Highest interrupt priority that can call FreeRTOS APIs */
#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY    5

/*
 * DO NOT MODIFY the macros below — they convert
 * "library" values to hardware register values.
 */
#define configKERNEL_INTERRUPT_PRIORITY \
    (configLIBRARY_LOWEST_INTERRUPT_PRIORITY << (8 - configPRIO_BITS))

#define configMAX_SYSCALL_INTERRUPT_PRIORITY \
    (configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY << (8 - configPRIO_BITS))


/*──────────── API Includes ────────────*/

#define INCLUDE_vTaskPrioritySet             1
#define INCLUDE_uxTaskPriorityGet            1
#define INCLUDE_vTaskDelete                  1
#define INCLUDE_vTaskSuspend                 1
#define INCLUDE_xResumeFromISR               1
#define INCLUDE_vTaskDelayUntil              1
#define INCLUDE_vTaskDelay                   1
#define INCLUDE_xTaskGetSchedulerState       1
#define INCLUDE_xTaskGetCurrentTaskHandle    1
#define INCLUDE_uxTaskGetStackHighWaterMark  1
#define INCLUDE_eTaskGetState                1

#endif /* FREERTOS_CONFIG_H */
```

### 4.3.2 The Interrupt Priority Diagram

This is the **most misunderstood** part of FreeRTOS configuration:

```
STM32F4 with 4 priority bits (0-15):
═════════════════════════════════════

Priority  NVIC     FreeRTOS
Value     Reg      Meaning
─────────────────────────────────────
  0      0x00     HIGHEST hardware priority
  1      0x10     │  These ISRs CANNOT call
  2      0x20     │  FreeRTOS FromISR APIs!
  3      0x30     │  (Used for ultra-fast
  4      0x40     ▼  timing-critical ISRs)
─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
  5      0x50     configMAX_SYSCALL... = 5
  6      0x60     │  These ISRs CAN call
  7      0x70     │  FreeRTOS FromISR APIs
  8      0x80     │  (xSemaphoreGiveFromISR,
  9      0x90     │   xQueueSendFromISR, etc.)
 10      0xA0     │
 11      0xB0     │
 12      0xC0     │
 13      0xD0     │
 14      0xE0     ▼
─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
 15      0xF0     configKERNEL... = 15
                  SysTick runs here
                  PendSV runs here
                  LOWEST hardware priority

 Rule: ALL ISRs that call FreeRTOS APIs MUST have
 priority >= configMAX_SYSCALL_INTERRUPT_PRIORITY (5)
 
 Rule: PendSV and SysTick MUST be at the LOWEST priority (15)
```

> **⚠️ Critical:** If an ISR at priority 0-4 calls a FreeRTOS `FromISR` API, you will get a **hard fault**. This is the #1 FreeRTOS configuration bug.

---

## 4.4 SysTick Configuration

### 4.4.1 HAL SysTick vs. FreeRTOS SysTick

By default, STM32 HAL uses SysTick for `HAL_Delay()` and `HAL_GetTick()`. FreeRTOS also needs SysTick for the kernel tick. **They conflict.**

**Solutions:**

**Option A: Use SysTick for FreeRTOS, a different timer for HAL (Recommended)**
```c
/* In stm32f4xx_hal_timebase_tim.c (generated by CubeMX):
 * HAL uses TIM6 (or another basic timer) for its timebase.
 * SysTick is free for FreeRTOS.
 */
```

When using CubeMX with FreeRTOS, this is configured automatically:
- `SYS → Timebase Source → TIM6` (or any available timer)

**Option B: Share SysTick (Not recommended but common)**
```c
/* In stm32f4xx_it.c */
void SysTick_Handler(void)
{
    HAL_IncTick();            /* HAL needs this for HAL_Delay() */

    #if (INCLUDE_xTaskGetSchedulerState == 1)
    if (xTaskGetSchedulerState() != taskSCHEDULER_NOT_STARTED)
    {
        xPortSysTickHandler();    /* FreeRTOS tick handler */
    }
    #endif
}
```

### 4.4.2 Using a Separate Timer for HAL

```
CubeMX Configuration:
1. System Core → SYS
2. Timebase Source: TIM6 (or TIM7, TIM14, etc.)
3. Generate Code

This creates stm32f4xx_hal_timebase_tim.c which initializes
TIM6 as the HAL timebase, freeing SysTick for FreeRTOS.
```

> **🏭 Industry Insight:** Always use a separate timer for HAL timebase in production systems. Sharing SysTick introduces subtle timing issues, especially with `HAL_Delay()` inside ISRs or before the scheduler starts.

---

## 4.5 Your First FreeRTOS Project

### 4.5.1 Complete Working Example

Here is a complete, tested, working example for NUCLEO-F446RE (or similar STM32F4 board):

```c
/* main.c — First FreeRTOS Project */

#include "main.h"
#include "FreeRTOS.h"
#include "task.h"
#include "semphr.h"

/* Handles */
UART_HandleTypeDef huart2;
TaskHandle_t xLEDHandle    = NULL;
TaskHandle_t xPrintHandle  = NULL;

/* Forward declarations */
void SystemClock_Config(void);
static void MX_GPIO_Init(void);
static void MX_USART2_UART_Init(void);

/*──────────── Task 1: Blink LED ────────────*/
void Task_BlinkLED(void *pvParameters)
{
    const TickType_t xDelay = pdMS_TO_TICKS(500);

    for (;;)
    {
        HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin);   /* Toggle onboard LED */
        vTaskDelay(xDelay);
    }
}

/*──────────── Task 2: Print to UART ────────────*/
void Task_Print(void *pvParameters)
{
    uint32_t counter = 0;
    char msg[64];

    for (;;)
    {
        int len = snprintf(msg, sizeof(msg),
                           "[%lu] FreeRTOS running! Tick: %lu\r\n",
                           counter++,
                           (unsigned long)xTaskGetTickCount());

        HAL_UART_Transmit(&huart2, (uint8_t *)msg, len, 100);

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

/*──────────── Required Hook Functions ────────────*/

void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName)
{
    /* Stack overflow detected! */
    taskDISABLE_INTERRUPTS();
    for (;;);
}

void vApplicationMallocFailedHook(void)
{
    /* Heap allocation failed! */
    taskDISABLE_INTERRUPTS();
    for (;;);
}

/*──────────── Main ────────────*/

int main(void)
{
    /* HAL initialization */
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_USART2_UART_Init();

    /* Print startup message BEFORE scheduler starts */
    const char *startup_msg = "\r\n=== FreeRTOS First Project ===\r\n\r\n";
    HAL_UART_Transmit(&huart2, (uint8_t *)startup_msg, strlen(startup_msg), 100);

    /* Create tasks */
    BaseType_t ret;

    ret = xTaskCreate(Task_BlinkLED, "LED", 128, NULL, 1, &xLEDHandle);
    configASSERT(ret == pdPASS);

    ret = xTaskCreate(Task_Print, "Print", 256, NULL, 2, &xPrintHandle);
    configASSERT(ret == pdPASS);

    /* Start the scheduler — this function should NEVER return */
    vTaskStartScheduler();

    /* If we get here, there was insufficient heap memory */
    Error_Handler();
}
```

### 4.5.2 Building and Flashing

```
Build:
1. Project → Build Project (Ctrl+B)
2. Check the Console for errors
3. Verify output:
   "arm-none-eabi-size" shows:
   text    data     bss     dec     hex
   12340    120    4564   17024    4280

Flash and Debug:
1. Run → Debug As → STM32 C/C++ Application
2. Click Resume (F8)
3. Open a serial terminal (115200 baud, 8N1) on the STM32's virtual COM port

Expected output:
  === FreeRTOS First Project ===
  [0] FreeRTOS running! Tick: 1001
  [1] FreeRTOS running! Tick: 2001
  [2] FreeRTOS running! Tick: 3001
  ...

LED should blink every 500 ms.
```

### 4.5.3 Memory Budget Analysis

```
RAM Usage Breakdown (STM32F446RE — 128 KB RAM):
═══════════════════════════════════════════════

Component                  Size (bytes)
──────────────────────────────────────
FreeRTOS Heap              15,360     (configTOTAL_HEAP_SIZE)
├── Task LED:
│   ├── TCB                ~100
│   └── Stack             512       (128 words × 4)
├── Task Print:
│   ├── TCB                ~100
│   └── Stack             1,024     (256 words × 4)
├── Idle Task:
│   ├── TCB                ~100
│   └── Stack             512       (128 words × 4)
├── Timer Daemon:
│   ├── TCB                ~100
│   └── Stack             1,024     (256 words × 4)
└── Free heap              ~11,888

MSP Stack (interrupts)     1,024     (from linker script)
Global variables           ~200
HAL/Driver data            ~500
──────────────────────────────────────
Total RAM used:            ~17,084   (13% of 128 KB)
```

---

## 4.6 Common Mistakes (Chapter 4)

### Mistake 1: Wrong SysTick Configuration
```
SYMPTOM: Tasks don't run, or timing is wrong
CAUSE:   HAL and FreeRTOS competing for SysTick
FIX:     Set SYS → Timebase Source → TIM6 in CubeMX
```

### Mistake 2: configTOTAL_HEAP_SIZE Too Small
```
SYMPTOM: vTaskStartScheduler() returns, or xTaskCreate fails
CAUSE:   Not enough heap for task TCBs + stacks + kernel objects
FIX:     Increase configTOTAL_HEAP_SIZE
         Check with xPortGetFreeHeapSize()
```

### Mistake 3: ISR Priority Above Maximum
```c
/* WRONG: Priority 1 is above configMAX_SYSCALL (5) — will hard fault */
HAL_NVIC_SetPriority(USART2_IRQn, 1, 0);

/* RIGHT: Priority 6 is within the safe range */
HAL_NVIC_SetPriority(USART2_IRQn, 6, 0);
```

### Mistake 4: Forgetting configASSERT
```c
/* Without configASSERT, errors silently corrupt state.
   Always define it during development: */
#define configASSERT(x) if ((x) == 0) { taskDISABLE_INTERRUPTS(); for(;;); }
```

### Mistake 5: Using HAL_Delay() in Tasks
```c
/* WRONG: HAL_Delay() busy-waits, blocking ALL lower-priority tasks */
void Task_Bad(void *pv)
{
    for (;;)
    {
        HAL_Delay(100);    /* DO NOT USE IN TASKS */
    }
}

/* RIGHT: vTaskDelay() yields CPU to other tasks while waiting */
void Task_Good(void *pv)
{
    for (;;)
    {
        vTaskDelay(pdMS_TO_TICKS(100));   /* CORRECT */
    }
}
```

---

## 4.7 Debugging Tips

1. **Verify SysTick is running:** Set a breakpoint in `xPortSysTickHandler()` — if it never hits, SysTick is misconfigured
2. **Check heap size:** Place a breakpoint at `vApplicationMallocFailedHook()` — if it hits, increase `configTOTAL_HEAP_SIZE`
3. **Verify interrupt priorities:** In the debugger, read `NVIC->IP[n]` for your interrupt and compare with `configMAX_SYSCALL_INTERRUPT_PRIORITY`
4. **Use the SFR view:** STM32CubeIDE's SFR (Special Function Register) view lets you inspect SysTick, NVIC, and SCB registers
5. **Enable all FreeRTOS hooks:** Stack overflow hook, malloc failed hook — they catch errors early
6. **Check linker output:** Verify `.text` (Flash) and `.bss`+`.data` (RAM) fit in your MCU

---

## 4.8 Hands-On Exercises

### Exercise 4.1: Project Creation

**Objective:** Create a FreeRTOS project from scratch using CubeMX.

1. Open STM32CubeIDE → New STM32 Project
2. Select your board
3. Enable FreeRTOS (CMSIS_V2)
4. Set Timebase Source to TIM6
5. Configure USART2 (Asynchronous, 115200 baud)
6. Generate code
7. Add the LED blink task from Section 4.5
8. Build, flash, verify LED blinks

### Exercise 4.2: Configuration Exploration

Modify `FreeRTOSConfig.h` one parameter at a time and observe the effect:

1. Set `configTICK_RATE_HZ` to `100` — what happens to `vTaskDelay(pdMS_TO_TICKS(500))`?
2. Set `configUSE_PREEMPTION` to `0` — does the system still work?
3. Set `configTOTAL_HEAP_SIZE` to `2048` — does the system boot?
4. Set `configMAX_PRIORITIES` to `2` — what changes?
5. Set `configCHECK_FOR_STACK_OVERFLOW` to `0` — is this a good idea? (No!)

### Exercise 4.3: Memory Budget

For your specific board:
1. Find the total RAM in the datasheet
2. Calculate `configTOTAL_HEAP_SIZE` as ~50% of available RAM
3. Create 3 tasks with stacks of 128, 256, and 512 words
4. Print `xPortGetFreeHeapSize()` after all tasks are created
5. Verify the math matches your expectation

---

## 4.9 Chapter Summary

| Concept | Key Takeaway |
|---|---|
| **STM32CubeIDE** | Free IDE with integrated CubeMX code generator |
| **FreeRTOS integration** | Enable via CubeMX Middleware or add source manually |
| **Port selection** | Must match your Cortex-M core (CM3, CM4F, CM7) |
| **FreeRTOSConfig.h** | Master configuration — tick rate, heap, priorities, hooks |
| **SysTick conflict** | Use TIM6 for HAL timebase, SysTick for FreeRTOS |
| **Interrupt priorities** | ISRs calling FreeRTOS API must be ≥ configMAX_SYSCALL |
| **configASSERT** | Must define it — catches errors during development |
| **HAL_Delay()** | Never use in tasks; use vTaskDelay() instead |
| **Heap size** | TCB + stack per task; verify with xPortGetFreeHeapSize() |

---

## 4.10 What's Next

In **Chapter 5: Tasks in FreeRTOS**, we will go deeper into:
- The `xTaskCreate` API in complete detail
- Task priorities and starvation
- The Idle task internals
- Task notifications as a lightweight signaling mechanism

---

*End of Chapter 4*
