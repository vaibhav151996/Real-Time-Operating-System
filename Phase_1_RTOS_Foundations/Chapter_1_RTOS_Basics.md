# Chapter 1: RTOS Basics

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Define what an RTOS is and explain how it differs from bare-metal programming
2. Distinguish between hard and soft real-time systems with concrete examples
3. Explain determinism and why it matters more than raw speed
4. Differentiate between throughput and response time
5. Describe what context switching is and why it is the heartbeat of multitasking
6. Identify when an RTOS is needed vs. when bare-metal is sufficient

---

## 1.1 What is an RTOS vs. Bare-Metal?

### 1.1.1 Starting from First Principles

Before we talk about an RTOS, let us understand the simplest way firmware runs on a microcontroller: **bare-metal programming**.

When you write a bare-metal program for an STM32 (or any microcontroller), your code typically looks like this:

```c
int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_USART2_UART_Init();

    while (1)
    {
        Read_Sensors();
        Process_Data();
        Update_Display();
        Send_UART_Data();
        HAL_Delay(100);
    }
}
```

This is called a **super-loop** (or main-loop) architecture. The processor executes each function in sequence, one after the other, forever.

#### The Super-Loop Mental Model

```
┌─────────────────────────────────────────────┐
│                  main()                      │
│                                              │
│   ┌──────────────┐                           │
│   │   HAL_Init   │                           │
│   └──────┬───────┘                           │
│          ▼                                   │
│   ┌──────────────┐                           │
│   │  Clock_Config │                          │
│   └──────┬───────┘                           │
│          ▼                                   │
│   ┌──────────────────────────────────────┐   │
│   │            while(1) Loop             │   │
│   │                                      │   │
│   │   Read_Sensors()  ──► Process_Data() │   │
│   │        ▲                    │        │   │
│   │        │                    ▼        │   │
│   │   HAL_Delay(100)  ◄── Send_UART()   │   │
│   │                                      │   │
│   └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

**Key characteristic:** Only ONE thing happens at a time. If `Process_Data()` takes 50 ms, the UART data will wait 50 ms before being sent. If a sensor needs to be read every 10 ms, but the loop takes 200 ms, you **miss** readings.

### 1.1.2 The Problem with Super-Loop

Imagine you are a chef who must:
- Stir the soup every 2 minutes
- Check the oven every 5 minutes
- Chop vegetables continuously

In a super-loop world, you would:
1. Stir the soup
2. Then check the oven
3. Then chop vegetables
4. Then go back to step 1

**What if chopping takes 10 minutes?** You miss stirring the soup (which burns) and you miss checking the oven (which overcooks). The super-loop cannot handle tasks with **different timing requirements**.

> **🔑 Key Insight:** The super-loop treats all tasks equally. In the real world, tasks have different priorities and deadlines.

### 1.1.3 Enter the RTOS

An **RTOS (Real-Time Operating System)** is a specialized operating system designed to run multiple tasks with **predictable timing guarantees**.

With an RTOS, the same program becomes:

```c
/* Task 1: Read sensors every 10 ms */
void Task_ReadSensors(void *pvParameters)
{
    for (;;)
    {
        Read_Sensors();
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}

/* Task 2: Process data every 50 ms */
void Task_ProcessData(void *pvParameters)
{
    for (;;)
    {
        Process_Data();
        vTaskDelay(pdMS_TO_TICKS(50));
    }
}

/* Task 3: Send UART data every 100 ms */
void Task_SendUART(void *pvParameters)
{
    for (;;)
    {
        Send_UART_Data();
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();

    xTaskCreate(Task_ReadSensors,  "Sensors",  128, NULL, 3, NULL);
    xTaskCreate(Task_ProcessData,  "Process",  128, NULL, 2, NULL);
    xTaskCreate(Task_SendUART,     "UART",     128, NULL, 1, NULL);

    vTaskStartScheduler();

    /* Should never reach here */
    for (;;);
}
```

#### The RTOS Mental Model

```
┌────────────────────────────────────────────────────────┐
│                   RTOS KERNEL                          │
│                                                        │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐          │
│   │  Task 1  │   │  Task 2  │   │  Task 3  │          │
│   │ Sensors  │   │ Process  │   │  UART    │          │
│   │ Pri: 3   │   │ Pri: 2   │   │ Pri: 1   │          │
│   │ 10 ms    │   │ 50 ms    │   │ 100 ms   │          │
│   └────┬─────┘   └────┬─────┘   └────┬─────┘          │
│        │              │              │                 │
│        └──────────────┼──────────────┘                 │
│                       ▼                                │
│              ┌──────────────┐                          │
│              │  SCHEDULER   │                          │
│              │  Decides who │                          │
│              │  runs next   │                          │
│              └──────────────┘                          │
│                       │                                │
│                       ▼                                │
│              ┌──────────────┐                          │
│              │     CPU      │                          │
│              │  (one core)  │                          │
│              └──────────────┘                          │
└────────────────────────────────────────────────────────┘
```

**What happens:** The RTOS **scheduler** rapidly switches between tasks, giving the **illusion** of parallelism on a single-core processor. But unlike a desktop OS, it does so with **predictable, bounded timing**.

### 1.1.4 RTOS vs. Bare-Metal: Side-by-Side Comparison

| Feature | Bare-Metal (Super-Loop) | RTOS |
|---|---|---|
| **Execution model** | Sequential, one-at-a-time | Concurrent, preemptive |
| **Timing control** | Manual delays, flags | Kernel-managed scheduling |
| **Priority handling** | None (all tasks equal) | Priority-based preemption |
| **Responsiveness** | Depends on loop duration | Bounded, deterministic |
| **Code structure** | One big loop | Independent task functions |
| **Complexity** | Simple for small systems | Higher initial learning curve |
| **RAM overhead** | Minimal | Stack per task + kernel data |
| **Scalability** | Poor for many tasks | Excellent |
| **Debugging** | Straightforward | Requires RTOS-aware tools |
| **Best for** | Simple, single-purpose | Complex, multi-deadline |

### 1.1.5 When to Use Bare-Metal vs. RTOS

**Use bare-metal when:**
- The system has only 1–3 simple tasks
- All tasks have similar timing requirements
- RAM is extremely constrained (< 4 KB)
- The application is a simple control loop
- Example: A blinking LED, a basic thermostat

**Use an RTOS when:**
- Multiple tasks have **different** deadlines
- You need to **guarantee** response times
- The system must handle asynchronous events (UART, sensors, buttons)
- Code modularity and maintainability matter
- Example: Industrial controller, drone flight system, medical device

> **📝 Beginner Note:** Don't use an RTOS just because it seems "professional." Many successful embedded products run on super-loops. Choose the right tool for the job.

> **🎓 Advanced Note:** Some systems use a hybrid approach — a super-loop with interrupt-driven critical tasks. This bridges the gap but becomes unmanageable as complexity grows.

---

## 1.2 Hard vs. Soft Real-Time Systems

### 1.2.1 What Does "Real-Time" Actually Mean?

A common misconception: **"Real-time" does not mean "fast."**

Real-time means: **The correctness of the system depends not only on producing the right result, but producing it at the right time.**

Consider two systems:
1. **A web server** that responds in 200 ms — fast, but not real-time. A 500 ms response is still acceptable.
2. **An airbag controller** that must deploy within 15 ms of impact — if it deploys at 20 ms, the passenger may already be injured. The system has **failed**, even though the airbag eventually deployed.

> **🔑 Key Insight:** In real-time systems, a late answer is a **wrong** answer.

### 1.2.2 Hard Real-Time Systems

A **hard real-time system** is one where missing a deadline results in **catastrophic failure**.

**Formal Definition:** A hard real-time system must meet every deadline. If even a single deadline is missed, the system is considered to have failed.

**Examples:**

| System | Deadline | Consequence of Missing |
|---|---|---|
| Airbag controller | 15 ms after impact | Passenger injury/death |
| Anti-lock braking (ABS) | 5 ms per wheel pulse | Loss of vehicle control |
| Pacemaker | Heart cycle timing | Cardiac arrest |
| Nuclear reactor control | Milliseconds | Meltdown |
| Aircraft fly-by-wire | 10–20 ms | Loss of aircraft |

**Characteristics:**
- Deadlines are **absolute** — no negotiation
- The system must be **provably** correct (mathematically verified)
- Worst-case execution time (WCET) must be analyzed
- Safety certification (DO-178C, IEC 61508, ISO 26262) is often required

### 1.2.3 Soft Real-Time Systems

A **soft real-time system** is one where missing a deadline degrades performance but does not cause catastrophe.

**Formal Definition:** A soft real-time system should meet most deadlines. Occasional misses reduce quality but the system remains functional.

**Examples:**

| System | Deadline | Consequence of Missing |
|---|---|---|
| Video streaming | 33 ms per frame (30 fps) | Frame drop, visual glitch |
| Audio playback | ~5 ms buffer | Audio crackle |
| Temperature display | 1 second update | Stale reading shown |
| Smart home sensor | 500 ms | Slightly delayed automation |

**Characteristics:**
- Deadlines are **desired** but not absolute
- Statistical guarantees (e.g., "99% of frames within 33 ms")
- Graceful degradation on overload
- Common in consumer electronics and IoT

### 1.2.4 Firm Real-Time Systems

Some textbooks define a third category: **firm real-time**. Here, a missed deadline means the result is **worthless** (discarded) but the system doesn't fail catastrophically.

**Example:** A stock trading system. If a trade order arrives 1 ms late, the price has changed — the late result is useless, but the system doesn't crash.

### 1.2.5 The Real-Time Spectrum

```
             HARD                FIRM               SOFT
         Real-Time           Real-Time           Real-Time
    ◄─────────────────────────────────────────────────────────►
    │                        │                        │
    │ Missed deadline =      │ Missed deadline =      │ Missed deadline =
    │ SYSTEM FAILURE         │ WASTED RESULT          │ REDUCED QUALITY
    │                        │                        │
    │ Airbag                 │ Stock trading          │ Video streaming
    │ Pacemaker              │ Radar tracking         │ IoT sensors
    │ ABS brakes             │ Packet routing         │ GUI updates
    │                        │                        │
    │ Must be PROVEN         │ Should be designed     │ Best effort with
    │ mathematically         │ for high reliability   │ statistical bounds
```

### 1.2.6 Where Does FreeRTOS Fit?

FreeRTOS is suitable for both **hard** and **soft** real-time systems, but with important caveats:

- FreeRTOS provides a **priority-based preemptive scheduler**, which is the foundation for hard real-time
- It does **not** provide formal verification out of the box
- **Amazon SafeRTOS** (commercial variant) is certified for IEC 61508 and ISO 26262
- For most STM32 applications (industrial, IoT, consumer), FreeRTOS is excellent
- For safety-critical (DO-178C Level A), you need SafeRTOS or a certified kernel

> **🏭 Industry Insight:** Most FreeRTOS deployments are in soft-to-firm real-time systems. When hard real-time guarantees are needed, the system design (priority assignment, WCET analysis) matters more than the kernel itself.

---

## 1.3 Determinism and Latency

### 1.3.1 What is Determinism?

**Determinism** means that the system's behavior is **predictable** and **repeatable**. Given the same inputs and the same state, the system will always respond in the same amount of time.

#### An Analogy: Two Restaurants

**Restaurant A (Non-deterministic):**
- Sometimes your food arrives in 5 minutes
- Sometimes it takes 45 minutes
- Depends on how busy the kitchen is
- You never know how long you'll wait

**Restaurant B (Deterministic):**
- Your food always arrives in exactly 10 minutes
- Regardless of how many other customers there are
- You can plan around this guarantee

An RTOS is Restaurant B. A general-purpose OS (Linux, Windows) is Restaurant A.

### 1.3.2 Why Determinism Matters More Than Speed

Consider two systems controlling a motor:

**System A:** Average response time = 1 ms, but occasionally takes 50 ms
**System B:** Average response time = 5 ms, maximum response time = 6 ms

**System B is better for real-time**, even though it is slower on average. Why?

Because you can **design around** a guaranteed 6 ms response. You cannot design around occasional 50 ms spikes — they will eventually cause failure.

```
System A (Fast but non-deterministic):
Time ──►
Response: 1ms  1ms  1ms  50ms  1ms  1ms  1ms  50ms  1ms
                          ^^^^                ^^^^
                          PROBLEM!            PROBLEM!

System B (Slower but deterministic):
Time ──►
Response: 5ms  5ms  6ms  5ms  5ms  6ms  5ms  5ms  5ms
          All responses within 6ms — PREDICTABLE ✓
```

> **🔑 Key Insight:** In real-time systems, **worst-case** behavior matters more than **average** behavior. An RTOS gives you bounded worst-case timing.

### 1.3.3 Sources of Non-Determinism

Understanding what makes a system non-deterministic helps you avoid these pitfalls:

| Source | Description | Impact |
|---|---|---|
| **Variable-length loops** | Iterating over data with unknown length | Execution time varies |
| **Dynamic memory allocation** | `malloc()` / `free()` | Fragmentation causes variable time |
| **Cache effects** | Cache hits vs. misses | 10x–100x speed variation |
| **Interrupt storms** | Bursts of interrupts | Delay task execution |
| **Priority inversion** | Low-priority task blocks high-priority | Unbounded delay |
| **Shared resources** | Waiting for a mutex held by another task | Unknown wait time |
| **Non-preemptive sections** | Critical sections disable interrupts | Delay all tasks |

### 1.3.4 What is Latency?

**Latency** is the time delay between a stimulus (input) and the response (output).

```
    Stimulus                      Response
    (event)                       (action)
       │                             │
       │◄──────── LATENCY ──────────►│
       │                             │
       ▼                             ▼
   ┌───────┐                    ┌───────── ┐
   │Button │                    │ Motor    │
   │Pressed│                    │ Starts   │
   └───────┘                    └──────────┘
```

#### Types of Latency in an RTOS

```
Total Latency Breakdown:
═══════════════════════════════════════════════════════════

│◄── Interrupt ──►│◄── Scheduling ──►│◄── Task ────────►│
│    Latency       │    Latency        │  Execution Time  │
│                  │                   │                  │

1. INTERRUPT LATENCY:
   Time from hardware event → ISR begins executing
   Typical: 1–12 clock cycles on Cortex-M

2. SCHEDULING LATENCY:
   Time from ISR exit → highest-priority ready task runs
   Depends on: kernel implementation, current critical sections

3. TASK EXECUTION TIME:
   Time for the task to process the event and produce output
   Depends on: algorithm complexity, data size
```

### 1.3.5 Latency on STM32 Cortex-M

The ARM Cortex-M processors are designed for low-latency interrupt handling:

| Metric | Cortex-M0 | Cortex-M3 | Cortex-M4 | Cortex-M7 |
|---|---|---|---|---|
| Interrupt latency | 16 cycles | 12 cycles | 12 cycles | 12 cycles |
| At 72 MHz | 222 ns | 167 ns | 167 ns | — |
| At 168 MHz | — | — | 71 ns | 71 ns |

FreeRTOS adds scheduling overhead on top of this:
- **Context switch time:** ~5–20 μs on STM32F4 at 168 MHz
- **Scheduler decision:** O(1) — constant time regardless of task count

### 1.3.6 Measuring Determinism: A Practical Test

```c
/* Toggle a GPIO pin at the start and end of a task's work.
   Measure jitter with an oscilloscope or logic analyzer. */

void Task_Determinism_Test(void *pvParameters)
{
    TickType_t xLastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET);   /* Pin HIGH */

        /* Simulate work */
        volatile uint32_t i;
        for (i = 0; i < 1000; i++);

        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_RESET); /* Pin LOW */

        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(10));    /* 10 ms period */
    }
}

/*
 * On an oscilloscope, you should see:
 * - Regular 10 ms period between rising edges
 * - Jitter (variation) should be < 1 tick (typically < 1 ms)
 * - Any jitter > 1 tick indicates a problem (higher-priority task
 *   or interrupt consuming too much time)
 */
```

---

## 1.4 Throughput vs. Response Time

### 1.4.1 Definitions

**Throughput:** The total amount of work completed per unit of time.
- Measured in: operations/second, bytes/second, samples/second
- Think: "How much total work gets done?"

**Response Time:** The time from a request to the beginning (or completion) of the response.
- Measured in: milliseconds, microseconds
- Think: "How quickly does the system react?"

### 1.4.2 The Trade-Off

These two metrics often conflict. Optimizing for one can harm the other.

#### Analogy: The Doctor's Office

**High Throughput, Bad Response Time:**
- The doctor sees 60 patients per day (high throughput)
- But your appointment wait time is 3 hours (bad response time)
- The doctor batches similar cases to be efficient

**Good Response Time, Lower Throughput:**
- The doctor takes patients immediately as they arrive
- Wait time is always under 5 minutes (good response time)
- But the doctor only sees 30 patients per day (lower throughput)
- Context-switching between different cases is expensive

### 1.4.3 In Embedded Systems

```
THROUGHPUT-OPTIMIZED DESIGN:
────────────────────────────

Task A runs for 100 ms  ████████████████████
Task B runs for 100 ms                      ████████████████████
Task A runs for 100 ms                                          ████████████████████

Total work done: HIGH (no context switch overhead)
Response to event for B: UP TO 100 ms DELAY


RESPONSE-TIME-OPTIMIZED DESIGN:
────────────────────────────────

Task A ██   ██   ██   ██   ██   ██   ██   ██   ██   ██
Task B   ██   ██   ██   ██   ██   ██   ██   ██   ██
         ↑↑   ↑↑   ↑↑  ... context switches

Total work done: LOWER (context switch overhead)
Response to event for B: < 20 ms ✓
```

### 1.4.4 RTOS Prioritizes Response Time

An RTOS, by default, optimizes for **response time** over throughput. This is the correct choice for real-time systems because:

1. **Missing a deadline by 1 ms is worse than wasting 10% of CPU on overhead**
2. The scheduler preempts lower-priority tasks to run higher-priority ones immediately
3. Context switches add overhead but guarantee responsiveness

**Design principle:** In real-time systems, predictable response time is more important than maximum throughput.

### 1.4.5 Practical Example: ADC Sampling vs. Data Processing

```c
/*
 * WRONG: Throughput-optimized (misses real-time requirements)
 * Collects 1000 samples, then processes all at once.
 */
void Task_Wrong(void *pvParameters)
{
    uint16_t buffer[1000];
    for (;;)
    {
        /* Collect 1000 samples - takes a long time */
        for (int i = 0; i < 1000; i++)
        {
            buffer[i] = Read_ADC();
        }
        /* Process all at once - blocks other tasks */
        Process_Buffer(buffer, 1000);
    }
}

/*
 * RIGHT: Response-time-optimized with RTOS
 * Separate tasks for collection and processing.
 */
void Task_Collect(void *pvParameters)      /* High priority */
{
    uint16_t sample;
    for (;;)
    {
        sample = Read_ADC();
        xQueueSend(adcQueue, &sample, portMAX_DELAY);
        vTaskDelay(pdMS_TO_TICKS(1));      /* 1 kHz sampling */
    }
}

void Task_Process(void *pvParameters)      /* Lower priority */
{
    uint16_t sample;
    for (;;)
    {
        xQueueReceive(adcQueue, &sample, portMAX_DELAY);
        Process_Sample(sample);
    }
}
```

> **🎯 Interview Insight:** "What's the difference between throughput and response time?" is a classic interview question. The answer: throughput is total work per second; response time is delay from stimulus to reaction. RTOS prioritizes response time. They are often inversely related due to context-switch overhead.

---

## 1.5 Context Switching

### 1.5.1 What is a Context Switch?

A **context switch** is the mechanism by which the RTOS saves the state of the currently running task and restores the state of the next task to run.

Think of it like a bookmark. When you're reading three books simultaneously:
1. You read Book A, place a bookmark at page 47
2. You pick up Book B, open to your bookmark at page 112
3. Later, you bookmark Book B at page 115, pick up Book A at page 47

The "bookmark" is the **context** — everything needed to resume exactly where you left off.

### 1.5.2 What Gets Saved (The Context)

On an ARM Cortex-M processor, the context includes:

```
┌──────────────────────────────────────────┐
│           TASK CONTEXT                    │
│                                           │
│  General-Purpose Registers:               │
│    R0–R3   (function arguments/return)    │
│    R4–R11  (callee-saved registers)       │
│    R12     (intra-procedure scratch)      │
│                                           │
│  Special Registers:                       │
│    R13 (SP)  — Stack Pointer              │
│    R14 (LR)  — Link Register (return addr)│
│    R15 (PC)  — Program Counter            │
│    xPSR      — Program Status Register    │
│                                           │
│  FPU Registers (if Cortex-M4F/M7):       │
│    S0–S31    — Floating-point registers   │
│    FPSCR     — FP status/control          │
│                                           │
│  Stack Contents:                          │
│    Local variables                        │
│    Function call chain                    │
│    Interrupt frames                       │
└──────────────────────────────────────────┘
```

### 1.5.3 How a Context Switch Works (Step by Step)

```
TIME ──────────────────────────────────────────────────────────►

TASK A (Running)         KERNEL              TASK B (Ready)
═══════════════         ════════             ═══════════════

  Executing code...
        │
        │  ◄── Tick interrupt (SysTick) fires
        │
        ▼
  ┌─────────────┐
  │ Save R0-R12 │ ◄── Hardware auto-saves R0-R3, R12, LR, PC, xPSR
  │  SP, LR, PC │     Software saves R4-R11 (FreeRTOS kernel code)
  │ onto Task A │
  │   stack     │
  └──────┬──────┘
         │
         ▼
  ┌──────────────┐
  │ Store Task A │
  │  SP into     │
  │  Task A TCB  │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  SCHEDULER:  │
  │  Which task  │
  │  runs next?  │──► Picks Task B (highest priority ready)
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ Load Task B  │
  │  SP from     │
  │  Task B TCB  │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐                      ┌──────────────┐
  │ Restore      │                      │              │
  │ R4-R11 from  │ ────────────────►    │ Task B       │
  │ Task B stack │                      │ Resumes      │
  │              │                      │ Executing    │
  └──────────────┘                      └──────────────┘
```

### 1.5.4 Context Switch on Cortex-M: The PendSV Handler

On ARM Cortex-M, FreeRTOS uses the **PendSV** (Pendable Service Call) exception for context switching. It is set to the **lowest priority** so it only runs when no other interrupt is active.

```c
/*
 * Simplified PendSV handler (what FreeRTOS does internally).
 * This is ARM assembly — you don't write this, but understanding it helps.
 */
PendSV_Handler:
    /* 1. Get current task's stack pointer */
    MRS     R0, PSP                 /* R0 = Process Stack Pointer */

    /* 2. Save remaining registers (R4-R11) onto task's stack */
    STMDB   R0!, {R4-R11}          /* Push R4-R11, decrement R0 */

    /* 3. Save updated SP into current task's TCB */
    LDR     R1, =pxCurrentTCB
    LDR     R1, [R1]
    STR     R0, [R1]               /* TCB->pxTopOfStack = R0 */

    /* 4. Call scheduler to select next task */
    BL      vTaskSwitchContext

    /* 5. Load new task's SP from its TCB */
    LDR     R1, =pxCurrentTCB
    LDR     R1, [R1]
    LDR     R0, [R1]               /* R0 = new task's stack pointer */

    /* 6. Restore R4-R11 from new task's stack */
    LDMIA   R0!, {R4-R11}          /* Pop R4-R11, increment R0 */

    /* 7. Set PSP to new task's stack */
    MSR     PSP, R0

    /* 8. Return — hardware will auto-restore R0-R3, R12, LR, PC, xPSR */
    BX      LR
```

### 1.5.5 Context Switch Timing

Context switch time depends on:
- **Processor speed:** Faster clock = faster switch
- **Number of registers:** FPU registers (Cortex-M4F) add overhead
- **Kernel complexity:** FreeRTOS is very lean

**Typical context switch times on STM32:**

| Processor | Clock | Context Switch Time |
|---|---|---|
| STM32F103 (Cortex-M3) | 72 MHz | ~10–15 μs |
| STM32F407 (Cortex-M4) | 168 MHz | ~5–8 μs |
| STM32H743 (Cortex-M7) | 480 MHz | ~2–4 μs |

> **📝 Beginner Note:** You don't need to write context switch code — FreeRTOS handles it. But understanding the mechanism helps you debug stack overflows, hard faults, and timing issues.

### 1.5.6 Context Switch Overhead: Why It Matters

Each context switch costs:
1. **Time:** CPU cycles saving/restoring registers (~5–20 μs)
2. **Cache pollution:** Cortex-M7 has caches; switching tasks may cause cache misses
3. **Pipeline flush:** The processor's instruction pipeline is flushed

**Rule of thumb:** If your system context-switches more than 10,000 times per second, you may be wasting significant CPU time. But for most applications, context switch overhead is negligible (< 1% of CPU time).

### 1.5.7 What Triggers a Context Switch?

| Trigger | Description | Example |
|---|---|---|
| **Tick interrupt** | Periodic timer (SysTick) | Every 1 ms (configTICK_RATE_HZ = 1000) |
| **Yield** | Task voluntarily gives up CPU | `taskYIELD()` |
| **Blocking API** | Task waits for resource | `xQueueReceive()`, `vTaskDelay()` |
| **Higher-priority task ready** | Preemption | ISR unblocks a high-priority task |
| **Task suspend/delete** | Current task stops | `vTaskSuspend()`, `vTaskDelete()` |

---

## 1.6 Common Mistakes (Chapter 1)

### Mistake 1: "Real-time means fast"
**Wrong:** Real-time means **predictable**. A 100 ms response is fine if it's **always** 100 ms.

### Mistake 2: "An RTOS guarantees perfect timing"
**Wrong:** The RTOS provides mechanisms for timing control. **You** must assign correct priorities, analyze worst-case execution times, and avoid priority inversion. The kernel cannot fix a bad design.

### Mistake 3: "Context switching is wasteful; super-loop is always more efficient"
**Nuanced:** Context switching has cost, but it **buys you responsiveness**. A 5 μs context switch that prevents a 200 ms response delay is an excellent trade-off.

### Mistake 4: "All my tasks should have different priorities"
**Not necessarily:** Tasks with similar importance can share the same priority and use round-robin scheduling. Over-differentiating priorities can complicate analysis.

### Mistake 5: "I need an RTOS for every embedded project"
**Wrong:** Many successful products use bare-metal. An RTOS adds complexity. Use it when the problem demands it — multiple tasks with different deadlines.

---

## 1.7 Debugging Tips

1. **Use a logic analyzer or oscilloscope** to measure actual task timing by toggling GPIO pins at task entry/exit
2. **Measure context switch time** by toggling a pin in the PendSV handler (before save and after restore)
3. **Check SysTick configuration** — a misconfigured tick rate will cause all timing to be wrong
4. **Verify clock configuration** — if the system clock isn't what you expect, all timing calculations are off
5. **Use FreeRTOS trace hooks** — `traceTASK_SWITCHED_IN()` and `traceTASK_SWITCHED_OUT()` macros let you instrument context switches without modifying kernel code

---

## 1.8 Hands-On Exercises

### Exercise 1.1: Super-Loop Timing Analysis

**Objective:** Demonstrate the timing problems of a super-loop.

**Setup:** STM32 board with 3 LEDs

```c
/*
 * Exercise: Run this bare-metal program and measure how often
 * each LED toggles. Then answer the questions below.
 */
int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();

    while (1)
    {
        /* Task A: Toggle LED1 every 100 ms */
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
        HAL_Delay(100);

        /* Task B: Toggle LED2 - simulates heavy processing */
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_6);
        HAL_Delay(500);   /* 500 ms processing */

        /* Task C: Toggle LED3 every 100 ms */
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_7);
        HAL_Delay(100);
    }
}
```

**Questions:**
1. What is the actual toggle period of LED1? (Answer: 700 ms, not 100 ms)
2. What is the actual toggle period of LED3? (Answer: 700 ms, not 100 ms)
3. If Task B's processing time increases to 1000 ms, what happens to LED1 and LED3?
4. How would you fix this in bare-metal? (Hint: Use timer interrupts and flags)
5. How would an RTOS solve this problem?

### Exercise 1.2: Identify Real-Time Requirements

**Objective:** Practice classifying systems as hard, soft, or firm real-time.

Classify each system and justify your answer:

1. A home thermostat that reads temperature every 5 seconds
2. An insulin pump that delivers precise doses
3. A music player on a Bluetooth speaker
4. A robotic arm in a factory assembly line
5. A smart doorbell that sends notifications
6. An engine control unit (ECU) managing fuel injection timing
7. A weather station logging data every minute
8. A CNC machine controlling cutting tool position

<details>
<summary><strong>Solutions</strong></summary>

1. **Soft real-time** — A few seconds' delay in reading temperature won't cause harm
2. **Hard real-time** — Wrong timing could cause diabetic shock or death
3. **Soft real-time** — A dropped audio sample causes a glitch, not a disaster
4. **Hard real-time** — A late movement command could cause collision or defective product
5. **Soft real-time** — A few seconds' delay in notification is acceptable
6. **Hard real-time** — Incorrect fuel injection timing damages the engine or causes failure
7. **Soft real-time** — A missed reading just means a gap in the log
8. **Hard real-time** — Incorrect positioning damages the workpiece or the machine

</details>

### Exercise 1.3: Context Switch Calculation

**Objective:** Calculate context switch overhead.

Given:
- STM32F407 running at 168 MHz
- Context switch takes 8 μs
- Tick rate = 1000 Hz (1 ms tick period)
- 5 tasks, each running for about 50 μs per tick

Questions:
1. In the worst case, how many context switches occur per tick? (Answer: up to 5)
2. What percentage of CPU time is spent context switching per tick? (Answer: 5 × 8 μs = 40 μs out of 1000 μs = 4%)
3. If you increase to 20 tasks, what is the context switch overhead? (Answer: 20 × 8 μs = 160 μs = 16%)
4. At what number of tasks does context switching consume > 50% of CPU? (Answer: 1000 / (2 × 8) = 62 tasks)
5. Is this a realistic concern for most embedded systems? (Answer: No — most systems have 5–15 tasks)

---

## 1.9 Mini-Project: Bare-Metal vs. RTOS Comparison

### Requirements

Build a system with 3 concurrent activities:
1. **Blink LED1** every 100 ms
2. **Read a button** and toggle LED2 on press (must respond within 50 ms)
3. **Send "Hello\r\n" over UART** every 1 second

### Part A: Bare-Metal Implementation

```c
/*
 * Bare-metal implementation using super-loop + hardware timer.
 * Note the complexity and fragility of manual timing management.
 */

volatile uint32_t tick_counter = 0;

void SysTick_Handler(void)
{
    tick_counter++;
    HAL_IncTick();
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_USART2_UART_Init();

    uint32_t last_led_toggle = 0;
    uint32_t last_uart_send  = 0;
    uint8_t  last_button_state = GPIO_PIN_SET;

    while (1)
    {
        uint32_t now = tick_counter;

        /* Task 1: Blink LED1 every 100 ms */
        if ((now - last_led_toggle) >= 100)
        {
            HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
            last_led_toggle = now;
        }

        /* Task 2: Check button (poll) */
        uint8_t btn = HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13);
        if (btn == GPIO_PIN_RESET && last_button_state == GPIO_PIN_SET)
        {
            HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_6);
        }
        last_button_state = btn;

        /* Task 3: Send UART every 1000 ms */
        if ((now - last_uart_send) >= 1000)
        {
            HAL_UART_Transmit(&huart2, (uint8_t *)"Hello\r\n", 7, 100);
            last_uart_send = now;
        }
    }
}

/*
 * PROBLEMS:
 * 1. If HAL_UART_Transmit blocks (it does!), LED blink and button
 *    response are delayed.
 * 2. No priority — a slow UART transmission delays everything.
 * 3. Adding more tasks makes the code increasingly complex.
 * 4. Button response depends on loop speed — if UART blocks for
 *    100 ms, button press is missed.
 */
```

### Part B: RTOS Implementation

```c
/*
 * RTOS implementation using FreeRTOS on STM32.
 * Note how each task is independent and has its own priority.
 */

/* Task 1: Blink LED1 — Medium priority */
void Task_BlinkLED(void *pvParameters)
{
    for (;;)
    {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

/* Task 2: Button handler — Highest priority (must respond fast) */
void Task_Button(void *pvParameters)
{
    uint8_t last_state = GPIO_PIN_SET;

    for (;;)
    {
        uint8_t current = HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13);
        if (current == GPIO_PIN_RESET && last_state == GPIO_PIN_SET)
        {
            HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_6);
        }
        last_state = current;
        vTaskDelay(pdMS_TO_TICKS(10));   /* Poll every 10 ms → < 50 ms response ✓ */
    }
}

/* Task 3: UART sender — Lowest priority */
void Task_UART(void *pvParameters)
{
    for (;;)
    {
        HAL_UART_Transmit(&huart2, (uint8_t *)"Hello\r\n", 7, 100);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_USART2_UART_Init();

    xTaskCreate(Task_BlinkLED, "LED",    128, NULL, 2, NULL);
    xTaskCreate(Task_Button,   "Button", 128, NULL, 3, NULL);  /* Highest */
    xTaskCreate(Task_UART,     "UART",   128, NULL, 1, NULL);  /* Lowest  */

    vTaskStartScheduler();

    for (;;);   /* Should never reach here */
}

/*
 * ADVANTAGES:
 * 1. Button task has highest priority — always responds within 10 ms
 * 2. UART transmission does NOT block LED blinking
 * 3. Adding a new task = adding a new function + xTaskCreate call
 * 4. Each task's timing is independent of others
 * 5. Code is modular and maintainable
 */
```

### Comparison Table

| Aspect | Bare-Metal | RTOS |
|---|---|---|
| LED timing accuracy | Depends on loop speed | Independent, accurate |
| Button response | Can be delayed by UART | Always < 10 ms |
| UART timing | Can be delayed by all tasks | Independent |
| Adding tasks | Increases complexity exponentially | Linear complexity |
| RAM usage | ~100 bytes | ~2 KB (kernel + stacks) |
| Code readability | Decreasing with complexity | Each task is self-contained |

---

## 1.10 Chapter Summary

| Concept | Key Takeaway |
|---|---|
| **Bare-metal** | Simple but cannot handle multiple deadlines |
| **RTOS** | Manages multiple tasks with different timing requirements |
| **Hard real-time** | Missing a deadline = catastrophic failure |
| **Soft real-time** | Missing a deadline = reduced quality |
| **Determinism** | Worst-case timing must be bounded and predictable |
| **Latency** | Time from event to response; must be minimized and bounded |
| **Throughput** | Total work per second; RTOS trades some throughput for responsiveness |
| **Response time** | RTOS optimizes for this over throughput |
| **Context switch** | Saves/restores task state; costs ~5–20 μs on STM32 |

---

## 1.11 What's Next

In **Chapter 2: RTOS Architecture**, we will look inside the RTOS kernel and understand:
- How the kernel is structured
- How the scheduler decides which task to run
- How the tick timer drives the entire system
- The difference between tick-based and tickless operation

---

*End of Chapter 1*
