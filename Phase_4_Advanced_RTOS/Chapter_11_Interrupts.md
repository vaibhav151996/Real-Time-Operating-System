# Chapter 11: Interrupts and RTOS

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Explain the rules for using FreeRTOS APIs from ISRs
2. Correctly use all `FromISR` API variants
3. Configure interrupt priorities for FreeRTOS on Cortex-M
4. Implement the deferred interrupt processing pattern
5. Avoid the most dangerous ISR-related bugs

---

## 11.1 ISR Rules in FreeRTOS

### 11.1.1 The Fundamental Rule

> **In an ISR, you MUST use the `FromISR` variant of any FreeRTOS API. Never call the task-level API from an ISR.**

```c
/* ═══ TASK CONTEXT ═══           ═══ ISR CONTEXT ═══ */
xQueueSend()                      xQueueSendFromISR()
xQueueReceive()                   xQueueReceiveFromISR()
xSemaphoreGive()                  xSemaphoreGiveFromISR()
xSemaphoreTake()                  /* NO Take in ISR — can't block! */
xEventGroupSetBits()              xEventGroupSetBitsFromISR()
xTaskNotifyGive()                 vTaskNotifyGiveFromISR()
xStreamBufferSend()               xStreamBufferSendFromISR()
```

**Why?** Task-level APIs can **block** (put the calling task to sleep). An ISR is not a task — there is no TCB, no stack to save, no state to block. Blocking in an ISR would crash the system.

### 11.1.2 ISR Priority Rules on Cortex-M

```
INTERRUPT PRIORITY MAP:
═══════════════════════

Priority 0 (HIGHEST)  ┐
Priority 1            │  These ISRs CANNOT call FreeRTOS APIs.
Priority 2            │  They run even when the kernel has
Priority 3            │  interrupts masked.
Priority 4            ┘  Use for ultra-fast, timing-critical ISRs.
─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ configMAX_SYSCALL_INTERRUPT_PRIORITY ─ ─
Priority 5            ┐
Priority 6            │  These ISRs CAN call FreeRTOS FromISR APIs.
Priority 7            │  The kernel can mask these during
  ...                 │  critical sections.
Priority 14           │
Priority 15 (LOWEST)  ┘  SysTick and PendSV run here.

RULE: ISRs calling FreeRTOS APIs must have NUMERICAL priority
      >= configMAX_SYSCALL_INTERRUPT_PRIORITY (e.g., >= 5)

REMEMBER: On Cortex-M, LOWER number = HIGHER priority!
```

### 11.1.3 Complete ISR Template

```c
void MY_IRQHandler(void)
{
    /* 1. MUST initialize to pdFALSE */
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    /* 2. Handle the interrupt (clear flags, read data) */
    uint32_t status = PERIPHERAL->SR;
    PERIPHERAL->SR = 0;  /* Clear flags */

    /* 3. Call FreeRTOS FromISR API */
    xQueueSendFromISR(xQueue, &data, &xHigherPriorityTaskWoken);
    /* OR: xSemaphoreGiveFromISR(xSem, &xHigherPriorityTaskWoken); */
    /* OR: vTaskNotifyGiveFromISR(xTask, &xHigherPriorityTaskWoken); */

    /* 4. ALWAYS call portYIELD_FROM_ISR at the END */
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

### 11.1.4 The Deferred Interrupt Pattern

```
DEFERRED INTERRUPT PROCESSING:
══════════════════════════════

  HARDWARE          ISR (fast)           TASK (slow)
  ─────────        ───────────          ──────────────
  Interrupt  ──►   Read data    ──►     Complex processing
  fires            Clear flag            State machines
                   Give semaphore        Protocol parsing
                   (< 1 μs)              Logging
                                         (may block, malloc, etc.)

Benefits:
- ISR runs for < 1 μs (fast interrupt response)
- Heavy work happens in task context (can use ALL APIs)
- Other interrupts are not delayed
- System remains responsive
```

```c
/* ISR: Minimal work */
void USART2_IRQHandler(void)
{
    BaseType_t xWoken = pdFALSE;

    if (USART2->SR & USART_SR_RXNE)
    {
        uint8_t byte = USART2->DR;  /* Reading DR clears RXNE */
        xQueueSendFromISR(xRxQueue, &byte, &xWoken);
    }

    portYIELD_FROM_ISR(xWoken);
}

/* TASK: All heavy processing */
void Task_UartHandler(void *pvParameters)
{
    uint8_t byte;
    for (;;)
    {
        xQueueReceive(xRxQueue, &byte, portMAX_DELAY);

        /* Complex processing — safe in task context */
        Parse_Protocol(&byte);
        Log_Data(&byte);
        Update_Display();
    }
}
```

---

## 11.2 Common ISR Mistakes

### Mistake 1: Using Task API in ISR
```c
/* Calling xSemaphoreTake in an ISR → CRASH or HANG */
void BAD_IRQHandler(void)
{
    xSemaphoreTake(xMutex, portMAX_DELAY);  /* FATAL! Can't block in ISR! */
}
```

### Mistake 2: Wrong Interrupt Priority
```c
/* ISR at priority 2 calling FreeRTOS API when MAX_SYSCALL = 5 */
HAL_NVIC_SetPriority(TIM2_IRQn, 2, 0);  /* TOO HIGH! */
/* Will cause assertion failure or silent corruption */

/* FIX: */
HAL_NVIC_SetPriority(TIM2_IRQn, 6, 0);  /* Below threshold */
```

### Mistake 3: Forgetting portYIELD_FROM_ISR
```c
/* Without yield, unblocked task waits until next tick (up to 1ms delay) */
/* ALWAYS call at END of ISR */
```

### Mistake 4: Spending Too Long in ISR
```c
/* Every microsecond in an ISR is a microsecond other interrupts are delayed.
   RULE: ISR should complete in < 10 μs ideally, < 100 μs maximum.
   Move heavy work to a task via deferred processing. */
```

---

## 11.3 Chapter Summary

| Concept | Key Takeaway |
|---|---|
| **FromISR APIs** | Must use in ISRs; task-level APIs crash in ISR context |
| **Priority rule** | ISRs calling FreeRTOS APIs: priority ≥ configMAX_SYSCALL |
| **Deferred processing** | ISR stores data + signals; task does heavy work |
| **portYIELD_FROM_ISR** | Call at ISR end to enable immediate context switch |
| **ISR duration** | Keep < 10 μs; defer everything possible to tasks |

---

*End of Chapter 11*
