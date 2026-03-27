# Chapter 8: Semaphores

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Explain the difference between binary and counting semaphores
2. Use semaphores for synchronization (signaling) between tasks and ISRs
3. Implement resource counting with counting semaphores
4. Distinguish between semaphores (signaling) and mutexes (locking) — and know when to use each
5. Debug semaphore-related issues

---

## 8.1 Binary Semaphores

### 8.1.1 What Is a Binary Semaphore?

A binary semaphore has two states: **taken** (0) and **given** (1). It is used for **synchronization** — one task signals another that an event has occurred.

```
BINARY SEMAPHORE:
═════════════════

State: 0 (taken)  or  1 (given)

    ┌─────────────┐                ┌─────────────┐
    │ ISR / Task A │               │   Task B     │
    │  (Signaler)  │               │  (Waiter)    │
    └──────┬──────┘                └──────┬──────┘
           │                              │
           │  xSemaphoreGive()            │  xSemaphoreTake()
           │  State: 0 → 1               │  BLOCKED until state = 1
           │─────────────────────────────►│  State: 1 → 0
           │                              │  UNBLOCKED → runs
           │                              ▼
           
Analogy: A notification bell.
  - The waiter sits until the bell rings (Take: blocks)
  - The signaler rings the bell (Give: unblocks waiter)
  - The bell can only be "rung" or "not rung" — binary
```

### 8.1.2 Creating and Using Binary Semaphores

```c
/* Create a binary semaphore — starts in "taken" state (0) */
SemaphoreHandle_t xBinarySem = xSemaphoreCreateBinary();
/* NOTE: Starts TAKEN — first Take will block! */

/* SIGNALER (ISR or Task): */
void EXTI0_IRQHandler(void)
{
    BaseType_t xWoken = pdFALSE;
    __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_0);

    xSemaphoreGiveFromISR(xBinarySem, &xWoken);
    portYIELD_FROM_ISR(xWoken);
}

/* WAITER (Task): */
void Task_Handler(void *pvParameters)
{
    for (;;)
    {
        /* Block until semaphore is given */
        if (xSemaphoreTake(xBinarySem, portMAX_DELAY) == pdTRUE)
        {
            /* Event occurred! Process it. */
            Handle_Event();
        }
    }
}
```

### 8.1.3 Binary Semaphore Timing Diagram

```
TIME ──────────────────────────────────────────────────────────►

ISR:        ↑Give         ↑Give                     ↑Give
            │             │                         │

Semaphore:  0→1  1→0      0→1  1→0                  0→1  1→0

Handler:    BLOCKED│RUN  BLOCKED│RUN              BLOCKED│RUN
                   │Handle()    │Handle()                │Handle()
```

### 8.1.4 Binary Semaphore as an ISR Deferral Mechanism

The most common use: **defer ISR processing to a task**.

```c
/*
 * Pattern: ISR does minimal work, signals task for heavy processing.
 *
 * Without RTOS: All processing happens in ISR (bad for latency)
 * With RTOS:    ISR stores data + gives semaphore; task does processing
 */

/* Shared data (written by ISR, read by task) */
volatile uint8_t rx_buffer[256];
volatile uint16_t rx_count = 0;

SemaphoreHandle_t xUartSem;

/* ISR: Fast — just store data and signal */
void USART2_IRQHandler(void)
{
    BaseType_t xWoken = pdFALSE;

    if (__HAL_UART_GET_FLAG(&huart2, UART_FLAG_RXNE))
    {
        rx_buffer[rx_count++] = huart2.Instance->DR;

        if (rx_count >= expected_length || rx_buffer[rx_count - 1] == '\n')
        {
            /* Full message received — signal the task */
            xSemaphoreGiveFromISR(xUartSem, &xWoken);
        }
    }

    portYIELD_FROM_ISR(xWoken);
}

/* Task: Heavy processing happens here */
void Task_UartProcess(void *pvParameters)
{
    for (;;)
    {
        xSemaphoreTake(xUartSem, portMAX_DELAY);

        /* Safe to process rx_buffer here (ISR won't modify until
           we start receiving the next message) */
        Parse_Message(rx_buffer, rx_count);
        rx_count = 0;
    }
}
```

### 8.1.5 Binary Semaphore Limitations

**Problem: Multiple Gives Before Take**

```
If the ISR fires TWICE before the task runs:

ISR:    ↑Give    ↑Give    (second Give is LOST!)
Sem:    0→1      1→1      (already 1, can't go higher)
Task:   BLOCKED  .......  →Take (1→0) — only ONE event processed!

RESULT: One event is LOST.

This is normal behavior for binary semaphores — they only track
"has an event occurred?" not "how many events occurred?"

SOLUTION: Use a counting semaphore or queue if event count matters.
```

---

## 8.2 Counting Semaphores

### 8.2.1 What Is a Counting Semaphore?

A counting semaphore maintains a count (0 to N). It is used for:

1. **Event counting:** Track how many events have occurred
2. **Resource management:** Track how many resources are available

```
COUNTING SEMAPHORE:
═══════════════════

State: 0, 1, 2, ... up to maxCount

  Give: count++  (if count < max)
  Take: count--  (if count > 0; else BLOCK)

TWO MAIN USES:

1. EVENT COUNTING:
   Create with initial count = 0
   Each ISR Give increments count
   Each Task Take decrements count
   → Ensures EVERY event is processed

2. RESOURCE COUNTING:
   Create with initial count = N (number of resources)
   Each Take "claims" a resource (count--)
   Each Give "releases" a resource (count++)
   → Limits concurrent access to N resources
```

### 8.2.2 Creating and Using Counting Semaphores

```c
/* Create a counting semaphore */
SemaphoreHandle_t xSemaphoreCreateCounting(
    UBaseType_t uxMaxCount,        /* Maximum count value */
    UBaseType_t uxInitialCount     /* Starting count */
);

/* EVENT COUNTING: Start at 0, max 20 events queued */
SemaphoreHandle_t xEventSem = xSemaphoreCreateCounting(20, 0);

/* RESOURCE COUNTING: 3 DMA channels available */
SemaphoreHandle_t xDmaSem = xSemaphoreCreateCounting(3, 3);
```

### 8.2.3 Event Counting Example

```c
/*
 * Ensure EVERY interrupt event is processed,
 * even if multiple fire before the task runs.
 */
SemaphoreHandle_t xEventCounter = xSemaphoreCreateCounting(20, 0);

/* ISR fires multiple times */
void TIM2_IRQHandler(void)
{
    BaseType_t xWoken = pdFALSE;
    __HAL_TIM_CLEAR_IT(&htim2, TIM_IT_UPDATE);

    /* Each Give increments the count */
    xSemaphoreGiveFromISR(xEventCounter, &xWoken);
    portYIELD_FROM_ISR(xWoken);
}

/* Task processes EVERY event */
void Task_EventProcessor(void *pvParameters)
{
    for (;;)
    {
        /* Blocks until count > 0, then decrements */
        xSemaphoreTake(xEventCounter, portMAX_DELAY);
        Process_Single_Event();

        /* If 5 events queued, this loops 5 times before blocking */
    }
}
```

### 8.2.4 Resource Counting Example

```c
/*
 * Limit concurrent SPI transactions to 2
 * (MCU has 2 SPI peripherals)
 */
SemaphoreHandle_t xSpiResource = xSemaphoreCreateCounting(2, 2);

void Task_SensorA(void *pvParameters)
{
    for (;;)
    {
        /* Wait for available SPI peripheral */
        xSemaphoreTake(xSpiResource, portMAX_DELAY);

        /* Use SPI (exclusive access to one peripheral) */
        SPI_Read_Sensor_A();

        /* Release the SPI peripheral */
        xSemaphoreGive(xSpiResource);

        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

/* Task_SensorB does the same — both can run simultaneously
   because there are 2 SPI peripherals (count = 2).
   A third task would have to wait. */
```

### 8.2.5 Binary vs. Counting Semaphore

| Feature | Binary Semaphore | Counting Semaphore |
|---|---|---|
| **Values** | 0 or 1 | 0 to uxMaxCount |
| **Multiple gives** | Extra gives lost | Count increments |
| **Primary use** | Signaling (ISR→Task) | Event counting, resources |
| **Give always succeeds** | Yes (sets to 1) | Only if count < max |
| **Ownership** | None (anyone can give/take) | None |
| **Priority inheritance** | No | No |
| **RAM overhead** | ~88 bytes | ~88 bytes |

---

## 8.3 Semaphore vs. Mutex (Critical Distinction)

This is one of the most important concepts in RTOS programming and a **top interview question**.

```
SEMAPHORE = SIGNALING (event notification)
═══════════════════════════════════════════

  Task A                    Task B
  (Producer)                (Consumer)
  
  "Data is ready!"          "OK, I'll process it"
  xSemaphoreGive() ──────► xSemaphoreTake()
  
  Task A GIVES              Task B TAKES
  A ≠ B (different tasks give and take)
  
  Analogy: A doorbell. Anyone can ring it. The resident answers.


MUTEX = LOCKING (mutual exclusion)
══════════════════════════════════

  Task A                    Task A (same task!)
  
  "I need exclusive access" "Done, releasing lock"
  xSemaphoreTake() ──────► xSemaphoreGive()
  
  SAME task takes AND gives
  
  Analogy: A bathroom lock. You lock it, you unlock it.
           No one else should unlock YOUR lock.
```

**Key differences:**

| Property | Semaphore | Mutex |
|---|---|---|
| **Purpose** | Signaling | Locking |
| **Who gives/takes** | Different tasks | Same task |
| **Priority inheritance** | No | Yes |
| **Can use from ISR** | Yes (FromISR) | No! |
| **Ownership** | None | Owner is the taker |

> **🎯 Interview Insight:** "What's the difference between a semaphore and a mutex?" — A semaphore is a signaling mechanism (one task gives, another takes). A mutex is a locking mechanism (same task takes and gives, with priority inheritance). They are NOT interchangeable.

---

## 8.4 Common Mistakes (Chapter 8)

### Mistake 1: Using Semaphore Where Mutex Is Needed
```c
/* WRONG: Using binary semaphore for mutual exclusion */
xSemaphoreTake(xBinarySem, portMAX_DELAY);
Access_Shared_Resource();
xSemaphoreGive(xBinarySem);
/* Problem: No priority inheritance → priority inversion possible */

/* RIGHT: Use mutex for mutual exclusion */
xSemaphoreTake(xMutex, portMAX_DELAY);
Access_Shared_Resource();
xSemaphoreGive(xMutex);
/* Mutex provides priority inheritance → safe */
```

### Mistake 2: Binary Semaphore Created Already Given
```c
/* xSemaphoreCreateBinary() starts TAKEN (count = 0) */
/* First Take will BLOCK until someone Gives! */

/* If you need it to start available, give it immediately: */
SemaphoreHandle_t xSem = xSemaphoreCreateBinary();
xSemaphoreGive(xSem);   /* Now count = 1 */
```

### Mistake 3: Giving Mutex from ISR
```c
/* WRONG: Mutex must not be used from ISR */
void ISR_Handler(void)
{
    xSemaphoreGiveFromISR(xMutex, &woken);   /* UNDEFINED BEHAVIOR! */
}
/* Use binary semaphore for ISR signaling, mutex for task locking */
```

---

## 8.5 Hands-On Exercises

### Exercise 8.1: ISR Deferral Pattern

Create a system where a button press ISR gives a binary semaphore, and a task waits for it to toggle an LED. Measure the latency from button press to LED toggle.

### Exercise 8.2: Resource Pool

Create a counting semaphore representing 3 available UART peripherals. Launch 5 tasks that each need a UART to transmit. Verify only 3 run concurrently.

---

## 8.6 Chapter Summary

| Concept | Key Takeaway |
|---|---|
| **Binary semaphore** | Two states (0/1); used for signaling |
| **Counting semaphore** | 0 to N; used for event counting or resource pools |
| **Give** | Increment count / set to 1 |
| **Take** | Decrement count / set to 0; blocks if count is 0 |
| **ISR usage** | Semaphores can be used from ISRs (FromISR variants) |
| **Semaphore vs Mutex** | Semaphore = signaling; Mutex = locking with ownership |
| **Priority inheritance** | Semaphores lack it; use Mutex when protecting resources |

---

*End of Chapter 8*
