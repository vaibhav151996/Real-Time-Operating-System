# Chapter 9: Mutex

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Explain why mutexes exist and how they differ from semaphores
2. Implement mutual exclusion with priority inheritance
3. Identify and prevent priority inversion
4. Detect and avoid deadlocks
5. Use recursive mutexes when needed

---

## 9.1 Mutual Exclusion — The Core Problem

### 9.1.1 What Is a Critical Section?

A **critical section** is a region of code that accesses a **shared resource** (global variable, peripheral, data structure) that must not be accessed by two tasks simultaneously.

```c
/* SHARED RESOURCE: global variable */
uint32_t shared_counter = 0;

/* Task A increments it */
void Task_A(void *pv)
{
    for (;;)
    {
        shared_counter++;   /* READ → INCREMENT → WRITE */
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}

/* Task B also increments it */
void Task_B(void *pv)
{
    for (;;)
    {
        shared_counter++;   /* READ → INCREMENT → WRITE */
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}

/*
 * RACE CONDITION:
 * shared_counter++ is NOT atomic on ARM. It compiles to:
 *   LDR R0, [shared_counter]   ← Read current value
 *   ADD R0, R0, #1             ← Increment
 *   STR R0, [shared_counter]   ← Write back
 *
 * If Task A is preempted BETWEEN the LDR and STR by Task B:
 *   Task A: LDR R0 = 5        (reads 5)
 *           <<<< PREEMPTED >>>>
 *   Task B: LDR R0 = 5        (reads 5)
 *   Task B: ADD R0 = 6
 *   Task B: STR = 6           (writes 6)
 *           <<<< TASK A RESUMES >>>>
 *   Task A: ADD R0 = 6        (still using old value 5!)
 *   Task A: STR = 6           (writes 6 — should be 7!)
 *
 * RESULT: One increment is LOST. Counter shows 6 instead of 7.
 */
```

### 9.1.2 Mutex Solution

```c
SemaphoreHandle_t xMutex = xSemaphoreCreateMutex();

void Task_A(void *pv)
{
    for (;;)
    {
        xSemaphoreTake(xMutex, portMAX_DELAY);  /* LOCK */
        shared_counter++;                         /* Safe! */
        xSemaphoreGive(xMutex);                   /* UNLOCK */
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}

void Task_B(void *pv)
{
    for (;;)
    {
        xSemaphoreTake(xMutex, portMAX_DELAY);  /* LOCK (blocks if A holds it) */
        shared_counter++;                         /* Safe! */
        xSemaphoreGive(xMutex);                   /* UNLOCK */
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

---

## 9.2 Priority Inheritance

### 9.2.1 The Priority Inversion Problem

```
PRIORITY INVERSION (The Classic Problem):
═════════════════════════════════════════

Tasks: H (High, pri 3), M (Medium, pri 2), L (Low, pri 1)
Shared resource protected by a mutex

Time ──────────────────────────────────────────────────►

L: ████│MUTEX LOCKED│██████████████████████████│UNLOCK│
                     │                          │
M:                   │█████████████████████████│  ← M preempts L!
                     │ M runs for a LONG time  │
                     │                          │
H:         ████│WAIT FOR MUTEX│░░░░░░░░░░░░░░░░░░░░░░│RUN
                ↑                                      ↑
           H needs mutex                          Finally gets it
           but L holds it!                        after M finishes!

PROBLEM:
  1. L takes mutex
  2. H becomes ready, preempts L, tries to take mutex → BLOCKED
  3. M becomes ready, preempts L (M > L priority)
  4. M runs for a long time — H is STILL blocked!
  5. H (highest priority) is effectively blocked by M (medium priority)

H's response time is determined by M's execution time — UNBOUNDED!
This is "priority inversion" — a high-priority task is blocked
by a lower-priority task that doesn't even hold the resource.
```

### 9.2.2 How Priority Inheritance Fixes It

```
PRIORITY INHERITANCE:
═════════════════════

Time ──────────────────────────────────────────────────►

L: ████│MUTEX│██│BOOSTED to pri 3│██│UNLOCK│──► back to pri 1
                │                 │
M:              │  CANNOT run!    │  ← L now has priority 3
                │  L is at pri 3  │     M (pri 2) can't preempt
                │                 │
H:    ████│WAIT│░░░░░░░░░░░░░░░░│RUN │████████████
           ↑                      ↑
     H wants mutex           L finishes quickly
     L holds it              H gets mutex fast!
     
FreeRTOS AUTOMATICALLY:
  1. L takes mutex
  2. H tries to take mutex → BLOCKED
  3. Kernel sees: H (pri 3) is blocked by L (pri 1)
  4. Kernel BOOSTS L's priority to 3 (H's priority)
  5. Now M (pri 2) cannot preempt L
  6. L finishes critical section, gives mutex
  7. L's priority is restored to 1
  8. H runs immediately (highest priority ready task)
```

> **🔑 Key Insight:** Priority inheritance is **automatic** in FreeRTOS mutexes. This is the primary reason to use a mutex instead of a binary semaphore for locking.

### 9.2.3 Mutex Creation

```c
/* Standard mutex — includes priority inheritance */
SemaphoreHandle_t xMutex = xSemaphoreCreateMutex();

/* The mutex uses the same API as semaphores: */
xSemaphoreTake(xMutex, timeout);    /* Lock */
xSemaphoreGive(xMutex);             /* Unlock */

/* BUT: Only the OWNER (task that took it) should give it.
   The kernel enforces this for recursive mutexes. */
```

---

## 9.3 Deadlocks

### 9.3.1 What Is a Deadlock?

A **deadlock** occurs when two or more tasks are each waiting for a resource held by the other, creating a circular dependency. Neither can proceed.

```
DEADLOCK:
═════════

Task A holds Mutex 1, wants Mutex 2
Task B holds Mutex 2, wants Mutex 1

  Task A                    Task B
  ──────                    ──────
  Take(Mutex1) ✓            Take(Mutex2) ✓
  Take(Mutex2) → BLOCKED   Take(Mutex1) → BLOCKED
       ↑                         │
       └─────────────────────────┘
       CIRCULAR DEPENDENCY!
       Both tasks wait FOREVER.

Analogy: Two people in a narrow hallway.
  Person A: "You go first."
  Person B: "No, YOU go first."
  Neither moves. Ever.
```

### 9.3.2 Preventing Deadlocks

**Strategy 1: Lock Ordering (Most common and effective)**
```c
/*
 * ALWAYS acquire mutexes in the SAME ORDER, everywhere.
 * Convention: Lock by address (lower address first)
 * or by a defined ordering (Mutex1 before Mutex2).
 */

/* WRONG: Different order in different tasks */
/* Task A: */ xSemaphoreTake(xMutex1, ...); xSemaphoreTake(xMutex2, ...);
/* Task B: */ xSemaphoreTake(xMutex2, ...); xSemaphoreTake(xMutex1, ...);
/* ^^^ DEADLOCK POSSIBLE ^^^ */

/* RIGHT: Same order everywhere */
/* Task A: */ xSemaphoreTake(xMutex1, ...); xSemaphoreTake(xMutex2, ...);
/* Task B: */ xSemaphoreTake(xMutex1, ...); xSemaphoreTake(xMutex2, ...);
/* Both tasks lock 1 → 2. No circular dependency possible. */
```

**Strategy 2: Timeout (Detection and recovery)**
```c
/* Use a timeout instead of portMAX_DELAY */
if (xSemaphoreTake(xMutex2, pdMS_TO_TICKS(100)) == pdFALSE)
{
    /* Could not acquire in 100 ms — possible deadlock! */
    xSemaphoreGive(xMutex1);   /* Release what we hold */
    Log_Warning("Deadlock averted");
    vTaskDelay(pdMS_TO_TICKS(10));  /* Back off and retry */
}
```

**Strategy 3: Single Mutex (Simplify)**
```c
/* Instead of two mutexes, use ONE mutex for a group of related resources */
xSemaphoreTake(xResourceGroupMutex, portMAX_DELAY);
Access_Resource_A();
Access_Resource_B();
xSemaphoreGive(xResourceGroupMutex);
/* No deadlock possible — only one mutex */
```

---

## 9.4 Recursive Mutexes

### 9.4.1 The Problem: Nested Locking

```c
/* A function that takes a mutex */
void Safe_Write(uint8_t *data, uint16_t len)
{
    xSemaphoreTake(xUartMutex, portMAX_DELAY);
    HAL_UART_Transmit(&huart2, data, len, 100);
    xSemaphoreGive(xUartMutex);
}

/* Another function that also takes the SAME mutex */
void Safe_WriteFormatted(const char *fmt, ...)
{
    char buf[128];
    va_list args;
    va_start(args, fmt);
    vsnprintf(buf, sizeof(buf), fmt, args);
    va_end(args);

    xSemaphoreTake(xUartMutex, portMAX_DELAY);   /* DEADLOCK! */
    /* Already holding the mutex if called from Safe_Write! */
    HAL_UART_Transmit(&huart2, (uint8_t *)buf, strlen(buf), 100);
    xSemaphoreGive(xUartMutex);
}
```

### 9.4.2 Solution: Recursive Mutex

A recursive mutex can be taken **multiple times** by the **same task**. It must be given the same number of times.

```c
/* Create recursive mutex */
SemaphoreHandle_t xRecMutex = xSemaphoreCreateRecursiveMutex();

/* Use recursive variants */
xSemaphoreTakeRecursive(xRecMutex, portMAX_DELAY);   /* count = 1 */
xSemaphoreTakeRecursive(xRecMutex, portMAX_DELAY);   /* count = 2 (same task, OK!) */
/* ... */
xSemaphoreGiveRecursive(xRecMutex);                   /* count = 1 */
xSemaphoreGiveRecursive(xRecMutex);                   /* count = 0 (released) */
```

> **📝 Beginner Note:** Needing a recursive mutex often indicates a design issue. Refactor so functions don't need to re-lock the same resource. Use it only when code structure makes it unavoidable (e.g., library calls).

---

## 9.5 Mutex Design Best Practices

### 9.5.1 The Guard Pattern

```c
/*
 * Encapsulate mutex with the resource it protects.
 * Other modules NEVER touch the mutex directly.
 */
/* uart_guard.h */
typedef struct {
    UART_HandleTypeDef *huart;
    SemaphoreHandle_t  mutex;
} UART_Guard_t;

void UART_Guard_Init(UART_Guard_t *guard, UART_HandleTypeDef *huart);
void UART_Guard_Send(UART_Guard_t *guard, const uint8_t *data, uint16_t len);

/* uart_guard.c */
void UART_Guard_Init(UART_Guard_t *guard, UART_HandleTypeDef *huart)
{
    guard->huart = huart;
    guard->mutex = xSemaphoreCreateMutex();
    configASSERT(guard->mutex != NULL);
}

void UART_Guard_Send(UART_Guard_t *guard, const uint8_t *data, uint16_t len)
{
    xSemaphoreTake(guard->mutex, portMAX_DELAY);
    HAL_UART_Transmit(guard->huart, data, len, 1000);
    xSemaphoreGive(guard->mutex);
}

/* Usage — other tasks just call the guarded function */
UART_Guard_t uart2_guard;

void Task_Logger(void *pv)
{
    for (;;) {
        UART_Guard_Send(&uart2_guard, (uint8_t *)"Log msg\r\n", 9);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

### 9.5.2 Hold Time Minimization

```c
/* WRONG: Holding mutex during long operation */
xSemaphoreTake(xMutex, portMAX_DELAY);
Compute_FFT(data, 1024);              /* 5 ms! */
shared_result = result;
xSemaphoreGive(xMutex);

/* RIGHT: Do computation outside mutex, only lock for the copy */
Compute_FFT(data, 1024);              /* No lock needed */
xSemaphoreTake(xMutex, portMAX_DELAY);
shared_result = result;                /* < 1 μs */
xSemaphoreGive(xMutex);
```

---

## 9.6 Common Mistakes (Chapter 9)

### Mistake 1: Giving a Mutex You Don't Own
```c
/* Task A takes, Task B gives — WRONG! */
/* Only the owner should give the mutex. */
/* FreeRTOS doesn't strictly enforce this for standard mutexes,
   but it breaks priority inheritance logic. */
```

### Mistake 2: Using Mutex from ISR
```c
/* NEVER use xSemaphoreTake/Give on a mutex from ISR.
   Mutexes use priority inheritance, which is meaningless in ISR context.
   Use binary semaphore for ISR signaling instead. */
```

### Mistake 3: Forgetting to Give the Mutex (Leak)
```c
/* If an error path skips xSemaphoreGive, the mutex stays locked FOREVER */
xSemaphoreTake(xMutex, portMAX_DELAY);
if (error_condition)
{
    return;   /* MUTEX LEAKED! Every other task blocks forever! */
}
xSemaphoreGive(xMutex);

/* FIX: Always give before returning */
xSemaphoreTake(xMutex, portMAX_DELAY);
if (error_condition)
{
    xSemaphoreGive(xMutex);   /* Release before return */
    return;
}
xSemaphoreGive(xMutex);
```

---

## 9.7 Chapter Summary

| Concept | Key Takeaway |
|---|---|
| **Mutex** | Mutual exclusion lock with priority inheritance |
| **Priority inversion** | High-priority task blocked by low, via medium |
| **Priority inheritance** | Low-priority holder is boosted to waiter's priority |
| **Deadlock** | Circular wait on mutexes — tasks blocked forever |
| **Prevention** | Lock ordering, timeouts, single mutex |
| **Recursive mutex** | Same task can lock multiple times; must unlock same count |
| **Best practice** | Minimize hold time; encapsulate mutex with resource |

---

*End of Chapter 9*
