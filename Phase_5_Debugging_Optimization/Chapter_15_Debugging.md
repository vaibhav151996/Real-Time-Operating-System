# Chapter 15: Debugging RTOS Applications

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Detect, diagnose, and fix stack overflow bugs
2. Analyze HardFault exceptions and identify their root cause
3. Use FreeRTOS-aware debugging tools in STM32CubeIDE
4. Implement systematic debugging strategies for RTOS applications

---

## 15.1 Stack Overflow

### 15.1.1 Why Stack Overflow Is the #1 RTOS Bug

Stack overflow corrupts adjacent memory — this could be another task's stack, a TCB, kernel data, or heap metadata. The symptoms are **non-deterministic**: random crashes, data corruption, HardFaults that appear unrelated to the actual cause.

### 15.1.2 Detection Methods

```c
/* Method 1: Check stack pointer at context switch */
#define configCHECK_FOR_STACK_OVERFLOW    1

/* Method 2: Pattern checking (more reliable) */
#define configCHECK_FOR_STACK_OVERFLOW    2
/* Last 20 bytes filled with 0xA5A5A5A5 at task creation.
   Checked at every context switch. If corrupted → overflow detected. */

/* The hook YOU must implement: */
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName)
{
    /* WARNING: Stack is already corrupted at this point! */
    
    /* Use direct register writes — HAL may need stack space */
    GPIOA->BSRR = GPIO_PIN_5;    /* Turn on error LED */
    
    /* Store task name for debugger inspection */
    volatile char *fault_task = pcTaskName;
    (void)fault_task;
    
    taskDISABLE_INTERRUPTS();
    for (;;);    /* Halt — connect debugger to inspect */
}
```

### 15.1.3 Monitoring Stack Usage

```c
/* Runtime stack monitoring — add to a diagnostic task */
void Task_StackMonitor(void *pvParameters)
{
    TaskHandle_t tasks[] = { xTask1, xTask2, xTask3 };
    const char *names[] = { "Task1", "Task2", "Task3" };
    
    for (;;)
    {
        for (int i = 0; i < 3; i++)
        {
            UBaseType_t hwm = uxTaskGetStackHighWaterMark(tasks[i]);
            uint32_t free_bytes = hwm * sizeof(StackType_t);
            
            char msg[64];
            snprintf(msg, sizeof(msg), "%s: %lu bytes free\r\n",
                     names[i], (unsigned long)free_bytes);
            HAL_UART_Transmit(&huart2, (uint8_t *)msg, strlen(msg), 100);
            
            if (free_bytes < 64)
            {
                /* DANGER: Less than 64 bytes remaining! */
                snprintf(msg, sizeof(msg), "*** WARNING: %s near overflow! ***\r\n",
                         names[i]);
                HAL_UART_Transmit(&huart2, (uint8_t *)msg, strlen(msg), 100);
            }
        }
        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}
```

### 15.1.4 Common Causes of Stack Overflow

| Cause | Example | Fix |
|---|---|---|
| **Large local arrays** | `char buf[1024]` in task | Use static/global or heap |
| **Deep call chains** | `A() → B() → C() → D() → E()` | Flatten or increase stack |
| **Recursive functions** | `factorial(1000)` | Use iterative version |
| **printf/sprintf** | Can use 200+ bytes of stack | Use snprintf with small buffer |
| **HAL functions** | Some HAL calls use significant stack | Increase stack size |
| **FPU context save** | Cortex-M4F saves 32 extra registers | Add 128 bytes to stack |

---

## 15.2 HardFault Analysis

### 15.2.1 What Causes HardFaults?

```
COMMON HARDFAULT CAUSES IN RTOS:
═════════════════════════════════

1. NULL pointer dereference      → Reading/writing to address 0x00000000
2. Stack overflow                → SP goes below stack boundary
3. Unaligned memory access       → 32-bit access to odd address
4. Invalid function pointer      → Task function pointer is NULL or corrupt
5. Wrong interrupt priority      → ISR at priority < configMAX_SYSCALL
6. Division by zero              → If div-by-zero trap is enabled
7. Execute from invalid address  → Branch to non-code memory
8. MPU violation                 → Access to protected region
```

### 15.2.2 HardFault Handler for Debugging

```c
/*
 * Enhanced HardFault handler that captures register state.
 * Place this in your exception handlers file.
 */

/* This function is called from the assembly wrapper below */
void HardFault_Handler_C(uint32_t *pulFaultStackAddress)
{
    /* These contain the register values at the time of the fault */
    volatile uint32_t r0  = pulFaultStackAddress[0];
    volatile uint32_t r1  = pulFaultStackAddress[1];
    volatile uint32_t r2  = pulFaultStackAddress[2];
    volatile uint32_t r3  = pulFaultStackAddress[3];
    volatile uint32_t r12 = pulFaultStackAddress[4];
    volatile uint32_t lr  = pulFaultStackAddress[5];  /* Return address */
    volatile uint32_t pc  = pulFaultStackAddress[6];  /* Faulting instruction */
    volatile uint32_t psr = pulFaultStackAddress[7];

    /* Fault status registers */
    volatile uint32_t cfsr  = SCB->CFSR;   /* Configurable Fault Status */
    volatile uint32_t hfsr  = SCB->HFSR;   /* HardFault Status */
    volatile uint32_t mmfar = SCB->MMFAR;  /* MemManage Fault Address */
    volatile uint32_t bfar  = SCB->BFAR;   /* Bus Fault Address */

    /* Which task was running? */
    volatile char *task_name = pcTaskGetName(NULL);

    /* Prevent optimizer from removing the variables */
    (void)r0; (void)r1; (void)r2; (void)r3;
    (void)r12; (void)lr; (void)pc; (void)psr;
    (void)cfsr; (void)hfsr; (void)mmfar; (void)bfar;
    (void)task_name;

    /*
     * SET A BREAKPOINT HERE.
     * In the debugger, inspect:
     *   pc    → The instruction that caused the fault
     *   lr    → Where the fault was called from
     *   cfsr  → Why the fault occurred
     *   task_name → Which FreeRTOS task faulted
     *
     * Look up the 'pc' value in the .map file or use
     * "addr2line -e firmware.elf <pc_value>" to find
     * the source code line.
     */
    __BKPT(0);    /* Trigger debugger breakpoint */
    for (;;);
}

/* Assembly wrapper to extract the correct stack pointer */
__attribute__((naked)) void HardFault_Handler(void)
{
    __asm volatile(
        " TST LR, #4          \n"    /* Test bit 2 of LR (EXC_RETURN) */
        " ITE EQ              \n"
        " MRSEQ R0, MSP       \n"    /* If 0: fault used MSP (in handler mode) */
        " MRSNE R0, PSP       \n"    /* If 1: fault used PSP (in task mode) */
        " B HardFault_Handler_C \n"  /* Branch to C handler with stack pointer */
    );
}
```

### 15.2.3 Decoding the CFSR Register

```
CFSR (Configurable Fault Status Register) = SCB->CFSR
═══════════════════════════════════════════════════════

Bits [7:0]   = MMFSR (MemManage Fault Status)
Bits [15:8]  = BFSR  (Bus Fault Status)
Bits [31:16] = UFSR  (Usage Fault Status)

Common bit meanings:
  CFSR & 0x0001  = IACCVIOL  → Instruction access violation
  CFSR & 0x0002  = DACCVIOL  → Data access violation
  CFSR & 0x0080  = MMARVALID → MMFAR holds the faulting address
  CFSR & 0x0100  = IBUSERR   → Instruction bus error
  CFSR & 0x0200  = PRECISERR → Precise data bus error
  CFSR & 0x0400  = IMPRECISERR → Imprecise data bus error (hard to locate)
  CFSR & 0x0800  = UNSTKERR  → Stacking error (stack overflow!)
  CFSR & 0x8000  = BFARVALID → BFAR holds the faulting address
  CFSR & 0x010000 = UNDEFINSTR → Undefined instruction
  CFSR & 0x020000 = INVSTATE  → Invalid state (Thumb bit not set)
  CFSR & 0x040000 = INVPC     → Invalid PC load
  CFSR & 0x080000 = NOCP      → No coprocessor
  CFSR & 0x01000000 = UNALIGNED → Unaligned access
  CFSR & 0x02000000 = DIVBYZERO → Division by zero
```

---

## 15.3 FreeRTOS-Aware Debugging

### 15.3.1 STM32CubeIDE RTOS Plugin

```
ENABLING FREERTOS AWARENESS IN STM32CubeIDE:
═════════════════════════════════════════════

1. Window → Show View → FreeRTOS
   - Task List: Shows all tasks, their states, priorities, stack usage
   - Queue List: Shows all queues, messages waiting, free space
   - Semaphore List: Shows semaphores and mutexes
   - Timer List: Shows software timers
   - Heap Usage: Shows heap statistics

2. Debug Configuration → Debugger tab:
   - Enable "Thread-aware debugging" (if available)
   
3. In the debugger:
   - Variables view: Watch pxCurrentTCB for the running task
   - Expressions: Add uxCurrentNumberOfTasks
   - Memory view: Inspect task stacks for 0xA5A5A5A5 pattern
```

### 15.3.2 configASSERT — Your First Line of Defense

```c
/* ESSENTIAL: Define this in FreeRTOSConfig.h */
#define configASSERT(x) do { \
    if ((x) == 0) { \
        taskDISABLE_INTERRUPTS(); \
        __BKPT(0); \
        for(;;); \
    } \
} while(0)

/* FreeRTOS uses configASSERT extensively to check:
 * - NULL pointers
 * - Invalid priorities
 * - Wrong interrupt priority for FromISR calls
 * - API called from wrong context (ISR vs Task)
 * - Stack pointer still valid
 *
 * With configASSERT defined, bugs are caught EARLY
 * with a breakpoint at the exact line of the error.
 * Without it, bugs cause SILENT CORRUPTION and
 * manifest as random crashes hours later.
 */
```

---

## 15.4 Systematic Debugging Methodology

```
RTOS BUG DIAGNOSIS FLOWCHART:
═════════════════════════════

System crashes / misbehaves
         │
         ▼
    HardFault?
    ├─ YES → Read pc (faulting instruction)
    │        Read cfsr (fault type)
    │        Read task_name (which task)
    │        └─ Stack overflow? → Increase stack
    │        └─ NULL pointer?  → Fix the pointer
    │        └─ Invalid ISR?   → Fix priority
    │
    ├─ NO → Task not running?
    │       ├─ Check vTaskList() output
    │       ├─ Task BLOCKED? → On what? (Queue? Semaphore?)
    │       ├─ Task SUSPENDED? → Who suspended it?
    │       ├─ Deadlock? → Check mutex ordering
    │       └─ Starvation? → Check priorities
    │
    └─ NO → Wrong behavior (not crash)?
            ├─ Race condition? → Missing mutex
            ├─ Data corruption? → Stack overflow (another task!)
            ├─ Timing wrong? → Check tick configuration
            └─ Queue full/empty? → Check queue sizing
```

---

## 15.5 Chapter Summary

| Concept | Key Takeaway |
|---|---|
| **Stack overflow** | #1 bug; enable detection (method 2); monitor high-water marks |
| **HardFault handler** | Capture registers (pc, lr, cfsr) to identify root cause |
| **CFSR register** | Tells you WHY the fault occurred |
| **configASSERT** | Must define; catches errors at the source |
| **RTOS-aware debugging** | Shows task states, queues, semaphores in debugger |

---

*End of Chapter 15*
