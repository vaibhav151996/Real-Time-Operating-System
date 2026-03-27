# Chapter 2: RTOS Architecture

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Describe the internal structure of an RTOS kernel
2. Explain the role of the scheduler, tick timer, and system services
3. Compare preemptive, cooperative, and hybrid scheduling algorithms
4. Understand how the SysTick timer drives the RTOS tick
5. Explain tick-based vs. tickless operation and the trade-offs
6. Map FreeRTOS kernel components to ARM Cortex-M hardware

---

## 2.1 Kernel Structure

### 2.1.1 What Is a Kernel?

The **kernel** is the core of the RTOS — the code that manages tasks, time, and resources. Everything else (your application tasks, drivers, middleware) runs **on top of** the kernel.

#### Analogy: The Air Traffic Controller

Think of the kernel as an air traffic controller (ATC) at a busy airport:

- **Aircraft** = Tasks (each with a destination and priority)
- **Runway** = CPU (only one aircraft can use it at a time)
- **ATC** = Kernel (decides who lands/takes off next)
- **Radar** = Tick timer (provides periodic updates)
- **Radio** = System calls (tasks communicate with ATC)

The ATC never flies an airplane — it just decides who gets the runway and when. Similarly, the kernel never does application work — it just manages tasks.

### 2.1.2 Kernel Components

```
┌──────────────────────────────────────────────────────────────────┐
│                         APPLICATION                              │
│    ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐          │
│    │ Task 1  │  │ Task 2  │  │ Task 3  │  │ Task N  │          │
│    └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘          │
│         │            │            │            │                 │
├─────────┴────────────┴────────────┴────────────┴─────────────────┤
│                         RTOS KERNEL                              │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │  SCHEDULER   │  │    TIMER     │  │   IPC SERVICES        │  │
│  │              │  │  MANAGEMENT  │  │                       │  │
│  │ • Ready list │  │              │  │ • Queues              │  │
│  │ • Priority   │  │ • Tick count │  │ • Semaphores          │  │
│  │   queues     │  │ • Delay list │  │ • Mutexes             │  │
│  │ • Context    │  │ • SW Timers  │  │ • Event Groups        │  │
│  │   switch     │  │              │  │ • Notifications       │  │
│  └──────┬───────┘  └──────┬───────┘  └───────────┬───────────┘  │
│         │                 │                      │               │
│  ┌──────┴─────────────────┴──────────────────────┴───────────┐  │
│  │                    MEMORY MANAGEMENT                       │  │
│  │   heap_1 │ heap_2 │ heap_3 │ heap_4 │ heap_5              │  │
│  └──────────────────────────┬────────────────────────────────┘  │
│                              │                                   │
├──────────────────────────────┴───────────────────────────────────┤
│                    HARDWARE ABSTRACTION                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │ SysTick  │  │ PendSV   │  │  SVC     │  │  NVIC    │        │
│  │ (Tick)   │  │ (Switch) │  │ (Start)  │  │ (IRQs)  │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
└──────────────────────────────────────────────────────────────────┘
│                    ARM CORTEX-M HARDWARE                         │
```

Let's examine each component:

### 2.1.3 The Scheduler

The scheduler is the **brain** of the kernel. It answers one question:

> **"Which task should the CPU run right now?"**

The scheduler maintains:

1. **Ready List** — Tasks that are ready to run, organized by priority
2. **Blocked List** — Tasks waiting for time, events, or resources
3. **Suspended List** — Tasks explicitly suspended by the application

```
SCHEDULER DATA STRUCTURES (FreeRTOS):
═════════════════════════════════════

Ready Lists (one per priority level):
┌─────────────────────────────────────────┐
│ Priority 0: ──► [Idle Task]             │
│ Priority 1: ──► [Task_UART] ──► [...]   │
│ Priority 2: ──► [Task_LED]              │
│ Priority 3: ──► [Task_Button]           │
│ ...                                     │
│ Priority N: ──► (empty)                 │
└─────────────────────────────────────────┘

The scheduler always picks the HIGHEST non-empty priority list
and runs the FIRST task in that list.

Delayed Task List (sorted by wake-up time):
┌─────────────────────────────────────────┐
│ Wake at tick 1050: Task_LED              │
│ Wake at tick 1100: Task_UART             │
│ Wake at tick 2000: Task_Logger           │
└─────────────────────────────────────────┘
```

### 2.1.4 Timer Management

The timer system provides:

- **Tick counter:** A monotonically increasing counter (incremented every tick)
- **Delay lists:** Tasks sorted by their wake-up time
- **Software timers:** Application-level timers managed by a dedicated daemon task

### 2.1.5 IPC (Inter-Process Communication) Services

The kernel provides synchronization and communication primitives:

| Primitive | Purpose | Chapter |
|---|---|---|
| Queue | Transfer data between tasks | 7 |
| Binary Semaphore | Signal events (one-shot) | 8 |
| Counting Semaphore | Track available resources | 8 |
| Mutex | Mutual exclusion with priority inheritance | 9 |
| Event Group | Synchronize on multiple conditions | 10 |
| Task Notification | Lightweight signaling (zero overhead) | 10 |

### 2.1.6 Memory Management

FreeRTOS offers 5 heap implementations (heap_1 through heap_5), each with different trade-offs between simplicity, fragmentation, and determinism. We cover this in detail in Chapter 12.

### 2.1.7 Hardware Abstraction Layer (Port Layer)

FreeRTOS is portable across many architectures. The port layer adapts the kernel to specific hardware. For ARM Cortex-M, this involves:

| Hardware Feature | RTOS Usage |
|---|---|
| **SysTick timer** | Generates the periodic tick interrupt |
| **PendSV exception** | Performs context switches |
| **SVC instruction** | Starts the first task |
| **NVIC** | Manages interrupt priorities |
| **PSP/MSP** | Process/Main Stack Pointer separation |

```
ARM Cortex-M Exception Priorities for FreeRTOS:
═══════════════════════════════════════════════

  HIGHEST PRIORITY (lowest number)
     │
     │  ┌──────────────────────────────┐
     │  │ HardFault, NMI, etc.         │  ◄── Cannot be masked
     │  ├──────────────────────────────┤
     │  │ Your hardware interrupts     │  ◄── MUST be above configMAX_SYSCALL_...
     │  │ (UART, SPI, ADC, etc.)       │     if they DON'T call FreeRTOS APIs
     │  ├──────────────────────────────┤
     │  │ configMAX_SYSCALL_INTERRUPT  │  ◄── BOUNDARY
     │  │ _PRIORITY                    │
     │  ├──────────────────────────────┤
     │  │ Hardware interrupts that     │  ◄── CAN call FromISR APIs
     │  │ call FreeRTOS APIs           │
     │  ├──────────────────────────────┤
     │  │ SysTick (kernel tick)        │  ◄── configKERNEL_INTERRUPT_PRIORITY
     │  ├──────────────────────────────┤
     │  │ PendSV (context switch)      │  ◄── LOWEST priority (always)
     │  └──────────────────────────────┘
     │
  LOWEST PRIORITY (highest number)

  NOTE: On Cortex-M, lower numerical value = higher priority!
  This is the OPPOSITE of FreeRTOS task priorities.
```

> **⚠️ Critical:** The interrupt priority numbering on Cortex-M is **inverted** compared to FreeRTOS task priorities. Cortex-M: 0 = highest priority. FreeRTOS tasks: 0 = lowest priority (idle task). This is a frequent source of bugs.

---

## 2.2 Scheduler Types

### 2.2.1 The Scheduling Problem

Given N tasks ready to run and 1 CPU, the scheduler must decide:
1. **Which** task runs?
2. **For how long** does it run?
3. **When** should the kernel switch to another task?

Different scheduling algorithms answer these questions differently.

### 2.2.2 Preemptive Priority-Based Scheduling

This is the **default** and most common scheduling algorithm in FreeRTOS.

**Rules:**
1. The task with the **highest priority** always runs
2. If a higher-priority task becomes ready, it **immediately preempts** (takes over from) the currently running task
3. Tasks of **equal priority** share time using **round-robin** (time-slicing)

```
PREEMPTIVE SCHEDULING EXAMPLE:
══════════════════════════════

Tasks: A (Priority 3), B (Priority 2), C (Priority 1)

Time ─────────────────────────────────────────────────────►

Task A: ████          ████               ████
Task B:     ████████       ████████████       ████████
Task C:                                               ████
         │         │         │         │         │
         ▲         ▲         ▲         ▲         ▲
       A ready   A blocks  A ready   A blocks  B blocks
                  B runs    preempts  B runs    C runs
                            B

KEY POINT: When Task A (highest priority) is ready, it
IMMEDIATELY preempts Task B, even mid-execution.
```

**Configuration in FreeRTOS:**
```c
/* FreeRTOSConfig.h */
#define configUSE_PREEMPTION        1    /* Enable preemptive scheduling */
#define configUSE_TIME_SLICING      1    /* Round-robin for equal priority */
```

### 2.2.3 Cooperative Scheduling

In cooperative scheduling, a task runs until it **voluntarily** yields the CPU. The kernel never forcibly preempts a task.

**Rules:**
1. The current task runs until it calls `taskYIELD()` or a blocking API
2. Even if a higher-priority task is ready, it must wait
3. Tasks are expected to be "well-behaved" and yield regularly

```
COOPERATIVE SCHEDULING EXAMPLE:
════════════════════════════════

Tasks: A (Priority 3), B (Priority 2)

Time ─────────────────────────────────────────────────────►

Task B: ██████████████████████████████████
Task A:                ← A becomes ready here, but B doesn't yield!
                        A is STARVED until B cooperates.

PROBLEM: If Task B has a bug (infinite loop, no yield),
Task A NEVER runs, even though it has higher priority.
```

**Configuration in FreeRTOS:**
```c
/* FreeRTOSConfig.h */
#define configUSE_PREEMPTION        0    /* Disable preemption = cooperative */
```

**When is cooperative scheduling useful?**
- Simpler systems where all tasks are trusted
- Avoiding the complexity of shared-resource protection (no preemption = no race conditions within task code)
- Educational purposes

> **🏭 Industry Insight:** Cooperative scheduling is rarely used in production real-time systems because it cannot guarantee response times. A single misbehaving task blocks the entire system. Preemptive scheduling is the industry standard.

### 2.2.4 Hybrid Scheduling (Preemptive Without Time-Slicing)

FreeRTOS can be configured for preemptive scheduling **without** round-robin time-slicing for equal-priority tasks.

**Rules:**
1. Higher-priority tasks preempt lower-priority tasks (same as preemptive)
2. Equal-priority tasks do NOT share time — the first one runs until it blocks or yields

```c
/* FreeRTOSConfig.h */
#define configUSE_PREEMPTION        1    /* Preemptive ON */
#define configUSE_TIME_SLICING      0    /* Time-slicing OFF */
```

```
HYBRID SCHEDULING WITH EQUAL-PRIORITY TASKS:
═════════════════════════════════════════════

Tasks: A (Priority 2), B (Priority 2)  — SAME priority

WITH time-slicing (configUSE_TIME_SLICING = 1):
Task A: ████    ████    ████    ████    ████
Task B:     ████    ████    ████    ████
        Each tick, the scheduler alternates (round-robin).

WITHOUT time-slicing (configUSE_TIME_SLICING = 0):
Task A: ████████████████████████████████████████
Task B:                                     ████
        Task A runs until it blocks/yields. Only then B runs.
```

### 2.2.5 Comparison of Scheduling Types

| Feature | Preemptive | Cooperative | Hybrid |
|---|---|---|---|
| **Responsiveness** | Excellent | Poor | Good |
| **Determinism** | High | Depends on tasks | High |
| **Complexity** | Medium | Low | Medium |
| **Race conditions** | Must protect shared data | None within tasks | Must protect shared data |
| **Starvation risk** | Low (if priorities correct) | High | Medium |
| **Time-slicing** | Yes (for equal priority) | No | No |
| **configUSE_PREEMPTION** | 1 | 0 | 1 |
| **configUSE_TIME_SLICING** | 1 | N/A | 0 |
| **Industry usage** | ★★★★★ | ★☆☆☆☆ | ★★★☆☆ |

### 2.2.6 Advanced: The Ready List Implementation

FreeRTOS implements the ready list as an **array of linked lists**, one per priority level:

```c
/*
 * FreeRTOS internal (simplified from tasks.c):
 *
 * INTERNAL_STATIC List_t pxReadyTasksLists[configMAX_PRIORITIES];
 *
 * This is an array where:
 *   pxReadyTasksLists[0] = list of ready tasks at priority 0
 *   pxReadyTasksLists[1] = list of ready tasks at priority 1
 *   ...
 *   pxReadyTasksLists[configMAX_PRIORITIES - 1] = highest priority
 */
```

The scheduler finds the highest-priority ready task in **O(1) time** using a leading-zero-count instruction on Cortex-M:

```c
/*
 * How FreeRTOS finds the highest-priority ready task:
 * (portGET_HIGHEST_PRIORITY macro)
 *
 * uxTopReadyPriority is a bitmap where bit N is set if there
 * are ready tasks at priority N.
 *
 * CLZ (Count Leading Zeros) instruction gives the position
 * of the highest set bit in one clock cycle.
 */
#define portGET_HIGHEST_PRIORITY(uxTopPriority, uxReadyPriorities) \
    uxTopPriority = (31UL - __clz((uxReadyPriorities)))
```

This means the scheduler's decision time is **constant**, regardless of how many tasks exist. This is critical for determinism.

---

## 2.3 Tick vs. Tickless

### 2.3.1 The RTOS Tick

The **tick** is the heartbeat of the RTOS. It is a periodic interrupt (typically every 1 ms) that gives the kernel an opportunity to:

1. **Increment** the tick counter
2. **Check** if any delayed tasks should wake up
3. **Perform** time-slicing (round-robin for equal-priority tasks)
4. **Trigger** a context switch if a higher-priority task became ready

#### Tick Timing Diagram

```
SysTick Interrupts (1 ms period, configTICK_RATE_HZ = 1000):

Time (ms):  0    1    2    3    4    5    6    7    8    9   10
            │    │    │    │    │    │    │    │    │    │    │
Ticks:      ▼    ▼    ▼    ▼    ▼    ▼    ▼    ▼    ▼    ▼    ▼
            0    1    2    3    4    5    6    7    8    9   10

Each tick, the kernel runs xTaskIncrementTick():
  - tickCount++
  - Check delayed task list → move woken tasks to ready list
  - If a woken task has higher priority → set pendSV flag
  
After SysTick ISR exits:
  - PendSV runs (if flagged) → context switch
```

### 2.3.2 SysTick Configuration on STM32

The SysTick timer is a 24-bit down-counter built into every Cortex-M core.

```c
/*
 * SysTick configuration for 1 ms tick at 168 MHz
 * (This is done automatically by FreeRTOS port layer)
 */

/* SysTick reload value = (SystemCoreClock / configTICK_RATE_HZ) - 1 */
/* = (168,000,000 / 1000) - 1 = 167,999 */

void vPortSetupTimerInterrupt(void)
{
    /* Configure SysTick to interrupt at the requested rate */
    SysTick->LOAD  = (SystemCoreClock / configTICK_RATE_HZ) - 1UL;
    SysTick->VAL   = 0UL;                             /* Clear counter */
    SysTick->CTRL  = SysTick_CTRL_CLKSOURCE_Msk |     /* Use processor clock */
                     SysTick_CTRL_TICKINT_Msk   |     /* Enable interrupt */
                     SysTick_CTRL_ENABLE_Msk;          /* Enable SysTick */
}
```

### 2.3.3 Tick Rate Selection

The tick rate (`configTICK_RATE_HZ`) determines the **time resolution** of the RTOS:

| Tick Rate | Period | Resolution | Overhead | Use Case |
|---|---|---|---|---|
| 100 Hz | 10 ms | 10 ms | Very low | Low-power, infrequent tasks |
| 250 Hz | 4 ms | 4 ms | Low | General purpose |
| **1000 Hz** | **1 ms** | **1 ms** | **Medium** | **Most common (default)** |
| 10000 Hz | 100 μs | 100 μs | High | High-frequency control loops |

**Trade-offs:**
- **Higher tick rate** = Better timing resolution, more CPU overhead
- **Lower tick rate** = Less overhead, coarser timing granularity
- Most STM32 + FreeRTOS projects use **1000 Hz** (1 ms tick)

> **⚠️ Warning:** Setting tick rate too high wastes CPU time on scheduler overhead. Too low and `vTaskDelay(1)` delays for too long (10 ms at 100 Hz). For most applications, 1000 Hz is the sweet spot.

### 2.3.4 The Tick ISR: What Actually Happens

```c
/*
 * Simplified view of what happens every tick
 * (inside xTaskIncrementTick, called from SysTick_Handler)
 */

BaseType_t xTaskIncrementTick(void)
{
    BaseType_t xSwitchRequired = pdFALSE;

    /* 1. Increment the tick count */
    xTickCount++;

    /* 2. Check if any tasks are waiting for this tick */
    if (listLIST_IS_EMPTY(&xDelayedTaskList) == pdFALSE)
    {
        TCB_t *pxTCB = listGET_OWNER_OF_HEAD_ENTRY(&xDelayedTaskList);

        if (xTickCount >= pxTCB->xTicksToWait)
        {
            /* 3. Move task from delayed list to ready list */
            uxListRemove(&(pxTCB->xStateListItem));
            prvAddTaskToReadyList(pxTCB);

            /* 4. If the woken task has higher priority, request switch */
            if (pxTCB->uxPriority >= pxCurrentTCB->uxPriority)
            {
                xSwitchRequired = pdTRUE;
            }
        }
    }

    /* 5. Time slicing: if time-slicing enabled and there are other
       tasks at the current priority, request a switch */
    #if (configUSE_TIME_SLICING == 1)
    if (listCURRENT_LIST_LENGTH(
            &pxReadyTasksLists[pxCurrentTCB->uxPriority]) > 1)
    {
        xSwitchRequired = pdTRUE;
    }
    #endif

    return xSwitchRequired;  /* If true, PendSV will fire */
}
```

### 2.3.5 Tickless Idle Mode

In many embedded systems (especially battery-powered), the processor spends most of its time **idle** — no tasks are ready to run. In normal tick mode, the SysTick interrupt still fires every 1 ms, waking the processor from sleep just to increment a counter.

**Tickless idle** suppresses the tick interrupt during idle periods, allowing the processor to stay in deep sleep for much longer.

```
NORMAL TICK MODE (wasteful for battery systems):
═════════════════════════════════════════════════

        Idle         Task A    Idle           Task A
Time: ──────────────│████████│──────────────│████████│──
       ↑ ↑ ↑ ↑ ↑ ↑            ↑ ↑ ↑ ↑ ↑ ↑
       Tick interrupts keep    Tick interrupts keep
       waking CPU (WASTED!)    waking CPU (WASTED!)


TICKLESS MODE (power-efficient):
═════════════════════════════════

        Deep Sleep       Task A    Deep Sleep       Task A
Time: ──────────────────│████████│──────────────────│████████│──
       ↑                          ↑
       SysTick suppressed,        SysTick suppressed,
       CPU stays asleep.          CPU stays asleep.
       Counter adjusted on        Counter adjusted on
       wake-up.                   wake-up.
```

**How it works:**

1. When all tasks are blocked, the idle task runs
2. The kernel calculates when the **next** task will wake up (e.g., 50 ms from now)
3. It reprograms SysTick for a **50 ms** period (instead of 1 ms)
4. Calls `__WFI()` (Wait For Interrupt) — CPU enters low-power mode
5. On wake-up (either at 50 ms or earlier from an external interrupt):
   - The kernel reads how long the CPU actually slept
   - Adjusts the tick counter by the elapsed time
   - Resumes normal operation

**Configuration:**
```c
/* FreeRTOSConfig.h */
#define configUSE_TICKLESS_IDLE     1    /* Enable tickless idle */
#define configEXPECTED_IDLE_TIME_BEFORE_SLEEP    2    /* Minimum ticks before sleeping */
```

### 2.3.6 Tickless Implementation on STM32

```c
/*
 * Simplified tickless idle implementation (port layer).
 * FreeRTOS provides a default one for Cortex-M.
 */

void vPortSuppressTicksAndSleep(TickType_t xExpectedIdleTime)
{
    /* 1. Calculate the reload value for the extended sleep */
    uint32_t ulReloadValue = (xExpectedIdleTime * ulTimerCountsPerTick) - 1UL;

    /* 2. Stop SysTick */
    SysTick->CTRL &= ~SysTick_CTRL_ENABLE_Msk;

    /* 3. Set new reload value for extended period */
    SysTick->LOAD = ulReloadValue;
    SysTick->VAL  = 0UL;

    /* 4. Re-enable SysTick */
    SysTick->CTRL |= SysTick_CTRL_ENABLE_Msk;

    /* 5. Enter sleep mode */
    __DSB();
    __WFI();    /* CPU sleeps until interrupt */
    __ISB();

    /* 6. On wake-up: calculate how long we actually slept */
    uint32_t ulCountAfterSleep = SysTick->VAL;
    uint32_t ulActualSleepTicks = (ulReloadValue - ulCountAfterSleep)
                                  / ulTimerCountsPerTick;

    /* 7. Compensate the tick counter */
    vTaskStepTick(ulActualSleepTicks);

    /* 8. Restore normal 1 ms tick period */
    SysTick->LOAD = ulTimerCountsPerTick - 1UL;
    SysTick->VAL  = 0UL;
}
```

### 2.3.7 Tick vs. Tickless: Comparison

| Feature | Tick Mode | Tickless Mode |
|---|---|---|
| **Power consumption** | Higher (constant wake-ups) | Much lower |
| **Timing accuracy** | 1 tick resolution always | Compensated on wake-up |
| **Implementation** | Simple | More complex |
| **Best for** | Always-on systems | Battery-powered IoT |
| **CPU wake-ups** | Every tick | Only when needed |
| **STM32 sleep modes** | Cannot use deep sleep | Can use STOP/STANDBY |
| **Jitter** | Low | Slightly higher after long sleep |

> **📝 Beginner Note:** Start with normal tick mode (configUSE_TICKLESS_IDLE = 0). Only enable tickless when you need power optimization. It adds complexity but saves significant battery life.

> **🏭 Industry Insight:** Tickless idle is essential for battery-powered IoT devices (LoRa sensors, BLE beacons, wearables). A device that wakes up 1000 times/second to increment a counter drains its battery significantly faster than one that sleeps for seconds at a time.

---

## 2.4 FreeRTOS Kernel Startup Sequence

Understanding the boot sequence helps with debugging and system bring-up:

```
STM32 + FreeRTOS Boot Sequence:
═══════════════════════════════

1. POWER ON / RESET
       │
       ▼
2. Reset_Handler (startup assembly)
   - Initialize .data section (copy from Flash to RAM)
   - Zero .bss section
   - Call SystemInit()
       │
       ▼
3. main()
   - HAL_Init()               ◄── Initialize HAL, configure SysTick for HAL
   - SystemClock_Config()      ◄── Configure HSE, PLL, AHB/APB clocks
   - Peripheral_Init()         ◄── Initialize GPIO, UART, SPI, etc.
       │
       ▼
4. Create Tasks
   - xTaskCreate(Task1, ...)
   - xTaskCreate(Task2, ...)
   - xTaskCreate(TaskN, ...)
       │
       ▼
5. vTaskStartScheduler()
   │
   ├── Creates Idle Task (priority 0)
   ├── Creates Timer Daemon Task (if SW timers enabled)
   ├── Configures SysTick for RTOS tick ◄── OVERRIDES HAL SysTick!
   ├── Sets PendSV priority to LOWEST
   ├── Sets SysTick priority to configKERNEL_INTERRUPT_PRIORITY
   ├── Calls xPortStartScheduler()
   │       │
   │       ├── Starts first task using SVC instruction
   │       │
   │       ▼
   │   FIRST TASK BEGINS EXECUTING
   │   (highest priority task created in step 4)
   │
   └── *** NEVER RETURNS ***
       (if it does, out of heap memory)

6. SysTick fires every 1 ms
   → Kernel checks for task wake-ups
   → Round-robin if applicable
   → PendSV triggers context switch if needed

7. SYSTEM IS NOW RUNNING
   Tasks execute, block, wake up, preempt each other...
```

> **⚠️ Critical:** `vTaskStartScheduler()` should **never return**. If it does, it means FreeRTOS could not allocate memory for the idle task. Always add an infinite loop after it as a safety net.

---

## 2.5 Common Mistakes (Chapter 2)

### Mistake 1: Wrong Interrupt Priority Configuration
```c
/* WRONG: UART interrupt at priority 0 (highest) calling FreeRTOS API */
HAL_NVIC_SetPriority(USART2_IRQn, 0, 0);
/* This will cause a hard fault! Priority must be >= configMAX_SYSCALL_INTERRUPT_PRIORITY */

/* RIGHT: UART interrupt at priority 5 (below max syscall) */
HAL_NVIC_SetPriority(USART2_IRQn, 5, 0);
/* Now FreeRTOS FromISR APIs can be safely called */
```

### Mistake 2: Confusing Cortex-M and FreeRTOS Priority Direction
- Cortex-M hardware: **0 = highest** priority
- FreeRTOS tasks: **0 = lowest** priority (idle task)
- This confusion causes hard-to-debug priority issues

### Mistake 3: Using HAL_Delay() After Scheduler Starts
```c
/* WRONG: HAL_Delay() uses SysTick, which is now owned by FreeRTOS */
void Task1(void *pvParameters)
{
    for (;;)
    {
        HAL_Delay(100);    /* BAD! Busy-waits, wastes CPU, may malfunction */
    }
}

/* RIGHT: Use FreeRTOS delay */
void Task1(void *pvParameters)
{
    for (;;)
    {
        vTaskDelay(pdMS_TO_TICKS(100));    /* Yields CPU while waiting */
    }
}
```

### Mistake 4: Setting Tick Rate Too High Without Justification
A tick rate of 10000 Hz means the SysTick ISR fires 10,000 times per second. On a 72 MHz Cortex-M3, each ISR takes ~2-5 μs = up to 5% CPU overhead just for ticks.

### Mistake 5: Not Placing an Infinite Loop After vTaskStartScheduler()
```c
int main(void)
{
    /* ... setup ... */
    vTaskStartScheduler();

    /* If we reach here, heap allocation for idle task failed */
    /* WITHOUT the loop below, the CPU executes random memory! */
    for (;;)
    {
        /* Error handler: turn on error LED, log, etc. */
    }
}
```

---

## 2.6 Debugging Tips

1. **Verify SysTick is firing:** Toggle a GPIO pin in `SysTick_Handler`. Measure with oscilloscope — period should match 1/configTICK_RATE_HZ
2. **Check PendSV priority:** Must be `0xFF` (lowest). Use the debugger to read `SCB->SHP[10]`
3. **Check SysTick priority:** Must be `configKERNEL_INTERRUPT_PRIORITY`. Read `SCB->SHP[11]`
4. **Verify heap size:** If `vTaskStartScheduler()` returns, `configTOTAL_HEAP_SIZE` is too small
5. **Use configASSERT():** Define it to catch configuration errors at runtime:
```c
#define configASSERT(x) if ((x) == 0) { taskDISABLE_INTERRUPTS(); for(;;); }
```

---

## 2.7 Hands-On Exercises

### Exercise 2.1: Identify the Scheduler Type

For each scenario, identify whether preemptive, cooperative, or hybrid scheduling is being used:

1. Task B is running. Task A (higher priority) becomes ready. Task A immediately starts running.
2. Task B is running. Task A (higher priority) becomes ready. Task B continues until it calls `taskYIELD()`.
3. Task A and Task B have the same priority. The scheduler alternates between them every tick.
4. Task A and Task B have the same priority. Task A runs until it calls `vTaskDelay()`.

<details>
<summary><strong>Solutions</strong></summary>

1. **Preemptive** — Higher-priority task preempts immediately
2. **Cooperative** — No preemption; tasks must yield voluntarily
3. **Preemptive with time-slicing** — Round-robin for equal priorities
4. **Hybrid (preemptive without time-slicing)** — Equal-priority tasks don't time-slice

</details>

### Exercise 2.2: SysTick Calculation

Given an STM32F407 running at 168 MHz:

1. What is the SysTick reload value for a 1 ms tick? (Answer: 168,000 - 1 = 167,999)
2. What is the maximum period the 24-bit SysTick can generate? (Answer: 2^24 / 168,000,000 = ~99.86 ms)
3. If you need a 100 μs tick, what is the reload value? (Answer: 16,800 - 1 = 16,799)
4. What percentage of CPU time does the tick ISR consume if it takes 2 μs per invocation at 1000 Hz? (Answer: 0.2%)
5. At what tick rate does the tick ISR consume 10% of CPU time? (Answer: 50,000 Hz)

### Exercise 2.3: Boot Sequence Verification

**Objective:** Verify the FreeRTOS boot sequence using GPIO toggles.

```c
/*
 * Place GPIO toggles at key points in the boot sequence.
 * Use a logic analyzer to verify the order and timing.
 */

void Toggle_Debug_Pin(GPIO_TypeDef *port, uint16_t pin, uint32_t count)
{
    for (uint32_t i = 0; i < count; i++)
    {
        HAL_GPIO_TogglePin(port, pin);
        volatile uint32_t delay = 1000;
        while (delay--);
        HAL_GPIO_TogglePin(port, pin);
    }
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();

    Toggle_Debug_Pin(GPIOA, GPIO_PIN_0, 1);  /* 1 pulse: after init */

    xTaskCreate(Task1, "T1", 128, NULL, 1, NULL);
    Toggle_Debug_Pin(GPIOA, GPIO_PIN_0, 2);  /* 2 pulses: tasks created */

    vTaskStartScheduler();
    /* If we get here: 3 pulses = scheduler failed */
    Toggle_Debug_Pin(GPIOA, GPIO_PIN_0, 3);
    for (;;);
}

void Task1(void *pvParameters)
{
    Toggle_Debug_Pin(GPIOA, GPIO_PIN_0, 4);  /* 4 pulses: first task running */
    for (;;)
    {
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

**Questions:**
1. What pattern should you see on the logic analyzer during normal boot? (1 pulse → 2 pulses → 4 pulses)
2. If you see 1 → 2 → 3, what went wrong? (Scheduler failed — out of heap)
3. What additional check can you add? (Verify SysTick interrupt is firing)

---

## 2.8 Chapter Summary

| Concept | Key Takeaway |
|---|---|
| **Kernel** | Core RTOS code: scheduler + timer + IPC + memory |
| **Scheduler** | Decides which task runs; O(1) on Cortex-M |
| **Preemptive** | Higher-priority tasks immediately take over CPU |
| **Cooperative** | Tasks must voluntarily yield; rarely used |
| **Ready list** | Array of linked lists, one per priority level |
| **SysTick** | 24-bit Cortex-M timer driving RTOS tick |
| **Tick rate** | Typically 1000 Hz; trade-off: resolution vs. overhead |
| **Tickless idle** | Suppresses ticks during sleep for power saving |
| **PendSV** | Lowest-priority exception used for context switching |
| **Boot sequence** | Init → Create tasks → Start scheduler → Never returns |

---

## 2.9 What's Next

In **Chapter 3: Task Management**, we will dive deep into:
- How to create and delete tasks with `xTaskCreate` and `vTaskDelete`
- The four task states: Running, Ready, Blocked, Suspended
- The Task Control Block (TCB) — what the kernel stores per task
- Stack management — how much stack each task needs and how to detect overflow

---

*End of Chapter 2*
