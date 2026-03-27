# Chapter 5: Tasks in FreeRTOS — Deep Dive

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Use `xTaskCreate()` and all its variants with confidence
2. Design correct priority schemes that avoid starvation
3. Understand the Idle task's role and how to use the idle hook
4. Implement task notifications for lightweight signaling
5. Apply task design best practices for production systems

---

## 5.1 xTaskCreate — Complete Reference

### 5.1.1 API Review

```c
BaseType_t xTaskCreate(
    TaskFunction_t        pvTaskCode,       /* Function to execute */
    const char * const    pcName,           /* Debug name string */
    configSTACK_DEPTH_TYPE usStackDepth,    /* Stack size (WORDS) */
    void * const          pvParameters,     /* Parameter for task */
    UBaseType_t           uxPriority,       /* Task priority */
    TaskHandle_t * const  pxCreatedTask     /* Output: task handle */
);
```

### 5.1.2 What Happens Inside xTaskCreate

When you call `xTaskCreate()`, FreeRTOS performs these steps:

```
xTaskCreate() Internal Flow:
════════════════════════════

1. ALLOCATE MEMORY
   ├── Allocate TCB from heap        (~100 bytes)
   └── Allocate stack from heap      (usStackDepth × 4 bytes)

2. INITIALIZE TCB
   ├── Set task name (pcTaskName)
   ├── Set priority (uxPriority)
   ├── Initialize list items
   └── Set base priority (for mutex inheritance)

3. INITIALIZE STACK
   ├── Fill with 0xA5A5A5A5 pattern (if configCHECK_FOR_STACK_OVERFLOW == 2)
   ├── Set up initial stack frame:
   │   ├── xPSR = 0x01000000        (Thumb bit set)
   │   ├── PC   = pvTaskCode         (entry point)
   │   ├── LR   = prvTaskExitError   (catch if task returns)
   │   ├── R12  = 0
   │   ├── R3   = 0
   │   ├── R2   = 0
   │   ├── R1   = 0
   │   ├── R0   = pvParameters       (parameter passed to task)
   │   └── R4-R11 = 0               (saved by software)
   └── Set pxTopOfStack

4. ADD TO SCHEDULER
   ├── Add task to the ready list at its priority level
   ├── Increment uxCurrentNumberOfTasks
   └── If task priority > current running task:
       └── Request context switch (portYIELD_WITHIN_API)

5. RETURN pdPASS (or pdFAIL if allocation failed)
```

### 5.1.3 The Initial Stack Frame

This is how FreeRTOS "tricks" the CPU into running the task for the first time:

```
Stack after xTaskCreate (Cortex-M4):
═════════════════════════════════════

High Address
┌──────────────────────────┐
│ xPSR    = 0x01000000    │  ◄── Thumb bit must be set
│ PC      = Task_Function │  ◄── Task entry point
│ LR      = ExitError     │  ◄── Catches accidental returns
│ R12     = 0             │
│ R3      = 0             │  ◄── These 8 registers are
│ R2      = 0             │      restored by hardware on
│ R1      = 0             │      exception return
│ R0      = pvParameters  │  ◄── Task's parameter!
├──────────────────────────┤
│ R11     = 0             │
│ R10     = 0             │  ◄── These 8 registers are
│ R9      = 0             │      restored by PendSV handler
│ R8      = 0             │
│ R7      = 0             │
│ R6      = 0             │
│ R5      = 0             │
│ R4      = 0             │
└──────────────────────────┘  ◄── pxTopOfStack points here
Low Address

When this task is first scheduled:
1. PendSV restores R4-R11 (all zeros)
2. Sets PSP to point after R0
3. Returns from exception → hardware restores R0-R3, R12, LR, PC, xPSR
4. CPU starts executing at PC = Task_Function
5. R0 = pvParameters (passed as first argument per ARM ABI)
```

> **🎯 Interview Insight:** "How does the first task start executing?" — FreeRTOS sets up a fake exception return frame on the task's stack. When the PendSV handler "returns," the hardware pops this frame, and the CPU jumps to the task function with the correct parameter in R0.

---

## 5.2 Priorities and Starvation

### 5.2.1 Priority Assignment Strategy

Assigning priorities correctly is one of the most important design decisions in an RTOS system.

**The Rate Monotonic Rule of Thumb:**
> Tasks with shorter deadlines (higher frequency) should get higher priorities.

```
PRIORITY ASSIGNMENT EXAMPLE — Industrial Controller:
════════════════════════════════════════════════════

Priority  Task            Period    Note
────────────────────────────────────────────────────
   6      Emergency_Stop   < 1 ms   SAFETY CRITICAL
   5      Motor_Control    5 ms     Real-time control loop
   4      Sensor_Read      10 ms    Data acquisition
   3      Communication    50 ms    CAN/UART protocol
   2      Display_Update   200 ms   Human interface
   1      Data_Logging     1 s      Non-critical recording
   0      IDLE             ∞        Housekeeping
```

### 5.2.2 What Is Starvation?

**Starvation** occurs when a task never gets CPU time because higher-priority tasks always preempt it.

```
STARVATION SCENARIO:
════════════════════

Task A (Priority 3): Never blocks — runs continuously
Task B (Priority 2): Ready to run — NEVER gets CPU time!
Task C (Priority 1): Ready to run — NEVER gets CPU time!
Idle   (Priority 0): Also starved

Time ──────────────────────────────────────────────────►
Task A: ████████████████████████████████████████████████
Task B:                     (STARVED — never runs)
Task C:                     (STARVED — never runs)

CAUSE: Task A never calls a blocking API (vTaskDelay,
       xQueueReceive, xSemaphoreTake, etc.)
```

### 5.2.3 Preventing Starvation

**Rule 1: Every task MUST block periodically**
```c
/* WRONG: Never blocks — starves all lower-priority tasks */
void Task_Bad(void *pvParameters)
{
    for (;;)
    {
        Process_Data();    /* Runs forever without yielding */
    }
}

/* RIGHT: Blocks periodically — gives lower tasks a chance */
void Task_Good(void *pvParameters)
{
    for (;;)
    {
        Process_Data();
        vTaskDelay(pdMS_TO_TICKS(10));    /* Block for 10 ms */
    }
}
```

**Rule 2: Use event-driven design instead of polling**
```c
/* WRONG: Busy-polling wastes CPU and starves others */
void Task_Poller(void *pvParameters)
{
    for (;;)
    {
        if (data_available)
        {
            Process_Data();
        }
        /* Tight loop — uses 100% CPU even when idle! */
    }
}

/* RIGHT: Block on event — uses 0% CPU when idle */
void Task_EventDriven(void *pvParameters)
{
    for (;;)
    {
        /* Block until data arrives — CPU is free for other tasks */
        xQueueReceive(dataQueue, &data, portMAX_DELAY);
        Process_Data();
    }
}
```

**Rule 3: Be careful with equal-priority round-robin**
```c
/* With time-slicing ON: Tasks A, B, C at priority 2 share time equally
   With time-slicing OFF: First task runs until it blocks! */

/* Verify your config: */
#define configUSE_PREEMPTION     1
#define configUSE_TIME_SLICING   1    /* Enable round-robin */
```

### 5.2.4 Starvation Detection

```c
/* Add watchdog-style monitoring */
void Task_Watchdog(void *pvParameters)
{
    for (;;)
    {
        /* Check that each task has run recently */
        if ((xTaskGetTickCount() - task_b_last_run) > pdMS_TO_TICKS(5000))
        {
            /* Task B hasn't run in 5 seconds — it's starved! */
            Log_Error("Task B is starved!");
        }

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

/* In each monitored task: */
volatile TickType_t task_b_last_run = 0;
void Task_B(void *pvParameters)
{
    for (;;)
    {
        task_b_last_run = xTaskGetTickCount();  /* Update timestamp */
        /* ... do work ... */
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

---

## 5.3 Idle Task Internals

### 5.3.1 What Is the Idle Task?

The **Idle Task** is automatically created by `vTaskStartScheduler()` at priority 0. It runs when no other task is ready.

**The Idle Task's responsibilities:**
1. Free memory from deleted tasks (their TCBs and stacks)
2. Call the idle hook function (if configured)
3. Execute tickless idle (if configured)
4. Yield to other priority-0 tasks (if `configIDLE_SHOULD_YIELD == 1`)

### 5.3.2 Idle Task Source Code (Simplified)

```c
/*
 * This is what the idle task actually does (simplified from tasks.c).
 */
static portTASK_FUNCTION(prvIdleTask, pvParameters)
{
    (void)pvParameters;    /* Unused */

    for (;;)
    {
        /* 1. Check for tasks waiting to be deleted and free their memory */
        prvCheckTasksWaitingTermination();

        /* 2. If there are other priority-0 tasks, yield to them */
        #if (configIDLE_SHOULD_YIELD == 1)
        {
            if (listCURRENT_LIST_LENGTH(
                    &pxReadyTasksLists[tskIDLE_PRIORITY]) > 1)
            {
                taskYIELD();
            }
        }
        #endif

        /* 3. Call the user's idle hook */
        #if (configUSE_IDLE_HOOK == 1)
        {
            vApplicationIdleHook();
        }
        #endif

        /* 4. If tickless idle is enabled, enter sleep */
        #if (configUSE_TICKLESS_IDLE != 0)
        {
            /* Calculate expected idle time, suppress ticks, sleep */
        }
        #endif
    }
}
```

### 5.3.3 The Idle Hook

The idle hook is your opportunity to run code when the system has nothing else to do.

```c
/* Enable in FreeRTOSConfig.h */
#define configUSE_IDLE_HOOK    1

/* YOU must implement this function */
void vApplicationIdleHook(void)
{
    /*
     * RULES FOR IDLE HOOK:
     * 1. MUST NOT block (no vTaskDelay, no xQueueReceive, etc.)
     * 2. MUST NOT call any API that could block
     * 3. Should be very short
     * 4. Called repeatedly in a tight loop
     */

    /* Good uses: */
    idle_counter++;                    /* Count idle cycles (CPU load est.) */
    __WFI();                           /* Enter sleep until next interrupt */

    /* BAD uses: */
    /* vTaskDelay(100);             ← WILL CRASH (blocks idle task!) */
    /* xQueueReceive(...);          ← WILL CRASH */
}
```

**Common Idle Hook Uses:**

| Use Case | Code | Notes |
|---|---|---|
| Power saving | `__WFI()` | Sleep until next interrupt |
| CPU load estimation | `idle_counter++` | Compare with total ticks |
| Background CRC check | Compute one block per call | Non-blocking incremental work |
| Watchdog kicking | `IWDG_Refresh()` | Only if idle should always run |

### 5.3.4 CPU Load Measurement Using Idle Hook

```c
volatile uint32_t ulIdleCount = 0;

void vApplicationIdleHook(void)
{
    ulIdleCount++;
}

/* Call this every second from a timer task */
void Calculate_CPU_Load(void)
{
    static uint32_t ulLastIdleCount = 0;
    static uint32_t ulMaxIdleCount  = 0;   /* Calibrate during startup */

    uint32_t ulDelta = ulIdleCount - ulLastIdleCount;
    ulLastIdleCount = ulIdleCount;

    if (ulMaxIdleCount == 0)
    {
        /* First measurement: assume system is idle → this is 100% idle */
        ulMaxIdleCount = ulDelta;
    }

    /* CPU load = (1 - idle/max_idle) × 100 */
    uint32_t cpu_load = 100 - (ulDelta * 100 / ulMaxIdleCount);

    char msg[32];
    snprintf(msg, sizeof(msg), "CPU Load: %lu%%\r\n", cpu_load);
    /* Send over UART */
}
```

> **📝 Beginner Note:** The idle hook is called in a tight loop whenever no task is running. If your system is 90% idle, the hook runs millions of times per second. Keep it fast!

---

## 5.4 Task Notifications (Lightweight IPC)

### 5.4.1 What Are Task Notifications?

Task notifications are a **direct-to-task** communication mechanism. Instead of using a queue or semaphore (which is a separate shared object), you send a notification directly to a specific task.

**Advantages over queues/semaphores:**
- **45% faster** than binary semaphore
- **Zero RAM overhead** (notification value is stored in the TCB)
- No separate object to create
- Ideal for single-producer, single-consumer patterns

**Limitations:**
- Only **one task** can receive (no broadcasting)
- Cannot be used from multiple senders easily (if the notification overwrites)
- Only one 32-bit notification value per task (FreeRTOS v10.4+ adds an array)

### 5.4.2 Task Notification as Binary Semaphore

```c
/* ISR notifies a task that data is ready */
TaskHandle_t xProcessingTask = NULL;

/* ISR — fast, lightweight notification */
void USART2_IRQHandler(void)
{
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    /* Clear interrupt flag */
    __HAL_UART_CLEAR_FLAG(&huart2, UART_FLAG_RXNE);

    /* Store received byte */
    rx_byte = huart2.Instance->DR;

    /* Notify the processing task */
    vTaskNotifyGiveFromISR(xProcessingTask, &xHigherPriorityTaskWoken);

    /* Yield if a higher-priority task was woken */
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

/* Task — waits for notification */
void Task_ProcessUART(void *pvParameters)
{
    for (;;)
    {
        /* Block until notified (like a binary semaphore take) */
        ulTaskNotifyTake(pdTRUE,       /* Clear count on exit */
                         portMAX_DELAY); /* Wait forever */

        /* Process the received byte */
        Process_Byte(rx_byte);
    }
}
```

### 5.4.3 Task Notification as Event Flags

```c
/* Define event bits */
#define EVENT_SENSOR_READY    (1 << 0)
#define EVENT_BUTTON_PRESSED  (1 << 1)
#define EVENT_TIMER_EXPIRED   (1 << 2)

/* Sender: set specific event bits */
void Task_Sensor(void *pvParameters)
{
    for (;;)
    {
        Read_Sensor();
        xTaskNotify(xMainTask,
                    EVENT_SENSOR_READY,    /* Bits to set */
                    eSetBits);              /* OR with existing value */
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

/* Receiver: wait for specific event combination */
void Task_Main(void *pvParameters)
{
    uint32_t ulNotifiedValue;

    for (;;)
    {
        /* Wait for ANY of the events */
        xTaskNotifyWait(
            0x00,                          /* Don't clear on entry */
            0xFFFFFFFF,                    /* Clear all bits on exit */
            &ulNotifiedValue,              /* Store notification value */
            portMAX_DELAY                  /* Wait forever */
        );

        if (ulNotifiedValue & EVENT_SENSOR_READY)
        {
            Handle_Sensor_Data();
        }
        if (ulNotifiedValue & EVENT_BUTTON_PRESSED)
        {
            Handle_Button();
        }
    }
}
```

### 5.4.4 Task Notification Summary

| Function | Direction | Purpose |
|---|---|---|
| `xTaskNotifyGive()` | Task → Task | Increment notification count (like semaphore give) |
| `vTaskNotifyGiveFromISR()` | ISR → Task | Same, from interrupt context |
| `ulTaskNotifyTake()` | Receiver | Wait for count > 0 (like semaphore take) |
| `xTaskNotify()` | Task → Task | Set value/bits (flexible) |
| `xTaskNotifyFromISR()` | ISR → Task | Same, from interrupt context |
| `xTaskNotifyWait()` | Receiver | Wait for notification with value |

---

## 5.5 Task Design Best Practices

### 5.5.1 Task Structure Template

```c
/*
 * Production-quality task template.
 * Copy this as a starting point for new tasks.
 */
void Task_Template(void *pvParameters)
{
    /* ──── 1. Task Initialization (runs once) ──── */
    MyConfig_t *config = (MyConfig_t *)pvParameters;

    /* Initialize local resources */
    uint8_t buffer[64];
    uint32_t error_count = 0;

    /* Perform one-time setup */
    Hardware_Init(config->device_id);

    /* Confirm initialization complete (optional) */
    xSemaphoreGive(xInitComplete);

    /* ──── 2. Task Main Loop (runs forever) ──── */
    TickType_t xLastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        /* ──── 3. Wait for Trigger ──── */
        /* Option A: Periodic execution */
        vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(100));

        /* Option B: Event-driven */
        /* xQueueReceive(myQueue, &data, portMAX_DELAY); */

        /* ──── 4. Do Work ──── */
        StatusCode_t status = Do_Work(buffer, sizeof(buffer));

        /* ──── 5. Handle Errors ──── */
        if (status != STATUS_OK)
        {
            error_count++;
            if (error_count > MAX_ERRORS)
            {
                Log_Error("Task exceeded max errors");
                /* Optionally: suspend self, signal watchdog, etc. */
            }
        }
        else
        {
            error_count = 0;   /* Reset on success */
        }

        /* ──── 6. Communicate Results ──── */
        /* xQueueSend(resultQueue, &result, 0); */
    }

    /* ──── 7. Never reached ──── */
    /* If task must exit, call vTaskDelete(NULL); */
}
```

### 5.5.2 Common Task Patterns

**Periodic Task (Sensor Reading):**
```c
void Task_Periodic(void *pv)
{
    TickType_t xLastWake = xTaskGetTickCount();

    for (;;)
    {
        Do_Work();
        vTaskDelayUntil(&xLastWake, pdMS_TO_TICKS(PERIOD));
    }
}
```

**Event-Driven Task (Command Processor):**
```c
void Task_EventDriven(void *pv)
{
    Command_t cmd;

    for (;;)
    {
        if (xQueueReceive(cmdQueue, &cmd, portMAX_DELAY) == pdPASS)
        {
            Execute_Command(&cmd);
        }
    }
}
```

**One-Shot Task (Initialization):**
```c
void Task_OneShot(void *pv)
{
    Run_Calibration();
    Load_Config_From_Flash();
    Signal_Init_Complete();
    vTaskDelete(NULL);
}
```

**Gateway Task (ISR to Task Bridge):**
```c
void Task_Gateway(void *pv)
{
    for (;;)
    {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);

        /* ISR only stores data in a buffer and notifies.
         * Processing happens here, in task context. */
        Process_ISR_Data();
    }
}
```

---

## 5.6 Common Mistakes (Chapter 5)

### Mistake 1: Creating Too Many Tasks
```
WRONG: 30 tasks each doing one tiny thing
RIGHT: Group related operations into fewer, well-designed tasks.
Rule of thumb: 5-15 tasks for most embedded systems.
Each task should represent a logical subsystem.
```

### Mistake 2: Using `vTaskDelay()` When `vTaskDelayUntil()` Is Needed
```c
/* vTaskDelay(100): Delays 100 ticks AFTER current point.
   If the task took 20 ms to execute, the period is 120 ms. */

/* vTaskDelayUntil(&last, 100): Delays until 100 ticks AFTER
   the last wake-up. Period is always 100 ms regardless of
   execution time. */

/* For PERIODIC tasks, always use vTaskDelayUntil() */
```

### Mistake 3: Notification Lost When Task Isn't Waiting
```c
/* If xTaskNotify is called before the receiver calls
   xTaskNotifyWait, the notification is latched (not lost).
   BUT if another notify comes before the wait, it may
   overwrite or merge with the first. Be aware of this. */
```

---

## 5.7 Hands-On Exercises

### Exercise 5.1: Priority Starvation Demonstration

```c
/*
 * Create this setup and observe:
 * 1. Task_High never blocks — what happens to Task_Low?
 * 2. Add vTaskDelay(1) to Task_High — what changes?
 * 3. Set both tasks to the same priority — what happens?
 */
void Task_High(void *pv) { for(;;) { /* busy work, no delay */ } }
void Task_Low(void *pv)  { for(;;) { HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_6);
                                      vTaskDelay(pdMS_TO_TICKS(100)); } }
/* Create Task_High at priority 3, Task_Low at priority 1 */
```

### Exercise 5.2: Task Notification Benchmark

Measure the time difference between:
1. Using a binary semaphore for ISR → Task signaling
2. Using task notification for the same

Toggle a GPIO pin to measure latency with an oscilloscope.

### Exercise 5.3: CPU Load Monitor

Implement the CPU load measurement technique from Section 5.3.4. Print the CPU load over UART every second. Then add progressively heavier work to a task and watch the load increase.

---

## 5.8 Chapter Summary

| Concept | Key Takeaway |
|---|---|
| **xTaskCreate internals** | Allocates TCB + stack, sets up initial stack frame |
| **Initial stack frame** | Fake exception return to "trick" CPU into running task |
| **Priority starvation** | Higher-priority task never blocks → lower tasks starve |
| **Prevention** | Every task must block periodically; use event-driven design |
| **Idle task** | Priority 0; frees deleted task memory; runs idle hook |
| **Idle hook** | Must not block; useful for power saving and CPU load |
| **Task notifications** | 45% faster than semaphores; zero RAM overhead |
| **vTaskDelayUntil** | Use for periodic tasks (vs. vTaskDelay for one-shot delays) |

---

*End of Chapter 5*
