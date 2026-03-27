# RTOS Interview Preparation Guide

## Complete Question Bank with Expert Answers

---

## Section A: Core Concepts (Beginner → Intermediate)

### Q1: What is an RTOS? How does it differ from a general-purpose OS?

**Answer:**

An RTOS (Real-Time Operating System) is an OS that guarantees **deterministic** response to events within defined time constraints.

| Aspect | RTOS | General-Purpose OS |
|--------|------|--------------------|
| Primary goal | **Determinism** (predictable timing) | **Throughput** (max work/time) |
| Scheduling | Priority-based preemptive | Fair/time-sharing |
| Latency | Microseconds (bounded) | Milliseconds (unbounded) |
| Memory | Static/fixed, no virtual memory | Virtual memory, paging |
| Footprint | 5-50 KB | Hundreds of MB |
| Context switch | 1-10 μs | 10-100 μs |
| Examples | FreeRTOS, Zephyr, VxWorks | Linux, Windows |

**Key point**: An RTOS does NOT mean "fast" — it means **predictable**. A slow RTOS that always meets deadlines is better than a fast system that occasionally misses them.

---

### Q2: Explain Hard Real-Time vs Soft Real-Time. Give examples.

**Answer:**

| Type | Deadline Miss Consequence | Example |
|------|--------------------------|---------|
| **Hard** | System failure / catastrophic | Airbag deployment, ABS braking, pacemaker |
| **Firm** | Result becomes worthless (but no damage) | Video frame rendering, factory robotics |
| **Soft** | Degraded quality (acceptable) | Audio streaming, UI responsiveness |

**Interview tip**: "The airbag has exactly 15ms to detect impact and deploy. Missing this deadline means the passenger is injured — that's hard real-time."

---

### Q3: What is a context switch? What happens during one?

**Answer:**

A context switch saves the state of the current task and restores the state of the next task.

**Steps on ARM Cortex-M:**

1. **Hardware** (automatic on exception entry):
   - Pushes R0-R3, R12, LR, PC, xPSR to the current task's stack
   
2. **Software** (RTOS PendSV handler):
   - Pushes R4-R11 to current task's stack
   - Saves current PSP to current TCB
   - Calls scheduler to select next task
   - Loads next task's PSP from its TCB
   - Pops R4-R11 from new task's stack

3. **Hardware** (automatic on exception return):
   - Pops R0-R3, R12, LR, PC, xPSR from new task's stack

**Cost**: ~12 μs on STM32F4 at 168 MHz (with FreeRTOS).

---

### Q4: Describe the FreeRTOS task states and transitions.

**Answer:**

```
                    vTaskSuspend()
  ┌──────────┐  ←─────────────────  ┌──────────────┐
  │ SUSPENDED │                      │   RUNNING    │
  │           │  ─────────────────► │ (executing)  │
  └──────────┘   vTaskResume()      └──────┬───────┘
                                           │
                            Preempted      │  Blocks on
                            (higher prio)  │  queue/sem/delay
                                    ▲      │
                                    │      ▼
                              ┌──────────────┐
                              │    READY     │
                              │  (waiting    │◄── Unblocked
                              │   to run)    │    (event/timeout)
                              └──────────────┘
                                    ▲
                                    │
                              ┌──────────────┐
                              │   BLOCKED    │
                              │ (waiting for │
                              │  event/time) │
                              └──────────────┘
```

---

### Q5: What is the difference between `vTaskDelay()` and `vTaskDelayUntil()`?

**Answer:**

- **`vTaskDelay(100)`**: Delays for 100 ticks **from NOW**. If the task was preempted for 20 ticks before calling delay, the total period is 120 ticks. Causes **drift**.

- **`vTaskDelayUntil(&lastWake, 100)`**: Delays until 100 ticks **from last wake time**. Compensates for execution time and preemption. **No drift**.

**Rule**: Use `vTaskDelayUntil()` for periodic tasks. Use `vTaskDelay()` for non-periodic waits.

---

## Section B: Synchronization & IPC (Intermediate)

### Q6: Semaphore vs Mutex — What's the difference?

**This is the #1 most asked RTOS interview question.**

**Answer:**

| Feature | Binary Semaphore | Mutex |
|---------|-----------------|-------|
| **Purpose** | Signaling (event notification) | Mutual exclusion (resource protection) |
| **Ownership** | No owner — anyone can give | Owner only can release |
| **Priority Inheritance** | ❌ No | ✅ Yes |
| **Give from ISR** | ✅ Yes | ❌ No |
| **Typical use** | ISR → Task notification | Protecting shared resource |
| **Can cause priority inversion** | Yes (no built-in fix) | No (inheritance prevents it) |

**Key insight**: A semaphore is like a **flag** (raise/lower by anyone). A mutex is like a **key** (only the holder can unlock).

```c
/* WRONG: Using semaphore to protect shared resource */
xSemaphoreTake(xSemaphore, portMAX_DELAY);
shared_variable++;  /* No priority inheritance! */
xSemaphoreGive(xSemaphore);

/* RIGHT: Using mutex to protect shared resource */
xSemaphoreTake(xMutex, portMAX_DELAY);
shared_variable++;  /* Priority inheritance protects us */
xSemaphoreGive(xMutex);
```

---

### Q7: What is Priority Inversion? How do you solve it?

**Answer:**

Priority inversion occurs when a high-priority task is blocked by a low-priority task holding a shared resource, while a medium-priority task preempts the low-priority task — making the high-priority task wait for the medium-priority task.

```
PRIORITY INVERSION:

  High  ──────┐ BLOCKED (waiting for mutex)       ┌── Runs
              │                                    │
  Med   ──────┼──────── RUNS (preempts Low) ──────┤
              │                                    │
  Low   ─ Has ┘ mutex ──── preempted ──────────── releases mutex
                                                    ↑
                                          UNBOUNDED DELAY!
```

**Solutions:**

1. **Priority Inheritance** (used by FreeRTOS mutexes):
   - Temporarily raise Low's priority to High's level while it holds the mutex
   - Low can't be preempted by Medium

2. **Priority Ceiling Protocol**:
   - Set mutex priority to highest possible user — any task taking the mutex immediately runs at that ceiling
   - Prevents all blocking chains

3. **Disable interrupts** (drastic, for very short critical sections only)

**Real-world example**: Mars Pathfinder (1997) — priority inversion caused system resets. Fixed by enabling priority inheritance.

---

### Q8: What is a Deadlock? How do you prevent it?

**Answer:**

Deadlock: Two or more tasks wait for resources held by each other, creating a circular wait. No task can proceed.

```
DEADLOCK:
  Task A: takes Mutex1 → tries to take Mutex2 (BLOCKED)
  Task B: takes Mutex2 → tries to take Mutex1 (BLOCKED)
  → Neither can proceed!
```

**Prevention strategies:**

1. **Lock ordering**: Always acquire mutexes in the same global order
   ```c
   /* ALWAYS take Mutex1 before Mutex2 */
   xSemaphoreTake(xMutex1, portMAX_DELAY);
   xSemaphoreTake(xMutex2, portMAX_DELAY);
   ```

2. **Timeout**: Use timed waits instead of infinite blocking
   ```c
   if (xSemaphoreTake(xMutex, pdMS_TO_TICKS(100)) == pdFAIL)
   {
       /* Release any held mutexes and retry */
   }
   ```

3. **Single mutex**: Combine resources under one mutex

4. **Resource hierarchy**: Assign numeric levels; only lock in ascending order

---

### Q9: Queue vs Task Notification — When to use each?

**Answer:**

| Feature | Queue | Task Notification |
|---------|-------|-------------------|
| Data transfer | Yes (any type, any size) | Limited (32-bit value) |
| Multiple senders | Yes | Yes |
| Multiple receivers | Yes | **No** (single task only) |
| RAM usage | ~76 + N × item_size bytes | **0 extra** (built into TCB) |
| Speed | Good | **45% faster** than queue |
| Buffering | Yes (N items) | **No** (one pending value) |

**Use Queue when**: Multiple consumers, need buffering, sending structs/large data.

**Use Notification when**: Single consumer, simple event/counter, lightweight signaling from ISR.

---

## Section C: Advanced & System Design (Advanced → Expert)

### Q10: How does FreeRTOS handle interrupts on ARM Cortex-M?

**Answer:**

FreeRTOS uses **two interrupt priority thresholds**:

```
INTERRUPT PRIORITY MODEL (Cortex-M, lower number = higher priority):

  Priority 0  ──► HardFault (always highest)
  Priority 1  ──► Reserved / critical
  ...
  Priority 4  ──► CAN, etc. (ABOVE FreeRTOS — can't use FreeRTOS API!)
  ─ ─ ─ ─ ─ ─ configMAX_SYSCALL_INTERRUPT_PRIORITY ─ ─ ─ ─ ─
  Priority 5  ──► UART ISR (CAN use FromISR APIs)
  Priority 6  ──► SPI ISR  (CAN use FromISR APIs)
  ...
  Priority 15 ──► Lowest
  Priority 255 ──► PendSV, SysTick (set by FreeRTOS to lowest)
```

**Rules:**
1. ISRs with priority **numerically lower** than `configMAX_SYSCALL_INTERRUPT_PRIORITY` must NOT call any FreeRTOS API
2. ISRs within the FreeRTOS range must use `...FromISR()` variants only
3. PendSV (context switch) is always at lowest priority — deferred until all ISRs complete

---

### Q11: Explain FreeRTOS memory management schemes.

**Answer:**

| Scheme | Allocation | Deallocation | Fragmentation | Best For |
|--------|-----------|--------------|---------------|----------|
| **heap_1** | ✅ | ❌ | None | Create once, never delete |
| **heap_2** | ✅ | ✅ | Yes (no coalescing) | Fixed-size alloc/free |
| **heap_3** | wraps `malloc()` | wraps `free()` | Depends on libc | If you trust libc |
| **heap_4** | ✅ | ✅ | **Minimal** (coalesces) | **Most applications** |
| **heap_5** | ✅ | ✅ | Minimal | Discontiguous RAM regions |

**Recommended**: heap_4 for most STM32 projects. heap_1 for safety-critical (deterministic, no fragmentation possible).

---

### Q12: Design a Real-Time Data Acquisition System

**System design interview question. Walk through your thinking process.**

**Requirements**: 4 analog channels at 1 kHz, log to SD card, UART interface, watchdog.

**Step 1: Task Decomposition**

```
ADC Task (ISR + DMA)     → Priority 4, Timer-triggered
Processing Task          → Priority 3, Queue-driven
SD Logger Task           → Priority 2, Batched writes
UART CLI Task            → Priority 2, Event-driven
Watchdog Task            → Priority 5, Periodic
```

**Step 2: IPC Design**

```
ADC DMA ──(notification)──► Processing Task
Processing ──(queue, 128 entries)──► SD Logger
Processing ──(queue, 16 entries)──► UART Display
All tasks ──(event group)──► Watchdog
```

**Step 3: Timing Analysis**

```
ADC: 4 channels × 1kHz = 250μs per sample set
Processing: ~50μs per sample set (filtering)
SD write: ~5ms per 512-byte block (batch 128 samples)
Total CPU: (250+50)/1000 = 30% → feasible
```

**Step 4: Memory Budget**

```
ADC DMA buffer:    4 × 2 × 128 = 1024 bytes (double-buffer)
Processing queue:  128 × 16    = 2048 bytes
SD write buffer:   512 bytes
Task stacks:       5 × 256 × 4 = 5120 bytes
Total:             ~8.7 KB (fits in 20KB+ SRAM)
```

---

### Q13: What is Rate Monotonic Scheduling (RMS)?

**Answer:**

RMS is an optimal **static priority** scheduling algorithm: **shorter period → higher priority**.

**Schedulability test** (Liu & Layland bound):

$$U = \sum_{i=1}^{n} \frac{C_i}{T_i} \leq n(2^{1/n} - 1)$$

Where $C_i$ = execution time, $T_i$ = period.

| Tasks (n) | Utilization Bound |
|-----------|------------------|
| 1 | 100% |
| 2 | 82.8% |
| 3 | 78.0% |
| ∞ | 69.3% |

**Example**: 3 tasks with periods 10ms, 20ms, 50ms and execution times 2ms, 3ms, 5ms:

$$U = \frac{2}{10} + \frac{3}{20} + \frac{5}{50} = 0.2 + 0.15 + 0.1 = 0.45$$

0.45 < 0.78 → **Schedulable** under RMS.

---

### Q14: How would you debug a system that randomly crashes?

**Systematic debugging approach (great interview answer):**

1. **Enable HardFault handler** with register capture (PC, LR, CFSR)
2. **Check stack overflow**: Enable `configCHECK_FOR_STACK_OVERFLOW = 2`
3. **Check heap exhaustion**: Monitor `xPortGetMinimumEverFreeHeapSize()`
4. **Check priority inversion**: Review all mutex usage and task priorities
5. **Check ISR priority**: Ensure no FreeRTOS API calls from priorities above `configMAX_SYSCALL_INTERRUPT_PRIORITY`
6. **Check uninitialized variables**: Use static analysis tools
7. **Check race conditions**: Review all shared data access — is it mutex-protected?
8. **Use `configASSERT()`**: Enable FreeRTOS assertions to catch API misuse
9. **Add watchdog monitoring**: Detect which task stops responding
10. **Use trace tools**: Tracealyzer or Segger SystemView to visualize task execution

---

### Q15: What happens when you call `xQueueSendFromISR()` in an ISR?

**Answer (show deep understanding):**

```c
void USART2_IRQHandler(void)
{
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    uint8_t data = USART2->DR;

    xQueueSendFromISR(xQueue, &data, &xHigherPriorityTaskWoken);

    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

**What happens internally:**

1. `xQueueSendFromISR()` copies `data` into the queue's storage
2. If a task is blocked waiting on this queue, it's moved to Ready list
3. If that unblocked task has **higher priority** than the currently interrupted task, `xHigherPriorityTaskWoken` is set to `pdTRUE`
4. `portYIELD_FROM_ISR(pdTRUE)` sets the PendSV pending bit
5. When ISR returns, PendSV fires (lowest priority), performing the context switch
6. The high-priority task runs **immediately** after the ISR — not after the interrupted task resumes

**Critical**: Without `portYIELD_FROM_ISR`, the unblocked task would wait until the next scheduler tick — adding up to 1ms latency.

---

## Section D: Quick-Fire Questions

| Question | One-Line Answer |
|----------|----------------|
| Max tasks in FreeRTOS? | Limited only by available RAM |
| What does the Idle task do? | Runs cleanup (deleted tasks), optionally enters low-power mode |
| Can a mutex be given from ISR? | **No** — only semaphores have `GiveFromISR` |
| `configTICK_RATE_HZ` default? | 1000 (1ms tick) |
| What is TCB? | Task Control Block — stores task state, priority, stack pointer |
| `configMINIMAL_STACK_SIZE`? | Minimum stack for Idle task (usually 128 words on Cortex-M) |
| Difference: preemptive vs cooperative? | Preemptive: scheduler can interrupt running task. Cooperative: task must yield |
| What is PendSV? | Pendable Service Call — ARM exception used for context switching |
| What does `taskYIELD()` do? | Forces immediate reschedule (triggers PendSV) |
| Counting semaphore use case? | Event counting or resource pool management |
| What is stack canary? | Known pattern at stack bottom to detect overflow |
| `portMAX_DELAY`? | Maximum possible block time (~49 days at 1kHz tick) |

---

## Section E: System Design Questions

### Design Exercise 1: "Design an elevator control system"

**Expected answer framework:**

1. **Tasks**: DoorControl, MotorControl, ButtonInput, DisplayUpdate, SafetyMonitor
2. **Priorities**: SafetyMonitor (highest) > MotorControl > DoorControl > ButtonInput > Display
3. **IPC**: Command queue (Button → Controller), Event groups (floor sensors), Mutex (motor state)
4. **Safety**: Watchdog on all tasks, door sensor interlock, emergency stop interrupt

### Design Exercise 2: "Design a medical infusion pump"

**Expected answer framework:**

1. **Tasks**: FlowControl (hard RT), PressureSensor, BubbleDetector, AlarmManager, UI
2. **Safety**: Triple redundancy on flow rate, watchdog, IWDG + WWDG, static allocation only (heap_1)
3. **Certification**: IEC 62304 compliance, MISRA C, 100% MC/DC coverage
4. **Critical**: FlowControl must never miss deadline (patient safety)

---

## Interview Tips

1. **Always mention trade-offs** — there's no perfect solution
2. **Start with requirements** before jumping to design
3. **Draw diagrams** — interviewers love ASCII art task diagrams
4. **Know the Mars Pathfinder story** — classic priority inversion example
5. **Quantify everything** — "STM32 context switch is ~12μs", not "it's fast"
6. **Mention testing** — how would you verify your design works?
7. **Think about failure modes** — what happens if a task crashes?

---

*End of Interview Preparation Guide*
