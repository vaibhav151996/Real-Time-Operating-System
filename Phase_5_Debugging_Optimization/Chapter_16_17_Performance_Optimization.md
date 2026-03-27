# Chapter 16: Performance — CPU Load and Timing Analysis

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Measure per-task CPU usage using FreeRTOS runtime stats
2. Set up DWT cycle counting for precise timing measurement
3. Identify performance bottlenecks
4. Analyze system timing with GPIO instrumentation

---

## 16.1 CPU Load Measurement

### 16.1.1 Runtime Stats Configuration

```c
/* FreeRTOSConfig.h */
#define configGENERATE_RUN_TIME_STATS            1
#define configUSE_STATS_FORMATTING_FUNCTIONS      1

/* You must provide a high-resolution timer (faster than tick) */
/* Use DWT Cycle Counter (available on Cortex-M3/M4/M7) */

/* Timer initialization macro */
#define portCONFIGURE_TIMER_FOR_RUN_TIME_STATS() do { \
    CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk; \
    DWT->CYCCNT = 0; \
    DWT->CTRL |= DWT_CTRL_CYCCNTENA_Msk; \
} while(0)

/* Timer read macro — returns a 32-bit timestamp */
#define portGET_RUN_TIME_COUNTER_VALUE()    (DWT->CYCCNT)
```

### 16.1.2 Displaying Runtime Stats

```c
void Task_Stats(void *pvParameters)
{
    char stats_buf[1024];

    for (;;)
    {
        vTaskGetRunTimeStats(stats_buf);
        
        /* Output format:
         * Task        Abs Time      % Time
         * ****************************************
         * Sensor      12345678      15%
         * Control     45678901      30%
         * Display      2345678       5%
         * IDLE        50000000      45%
         * Monitor      5000000       5%
         */
        
        const char *header = "\r\n--- CPU Usage ---\r\n";
        HAL_UART_Transmit(&huart2, (uint8_t *)header, strlen(header), 100);
        HAL_UART_Transmit(&huart2, (uint8_t *)stats_buf, strlen(stats_buf), 1000);

        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}
```

---

## 16.2 Precise Timing with DWT

### 16.2.1 Cycle-Accurate Measurement

```c
/* Measure execution time of any code block */
static inline uint32_t DWT_GetCycles(void)
{
    return DWT->CYCCNT;
}

void Task_TimingTest(void *pvParameters)
{
    for (;;)
    {
        uint32_t start = DWT_GetCycles();
        
        /* Code to measure */
        Process_Data();
        
        uint32_t end = DWT_GetCycles();
        uint32_t cycles = end - start;
        
        /* Convert to microseconds */
        float us = (float)cycles / (SystemCoreClock / 1000000);
        
        char msg[64];
        snprintf(msg, sizeof(msg), "Process_Data: %lu cycles (%.1f us)\r\n",
                 (unsigned long)cycles, us);
        HAL_UART_Transmit(&huart2, (uint8_t *)msg, strlen(msg), 100);
        
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

---

## 16.3 GPIO Instrumentation

```c
/*
 * Toggle GPIO pins at task entry/exit for oscilloscope measurement.
 * Each task gets a dedicated pin.
 */

void Task_Sensor(void *pvParameters)
{
    TickType_t xLastWake = xTaskGetTickCount();

    for (;;)
    {
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);   /* Pin HIGH */
        
        Read_Sensors();
        Process_Data();
        
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET); /* Pin LOW */
        
        xTaskDelayUntil(&xLastWake, pdMS_TO_TICKS(10));
    }
}

/*
 * On oscilloscope:
 *   HIGH duration = task execution time
 *   LOW  duration = task idle time (blocked/waiting)
 *   Period        = task period
 *   Jitter        = variation in period (should be < 1 tick)
 *   CPU usage     = HIGH% / (HIGH% + LOW%)
 */
```

---

## 16.4 Chapter Summary

| Concept | Key Takeaway |
|---|---|
| **Runtime stats** | Per-task CPU usage via DWT cycle counter |
| **DWT** | Hardware cycle counter; cycle-accurate timing |
| **GPIO instrumentation** | Toggle pins for oscilloscope-based analysis |
| **CPU load** | Sum of all task execution times / total time × 100 |
| **Target** | Keep CPU load < 60-70% for headroom |

---

*End of Chapter 16*


# Chapter 17: Optimization

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Design efficient task architectures that minimize overhead
2. Optimize memory usage (RAM and Flash)
3. Reduce context switch overhead
4. Apply compiler optimization settings effectively

---

## 17.1 Task Design Optimization

### 17.1.1 Consolidate Related Tasks

```c
/* WASTEFUL: One task per LED (3 tasks × 512 bytes stack = 1.5 KB) */
xTaskCreate(Task_LED1, "LED1", 128, NULL, 1, NULL);
xTaskCreate(Task_LED2, "LED2", 128, NULL, 1, NULL);
xTaskCreate(Task_LED3, "LED3", 128, NULL, 1, NULL);

/* EFFICIENT: One task manages all LEDs (512 bytes total) */
void Task_AllLEDs(void *pv)
{
    uint32_t counter = 0;
    for (;;)
    {
        counter++;
        if (counter % 1 == 0)   Toggle_LED1();  /* Every 10 ms */
        if (counter % 5 == 0)   Toggle_LED2();  /* Every 50 ms */
        if (counter % 10 == 0)  Toggle_LED3();  /* Every 100 ms */
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

### 17.1.2 Reduce Stack Usage

```c
/* WASTEFUL: Large local buffer on stack */
void Task_Bad(void *pv)
{
    char buffer[2048];    /* 2 KB on stack! */
    for (;;) { /* ... */ }
}

/* EFFICIENT: Static buffer (in BSS, not stack) */
static char buffer[2048];    /* In global memory */
void Task_Good(void *pv)
{
    for (;;) { /* use buffer */ }
}

/* EFFICIENT: Dynamic allocation (on heap, not stack) */
void Task_Alternative(void *pv)
{
    char *buffer = pvPortMalloc(2048);
    for (;;) { /* use buffer */ }
}
```

---

## 17.2 Memory Optimization

### 17.2.1 RAM Budget Checklist

```
RAM USAGE BREAKDOWN:
════════════════════

Per Task:
  TCB:            ~100 bytes
  Stack:          N × 4 bytes (N = configurable words)
  ─────────────────────────────────
  Total per task: 100 + (stack_words × 4)

Kernel:
  Ready lists:    configMAX_PRIORITIES × 20 bytes
  Delayed lists:  2 × 20 bytes
  Other lists:    ~100 bytes

Per Queue:        76 + (length × item_size) bytes
Per Semaphore:    88 bytes
Per Mutex:        88 bytes
Per Timer:        ~50 bytes
Per Event Group:  ~28 bytes

OPTIMIZATION CHECKLIST:
□ Minimize configMAX_PRIORITIES (5-7 is usually enough)
□ Use task notifications instead of semaphores where possible (0 bytes vs 88)
□ Right-size stacks (measure high-water mark, add 25%)
□ Use configMINIMAL_STACK_SIZE for simple tasks
□ Disable unused features (configUSE_COUNTING_SEMAPHORES = 0, etc.)
□ Use configUSE_16_BIT_TICKS if tick count doesn't need 32 bits
```

### 17.2.2 Flash Optimization

```c
/* Disable FreeRTOS features you don't use */
#define configUSE_MUTEXES                  0    /* Saves ~1 KB Flash */
#define configUSE_RECURSIVE_MUTEXES        0    /* Saves ~500 bytes */
#define configUSE_COUNTING_SEMAPHORES      0    /* Saves ~500 bytes */
#define configUSE_QUEUE_SETS               0    /* Saves ~500 bytes */
#define configUSE_TIMERS                   0    /* Saves ~2 KB + daemon stack */
#define configUSE_TRACE_FACILITY           0    /* Saves ~1 KB */
#define configGENERATE_RUN_TIME_STATS      0    /* Saves ~500 bytes */

/* Use compiler optimization */
/* -Os (optimize for size) or -O2 (optimize for speed) */
/* GCC: arm-none-eabi-gcc -Os ... */
```

---

## 17.3 Context Switch Optimization

```
REDUCE CONTEXT SWITCHES:
════════════════════════

1. Use event-driven design (block on queues/semaphores)
   instead of periodic polling

2. Consolidate tasks that run at the same priority
   and similar frequency

3. Don't over-slice: configUSE_TIME_SLICING = 0 if
   equal-priority tasks can take turns naturally

4. Reduce tick rate if 1ms resolution isn't needed:
   configTICK_RATE_HZ = 250 (4ms resolution, 75% fewer ticks)

5. Use task notifications instead of queues for simple
   signaling (fewer kernel operations)
```

---

## 17.4 Chapter Summary

| Concept | Key Takeaway |
|---|---|
| **Task consolidation** | Fewer tasks = less overhead; group related operations |
| **Stack optimization** | Measure, right-size, use static buffers for large data |
| **Feature disabling** | Turn off unused FreeRTOS features to save Flash/RAM |
| **Notifications > Semaphores** | Zero RAM, 45% faster for 1:1 signaling |
| **Tick rate** | Lower tick rate = less overhead (if timing allows) |

---

*End of Chapter 17*
