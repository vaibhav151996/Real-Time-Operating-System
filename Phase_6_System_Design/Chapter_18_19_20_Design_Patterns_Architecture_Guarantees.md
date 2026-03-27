# Chapter 18: Design Patterns for RTOS

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Implement the Producer-Consumer pattern with FreeRTOS queues
2. Build Event-Driven Architectures for embedded systems
3. Apply the Gateway pattern to protect shared peripherals
4. Use state machines within RTOS tasks

---

## 18.1 Producer-Consumer Pattern

### 18.1.1 Architecture

```
PRODUCER-CONSUMER:
══════════════════

  ┌──────────┐     ┌───────────┐     ┌──────────┐
  │ Producer │────►│   QUEUE   │────►│ Consumer │
  │ (Task)   │     │ (Buffer)  │     │ (Task)   │
  └──────────┘     └───────────┘     └──────────┘

  Producer: Generates data (sensor readings, UART bytes, events)
  Queue:    Buffered, thread-safe data transfer
  Consumer: Processes data (filtering, logging, transmission)

  ADVANTAGES:
  - Decoupled: Producer and Consumer run at different speeds
  - Buffered:  Queue absorbs data bursts
  - Safe:      No shared variables or race conditions
```

### 18.1.2 Implementation

```c
#define QUEUE_LENGTH    32

typedef struct {
    uint16_t sensor_id;
    uint16_t value;
    uint32_t timestamp;
} SensorReading_t;

QueueHandle_t xSensorQueue;

/* PRODUCER: Reads sensors and enqueues data */
void Task_SensorProducer(void *pvParameters)
{
    TickType_t xLastWake = xTaskGetTickCount();

    for (;;)
    {
        SensorReading_t reading;
        reading.sensor_id = 1;
        reading.value     = Read_ADC(ADC_CHANNEL_0);
        reading.timestamp = xTaskGetTickCount();

        if (xQueueSend(xSensorQueue, &reading, pdMS_TO_TICKS(10)) != pdPASS)
        {
            /* Queue full — data LOST! Log this event. */
            overflow_counter++;
        }

        xTaskDelayUntil(&xLastWake, pdMS_TO_TICKS(10));   /* 100 Hz */
    }
}

/* CONSUMER: Processes and logs readings */
void Task_DataConsumer(void *pvParameters)
{
    SensorReading_t reading;

    for (;;)
    {
        /* Block until data arrives */
        if (xQueueReceive(xSensorQueue, &reading, portMAX_DELAY) == pdPASS)
        {
            /* Process: filter, check thresholds, log */
            float filtered = Apply_LowPass_Filter(reading.value);

            if (filtered > ALARM_THRESHOLD)
            {
                Trigger_Alarm(reading.sensor_id);
            }

            Log_To_SD(&reading);
        }
    }
}

/* SETUP */
void Setup(void)
{
    xSensorQueue = xQueueCreate(QUEUE_LENGTH, sizeof(SensorReading_t));
    configASSERT(xSensorQueue != NULL);

    xTaskCreate(Task_SensorProducer, "SensProd", 256, NULL, 3, NULL);
    xTaskCreate(Task_DataConsumer,   "DataCons", 512, NULL, 2, NULL);
}
```

---

## 18.2 Event-Driven Architecture

### 18.2.1 Concept

```
EVENT-DRIVEN ARCHITECTURE:
══════════════════════════

Instead of periodic polling, tasks REACT to events.

  ┌────────────┐
  │ ISR:Button │ ──EVT_BUTTON──┐
  └────────────┘                │
  ┌────────────┐                ▼
  │ ISR:UART   │ ──EVT_UART──► ┌──────────────┐     ┌──────────┐
  └────────────┘                │ Event Queue  │────►│ Dispatcher│
  ┌────────────┐                │              │     │  Task     │
  │ Timer CB   │ ──EVT_TIMER──►│              │     └──────────┘
  └────────────┘                └──────────────┘          │
  ┌────────────┐                                    ┌─────┼─────┐
  │ Task:Sensor│ ──EVT_DATA──────────────────────►  │ Handler   │
  └────────────┘                                    │ Functions │
                                                    └───────────┘

ADVANTAGES:
- CPU idle when no events → power efficient
- Clean, modular code
- Easy to add new event types
```

### 18.2.2 Implementation

```c
typedef enum {
    EVT_BUTTON_PRESS,
    EVT_UART_RECEIVED,
    EVT_SENSOR_READY,
    EVT_TIMER_EXPIRED,
    EVT_ERROR,
} EventType_t;

typedef struct {
    EventType_t type;
    uint32_t    data;
    uint32_t    timestamp;
} Event_t;

QueueHandle_t xEventQueue;

/* Event Dispatcher Task */
void Task_EventDispatcher(void *pvParameters)
{
    Event_t event;

    for (;;)
    {
        if (xQueueReceive(xEventQueue, &event, portMAX_DELAY) == pdPASS)
        {
            switch (event.type)
            {
                case EVT_BUTTON_PRESS:
                    Handle_ButtonPress(event.data);
                    break;

                case EVT_UART_RECEIVED:
                    Handle_UartData(event.data);
                    break;

                case EVT_SENSOR_READY:
                    Handle_SensorData(event.data);
                    break;

                case EVT_TIMER_EXPIRED:
                    Handle_Timeout(event.data);
                    break;

                case EVT_ERROR:
                    Handle_Error(event.data);
                    break;
            }
        }
    }
}

/* Any ISR or task can post events */
void EXTI0_IRQHandler(void)
{
    BaseType_t xWoken = pdFALSE;
    __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_0);

    Event_t evt = {
        .type = EVT_BUTTON_PRESS,
        .data = 0,
        .timestamp = xTaskGetTickCountFromISR()
    };

    xQueueSendFromISR(xEventQueue, &evt, &xWoken);
    portYIELD_FROM_ISR(xWoken);
}
```

---

## 18.3 Gateway Pattern

```c
/*
 * GATEWAY PATTERN: One task owns a peripheral. Other tasks
 * send requests through a queue. Eliminates mutex contention.
 */

typedef struct {
    const uint8_t *data;
    uint16_t       length;
    SemaphoreHandle_t xDoneSem;  /* Signal completion */
} UartRequest_t;

QueueHandle_t xUartRequestQueue;

/* Gateway task — ONLY this task touches the UART */
void Task_UartGateway(void *pvParameters)
{
    UartRequest_t request;

    for (;;)
    {
        xQueueReceive(xUartRequestQueue, &request, portMAX_DELAY);
        HAL_UART_Transmit(&huart2, request.data, request.length, 1000);
        xSemaphoreGive(request.xDoneSem);   /* Signal caller */
    }
}

/* Any task can request UART transmission */
void UART_SendBlocking(const uint8_t *data, uint16_t len)
{
    SemaphoreHandle_t xDone = xSemaphoreCreateBinary();

    UartRequest_t request = {
        .data = data,
        .length = len,
        .xDoneSem = xDone
    };

    xQueueSend(xUartRequestQueue, &request, portMAX_DELAY);
    xSemaphoreTake(xDone, portMAX_DELAY);   /* Wait for completion */
    vSemaphoreDelete(xDone);
}
```

---

## 18.4 Chapter Summary

| Pattern | Use Case | Mechanism |
|---|---|---|
| **Producer-Consumer** | Data pipeline (sensor → processor) | Queue between tasks |
| **Event-Driven** | Reactive systems (IoT, UI) | Event queue + dispatcher |
| **Gateway** | Peripheral access without mutex | Request queue to owner task |
| **State Machine** | Protocol parsing, mode control | Switch/case in task loop |

---

*End of Chapter 18*


# Chapter 19: System Architecture

---

## 19.1 Task Decomposition

### 19.1.1 How to Decompose a System into Tasks

```
DECOMPOSITION RULES:
════════════════════

1. ONE TASK per independent timing requirement
   Example: Motor PWM at 10 kHz ≠ Display update at 5 Hz
   → Two separate tasks

2. ONE TASK per independent event source
   Example: UART receive, CAN receive
   → One task per protocol

3. MERGE tasks with SAME frequency and priority
   Example: Read temperature and humidity at 1 Hz
   → One "Environment" task

4. SEPARATE computational work from I/O
   Example: FFT computation vs. data display
   → Compute task feeds display task via queue

5. LIMIT total tasks to 5–15 for most embedded systems
   More tasks = more RAM (stacks), more complexity
```

### 19.1.2 Priority Assignment Method

```
STEP 1: List all tasks with their timing requirements

STEP 2: Apply RMS — shorter period = higher priority

STEP 3: Adjust for functional priority:
   - Safety tasks: ALWAYS highest
   - Communication: High (data loss on timeout)
   - Processing: Medium
   - Display/UI: Low (humans are slow)
   - Logging: Lowest (can tolerate delays)

STEP 4: Verify with utilization analysis:
   Total U = Σ(WCET / Period) < 69.3% (RMS bound)

STEP 5: Test under stress (all worst cases simultaneously)
```

---

## 19.2 Architecture Examples

### 19.2.1 Industrial Data Logger

```
┌───────────────────────────────────────────────────┐
│              SYSTEM ARCHITECTURE                   │
│                                                    │
│  ┌──────────┐  Queue   ┌──────────┐              │
│  │ Sensor   │─────────►│ Process  │              │
│  │ Task     │          │ Task     │              │
│  │ Pri: 4   │          │ Pri: 3   │              │
│  │ 10ms     │          │ Event    │              │
│  └──────────┘          └────┬─────┘              │
│                              │ Queue              │
│  ┌──────────┐          ┌────▼─────┐              │
│  │ Comm     │◄── Cmd ──│ Logger   │              │
│  │ Task     │   Queue  │ Task     │              │
│  │ Pri: 3   │          │ Pri: 2   │              │
│  │ 50ms     │          │ 1s       │              │
│  └──────────┘          └──────────┘              │
│                                                    │
│  ┌──────────┐   ┌──────────┐                      │
│  │ Display  │   │ Watchdog │                      │
│  │ Task     │   │ Task     │                      │
│  │ Pri: 1   │   │ Pri: 5   │                      │
│  │ 200ms    │   │ 100ms    │                      │
│  └──────────┘   └──────────┘                      │
└───────────────────────────────────────────────────┘
```

---

## 19.3 Chapter Summary

| Concept | Key Takeaway |
|---|---|
| **Task decomposition** | One task per timing requirement or event source |
| **Priority assignment** | RMS + functional importance adjustment |
| **Target task count** | 5–15 tasks for most systems |
| **Utilization budget** | Keep < 60% for margin |
| **Communication** | Queues between tasks; minimize shared state |

---

*End of Chapter 19*


# Chapter 20: Real-Time Guarantees

---

## 20.1 Meeting Deadlines

### 20.1.1 Deadline Analysis

For each task, verify:

$$ResponseTime_i = C_i + \sum_{j \in hp(i)} \lceil \frac{ResponseTime_i}{T_j} \rceil \times C_j$$

Where $hp(i)$ = set of tasks with higher priority than task $i$.

If $ResponseTime_i \leq Deadline_i$ for all tasks, the system is schedulable.

### 20.1.2 Worst-Case Execution Time (WCET)

```
MEASURING WCET:
═══════════════

1. MEASUREMENT: Run code under stress, measure maximum time
   - Toggle GPIO, use DWT, or logic analyzer
   - Must exercise ALL code paths
   - Add 20-30% margin for untested paths

2. ANALYSIS: Use static analysis tools
   - Count instructions on longest path
   - Account for cache misses, pipeline stalls
   - Tools: aiT, Bound-T, OTAWA

3. HYBRID: Measure + analyze for completeness
   - Most practical approach for embedded systems
```

---

## 20.2 Jitter

### 20.2.1 Sources and Mitigation

```
JITTER = Variation in task period or response time

Sources:
  1. Tick resolution (±1 tick)
  2. Higher-priority task preemption
  3. ISR processing time
  4. Critical section duration

Mitigation:
  1. Use vTaskDelayUntil (not vTaskDelay)
  2. Keep high-priority tasks short
  3. Minimize ISR duration
  4. Minimize critical section length
  5. Use hardware timers for microsecond-critical operations
```

---

## 20.3 Chapter Summary

| Concept | Key Takeaway |
|---|---|
| **Deadline** | Response time must be ≤ deadline for correctness |
| **WCET** | Always analyze worst-case, not average |
| **Jitter** | ±1 tick typical; minimize with good design |
| **Verification** | Response time analysis for each task |
| **Margin** | Design for < 60% CPU; leave 30% WCET margin |

---

*End of Chapter 20*
