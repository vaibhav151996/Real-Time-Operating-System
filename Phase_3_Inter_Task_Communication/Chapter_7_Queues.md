# Chapter 7: Queues

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Explain FIFO principles and why queues are fundamental to RTOS communication
2. Create, send to, and receive from FreeRTOS queues
3. Implement ISR-to-task communication using queues
4. Design queue-based producer-consumer systems
5. Debug queue-related issues (full queues, data corruption, deadlock)

---

## 7.1 FIFO Principles

### 7.1.1 What Is a Queue?

A **queue** is a thread-safe FIFO (First-In, First-Out) data structure that allows tasks (and ISRs) to exchange data safely.

```
QUEUE: First-In, First-Out (FIFO)
══════════════════════════════════

   SEND                                        RECEIVE
    │                                             │
    ▼                                             ▼
┌───────┬───────┬───────┬───────┬───────┬───────┐
│ Item6 │ Item5 │ Item4 │ Item3 │ Item2 │ Item1 │ ──► Item1 out first
└───────┴───────┴───────┴───────┴───────┴───────┘
  BACK                                   FRONT

Analogy: A line at a bank — first person in line
         is the first person served.
```

### 7.1.2 Why Not Just Use Global Variables?

```c
/* WRONG: Shared global variable — RACE CONDITION */
volatile uint16_t shared_data = 0;

void Task_Producer(void *pv)
{
    for (;;)
    {
        shared_data = Read_ADC();    /* Write */
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}

void Task_Consumer(void *pv)
{
    for (;;)
    {
        uint16_t value = shared_data;    /* Read */
        Process(value);
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}

/*
 * PROBLEMS:
 * 1. DATA RACE: Producer may write while consumer reads
 *    → Consumer gets torn data (half old, half new)
 * 2. LOST DATA: If producer writes twice before consumer reads,
 *    first value is lost
 * 3. NO SYNCHRONIZATION: Consumer doesn't know when new data arrives
 * 4. NO BUFFERING: Only one value stored at a time
 */
```

**Queues solve ALL of these problems:**

```c
/* RIGHT: Queue provides safe, synchronized, buffered communication */
QueueHandle_t xQueue = xQueueCreate(10, sizeof(uint16_t));

void Task_Producer(void *pv)
{
    for (;;)
    {
        uint16_t value = Read_ADC();
        xQueueSend(xQueue, &value, portMAX_DELAY);  /* Thread-safe */
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}

void Task_Consumer(void *pv)
{
    uint16_t value;
    for (;;)
    {
        xQueueReceive(xQueue, &value, portMAX_DELAY);  /* Blocks until data */
        Process(value);
    }
}
```

### 7.1.3 Creating a Queue

```c
QueueHandle_t xQueueCreate(
    UBaseType_t uxQueueLength,     /* Maximum number of items */
    UBaseType_t uxItemSize          /* Size of each item in bytes */
);

/* Examples */
QueueHandle_t xByteQueue   = xQueueCreate(64, sizeof(uint8_t));     /* 64 bytes */
QueueHandle_t xSensorQueue = xQueueCreate(10, sizeof(SensorData_t)); /* 10 structs */
QueueHandle_t xPointerQueue = xQueueCreate(5, sizeof(char *));       /* 5 pointers */
```

**Memory allocated:** `uxQueueLength × uxItemSize + queue overhead (~76 bytes)`

> **⚠️ Critical:** FreeRTOS queues **copy** data, not pass pointers. When you send a struct, the entire struct is copied into the queue. When you receive, the struct is copied out. This is safe (no dangling pointers) but uses more memory for large items.

### 7.1.4 Queue Operations

```c
/* ──── SEND (Enqueue) ──── */

/* From a Task: */
BaseType_t xQueueSend(
    QueueHandle_t xQueue,       /* Queue handle */
    const void *pvItemToQueue,  /* Pointer to item to copy */
    TickType_t xTicksToWait     /* Max ticks to wait if queue full */
);

/* Send to back (default — same as xQueueSend) */
xQueueSendToBack(xQueue, &data, portMAX_DELAY);

/* Send to front (cut in line — useful for urgent items) */
xQueueSendToFront(xQueue, &urgent_data, portMAX_DELAY);

/* Overwrite (queue of length 1 — always succeeds, replaces data) */
xQueueOverwrite(xQueue, &latest_data);


/* ──── RECEIVE (Dequeue) ──── */

/* From a Task: */
BaseType_t xQueueReceive(
    QueueHandle_t xQueue,       /* Queue handle */
    void *pvBuffer,             /* Buffer to receive item into */
    TickType_t xTicksToWait     /* Max ticks to wait if queue empty */
);

/* Peek (read without removing) */
xQueuePeek(xQueue, &data, portMAX_DELAY);


/* ──── ISR Variants ──── */
xQueueSendFromISR(xQueue, &data, &xHigherPriorityTaskWoken);
xQueueReceiveFromISR(xQueue, &data, &xHigherPriorityTaskWoken);

/* ──── Utility ──── */
UBaseType_t uxQueueMessagesWaiting(xQueue);  /* Items in queue */
UBaseType_t uxQueueSpacesAvailable(xQueue);  /* Free space */
void vQueueDelete(QueueHandle_t xQueue);     /* Delete queue */
```

### 7.1.5 Blocking Behavior

```
BLOCKING ON SEND (queue full):
══════════════════════════════

Task calls xQueueSend() but queue is full.

  xTicksToWait = 0:
    Returns errQUEUE_FULL immediately (non-blocking)

  xTicksToWait = pdMS_TO_TICKS(100):
    Task BLOCKS for up to 100 ms
    If space becomes available → item sent, returns pdPASS
    If timeout expires → returns errQUEUE_FULL

  xTicksToWait = portMAX_DELAY:
    Task BLOCKS indefinitely until space becomes available
    (Requires INCLUDE_vTaskSuspend = 1)


BLOCKING ON RECEIVE (queue empty):
══════════════════════════════════

Task calls xQueueReceive() but queue is empty.

  xTicksToWait = 0:
    Returns pdFALSE immediately (non-blocking)

  xTicksToWait = pdMS_TO_TICKS(100):
    Task BLOCKS for up to 100 ms
    If data arrives → item received, returns pdPASS
    If timeout expires → returns pdFALSE

  xTicksToWait = portMAX_DELAY:
    Task BLOCKS indefinitely until data is available
```

---

## 7.2 ISR → Task Communication

### 7.2.1 The Problem: You Can't Do Heavy Work in an ISR

```c
/* WRONG: Too much processing in ISR */
void USART2_IRQHandler(void)
{
    uint8_t byte = USART2->DR;

    /* BAD: Long processing in ISR blocks ALL interrupts */
    Parse_Protocol(byte);        /* Could take 500 μs */
    Update_State_Machine(byte);  /* Could take 200 μs */
    Log_To_Buffer(byte);         /* Variable time */
}

/* RIGHT: ISR stores data, task processes it */
void USART2_IRQHandler(void)
{
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    uint8_t byte = USART2->DR;

    /* Fast: just put data in queue */
    xQueueSendFromISR(xUartQueue, &byte, &xHigherPriorityTaskWoken);

    /* If send woke a higher-priority task, yield */
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

void Task_UartProcess(void *pvParameters)
{
    uint8_t byte;
    for (;;)
    {
        /* Block until byte arrives from ISR */
        xQueueReceive(xUartQueue, &byte, portMAX_DELAY);

        /* Now we can take our time processing */
        Parse_Protocol(byte);
        Update_State_Machine(byte);
        Log_To_Buffer(byte);
    }
}
```

### 7.2.2 The xHigherPriorityTaskWoken Pattern

Every `FromISR` API has a `pxHigherPriorityTaskWoken` parameter. This is critical for responsiveness:

```c
void EXTI0_IRQHandler(void)
{
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;   /* MUST initialize */

    /* Clear interrupt */
    __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_0);

    /* Send data to queue */
    uint32_t event = EVENT_BUTTON;
    xQueueSendFromISR(xEventQueue, &event, &xHigherPriorityTaskWoken);

    /*
     * If xQueueSendFromISR unblocked a task with higher priority
     * than the currently running task, xHigherPriorityTaskWoken = pdTRUE.
     *
     * portYIELD_FROM_ISR will trigger a context switch at ISR exit,
     * so the high-priority task runs IMMEDIATELY.
     *
     * Without this, the task would only run at the next tick (up to 1 ms delay).
     */
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

```
WITHOUT portYIELD_FROM_ISR:
═══════════════════════════

Time ──────────────────────────────────────────────────►

         ISR fires    Task A    Next tick    Task B runs
Low-Pri  ████│████████████████ │████████████│
               ↑ Queue sent     ↑ Scheduler    ↑ Finally!
                                  runs          Up to 1ms delay

WITH portYIELD_FROM_ISR:
════════════════════════

Time ──────────────────────────────────────────────────►

         ISR fires    Task B runs IMMEDIATELY
Low-Pri  ████│████████│
High-Pri             │████████████████████████
               ↑ Queue sent + yield → instant switch
               Zero delay!
```

### 7.2.3 Complete ISR → Task Example: ADC with Queue

```c
/* Queue holds ADC readings */
QueueHandle_t xAdcQueue;

/* ADC conversion complete interrupt */
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    BaseType_t xWoken = pdFALSE;
    uint16_t adc_value = HAL_ADC_GetValue(hadc);

    xQueueSendFromISR(xAdcQueue, &adc_value, &xWoken);
    portYIELD_FROM_ISR(xWoken);
}

/* Processing task */
void Task_AdcProcess(void *pvParameters)
{
    uint16_t value;
    float filtered = 0.0f;

    for (;;)
    {
        if (xQueueReceive(xAdcQueue, &value, pdMS_TO_TICKS(1000)) == pdPASS)
        {
            /* Simple low-pass filter */
            filtered = 0.9f * filtered + 0.1f * (float)value;

            if (filtered > THRESHOLD)
            {
                Trigger_Alarm();
            }
        }
        else
        {
            /* Timeout — no ADC data for 1 second */
            Log_Error("ADC timeout");
        }
    }
}

int main(void)
{
    /* ... init ... */

    xAdcQueue = xQueueCreate(32, sizeof(uint16_t));
    configASSERT(xAdcQueue != NULL);

    xTaskCreate(Task_AdcProcess, "ADC", 256, NULL, 3, NULL);

    /* Start ADC in interrupt mode */
    HAL_ADC_Start_IT(&hadc1);

    vTaskStartScheduler();
    for (;;);
}
```

---

## 7.3 Queue Design Patterns

### 7.3.1 Command Queue Pattern

```c
/* Define command types */
typedef enum {
    CMD_MOTOR_START,
    CMD_MOTOR_STOP,
    CMD_MOTOR_SET_SPEED,
    CMD_LED_ON,
    CMD_LED_OFF,
} CommandType_t;

typedef struct {
    CommandType_t type;
    uint32_t      parameter;
} Command_t;

QueueHandle_t xCmdQueue;

/* Sender tasks enqueue commands */
void Task_UI(void *pv)
{
    for (;;)
    {
        if (Button_Pressed())
        {
            Command_t cmd = { .type = CMD_MOTOR_START, .parameter = 1500 };
            xQueueSend(xCmdQueue, &cmd, pdMS_TO_TICKS(100));
        }
        vTaskDelay(pdMS_TO_TICKS(50));
    }
}

/* Single executor task processes commands */
void Task_CommandExecutor(void *pv)
{
    Command_t cmd;
    for (;;)
    {
        if (xQueueReceive(xCmdQueue, &cmd, portMAX_DELAY) == pdPASS)
        {
            switch (cmd.type)
            {
                case CMD_MOTOR_START:
                    Motor_Start(cmd.parameter);
                    break;
                case CMD_MOTOR_STOP:
                    Motor_Stop();
                    break;
                case CMD_MOTOR_SET_SPEED:
                    Motor_SetSpeed(cmd.parameter);
                    break;
                case CMD_LED_ON:
                    LED_On(cmd.parameter);
                    break;
                case CMD_LED_OFF:
                    LED_Off(cmd.parameter);
                    break;
            }
        }
    }
}
```

### 7.3.2 Queue of Pointers (Large Data)

For large data structures, send **pointers** instead of copies to avoid memcpy overhead:

```c
/* Queue holds pointers to data blocks */
QueueHandle_t xPtrQueue = xQueueCreate(10, sizeof(DataBlock_t *));

/* Producer allocates and sends pointer */
void Task_Producer(void *pv)
{
    for (;;)
    {
        DataBlock_t *pBlock = pvPortMalloc(sizeof(DataBlock_t));
        if (pBlock != NULL)
        {
            Fill_DataBlock(pBlock);
            xQueueSend(xPtrQueue, &pBlock, portMAX_DELAY);
            /* NOTE: Producer must NOT use pBlock after sending! */
        }
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

/* Consumer receives pointer, uses data, frees memory */
void Task_Consumer(void *pv)
{
    DataBlock_t *pBlock;
    for (;;)
    {
        xQueueReceive(xPtrQueue, &pBlock, portMAX_DELAY);
        Process_DataBlock(pBlock);
        vPortFree(pBlock);    /* Consumer frees the memory */
    }
}
```

> **⚠️ Warning:** With pointer queues, you must manage memory ownership carefully. Only ONE task should free the memory. Double-free or use-after-free causes heap corruption.

---

## 7.4 Common Mistakes (Chapter 7)

### Mistake 1: Calling Queue API (Not FromISR) in an ISR
```c
/* WRONG: Will corrupt kernel state or crash */
void EXTI_IRQHandler(void)
{
    xQueueSend(xQueue, &data, portMAX_DELAY);  /* CRASH! */
}

/* RIGHT: Use FromISR variant */
void EXTI_IRQHandler(void)
{
    BaseType_t xWoken = pdFALSE;
    xQueueSendFromISR(xQueue, &data, &xWoken);
    portYIELD_FROM_ISR(xWoken);
}
```

### Mistake 2: Queue Too Small (Data Loss)
```c
/* If producer is faster than consumer, queue fills up */
xQueueCreate(5, sizeof(uint16_t));  /* Only 5 items! */

/* At 1000 Hz producer, 100 Hz consumer:
   Queue fills in 5 ms, then data is LOST unless
   xTicksToWait > 0 (which BLOCKS the producer) */

/* FIX: Size queue for the worst-case burst */
xQueueCreate(100, sizeof(uint16_t));
```

### Mistake 3: Forgetting portYIELD_FROM_ISR
```c
/* Without yield, unblocked high-priority task waits up to 1 tick */
/* Always call portYIELD_FROM_ISR at the END of the ISR */
```

---

## 7.5 Hands-On Exercises

### Exercise 7.1: UART Echo with Queue

Build a system where:
1. UART receive ISR puts bytes into a queue
2. A task reads from the queue and echoes bytes back over UART
3. Bonus: Add a command parser (commands terminated by '\n')

### Exercise 7.2: Multi-Sensor Data Aggregator

Create 3 "sensor" tasks that generate random data at different rates (10 Hz, 5 Hz, 1 Hz). All send to a single queue. A display task reads the queue and prints all values.

---

## 7.6 Chapter Summary

| Concept | Key Takeaway |
|---|---|
| **Queue** | Thread-safe FIFO; copies data; supports blocking |
| **xQueueSend** | Add to back; blocks if full (configurable timeout) |
| **xQueueReceive** | Remove from front; blocks if empty |
| **FromISR variants** | Must use in ISRs; include pxHigherPriorityTaskWoken |
| **portYIELD_FROM_ISR** | Triggers immediate context switch at ISR exit |
| **Queue sizing** | Must handle worst-case burst; oversizing wastes RAM |
| **Copy vs. pointer** | Small items: copy; large items: send pointers |

---

*End of Chapter 7*
