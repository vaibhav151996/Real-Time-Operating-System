# Chapter 10: Event Groups

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Explain what event groups are and when to use them over other IPC
2. Set, clear, and wait for event bits
3. Implement multi-condition synchronization (AND/OR)
4. Use event groups for task rendezvous (barrier synchronization)

---

## 10.1 What Are Event Groups?

### 10.1.1 The Problem: Waiting for Multiple Conditions

Sometimes a task needs to wait for **combination** of events:
- "Wait until sensor data is ready **AND** the display is free"
- "Wake up when **either** a button is pressed **OR** a timer expires"

Queues and semaphores are **one-to-one**. Event groups allow **many-to-one** synchronization.

```
EVENT GROUP (24 usable bits):
═════════════════════════════

Bit:  23 22 21 ... 7  6  5  4  3  2  1  0
      [0][0][0]   [0][0][1][0][1][0][1][0]
                       │     │     │
                 SENSOR_READY │  BUTTON_PRESSED
                         DMA_COMPLETE

Multiple tasks or ISRs SET bits.
One or more tasks WAIT for specific BIT PATTERNS.
```

### 10.1.2 Creating and Using Event Groups

```c
#include "event_groups.h"

/* Define event bits */
#define EVT_SENSOR_READY    (1 << 0)   /* Bit 0 */
#define EVT_BUTTON_PRESSED  (1 << 1)   /* Bit 1 */
#define EVT_DMA_COMPLETE    (1 << 2)   /* Bit 2 */
#define EVT_UART_RX         (1 << 3)   /* Bit 3 */

EventGroupHandle_t xEvents = xEventGroupCreate();

/* SETTER: Set bits when events occur */
void Task_Sensor(void *pv)
{
    for (;;)
    {
        Read_Sensor();
        xEventGroupSetBits(xEvents, EVT_SENSOR_READY);
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

/* WAITER: Wait for specific combination */
void Task_Main(void *pv)
{
    for (;;)
    {
        /* Wait for BOTH sensor AND DMA (AND logic) */
        EventBits_t bits = xEventGroupWaitBits(
            xEvents,
            EVT_SENSOR_READY | EVT_DMA_COMPLETE,   /* Bits to wait for */
            pdTRUE,                                  /* Clear bits on exit */
            pdTRUE,                                  /* Wait for ALL (AND) */
            portMAX_DELAY                            /* Wait forever */
        );

        /* Both events have occurred! */
        if (bits & EVT_SENSOR_READY) { /* Sensor data available */ }
        if (bits & EVT_DMA_COMPLETE) { /* DMA transfer done */ }
    }
}
```

### 10.1.3 Wait Modes: AND vs. OR

```c
/* AND: Wait until ALL specified bits are set */
xEventGroupWaitBits(xEvents,
    EVT_SENSOR_READY | EVT_DMA_COMPLETE,
    pdTRUE,     /* Clear on exit */
    pdTRUE,     /* ← AND: wait for ALL bits */
    portMAX_DELAY);

/* OR: Wait until ANY specified bit is set */
xEventGroupWaitBits(xEvents,
    EVT_BUTTON_PRESSED | EVT_UART_RX | EVT_SENSOR_READY,
    pdTRUE,     /* Clear on exit */
    pdFALSE,    /* ← OR: wait for ANY bit */
    portMAX_DELAY);
```

### 10.1.4 Setting Bits from ISR

```c
void DMA1_Stream0_IRQHandler(void)
{
    BaseType_t xWoken = pdFALSE;

    HAL_DMA_IRQHandler(&hdma);

    /* Set event bit from ISR */
    xEventGroupSetBitsFromISR(xEvents, EVT_DMA_COMPLETE, &xWoken);
    portYIELD_FROM_ISR(xWoken);
}

/* NOTE: xEventGroupSetBitsFromISR requires configUSE_TIMERS = 1
   because it sends a command through the timer daemon task. */
```

---

## 10.2 Task Synchronization (Rendezvous)

### 10.2.1 The Barrier Pattern

Wait for multiple tasks to reach the same point before any proceed:

```c
#define TASK_A_BIT   (1 << 0)
#define TASK_B_BIT   (1 << 1)
#define TASK_C_BIT   (1 << 2)
#define ALL_TASKS    (TASK_A_BIT | TASK_B_BIT | TASK_C_BIT)

void Task_A(void *pv)
{
    /* Do initialization... */
    Init_Subsystem_A();

    /* Signal ready and wait for ALL tasks */
    xEventGroupSync(xEvents, TASK_A_BIT, ALL_TASKS, portMAX_DELAY);

    /* All tasks are here — start working together */
    for (;;) { /* ... */ }
}

/* Task_B and Task_C do the same with their respective bits */
```

```
RENDEZVOUS TIMING:
══════════════════

Task A: ██ Init ██│──── WAIT ──── │ All ready! ████
Task B: ██████ Init ██████│ WAIT │ All ready! ████
Task C: ████ Init ████│── WAIT ──│ All ready! ████
                                  ↑
                          All three tasks
                          synchronized here
```

---

## 10.3 Event Groups vs. Other IPC

| Feature | Event Group | Semaphore | Queue | Notification |
|---|---|---|---|---|
| **Data transfer** | No (bits only) | No | Yes | 32-bit value |
| **Multiple conditions** | Yes (AND/OR) | No | No | Limited |
| **Multiple waiters** | Yes | Yes | Yes | No (1 receiver) |
| **ISR usage** | SetBitsFromISR | GiveFromISR | SendFromISR | GiveFromISR |
| **RAM per instance** | ~28 bytes | ~88 bytes | 76 + data | 0 (in TCB) |
| **Best for** | Multi-event sync | Signaling | Data passing | Fast 1:1 signal |

---

## 10.4 Common Mistakes (Chapter 10)

### Mistake 1: Using Top 8 Bits
```c
/* On 32-bit systems, only bits 0-23 are available for events.
   Bits 24-31 are used internally by FreeRTOS. */
#define BAD_BIT  (1 << 25)    /* RESERVED — don't use! */
```

### Mistake 2: Not Clearing Bits
```c
/* If pdTRUE is passed for xClearOnExit, bits are auto-cleared.
   If pdFALSE, you must clear manually: */
xEventGroupClearBits(xEvents, EVT_SENSOR_READY);
```

---

## 10.5 Chapter Summary

| Concept | Key Takeaway |
|---|---|
| **Event group** | Bit-field for multi-condition synchronization |
| **WaitBits AND** | Blocks until ALL specified bits are set |
| **WaitBits OR** | Blocks until ANY specified bit is set |
| **Sync** | Barrier/rendezvous — all tasks meet at a point |
| **Usable bits** | 0–23 on 32-bit systems (24 bits) |
| **From ISR** | xEventGroupSetBitsFromISR (requires timer daemon) |
| **Best use case** | Multi-condition waits; task synchronization barriers |

---

*End of Chapter 10*
