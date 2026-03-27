# Chapter 3: Task Management

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Create and delete tasks using FreeRTOS APIs
2. Explain the four task states and transitions between them
3. Describe the Task Control Block (TCB) and its contents
4. Calculate stack requirements and detect stack overflow
5. Implement proper task lifecycle management in production systems
6. Debug common task-related issues

---

## 3.1 Task Creation and Deletion

### 3.1.1 What Is a Task?

A **task** (called a "thread" in general-purpose OSes) is an independent unit of execution. Each task:

- Has its **own stack** (local variables, function call chain)
- Has its **own context** (registers, program counter)
- Runs as if it **owns** the entire CPU
- Is managed by the kernel (scheduled, preempted, delayed)

#### Analogy: Workers in a Factory

Think of tasks as workers on a factory floor with one machine (CPU):
- Each worker has a **workbench** (stack) with their personal tools
- Each worker has a **clipboard** (TCB) tracking their assignment
- The **foreman** (scheduler) tells workers when to use the machine
- Workers can **wait** for parts (blocked), **be ready** to work (ready), or **be using the machine** (running)

### 3.1.2 The xTaskCreate API

```c
BaseType_t xTaskCreate(
    TaskFunction_t        pvTaskCode,       /* Function pointer */
    const char * const    pcName,           /* Human-readable name */
    configSTACK_DEPTH_TYPE usStackDepth,    /* Stack size in WORDS */
    void * const          pvParameters,     /* Parameter passed to task */
    UBaseType_t           uxPriority,       /* Task priority */
    TaskHandle_t * const  pxCreatedTask     /* Handle (optional) */
);
```

**Return value:**
- `pdPASS` — Task created successfully
- `pdFAIL` — Not enough heap memory

**Detailed parameter breakdown:**

| Parameter | Description | Typical Value |
|---|---|---|
| `pvTaskCode` | Pointer to the task function | `Task_LED` |
| `pcName` | Debug name (max `configMAX_TASK_NAME_LEN` chars) | `"LED"` |
| `usStackDepth` | Stack size in **words** (not bytes!) | `128` (= 512 bytes on 32-bit) |
| `pvParameters` | Pointer passed to task function | `NULL` or `&config` |
| `uxPriority` | Priority (0 = lowest, `configMAX_PRIORITIES-1` = highest) | `1` |
| `pxCreatedTask` | Output: handle to the task (for later control) | `&xTaskHandle` or `NULL` |

> **⚠️ Critical:** `usStackDepth` is in **words** (4 bytes each on ARM Cortex-M), not bytes! `128` means 128 × 4 = 512 bytes.

### 3.1.3 Complete Task Creation Example

```c
/* Task handle — allows controlling the task later */
TaskHandle_t xSensorTaskHandle = NULL;

/* Task function — must never return! */
void Task_ReadSensor(void *pvParameters)
{
    /* Cast the parameter if needed */
    uint32_t sensor_id = (uint32_t)pvParameters;

    /* Local variables live on this task's stack */
    uint16_t reading;
    uint32_t count = 0;

    /* Task initialization (runs once) */
    Sensor_Init(sensor_id);

    /* Infinite loop — the task's "main loop" */
    for (;;)
    {
        reading = Sensor_Read(sensor_id);
        count++;

        /* Process reading... */

        vTaskDelay(pdMS_TO_TICKS(100));   /* Run every 100 ms */
    }

    /* Should NEVER reach here.
     * If the task must end, call vTaskDelete(NULL) instead. */
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();

    /* Create the sensor task */
    BaseType_t result = xTaskCreate(
        Task_ReadSensor,          /* Task function */
        "Sensor",                 /* Name for debugging */
        256,                      /* Stack: 256 words = 1024 bytes */
        (void *)1,                /* Parameter: sensor_id = 1 */
        2,                        /* Priority: 2 */
        &xSensorTaskHandle        /* Store handle */
    );

    if (result != pdPASS)
    {
        /* Task creation failed — not enough heap */
        Error_Handler();
    }

    vTaskStartScheduler();

    for (;;);  /* Never reached */
}
```

### 3.1.4 Creating Multiple Tasks from One Function

A powerful pattern: use a single function for similar tasks, differentiated by parameters.

```c
typedef struct
{
    GPIO_TypeDef *port;
    uint16_t      pin;
    uint32_t      period_ms;
    const char   *name;
} LED_Config_t;

/* One function, multiple instances */
void Task_BlinkLED(void *pvParameters)
{
    LED_Config_t *config = (LED_Config_t *)pvParameters;

    for (;;)
    {
        HAL_GPIO_TogglePin(config->port, config->pin);
        vTaskDelay(pdMS_TO_TICKS(config->period_ms));
    }
}

/* Configuration MUST be static/global — not local to main()! */
static LED_Config_t led_configs[] = {
    { GPIOA, GPIO_PIN_5,  100, "LED_Fast" },
    { GPIOA, GPIO_PIN_6,  500, "LED_Med"  },
    { GPIOA, GPIO_PIN_7, 1000, "LED_Slow" },
};

int main(void)
{
    /* ... init ... */

    for (int i = 0; i < 3; i++)
    {
        xTaskCreate(
            Task_BlinkLED,
            led_configs[i].name,
            128,
            &led_configs[i],     /* Each task gets different config */
            1,
            NULL
        );
    }

    vTaskStartScheduler();
    for (;;);
}
```

> **⚠️ Critical:** The parameter data (`led_configs`) must persist for the lifetime of the task. **Never** pass a pointer to a local variable — it will be destroyed when `main()` reaches `vTaskStartScheduler()`, and the task will access garbage memory.

### 3.1.5 Static vs. Dynamic Task Creation

FreeRTOS supports two methods of task creation:

**Dynamic (xTaskCreate):** Memory allocated from the FreeRTOS heap at runtime.
```c
/* Dynamic: kernel allocates TCB and stack from heap */
xTaskCreate(Task_LED, "LED", 128, NULL, 1, NULL);
```

**Static (xTaskCreateStatic):** You provide pre-allocated memory.
```c
/* Static: YOU provide the memory — no heap needed */
StaticTask_t  xTaskBuffer;
StackType_t   xStack[128];

xTaskCreateStatic(
    Task_LED,          /* Function */
    "LED",             /* Name */
    128,               /* Stack size */
    NULL,              /* Parameters */
    1,                 /* Priority */
    xStack,            /* Stack buffer (you provide) */
    &xTaskBuffer       /* TCB buffer (you provide) */
);
```

**Comparison:**

| Feature | Dynamic (`xTaskCreate`) | Static (`xTaskCreateStatic`) |
|---|---|---|
| Memory source | FreeRTOS heap | Your arrays (BSS/data section) |
| Heap required | Yes | No (can use heap_0 or no heap) |
| Fragmentation risk | Yes | No |
| Memory known at compile | No | Yes |
| RAM usage analysis | Runtime only | Linker map shows usage |
| Safety-critical systems | Less preferred | Preferred (deterministic) |
| Ease of use | Simpler | More boilerplate |

> **🏭 Industry Insight:** Safety-critical systems (IEC 61508, ISO 26262) often mandate static allocation. You can verify at compile time that all memory fits. Dynamic allocation can fail at runtime — an unacceptable risk in safety systems.

### 3.1.6 Task Deletion

```c
void vTaskDelete(TaskHandle_t xTaskToDelete);
```

- Pass `NULL` to delete the calling task (itself)
- Pass a handle to delete another task
- The **idle task** is responsible for freeing memory of deleted tasks

```c
/* Example: Task that deletes itself after initialization */
void Task_OneShot(void *pvParameters)
{
    /* Perform one-time initialization */
    Calibrate_Sensors();
    Load_Configuration();

    /* Done — delete self */
    vTaskDelete(NULL);
    /* Code after vTaskDelete(NULL) never executes */
}

/* Example: Deleting another task */
TaskHandle_t xMonitorHandle = NULL;

void Task_Monitor(void *pvParameters)
{
    for (;;)
    {
        Monitor_System();
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

void Task_Manager(void *pvParameters)
{
    for (;;)
    {
        if (system_error_detected)
        {
            vTaskDelete(xMonitorHandle);    /* Kill the monitor task */
            xMonitorHandle = NULL;
        }
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

> **⚠️ Warning:** Avoid frequent task creation/deletion in production systems. It causes heap fragmentation (with heap_2/heap_4) and is an anti-pattern. Instead, suspend/resume or use event-driven blocking.

---

## 3.2 Task States

### 3.2.1 The Four Task States

Every task in FreeRTOS is in exactly one of four states:

```
┌─────────────────────────────────────────────────────────────┐
│                    TASK STATE MACHINE                        │
│                                                              │
│                    ┌───────────┐                             │
│        ┌──────────►│  RUNNING  │◄──────────┐                │
│        │           └─────┬─────┘           │                │
│        │                 │                 │                │
│   Scheduler         Preempted /        Scheduler            │
│   selects          Yield/Tick          selects              │
│   this task             │              this task            │
│        │                ▼                  │                │
│        │           ┌───────────┐           │                │
│        └───────────│   READY   │───────────┘                │
│                    └─────┬─────┘                            │
│                          │ ▲                                │
│              Block:      │ │  Event / Timeout               │
│              Delay,      │ │  occurs                        │
│              Queue,      │ │                                │
│              Semaphore   ▼ │                                │
│                    ┌───────────┐                            │
│                    │  BLOCKED  │                             │
│                    └───────────┘                            │
│                          │ ▲                                │
│                          │ │                                │
│                          │ │                                │
│                    ┌───────────┐                            │
│                    │ SUSPENDED │                             │
│                    └───────────┘                            │
│              (Only by explicit vTaskSuspend/Resume)          │
└─────────────────────────────────────────────────────────────┘
```

### 3.2.2 State Descriptions

**RUNNING:**
- The task is currently executing on the CPU
- Only **one** task can be Running at a time (single-core)
- The kernel tracks this via `pxCurrentTCB`

**READY:**
- The task is able to run (not waiting for anything)
- It is waiting for the scheduler to give it the CPU
- Stored in `pxReadyTasksLists[priority]`

**BLOCKED:**
- The task is waiting for an event or timeout
- Examples: `vTaskDelay()`, `xQueueReceive()`, `xSemaphoreTake()`
- The task specifies a **maximum wait time** (or `portMAX_DELAY` for forever)
- Stored in `xDelayedTaskList` or resource's wait list
- A blocked task uses **zero CPU time**

**SUSPENDED:**
- The task is explicitly paused by `vTaskSuspend()`
- It will not run until `vTaskResume()` is called
- Unlike Blocked, there is **no timeout** — it stays suspended until explicitly resumed
- Stored in `xSuspendedTaskList`

### 3.2.3 State Transitions with Examples

```c
void Task_Example(void *pvParameters)
{
    /* Task is now RUNNING */

    /* Transition: RUNNING → BLOCKED (waiting for 100 ms) */
    vTaskDelay(pdMS_TO_TICKS(100));
    /* After 100 ms, kernel moves task: BLOCKED → READY */
    /* When scheduler picks it: READY → RUNNING */

    /* Transition: RUNNING → BLOCKED (waiting for queue data) */
    uint32_t value;
    xQueueReceive(myQueue, &value, pdMS_TO_TICKS(500));
    /* BLOCKED → READY when data arrives or timeout */

    /* Transition: RUNNING → BLOCKED (waiting for semaphore) */
    xSemaphoreTake(mySemaphore, portMAX_DELAY);
    /* BLOCKED → READY when semaphore given */
}

/* From another task: */
void Task_Controller(void *pvParameters)
{
    /* Suspend a task: moves it to SUSPENDED regardless of current state */
    vTaskSuspend(xExampleHandle);

    /* Resume a task: moves it to READY */
    vTaskResume(xExampleHandle);
}
```

### 3.2.4 Timing Diagram of State Transitions

```
Time ────────────────────────────────────────────────────────────►

Task A   RUNNING  RUNNING    READY    BLOCKED   READY  RUNNING
(Pri 3) ████████ ████████           ░░░░░░░░░░        ████████

Task B    READY   READY   RUNNING    RUNNING   RUNNING  READY
(Pri 2)                   ████████  ████████  ████████

Idle     READY   READY    READY     READY     READY   READY
(Pri 0)

Events:
         │       │        │         │          │       │
         ▼       ▼        ▼         ▼          ▼       ▼
      A starts  A runs  A calls   B runs     A's    A preempts
               normally vTaskDelay while A   delay   B
                        (BLOCKED)  sleeps   expires
```

### 3.2.5 Querying Task State

```c
/* Get the current state of a task */
eTaskState eTaskGetState(TaskHandle_t xTask);

/*
 * Returns one of:
 *   eRunning   = 0
 *   eReady     = 1
 *   eBlocked   = 2
 *   eSuspended = 3
 *   eDeleted   = 4
 */

/* Example: Check if a task is running */
if (eTaskGetState(xSensorHandle) == eBlocked)
{
    /* Task is waiting for something */
}

/* Get detailed info about all tasks (for debugging) */
char buffer[512];
vTaskList(buffer);
/* buffer now contains a formatted table of all tasks:
 *
 * Name          State  Pri  Stack  Num
 * Sensor        B      2    180    1
 * LED           R      1    200    2
 * IDLE          X      0    110    3
 *
 * State: X=Running, R=Ready, B=Blocked, S=Suspended, D=Deleted
 */
```

> **📝 Beginner Note:** Enable `configUSE_TRACE_FACILITY` and `configUSE_STATS_FORMATTING_FUNCTIONS` in `FreeRTOSConfig.h` to use `vTaskList()`. It's invaluable for debugging.

---

## 3.3 Task Control Block (TCB)

### 3.3.1 What Is the TCB?

The **Task Control Block (TCB)** is a data structure that the kernel maintains for each task. It stores everything the kernel needs to manage the task.

#### Analogy: An Employee Record

The TCB is like an employee's HR file:
- Name and ID
- Current assignment (what they're working on — the stack pointer)
- Pay grade (priority)
- Status (working, on break, on leave)
- Desk location (stack memory)

### 3.3.2 TCB Contents (FreeRTOS)

```c
/*
 * Simplified TCB structure from FreeRTOS tasks.c
 * (actual structure has more fields and conditional compilation)
 */
typedef struct tskTaskControlBlock
{
    /* CRITICAL: pxTopOfStack MUST be the first member.
     * The context switch code depends on this position. */
    volatile StackType_t *pxTopOfStack;      /* Current stack pointer */

    /* List items for linking into kernel lists */
    ListItem_t          xStateListItem;       /* Ready/Blocked/Suspended list */
    ListItem_t          xEventListItem;       /* Event wait list */

    /* Task properties */
    UBaseType_t         uxPriority;            /* Current priority */
    UBaseType_t         uxBasePriority;         /* Original priority (for mutex) */

    /* Stack information */
    StackType_t         *pxStack;              /* Start of stack memory */
    uint16_t            usStackDepth;          /* Stack size in words */

    /* Task identification */
    char                pcTaskName[configMAX_TASK_NAME_LEN];

    /* Mutual exclusion tracking */
    UBaseType_t         uxMutexesHeld;         /* Number of mutexes held */

    /* Notification */
    volatile uint32_t   ulNotifiedValue;       /* Task notification value */
    volatile uint8_t    ucNotifyState;          /* Notification state */

    /* Stack overflow detection */
    #if (configCHECK_FOR_STACK_OVERFLOW > 0)
        StackType_t     *pxEndOfStack;          /* Stack boundary */
    #endif

    /* Runtime stats */
    #if (configGENERATE_RUN_TIME_STATS == 1)
        uint32_t        ulRunTimeCounter;       /* Total CPU time used */
    #endif

} TCB_t;
```

### 3.3.3 TCB Memory Layout

```
Memory layout for one task (approximate):
═════════════════════════════════════════

TCB (Task Control Block) — ~80-120 bytes:
┌──────────────────────────────────────┐  ◄── TCB start
│ pxTopOfStack  (4 bytes)              │  ◄── Points to current stack top
│ xStateListItem (20 bytes)            │  ◄── Links into ready/blocked list
│ xEventListItem (20 bytes)            │  ◄── Links into event wait list
│ uxPriority (4 bytes)                 │
│ pxStack (4 bytes)                    │  ◄── Points to stack start
│ pcTaskName[] (configMAX_TASK_NAME_LEN)│
│ ... other fields ...                 │
└──────────────────────────────────────┘

Stack — usStackDepth × 4 bytes:
┌──────────────────────────────────────┐  ◄── pxStack (stack bottom)
│ 0xA5A5A5A5  (fill pattern)           │  ◄── For overflow detection
│ 0xA5A5A5A5                           │
│ ...                                  │
│ (unused stack space)                 │
│ ...                                  │
│ Local variables                      │
│ Function call stack frames           │
│ Saved context (R4-R11)              │
│ Hardware-saved (R0-R3, R12, LR, PC) │
└──────────────────────────────────────┘  ◄── pxTopOfStack (grows DOWN)

Total RAM per task ≈ TCB size + (usStackDepth × 4) bytes
Example: 100 + (256 × 4) = 1124 bytes per task
```

### 3.3.4 The pxCurrentTCB Pointer

The kernel maintains a global pointer to the currently running task's TCB:

```c
/* THE most important variable in FreeRTOS */
TCB_t * volatile pxCurrentTCB = NULL;

/*
 * At any point in time:
 *   pxCurrentTCB->pxTopOfStack  = the running task's stack pointer
 *   pxCurrentTCB->uxPriority    = the running task's priority
 *   pxCurrentTCB->pcTaskName    = the running task's name
 *
 * During context switch:
 *   1. Save CPU registers to pxCurrentTCB->pxTopOfStack location
 *   2. Call scheduler → pxCurrentTCB now points to a different TCB
 *   3. Restore CPU registers from new pxCurrentTCB->pxTopOfStack
 */
```

> **🎯 Interview Insight:** "What is pxCurrentTCB?" — It's the kernel's pointer to the TCB of the currently executing task. It's the most critical variable in FreeRTOS. The context switch saves/restores state through this pointer.

---

## 3.4 Stack Management

### 3.4.1 Why Each Task Needs Its Own Stack

In bare-metal, there is one stack for the entire program. In an RTOS, each task needs its own stack because:

1. **Context isolation:** Each task has its own local variables
2. **Independent call chains:** Task A might be 5 levels deep in function calls while Task B is only 2 levels deep
3. **Preemption safety:** When Task A is preempted, its stack remains intact
4. **Register save area:** The context switch pushes saved registers onto the task's stack

### 3.4.2 Stack Growth on ARM Cortex-M

ARM uses a **full-descending stack**: the stack pointer starts high and grows downward.

```
Stack Memory for Task A (256 words = 1024 bytes):
═══════════════════════════════════════════════════

High Address (0x20001400)
┌──────────────────────────────────────┐
│  Initial setup: xPSR, PC, LR, R12,  │  ◄── Created by xTaskCreate
│  R3, R2, R1, R0 (simulated          │      (simulates a return from
│  exception frame)                    │       exception to start task)
├──────────────────────────────────────┤
│  R4-R11 (saved by kernel)           │  ◄── pxTopOfStack points here
├──────────────────────────────────────┤      after first context switch
│                                      │
│  Local variables of task function    │
│  Function call parameters            │
│  Return addresses                    │
│                                      │
│  ▼ Stack grows downward ▼           │
│                                      │
│  (Unused stack space)                │
│                                      │
│  0xA5A5A5A5 (fill pattern for       │
│   stack high-water mark detection)   │
│                                      │
├──────────────────────────────────────┤
│  Stack overflow sentinel             │  ◄── pxStack (stack bottom)
└──────────────────────────────────────┘
Low Address (0x20001000)

If the stack pointer ever reaches below pxStack,
you have a STACK OVERFLOW — one of the hardest
bugs to diagnose in embedded systems.
```

### 3.4.3 How to Determine Stack Size

This is one of the most common questions in RTOS development. The answer depends on:

1. **Local variables** in the task function and all functions it calls
2. **Function call depth** (each call pushes a return address + saved registers)
3. **Interrupt nesting** (interrupts can push more context on the main stack)
4. **Compiler behavior** (optimization level affects stack usage)

#### Method 1: Start Large, Measure, Then Reduce

```c
/* 1. Create the task with a generous stack */
xTaskCreate(Task_Sensor, "Sensor", 512, NULL, 2, NULL);  /* 2048 bytes */

/* 2. Run the system under stress (all code paths exercised) */

/* 3. Check high-water mark (minimum free stack ever) */
UBaseType_t uxHighWaterMark = uxTaskGetStackHighWaterMark(xSensorHandle);
/* Returns the minimum number of WORDS of free stack space */

/* 4. Calculate margin:
 *    If 512 words allocated and high-water mark = 340 words,
 *    then max usage = 512 - 340 = 172 words = 688 bytes
 *    Set stack to: max_usage + 25% safety margin
 *    = 172 * 1.25 = 215 words → round to 256 words
 */
```

#### Method 2: Analytical Calculation

```
Stack Size = Context Save Size
           + ISR Stack Frame (if applicable)
           + All Local Variables
           + Deepest Function Call Chain
           + Safety Margin (25-50%)

Context Save Size (Cortex-M4):
  Hardware auto-save:  8 registers × 4 bytes = 32 bytes
  Software save:       8 registers × 4 bytes = 32 bytes
  ─────────────────────────────────────────────────────
  Total:               64 bytes per context switch

Example calculation:
  Context save:        64 bytes
  Task function locals: 48 bytes (uint16_t buffer[24])
  Called functions:     
    → ProcessData():   32 bytes (local vars)
      → FilterData():  16 bytes (local vars)
        → sqrt():      24 bytes (lib function)
  Call chain overhead:  4 levels × 8 bytes = 32 bytes (saved LR + alignment)
  ─────────────────────────────────────────────────────
  Subtotal:            216 bytes
  + 30% safety margin: 281 bytes ≈ 72 words → round to 128 words
```

#### Method 3: Compiler Stack Analysis

```bash
# GCC can analyze static stack usage (does NOT include dynamic allocation)
arm-none-eabi-gcc -fstack-usage -c task_sensor.c
# Produces task_sensor.su file with per-function stack usage

# Example output (task_sensor.su):
# Task_ReadSensor    128    static
# ProcessData         64    static
# FilterData          32    static
```

### 3.4.4 Stack Overflow Detection

FreeRTOS provides two overflow detection methods:

**Method 1: Check on context switch**
```c
/* FreeRTOSConfig.h */
#define configCHECK_FOR_STACK_OVERFLOW    1

/*
 * At every context switch, the kernel checks if the stack pointer
 * is still within bounds. Fast but can miss transient overflow
 * (stack could overflow and return within a single tick).
 */
```

**Method 2: Pattern checking**
```c
/* FreeRTOSConfig.h */
#define configCHECK_FOR_STACK_OVERFLOW    2

/*
 * At task creation, the last 20 bytes of the stack are filled with
 * a known pattern (0xA5A5A5A5). At every context switch, the kernel
 * checks if this pattern is still intact. More reliable but slower.
 */
```

**Stack overflow hook:**
```c
/*
 * YOU must implement this function.
 * It is called when a stack overflow is detected.
 */
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName)
{
    /* WARNING: Stack is already corrupted at this point!
     * Avoid doing anything that uses the stack heavily. */

    /* Option 1: Halt and indicate error */
    taskDISABLE_INTERRUPTS();

    /* Turn on error LED (direct register access, no HAL) */
    GPIOA->BSRR = GPIO_PIN_5;

    /* Log the task name if possible */
    /* (Use a fixed-size buffer, not printf!) */

    for (;;);  /* Halt — requires power cycle */
}
```

### 3.4.5 Stack Overflow: What Actually Happens

```
STACK OVERFLOW SCENARIO:
════════════════════════

Before overflow (normal):                After overflow (corrupted):
┌────────────────────────┐               ┌────────────────────────┐
│  Task B's data         │  ◄── Adjacent │  Task B's data         │
│  (or kernel data)      │      memory   │  CORRUPTED!!! ████████ │
├────────────────────────┤               ├────────────────────────┤
│  Stack sentinel        │               │  OVERWRITTEN! ████████ │
├────────────────────────┤               ├────────────────────────┤
│  (empty)               │               │  █ Overflow data █████ │
│  (empty)               │               │  █ Overflow data █████ │
│  Task A's data         │               │  Task A stack data     │
│  Saved context         │               │  Saved context         │
└────────────────────────┘               └────────────────────────┘

Symptoms of stack overflow:
 ✗ Random HardFaults
 ✗ Task B behaves erratically (its data is corrupted)
 ✗ Kernel crashes (if kernel data is adjacent)
 ✗ System resets unexpectedly
 ✗ Data corruption that appears "random"
```

> **⚠️ Critical:** Stack overflow is the **#1 cause** of mysterious crashes in RTOS applications. Always enable stack overflow detection during development. Size your stacks carefully and monitor high-water marks.

### 3.4.6 ARM Cortex-M Dual Stack Pointers

ARM Cortex-M has two stack pointers:

```
┌─────────────────────────────────────────────────────────┐
│ MSP (Main Stack Pointer)                                │
│ - Used by exception handlers (interrupts, SysTick)      │
│ - Used before the scheduler starts                      │
│ - Size defined in linker script (_Min_Stack_Size)       │
│ - Shared by ALL interrupts (they nest on the MSP)       │
│                                                         │
│ PSP (Process Stack Pointer)                             │
│ - Used by FreeRTOS tasks                                │
│ - Each task has its own PSP value (stored in TCB)       │
│ - Context switch swaps PSP between tasks                │
└─────────────────────────────────────────────────────────┘

IMPORTANT: Interrupts ALWAYS use MSP, regardless of which
task was running. This means interrupt stack usage comes
from the MSP, not the task stack.

Total interrupt stack needed = deepest interrupt nesting
  × (exception frame + ISR local variables)
```

---

## 3.5 Task Priorities in Detail

### 3.5.1 Priority Numbering

```
PRIORITY ASSIGNMENTS IN FREERTOS:
═════════════════════════════════

Priority 0:    [IDLE TASK]          ◄── LOWEST (always exists)
Priority 1:    [Task_Logger]        ◄── Low importance
Priority 2:    [Task_Display]
Priority 3:    [Task_Sensor]
Priority 4:    [Task_Control]       ◄── High importance
...
Priority (configMAX_PRIORITIES-1):  ◄── HIGHEST

Default configMAX_PRIORITIES = 5 (can be increased)

NOTE: Using too many priority levels wastes RAM (one ready list
per level). 5-10 levels is sufficient for most applications.
```

### 3.5.2 Dynamic Priority Changing

```c
/* Change a task's priority at runtime */
void vTaskPrioritySet(TaskHandle_t xTask, UBaseType_t uxNewPriority);

/* Get a task's current priority */
UBaseType_t uxTaskPriorityGet(TaskHandle_t xTask);

/* Example: Temporarily boost priority for critical work */
void Task_DataTransfer(void *pvParameters)
{
    for (;;)
    {
        xSemaphoreTake(xDataReady, portMAX_DELAY);

        /* Boost priority during critical transfer */
        UBaseType_t origPriority = uxTaskPriorityGet(NULL);
        vTaskPrioritySet(NULL, configMAX_PRIORITIES - 1);

        Transfer_Critical_Data();

        /* Restore original priority */
        vTaskPrioritySet(NULL, origPriority);
    }
}
```

### 3.5.3 The Idle Task

FreeRTOS automatically creates the **idle task** at priority 0. It runs when no other task is ready.

```c
/* The idle task does critical housekeeping: */
static void prvIdleTask(void *pvParameters)
{
    for (;;)
    {
        /* 1. Free memory from deleted tasks */
        prvCheckTasksWaitingTermination();

        /* 2. Call the idle hook (if configured) */
        #if (configUSE_IDLE_HOOK == 1)
            vApplicationIdleHook();
        #endif

        /* 3. Yield to other priority-0 tasks (if any) */
        #if (configIDLE_SHOULD_YIELD == 1)
            taskYIELD();
        #endif

        /* 4. Enter tickless sleep (if configured) */
        #if (configUSE_TICKLESS_IDLE == 1)
            /* Suppresses tick and sleeps */
        #endif
    }
}
```

> **⚠️ Warning:** Never starve the idle task completely. If no task ever blocks, the idle task never runs, and deleted task memory is never freed, causing a memory leak.

---

## 3.6 Common Mistakes (Chapter 3)

### Mistake 1: Stack Size in Bytes Instead of Words

```c
/* WRONG: 128 bytes — way too small! */
xTaskCreate(MyTask, "Task", 128, NULL, 1, NULL);
/* This actually allocates 128 × 4 = 512 bytes, which HAPPENS to be
   enough for simple tasks. But if you INTENDED 128 bytes, you'll
   have 4× the memory you planned. */

/* CLEAR: Use pdMS_TO_TICKS pattern for clarity */
#define STACK_SIZE_BYTES(x)    ((x) / sizeof(StackType_t))
xTaskCreate(MyTask, "Task", STACK_SIZE_BYTES(1024), NULL, 1, NULL);
```

### Mistake 2: Task Function Returns

```c
/* WRONG: Task function returns — undefined behavior! */
void Task_Bad(void *pvParameters)
{
    Do_Some_Work();
    /* Falls off the end — CPU jumps to garbage address! */
}

/* RIGHT: Infinite loop or explicit delete */
void Task_Good(void *pvParameters)
{
    for (;;)
    {
        Do_Some_Work();
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

/* RIGHT: One-shot task that deletes itself */
void Task_OneShot(void *pvParameters)
{
    Do_Some_Work();
    vTaskDelete(NULL);    /* Proper cleanup */
}
```

### Mistake 3: Passing Local Variable Address as Parameter

```c
/* WRONG: config is on the stack and gets destroyed */
void CreateTasks(void)
{
    LED_Config_t config = { GPIOA, GPIO_PIN_5, 100 };
    xTaskCreate(Task_LED, "LED", 128, &config, 1, NULL);
    /* config is destroyed when CreateTasks returns!
     * Task_LED will read garbage memory. */
}

/* RIGHT: Use static or global storage */
static LED_Config_t config = { GPIOA, GPIO_PIN_5, 100 };
void CreateTasks(void)
{
    xTaskCreate(Task_LED, "LED", 128, &config, 1, NULL);
    /* config persists because it's static */
}
```

### Mistake 4: Not Checking xTaskCreate Return Value

```c
/* DANGEROUS: Ignoring return value */
xTaskCreate(Task_Critical, "Crit", 512, NULL, 4, NULL);
/* If this fails (out of heap), no task is created, but you don't know! */

/* SAFE: Check return value */
if (xTaskCreate(Task_Critical, "Crit", 512, NULL, 4, NULL) != pdPASS)
{
    Error_Handler();   /* Handle the failure */
}
```

---

## 3.7 Debugging Tips

1. **Use `vTaskList()` to dump task states:** Shows name, state, priority, and free stack
2. **Monitor stack high-water marks:** Call `uxTaskGetStackHighWaterMark()` periodically
3. **Enable `configASSERT()`:** Catches many errors early (NULL pointers, invalid parameters)
4. **Watch `pxCurrentTCB` in the debugger:** See which task is running at any breakpoint
5. **Set breakpoint in `vApplicationStackOverflowHook()`:** Catches stack overflows
6. **Check the heap:** `xPortGetFreeHeapSize()` shows remaining heap memory
7. **UART debug output:** Print task status periodically to a serial terminal

```c
/* Periodic task status dump over UART */
void Task_Debug(void *pvParameters)
{
    char buffer[512];

    for (;;)
    {
        vTaskList(buffer);
        HAL_UART_Transmit(&huart2, (uint8_t *)buffer, strlen(buffer), 1000);

        /* Also print heap info */
        char heap_info[64];
        snprintf(heap_info, sizeof(heap_info),
                 "\r\nFree Heap: %lu bytes\r\n\r\n",
                 (unsigned long)xPortGetFreeHeapSize());
        HAL_UART_Transmit(&huart2, (uint8_t *)heap_info, strlen(heap_info), 1000);

        vTaskDelay(pdMS_TO_TICKS(5000));   /* Every 5 seconds */
    }
}
```

---

## 3.8 Hands-On Exercises

### Exercise 3.1: Task Creation and State Observation

**Objective:** Create three tasks and observe their states using `vTaskList()`.

```c
/*
 * Create three tasks with different priorities.
 * Use a debug task to print task states over UART.
 *
 * Expected output:
 *   Name          State  Priority  Stack  Num
 *   High          B      3         XXX    1
 *   Medium        B      2         XXX    2
 *   Low           R      1         XXX    3
 *   Debug         X      4         XXX    4
 *   IDLE          R      0         XXX    5
 */

void Task_High(void *pv)   { for(;;) { vTaskDelay(pdMS_TO_TICKS(500)); } }
void Task_Medium(void *pv) { for(;;) { vTaskDelay(pdMS_TO_TICKS(300)); } }
void Task_Low(void *pv)    { for(;;) { vTaskDelay(pdMS_TO_TICKS(100)); } }

void Task_Debug(void *pvParameters)
{
    char buf[512];
    for (;;)
    {
        vTaskList(buf);
        /* Send buf over UART */
        HAL_UART_Transmit(&huart2, (uint8_t *)buf, strlen(buf), 1000);
        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}
```

**Questions:**
1. Which task is shown as "Running" (X) when the Debug task prints? (Answer: Debug — it's running the print code)
2. What state are High, Medium, Low in most of the time? (Answer: Blocked — they're in vTaskDelay)
3. What is the Idle task doing? (Answer: Ready or Running when all others are blocked)

### Exercise 3.2: Stack High-Water Mark Monitoring

```c
/*
 * Monitor stack usage of all tasks.
 * Deliberately cause a near-overflow to see the high-water mark change.
 */
void Task_StackTest(void *pvParameters)
{
    /* Start with small stack: 128 words = 512 bytes */

    for (;;)
    {
        /* Allocate a local buffer — uses stack! */
        uint8_t buffer[200];  /* 200 bytes on stack */

        /* Simulate processing */
        memset(buffer, 0xAA, sizeof(buffer));

        /* Report stack usage */
        UBaseType_t hwm = uxTaskGetStackHighWaterMark(NULL);
        /* hwm = minimum free words EVER remaining */

        char msg[64];
        snprintf(msg, sizeof(msg), "Stack HWM: %lu words free\r\n",
                 (unsigned long)hwm);
        /* Send msg over UART */

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

**Questions:**
1. With a 128-word stack and a 200-byte local buffer, how much stack is the buffer using? (Answer: 200/4 = 50 words)
2. If you change the buffer to 400 bytes, will it overflow? (Answer: Likely yes — 400/4 = 100 words + context save + overhead ≈ 120+ words out of 128)
3. What high-water mark value indicates danger? (Answer: < 20 words remaining)

### Exercise 3.3: Task Parameter Passing

**Objective:** Create 4 LED tasks from one function using different parameters.

```c
/*
 * Complete this exercise:
 * 1. Define a struct with GPIO port, pin, and blink period
 * 2. Create 4 static configurations
 * 3. Create 4 tasks using the same function
 * 4. Verify each LED blinks at its own rate
 */

/* YOUR CODE HERE */
```

<details>
<summary><strong>Solution</strong></summary>

```c
typedef struct {
    GPIO_TypeDef *port;
    uint16_t      pin;
    uint32_t      period_ms;
} BlinkConfig_t;

static BlinkConfig_t configs[4] = {
    { GPIOA, GPIO_PIN_5,  100 },  /* Fast */
    { GPIOA, GPIO_PIN_6,  250 },  /* Medium-fast */
    { GPIOA, GPIO_PIN_7,  500 },  /* Medium */
    { GPIOB, GPIO_PIN_0, 1000 },  /* Slow */
};

void Task_Blink(void *pvParameters)
{
    BlinkConfig_t *cfg = (BlinkConfig_t *)pvParameters;

    for (;;)
    {
        HAL_GPIO_TogglePin(cfg->port, cfg->pin);
        vTaskDelay(pdMS_TO_TICKS(cfg->period_ms));
    }
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();

    for (int i = 0; i < 4; i++)
    {
        char name[12];
        snprintf(name, sizeof(name), "LED%d", i);
        xTaskCreate(Task_Blink, name, 128, &configs[i], 1, NULL);
    }

    vTaskStartScheduler();
    for (;;);
}
```

</details>

---

## 3.9 Mini-Project: Task Monitor System

### Requirements

Build a system that:
1. Creates 3 worker tasks (different priorities)
2. A monitor task that periodically reports:
   - Each task's state
   - Each task's stack high-water mark
   - Total free heap memory
   - System uptime in seconds
3. Output is sent over UART to a serial terminal

### Full Annotated Code

```c
#include "FreeRTOS.h"
#include "task.h"
#include <stdio.h>
#include <string.h>

/* Task handles for monitoring */
TaskHandle_t xTask1Handle, xTask2Handle, xTask3Handle;

/*──────────── Worker Tasks ────────────*/

/* Task 1: Simulates sensor reading (high priority) */
void Task_Sensor(void *pvParameters)
{
    for (;;)
    {
        /* Simulate sensor read: toggle GPIO, small delay */
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);

        /* Simulate variable processing time */
        volatile uint32_t work;
        for (work = 0; work < 5000; work++);

        vTaskDelay(pdMS_TO_TICKS(50));
    }
}

/* Task 2: Simulates display update (medium priority) */
void Task_Display(void *pvParameters)
{
    uint8_t frame_buffer[128];  /* 128 bytes on stack */

    for (;;)
    {
        memset(frame_buffer, 0, sizeof(frame_buffer));
        /* Simulate rendering */
        volatile uint32_t work;
        for (work = 0; work < 20000; work++);

        vTaskDelay(pdMS_TO_TICKS(200));
    }
}

/* Task 3: Simulates data logging (low priority) */
void Task_Logger(void *pvParameters)
{
    for (;;)
    {
        /* Simulate writing to storage */
        volatile uint32_t work;
        for (work = 0; work < 50000; work++);

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

/*──────────── Monitor Task ────────────*/

static const char *state_to_string(eTaskState state)
{
    switch (state)
    {
        case eRunning:   return "RUNNING";
        case eReady:     return "READY  ";
        case eBlocked:   return "BLOCKED";
        case eSuspended: return "SUSPEND";
        case eDeleted:   return "DELETED";
        default:         return "UNKNOWN";
    }
}

void Task_Monitor(void *pvParameters)
{
    char buf[256];
    TickType_t start_tick = xTaskGetTickCount();

    for (;;)
    {
        /* Clear terminal */
        const char *cls = "\033[2J\033[H";  /* ANSI escape: clear screen */
        HAL_UART_Transmit(&huart2, (uint8_t *)cls, strlen(cls), 100);

        /* Header */
        snprintf(buf, sizeof(buf),
                 "=== RTOS Task Monitor ===\r\n"
                 "Uptime: %lu s | Free Heap: %lu bytes\r\n"
                 "──────────────────────────────────────\r\n"
                 "%-10s %-8s %-5s %-10s\r\n"
                 "──────────────────────────────────────\r\n",
                 (unsigned long)((xTaskGetTickCount() - start_tick) / configTICK_RATE_HZ),
                 (unsigned long)xPortGetFreeHeapSize(),
                 "TASK", "STATE", "PRI", "STACK FREE");
        HAL_UART_Transmit(&huart2, (uint8_t *)buf, strlen(buf), 500);

        /* Task info */
        struct { TaskHandle_t handle; const char *name; } tasks[] = {
            { xTask1Handle, "Sensor" },
            { xTask2Handle, "Display" },
            { xTask3Handle, "Logger" },
        };

        for (int i = 0; i < 3; i++)
        {
            snprintf(buf, sizeof(buf), "%-10s %-8s %-5lu %-10lu\r\n",
                     tasks[i].name,
                     state_to_string(eTaskGetState(tasks[i].handle)),
                     (unsigned long)uxTaskPriorityGet(tasks[i].handle),
                     (unsigned long)uxTaskGetStackHighWaterMark(tasks[i].handle) * 4);
            HAL_UART_Transmit(&huart2, (uint8_t *)buf, strlen(buf), 500);
        }

        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}

/*──────────── Main ────────────*/

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_USART2_UART_Init();

    xTaskCreate(Task_Sensor,  "Sensor",  256, NULL, 3, &xTask1Handle);
    xTaskCreate(Task_Display, "Display", 256, NULL, 2, &xTask2Handle);
    xTaskCreate(Task_Logger,  "Logger",  256, NULL, 1, &xTask3Handle);
    xTaskCreate(Task_Monitor, "Monitor", 512, NULL, 4, NULL);  /* Highest: always runs */

    vTaskStartScheduler();
    for (;;);
}

/*
 * Expected UART output (updates every 2 seconds):
 *
 * === RTOS Task Monitor ===
 * Uptime: 12 s | Free Heap: 3456 bytes
 * ──────────────────────────────────────
 * TASK       STATE    PRI   STACK FREE
 * ──────────────────────────────────────
 * Sensor     BLOCKED  3     712
 * Display    BLOCKED  2     424
 * Logger     BLOCKED  1     852
 */
```

### Debugging Strategy

1. If no UART output: Check `SystemClock_Config()` and UART baud rate
2. If stack free values are very low: Increase `usStackDepth` for that task
3. If free heap is 0: Increase `configTOTAL_HEAP_SIZE` or reduce stack sizes
4. If a task shows RUNNING state: Unlikely in the monitor (it only prints when others are blocked); check priorities

### Extensions

1. Add a button-press counter task and display in the monitor
2. Add color coding (ANSI escape codes) for stack warnings (red if < 100 bytes free)
3. Add CPU usage percentage using `configGENERATE_RUN_TIME_STATS`

---

## 3.10 Chapter Summary

| Concept | Key Takeaway |
|---|---|
| **xTaskCreate** | Creates a task; stack size in words, not bytes |
| **xTaskCreateStatic** | Static allocation; preferred for safety-critical |
| **vTaskDelete** | Deletes a task; idle task frees memory |
| **Task states** | Running, Ready, Blocked, Suspended |
| **TCB** | Kernel's per-task data structure (~100 bytes) |
| **pxCurrentTCB** | Pointer to the running task's TCB |
| **Stack per task** | Each task has its own stack; grows downward on ARM |
| **Stack sizing** | Start large, measure high-water mark, add margin |
| **Stack overflow** | #1 cause of crashes; always enable detection |
| **Idle task** | Priority 0; frees deleted task memory |

---

## 3.11 What's Next

In **Chapter 4: FreeRTOS on STM32 — Setup**, we will:
- Walk through STM32CubeIDE installation and project creation
- Integrate FreeRTOS step by step
- Configure SysTick, heap, and priorities
- Create and run your first FreeRTOS project on real hardware

---

*End of Chapter 3*
