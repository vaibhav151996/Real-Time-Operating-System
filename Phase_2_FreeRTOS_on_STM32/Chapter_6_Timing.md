# Chapter 6: Timing — vTaskDelay and Software Timers

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Use `vTaskDelay()` and `vTaskDelayUntil()` correctly and understand the difference
2. Explain how the FreeRTOS tick drives all timing operations
3. Create, manage, and debug software timers
4. Implement precise periodic execution
5. Avoid common timing pitfalls in RTOS applications

---

## 6.1 vTaskDelay — Relative Delay

### 6.1.1 How It Works

```c
void vTaskDelay(const TickType_t xTicksToDelay);
```

`vTaskDelay()` places the calling task into the **Blocked** state for a specified number of ticks. The task uses **zero CPU** while blocked.

```
vTaskDelay(pdMS_TO_TICKS(100)):
════════════════════════════════

Time  ──────────────────────────────────────────────────►

Task:  ████████████        ████████████        ████████████
       (executing)  BLOCKED (executing)  BLOCKED (executing)
                    100 ms               100 ms

       │ 20 ms │ 100 ms  │ 20 ms │ 100 ms  │
       ├───────┤          ├───────┤
       work time          work time

       Total period = work_time + delay_time = 120 ms
       ^^^                              ^^^
       NOT exactly 100 ms period!       DRIFT!
```

**Key characteristic:** The delay starts from when `vTaskDelay()` is called, not from when the task last woke up. If the task's work takes variable time, the period **drifts**.

### 6.1.2 When to Use vTaskDelay

- **Non-time-critical delays:** Debouncing buttons, LED animations
- **Approximate timing:** "About every 100 ms" is good enough
- **One-shot waits:** Wait before retrying an operation

```c
/* Example: Button debounce — exact timing doesn't matter */
void Task_Button(void *pvParameters)
{
    for (;;)
    {
        if (HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_RESET)
        {
            vTaskDelay(pdMS_TO_TICKS(50));   /* Debounce delay */

            if (HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_RESET)
            {
                Handle_Button_Press();
            }
        }
        vTaskDelay(pdMS_TO_TICKS(10));   /* Poll every ~10 ms */
    }
}
```

---

## 6.2 vTaskDelayUntil — Absolute Periodic Delay

### 6.2.1 How It Works

```c
BaseType_t xTaskDelayUntil(
    TickType_t * const pxPreviousWakeTime,    /* In/Out: last wake time */
    const TickType_t   xTimeIncrement          /* Period in ticks */
);
```

`vTaskDelayUntil()` / `xTaskDelayUntil()` calculates the delay relative to the **last wake time**, not the current time. This compensates for variable execution time.

```
vTaskDelayUntil — Period = 100 ms:
═══════════════════════════════════

Time  ──────────────────────────────────────────────────────►

Task:  ████         ████████         ████           ████████
       10ms         40ms             15ms            40ms
       │←─ 100 ms ──►│←── 100 ms ──►│←── 100 ms ──►│

       Wake at       Wake at         Wake at         Wake at
       tick 0        tick 100        tick 200        tick 300

       Even though work time varies (10, 40, 15, 40 ms),
       the wake-up time is ALWAYS exactly 100 ms apart.
       NO DRIFT! ✓
```

### 6.2.2 The pdMS_TO_TICKS Macro

```c
/* Converts milliseconds to ticks based on configTICK_RATE_HZ */
#define pdMS_TO_TICKS(xTimeInMs) \
    ((TickType_t)(((TickType_t)(xTimeInMs) * (TickType_t)configTICK_RATE_HZ) / (TickType_t)1000U))

/* At 1000 Hz tick rate: pdMS_TO_TICKS(100) = 100 ticks = 100 ms ✓ */
/* At 100 Hz tick rate:  pdMS_TO_TICKS(100) = 10 ticks = 100 ms ✓ */
/* At 1000 Hz tick rate: pdMS_TO_TICKS(5) = 5 ticks = 5 ms ✓ */
/* At 100 Hz tick rate:  pdMS_TO_TICKS(5) = 0 ticks = 0 ms ✗ PROBLEM! */
```

> **⚠️ Warning:** `pdMS_TO_TICKS()` uses integer division. At low tick rates, small delays round to zero! At 100 Hz, you cannot create delays shorter than 10 ms.

### 6.2.3 Complete Periodic Task Example

```c
/* Precise 100 Hz sensor sampling task */
void Task_SensorSample(void *pvParameters)
{
    TickType_t xLastWakeTime;

    /* Initialize the last wake time to the current tick count.
     * This MUST be done ONCE before the loop. */
    xLastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        /* Do the work */
        uint16_t sample = Read_ADC_Channel(0);
        Store_Sample(sample);

        /* Wait exactly 10 ms from last wake (100 Hz) */
        xTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(10));
    }
}
```

### 6.2.4 vTaskDelay vs. vTaskDelayUntil — Comparison

| Feature | `vTaskDelay()` | `vTaskDelayUntil()` |
|---|---|---|
| **Delay reference** | From NOW | From last wake-up |
| **Period accuracy** | Drifts with work time | Constant period |
| **Compensates work time** | No | Yes |
| **When to use** | Approximate delays | Precise periodic tasks |
| **Code complexity** | Simpler | Needs `xLastWakeTime` variable |
| **If work > period** | Never a problem | Returns immediately (no delay) |

### 6.2.5 What If Work Time Exceeds the Period?

```c
/*
 * If the task's work takes LONGER than the period:
 * vTaskDelayUntil() returns immediately (no delay).
 * The next wake time is calculated correctly.
 *
 * Example: Period = 10 ms, work takes 15 ms
 */
void Task_Problem(void *pvParameters)
{
    TickType_t xLastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        /* Work takes 15 ms — longer than the 10 ms period! */
        Heavy_Processing();       /* 15 ms */

        xTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(10));
        /* Returns immediately — already past the deadline!
         * xLastWakeTime is updated, so the next cycle starts
         * immediately but from the correct reference point. */
    }
}

/*
 * Timeline:
 * Tick 0:    Wake, start work
 * Tick 15:   Work done, DelayUntil: target was tick 10 — already passed!
 *            Returns immediately. xLastWakeTime = 10.
 *            Starts next work immediately.
 * Tick 30:   Work done, DelayUntil: target was tick 20 — already passed!
 *            Returns immediately. xLastWakeTime = 20.
 *
 * RESULT: Task runs at 100% CPU and never meets its deadline.
 * The task is OVERLOADED — its work exceeds its period.
 */
```

> **🏭 Industry Insight:** In production systems, monitor for this condition. If `vTaskDelayUntil()` returns without actually delaying, your task is overloaded. Log it and alert the system.

---

## 6.3 Software Timers

### 6.3.1 What Are Software Timers?

Software timers execute a callback function at a specified time, managed entirely by the RTOS kernel (no hardware timer needed).

```
SOFTWARE TIMER ARCHITECTURE:
════════════════════════════

Your Task          Timer Daemon Task          Your Callback
─────────         ──────────────────          ─────────────
                  (Created by kernel)
                  
xTimerStart() ──► Timer command queue ──► Daemon processes ──► callback()
                  │                     │ commands, tracks   │
xTimerStop()  ──► │                     │ expirations        │
                  │                     │                    │
xTimerReset() ──► │                     │ When timer expires:│
                  └─────────────────────┘ calls your callback└──────────
```

**Key characteristics:**
- Callbacks run in the **Timer Daemon Task** context (not the calling task)
- Callbacks must **not block** (no `vTaskDelay`, no `xQueueReceive`)
- Callbacks must be **fast** (they block other timer callbacks)
- Timer daemon priority is `configTIMER_TASK_PRIORITY`

### 6.3.2 Creating a Software Timer

```c
TimerHandle_t xTimerCreate(
    const char * const    pcTimerName,     /* Debug name */
    const TickType_t      xTimerPeriodInTicks, /* Period */
    const UBaseType_t     uxAutoReload,    /* pdTRUE = repeating */
    void * const          pvTimerID,       /* User-defined ID */
    TimerCallbackFunction_t pxCallbackFunction /* Callback */
);
```

### 6.3.3 One-Shot vs. Auto-Reload Timers

```c
/* ONE-SHOT TIMER: Fires once, then stops */
TimerHandle_t xOneShotTimer = xTimerCreate(
    "OneShot",                     /* Name */
    pdMS_TO_TICKS(5000),           /* 5-second timeout */
    pdFALSE,                       /* One-shot */
    NULL,                          /* Timer ID */
    vOneShotCallback               /* Callback function */
);

void vOneShotCallback(TimerHandle_t xTimer)
{
    /* Fires once after 5 seconds */
    Turn_Off_Backlight();
}


/* AUTO-RELOAD TIMER: Fires repeatedly at the specified period */
TimerHandle_t xPeriodicTimer = xTimerCreate(
    "Periodic",                    /* Name */
    pdMS_TO_TICKS(1000),           /* 1-second period */
    pdTRUE,                        /* Auto-reload */
    NULL,                          /* Timer ID */
    vPeriodicCallback              /* Callback function */
);

void vPeriodicCallback(TimerHandle_t xTimer)
{
    /* Fires every 1 second */
    static uint32_t seconds = 0;
    seconds++;
    Update_Clock_Display(seconds);
}
```

### 6.3.4 Timer Operations

```c
/* Start a timer (begins counting from NOW) */
xTimerStart(xMyTimer, pdMS_TO_TICKS(100));   /* 100 ms to send command */

/* Stop a running timer */
xTimerStop(xMyTimer, pdMS_TO_TICKS(100));

/* Reset a timer (restart the countdown) */
xTimerReset(xMyTimer, pdMS_TO_TICKS(100));

/* Change the period */
xTimerChangePeriod(xMyTimer, pdMS_TO_TICKS(2000), pdMS_TO_TICKS(100));

/* From ISR */
BaseType_t xHigherPriorityTaskWoken = pdFALSE;
xTimerStartFromISR(xMyTimer, &xHigherPriorityTaskWoken);
portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
```

### 6.3.5 Practical Example: Inactivity Timeout

```c
/*
 * Turn off LCD backlight after 30 seconds of no button presses.
 * Any button press resets the timer.
 */

TimerHandle_t xBacklightTimer;

void vBacklightTimeoutCallback(TimerHandle_t xTimer)
{
    LCD_Backlight_Off();
}

void Setup_Backlight_Timer(void)
{
    xBacklightTimer = xTimerCreate(
        "Backlight",
        pdMS_TO_TICKS(30000),      /* 30 seconds */
        pdFALSE,                    /* One-shot */
        NULL,
        vBacklightTimeoutCallback
    );

    xTimerStart(xBacklightTimer, 0);
    LCD_Backlight_On();
}

/* Call this whenever user presses a button */
void On_Button_Press(void)
{
    LCD_Backlight_On();
    /* Reset the timer — restart the 30-second countdown */
    xTimerReset(xBacklightTimer, 0);
}
```

### 6.3.6 Using Timer ID for Multiple Timers with One Callback

```c
/*
 * Use Timer ID to differentiate timers sharing one callback.
 */
#define TIMER_ID_LED1    ((void *)1)
#define TIMER_ID_LED2    ((void *)2)
#define TIMER_ID_LED3    ((void *)3)

void vLEDTimerCallback(TimerHandle_t xTimer)
{
    uint32_t id = (uint32_t)pvTimerGetTimerID(xTimer);

    switch (id)
    {
        case 1: HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5); break;
        case 2: HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_6); break;
        case 3: HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_7); break;
    }
}

void Create_LED_Timers(void)
{
    xTimerCreate("LED1", pdMS_TO_TICKS(100),  pdTRUE, TIMER_ID_LED1, vLEDTimerCallback);
    xTimerCreate("LED2", pdMS_TO_TICKS(500),  pdTRUE, TIMER_ID_LED2, vLEDTimerCallback);
    xTimerCreate("LED3", pdMS_TO_TICKS(1000), pdTRUE, TIMER_ID_LED3, vLEDTimerCallback);
}
```

### 6.3.7 Software Timers vs. Hardware Timers

| Feature | Software Timer | Hardware Timer |
|---|---|---|
| **Resolution** | 1 tick (typically 1 ms) | Clock cycle (nanoseconds) |
| **Quantity** | Unlimited (limited by RAM) | Limited by MCU (3-15 typically) |
| **Accuracy** | ±1 tick jitter | Extremely precise |
| **CPU overhead** | Timer daemon task | Interrupt only |
| **When to use** | Timeouts, periodic non-critical | PWM, precise timing, counting |
| **Maximum precision** | ~1 ms | Sub-microsecond |

### 6.3.8 Timer Daemon Task

```
TIMER DAEMON INTERNALS:
═══════════════════════

                    ┌─────────────────────────┐
 xTimerStart() ──►  │   Timer Command Queue    │
 xTimerStop()  ──►  │   (configTIMER_QUEUE_    │
 xTimerReset() ──►  │    LENGTH deep)          │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │   Timer Daemon Task      │
                    │   Priority: configTIMER_ │
                    │   TASK_PRIORITY          │
                    │                          │
                    │   1. Process commands    │
                    │   2. Check timer list    │
                    │   3. Call callbacks       │
                    │   4. Block until next    │
                    │      timer or command    │
                    └─────────────────────────┘

 IMPORTANT: If the queue is full, xTimerStart() will BLOCK
 (up to the specified timeout). If the daemon's priority
 is too low, timer callbacks may be delayed.
```

---

## 6.4 Timing Precision and Jitter

### 6.4.1 Understanding Tick Jitter

```
TICK JITTER:
════════════

Ideal tick timing (1ms):
│    │    │    │    │    │    │    │    │
0    1    2    3    4    5    6    7    8  ms

Actual tick timing (with jitter):
│    │   │     │   │    │    │   │     │
0   1   1.8   3   3.9  5    6  6.9   8  ms
    ↑   ↑         ↑              ↑
 OK  early       early          early

Jitter sources:
1. Higher-priority interrupts delaying SysTick ISR
2. Critical sections (interrupts disabled)
3. PendSV processing time
4. Flash wait states (variable instruction fetch time)
```

### 6.4.2 Measuring Timing Precision

```c
/*
 * Measure actual period of a periodic task.
 * Toggle a GPIO pin and measure with oscilloscope/logic analyzer.
 */
void Task_TimingTest(void *pvParameters)
{
    TickType_t xLastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        /* Toggle pin — measure period on oscilloscope */
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_0);

        xTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(10));
    }
}

/*
 * Expected: Pin toggles every 10 ms
 * 
 * Typical measurement:
 *   Average period: 10.000 ms
 *   Min period:     9.998 ms
 *   Max period:     10.002 ms
 *   Jitter:         ±2 μs (< 1 tick)
 *
 * If jitter is > 1 tick, check for:
 *   - Long ISRs blocking SysTick
 *   - Critical sections disabling interrupts
 *   - Higher-priority tasks consuming too much time
 */
```

### 6.4.3 Sub-Tick Precision

FreeRTOS tick resolution is typically 1 ms. For sub-millisecond timing:

```c
/* WRONG: vTaskDelay can't do sub-tick delays */
vTaskDelay(pdMS_TO_TICKS(0));    /* = 0 ticks = no delay */

/* For sub-millisecond precision, use hardware timers */
void Delay_Microseconds(uint32_t us)
{
    /* Use a hardware timer (TIM2, etc.) for μs precision */
    __HAL_TIM_SET_COUNTER(&htim2, 0);
    while (__HAL_TIM_GET_COUNTER(&htim2) < us);
}

/* Or use DWT cycle counter (Cortex-M3/M4/M7) */
void Delay_Microseconds_DWT(uint32_t us)
{
    uint32_t start = DWT->CYCCNT;
    uint32_t cycles = us * (SystemCoreClock / 1000000);
    while ((DWT->CYCCNT - start) < cycles);
}

/* WARNING: These are BUSY-WAIT delays — they consume CPU!
   Only use for very short delays (< 100 μs) within a task. */
```

---

## 6.5 Common Mistakes (Chapter 6)

### Mistake 1: Using vTaskDelay for Periodic Tasks
```c
/* WRONG: Period = work_time + delay → DRIFTS */
for (;;) { Do_Work(); vTaskDelay(pdMS_TO_TICKS(100)); }

/* RIGHT: Period = constant 100 ms → NO DRIFT */
TickType_t last = xTaskGetTickCount();
for (;;) { Do_Work(); xTaskDelayUntil(&last, pdMS_TO_TICKS(100)); }
```

### Mistake 2: Blocking in a Timer Callback
```c
/* WRONG: Timer callbacks run in the daemon task — blocking stalls ALL timers */
void vMyCallback(TimerHandle_t xTimer)
{
    vTaskDelay(pdMS_TO_TICKS(100));   /* BLOCKS TIMER DAEMON! */
}

/* RIGHT: Timer callbacks must be non-blocking */
void vMyCallback(TimerHandle_t xTimer)
{
    xQueueSend(workQueue, &data, 0);  /* Non-blocking send to a worker task */
}
```

### Mistake 3: Forgetting to Initialize xLastWakeTime
```c
/* WRONG: Uninitialized variable — undefined behavior */
TickType_t xLastWakeTime;   /* GARBAGE VALUE! */
for (;;) { xTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(100)); }

/* RIGHT: Initialize before the loop */
TickType_t xLastWakeTime = xTaskGetTickCount();
for (;;) { Do_Work(); xTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(100)); }
```

### Mistake 4: Timer Queue Too Short
```c
/* If multiple parts of the code start/stop timers simultaneously,
   and configTIMER_QUEUE_LENGTH is too small, commands are lost! */
#define configTIMER_QUEUE_LENGTH    10   /* Usually sufficient */
/* Increase if you get timeout errors from xTimerStart() */
```

---

## 6.6 Hands-On Exercises

### Exercise 6.1: Drift Demonstration

Create two tasks with the same intended period (100 ms):
- Task A uses `vTaskDelay(pdMS_TO_TICKS(100))`
- Task B uses `vTaskDelayUntil(&last, pdMS_TO_TICKS(100))`

Both toggle different GPIO pins. Add a variable-length inner loop to simulate work. Measure periods with a logic analyzer. You should see Task A drift while Task B remains precise.

### Exercise 6.2: Software Timer LED Pattern

Create a set of software timers that produce a specific LED blinking pattern:
1. LED1: Blink at 2 Hz (on 250ms, off 250ms)
2. LED2: Blink at 1 Hz
3. LED3: Single pulse every 3 seconds (100ms on, 2900ms off)

Use a single callback function with Timer IDs.

### Exercise 6.3: Watchdog Timer Implementation

Implement a software watchdog: if a task doesn't "feed" the watchdog within 5 seconds, trigger an error. Use a one-shot timer that's reset each time the task reports healthy.

---

## 6.7 Chapter Summary

| Concept | Key Takeaway |
|---|---|
| **vTaskDelay** | Relative delay; period = work + delay (drifts) |
| **vTaskDelayUntil** | Absolute delay; constant period (no drift) |
| **pdMS_TO_TICKS** | Converts ms to ticks; rounds down (beware at low tick rates) |
| **Software timers** | Callbacks in daemon task; must not block; 1-tick resolution |
| **One-shot timer** | Fires once; useful for timeouts |
| **Auto-reload timer** | Fires repeatedly; useful for periodic callbacks |
| **Timer daemon** | Task that manages all software timers; priority matters |
| **Tick jitter** | Typically < 1 tick; caused by ISRs, critical sections |
| **Sub-tick timing** | Use hardware timers or DWT counter (busy-wait) |

---

*End of Chapter 6*
