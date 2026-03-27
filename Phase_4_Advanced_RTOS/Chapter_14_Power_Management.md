# Chapter 14: Power Management

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Implement tickless idle mode for low-power applications
2. Use STM32 low-power modes (Sleep, Stop, Standby) with FreeRTOS
3. Design power-aware task architectures
4. Measure and optimize power consumption

---

## 14.1 Tickless Idle Mode

### 14.1.1 The Power Problem

In normal operation, SysTick fires every 1 ms, waking the CPU from sleep. For a battery-powered IoT sensor that only needs to sample every 30 seconds, this means 30,000 unnecessary wake-ups.

```
NORMAL MODE (wasteful):
═══════════════════════
Power ▲
      │ ██  ██  ██  ██  ██  ██  ██  ██  ██  ██  Tick interrupts
      │ │↑│ │↑│ │↑│ │↑│ │↑│ │↑│ │↑│ │↑│ │↑│ │↑│ every 1ms
      │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │ │
      └─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴──► Time

TICKLESS MODE (efficient):
══════════════════════════
Power ▲
      │ ██                                    ██
      │ │ │      Deep Sleep (30 seconds)     │ │
      │ │ │                                  │ │
      └─┴─┴──────────────────────────────────┴─┴──► Time
      Wakes only when task needs to run
```

### 14.1.2 Configuration

```c
/* FreeRTOSConfig.h */
#define configUSE_TICKLESS_IDLE              2    /* 1=default, 2=custom */
#define configEXPECTED_IDLE_TIME_BEFORE_SLEEP  2  /* Min ticks before sleep */

/* Custom implementation for STM32 low-power modes */
/* See port layer: vPortSuppressTicksAndSleep() */
```

### 14.1.3 Pre-Sleep and Post-Sleep Hooks

```c
/* Called BEFORE entering sleep */
void configPRE_SLEEP_PROCESSING(TickType_t xExpectedIdleTime)
{
    /* Disable peripherals to save power */
    __HAL_RCC_GPIOA_CLK_DISABLE();
    __HAL_RCC_USART2_CLK_DISABLE();
    
    /* Configure wake-up sources */
    HAL_PWR_EnableWakeUpPin(PWR_WAKEUP_PIN1);
}

/* Called AFTER waking from sleep */
void configPOST_SLEEP_PROCESSING(TickType_t xExpectedIdleTime)
{
    /* Re-enable peripherals */
    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_USART2_CLK_ENABLE();
    
    /* Reconfigure clocks if exiting STOP mode */
    SystemClock_Config();
}
```

---

## 14.2 STM32 Low-Power Modes

### 14.2.1 Power Mode Comparison

```
STM32 Low-Power Modes:
═════════════════════

Mode          CPU    RAM    Peripherals    Wake-up time    Current
──────────────────────────────────────────────────────────────────
RUN           ON     ON     ON             —               ~100 mA
SLEEP         OFF    ON     ON             ~1 μs           ~20 mA
STOP          OFF    ON     OFF*           ~5 μs           ~20 μA
STANDBY       OFF    OFF    OFF            ~50 μs          ~2 μA

* In STOP mode, RTC and some wake-up peripherals remain active
```

### 14.2.2 Sleep Mode with FreeRTOS

```c
/* Simple approach: WFI in idle hook (Sleep mode only) */
#define configUSE_IDLE_HOOK    1

void vApplicationIdleHook(void)
{
    /* Enter Sleep mode — SysTick wakes us every tick */
    __WFI();
}
```

### 14.2.3 STOP Mode with Tickless

```c
/* Custom tickless implementation using LPTIM for STOP mode */
void vPortSuppressTicksAndSleep(TickType_t xExpectedIdleTime)
{
    /* 1. Calculate sleep duration */
    uint32_t ulSleepTicks = xExpectedIdleTime;
    
    /* 2. Configure LPTIM to wake after ulSleepTicks */
    HAL_LPTIM_Counter_Start_IT(&hlptim1, ulSleepTicks);
    
    /* 3. Enter STOP mode */
    HAL_SuspendTick();
    HAL_PWR_EnterSTOPMode(PWR_LOWPOWERREGULATOR_ON, PWR_STOPENTRY_WFI);
    
    /* 4. Woken up! Reconfigure clocks */
    SystemClock_Config();
    HAL_ResumeTick();
    
    /* 5. Compensate tick counter */
    uint32_t ulActualSleep = HAL_LPTIM_ReadCounter(&hlptim1);
    vTaskStepTick(ulActualSleep);
}
```

---

## 14.3 Power-Aware Design Patterns

```c
/* Event-driven design = natural power efficiency */
void Task_Sensor(void *pv)
{
    for (;;)
    {
        /* Block for 30 seconds — CPU can sleep entire time */
        vTaskDelay(pdMS_TO_TICKS(30000));
        
        /* Wake, sample, sleep */
        uint16_t sample = Read_Sensor();
        xQueueSend(xDataQueue, &sample, 0);
    }
}

/* ANTI-PATTERN: Busy polling (wastes power) */
void Task_Bad(void *pv)
{
    for (;;)
    {
        if (Is_Data_Ready()) { Process(); }
        /* CPU runs at full speed, spinning! */
    }
}
```

---

## 14.4 Chapter Summary

| Concept | Key Takeaway |
|---|---|
| **Tickless idle** | Suppresses tick during idle; enables deep sleep |
| **Sleep mode** | CPU off, peripherals on; ~1 μs wake-up |
| **STOP mode** | CPU + peripherals off; ~20 μA; needs clock reconfiguration |
| **Standby** | Everything off; RAM lost; lowest power |
| **Event-driven** | Tasks block → CPU sleeps → power saved naturally |
| **Design rule** | Never busy-poll; always block on events/delays |

---

*End of Chapter 14*
