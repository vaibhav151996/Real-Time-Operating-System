# Phase 9: Complete Projects

---

# Project 1: Multitasking LED Controller

## Requirements

- 4 LEDs with independent blink patterns (different frequencies)
- UART command interface to change patterns at runtime
- Button input to cycle through preset modes
- Clean task architecture with proper IPC

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                  LED CONTROLLER                      │
│                                                      │
│  ┌──────────┐   Command    ┌──────────────┐          │
│  │  UART    │──► Queue ───►│  Controller  │          │
│  │  Parser  │              │  Task        │          │
│  └──────────┘              │  (Priority 3)│          │
│                            └──────┬───────┘          │
│  ┌──────────┐                     │                  │
│  │  Button  │── Notification ─────┘                  │
│  │  ISR     │                     │                  │
│  └──────────┘              ┌──────▼───────┐          │
│                            │  LED Config  │          │
│                            │  (Shared     │          │
│                            │   struct +   │          │
│                            │   mutex)     │          │
│                            └──────┬───────┘          │
│                                   │                  │
│                  ┌────────────────┼────────────┐     │
│                  ▼                ▼            ▼     │
│            ┌──────────┐   ┌──────────┐  ┌─────────┐ │
│            │  LED 1   │   │  LED 2   │  │ LED 3/4 │ │
│            │  Task    │   │  Task    │  │ Task    │ │
│            └──────────┘   └──────────┘  └─────────┘ │
└─────────────────────────────────────────────────────┘
```

## Task Breakdown

| Task | Priority | Stack | Period |
|------|----------|-------|--------|
| UART Parser | 3 | 256 words | Event-driven |
| Controller | 3 | 128 words | Event-driven |
| LED 1 | 1 | 64 words | Configurable (100-2000ms) |
| LED 2 | 1 | 64 words | Configurable |
| LED 3/4 | 1 | 64 words | Configurable |

## Full Annotated Code

```c
/* ═══════════════════════════════════════════════════
 * PROJECT 1: MULTITASKING LED CONTROLLER
 * Platform: STM32F4xx + FreeRTOS
 * ═══════════════════════════════════════════════════ */

#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"
#include "semphr.h"
#include "stm32f4xx_hal.h"
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

/* ──── LED Configuration ──── */
typedef struct {
    GPIO_TypeDef *port;
    uint16_t      pin;
    uint32_t      period_ms;    /* Blink period in ms */
    uint8_t       enabled;
} LED_Config_t;

#define NUM_LEDS    4
LED_Config_t led_config[NUM_LEDS] = {
    { GPIOA, GPIO_PIN_5,  500, 1 },   /* LED1: 500ms default */
    { GPIOA, GPIO_PIN_6, 1000, 1 },   /* LED2: 1000ms default */
    { GPIOA, GPIO_PIN_7,  250, 1 },   /* LED3: 250ms */
    { GPIOB, GPIO_PIN_0, 2000, 1 },   /* LED4: 2000ms */
};

SemaphoreHandle_t xConfigMutex;

/* ──── Command Queue ──── */
typedef struct {
    uint8_t led_index;
    uint32_t new_period;
    uint8_t  enable;
} Command_t;

QueueHandle_t xCommandQueue;

/* ──── Preset Modes ──── */
typedef struct {
    uint32_t periods[NUM_LEDS];
} Mode_t;

const Mode_t modes[] = {
    {{ 500, 1000,  250, 2000 }},   /* Mode 0: Default */
    {{ 100,  100,  100,  100 }},   /* Mode 1: Fast all */
    {{ 1000, 1000, 1000, 1000 }},  /* Mode 2: Slow all */
    {{ 200,  400,  800, 1600 }},   /* Mode 3: Cascade */
};
#define NUM_MODES (sizeof(modes) / sizeof(modes[0]))

volatile uint8_t current_mode = 0;
TaskHandle_t xControllerTask;

/* ──── Button ISR ──── */
void EXTI0_IRQHandler(void)
{
    if (__HAL_GPIO_EXTI_GET_IT(GPIO_PIN_0))
    {
        __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_0);

        BaseType_t xWoken = pdFALSE;
        vTaskNotifyGiveFromISR(xControllerTask, &xWoken);
        portYIELD_FROM_ISR(xWoken);
    }
}

/* ──── Controller Task ──── */
void Task_Controller(void *pvParameters)
{
    Command_t cmd;

    for (;;)
    {
        /* Check button press (notification) or UART command (queue) */
        if (ulTaskNotifyTake(pdTRUE, 0))
        {
            /* Button pressed → cycle mode */
            current_mode = (current_mode + 1) % NUM_MODES;

            xSemaphoreTake(xConfigMutex, portMAX_DELAY);
            for (int i = 0; i < NUM_LEDS; i++)
            {
                led_config[i].period_ms = modes[current_mode].periods[i];
            }
            xSemaphoreGive(xConfigMutex);
        }

        if (xQueueReceive(xCommandQueue, &cmd, pdMS_TO_TICKS(100)) == pdPASS)
        {
            if (cmd.led_index < NUM_LEDS)
            {
                xSemaphoreTake(xConfigMutex, portMAX_DELAY);
                led_config[cmd.led_index].period_ms = cmd.new_period;
                led_config[cmd.led_index].enabled   = cmd.enable;
                xSemaphoreGive(xConfigMutex);
            }
        }
    }
}

/* ──── LED Task (one per LED or shared) ──── */
void Task_LED(void *pvParameters)
{
    uint8_t idx = (uint8_t)(uintptr_t)pvParameters;

    for (;;)
    {
        uint32_t period;
        uint8_t  enabled;

        xSemaphoreTake(xConfigMutex, portMAX_DELAY);
        period  = led_config[idx].period_ms;
        enabled = led_config[idx].enabled;
        xSemaphoreGive(xConfigMutex);

        if (enabled)
        {
            HAL_GPIO_TogglePin(led_config[idx].port, led_config[idx].pin);
        }
        else
        {
            HAL_GPIO_WritePin(led_config[idx].port,
                              led_config[idx].pin, GPIO_PIN_RESET);
        }

        vTaskDelay(pdMS_TO_TICKS(period / 2));
    }
}

/* ──── UART Parser Task ──── */
void Task_UartParser(void *pvParameters)
{
    char line[64];
    uint16_t idx = 0;
    uint8_t byte;

    for (;;)
    {
        if (HAL_UART_Receive(&huart2, &byte, 1, portMAX_DELAY) == HAL_OK)
        {
            if (byte == '\n' || idx >= sizeof(line) - 1)
            {
                line[idx] = '\0';

                /* Parse: "LED <n> <period> <enable>" */
                int led_n, period, enable;
                if (sscanf(line, "LED %d %d %d", &led_n, &period, &enable) == 3)
                {
                    Command_t cmd = {
                        .led_index  = led_n,
                        .new_period = period,
                        .enable     = enable,
                    };
                    xQueueSend(xCommandQueue, &cmd, 0);
                }

                idx = 0;
            }
            else if (byte != '\r')
            {
                line[idx++] = byte;
            }
        }
    }
}

/* ──── Initialization ──── */
void Project1_Init(void)
{
    xConfigMutex = xSemaphoreCreateMutex();
    xCommandQueue = xQueueCreate(8, sizeof(Command_t));

    xTaskCreate(Task_Controller, "CTRL", 128, NULL, 3, &xControllerTask);
    xTaskCreate(Task_UartParser, "UART", 256, NULL, 3, NULL);

    for (int i = 0; i < NUM_LEDS; i++)
    {
        char name[8];
        snprintf(name, sizeof(name), "LED%d", i);
        xTaskCreate(Task_LED, name, 64, (void *)(uintptr_t)i, 1, NULL);
    }
}
```

## Debugging Strategy

1. **LED not blinking**: Check GPIO init, verify `period_ms` with debugger
2. **Commands not working**: Print received bytes to confirm UART reception
3. **All LEDs same rate**: Verify each task gets different `pvParameters` index
4. **Button bounce**: Add software debounce (50ms delay after first detection)

---

# Project 2: UART Command-Line Interface (CLI)

## Requirements

- Interactive CLI over UART (115200 baud)
- Commands: `help`, `status`, `led <on|off>`, `freq <hz>`, `stats`, `reset`
- Tab completion and command history (optional)
- `stats` shows FreeRTOS runtime statistics

## Full Code

```c
/* ═══════════════════════════════════════════════════
 * PROJECT 2: UART CLI WITH FREERTOS
 * ═══════════════════════════════════════════════════ */

#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"
#include <stdio.h>
#include <string.h>

extern UART_HandleTypeDef huart2;

/* Thread-safe printf */
SemaphoreHandle_t xPrintMutex;

void cli_printf(const char *fmt, ...)
{
    char buf[256];
    va_list args;
    va_start(args, fmt);
    int len = vsnprintf(buf, sizeof(buf), fmt, args);
    va_end(args);

    xSemaphoreTake(xPrintMutex, portMAX_DELAY);
    HAL_UART_Transmit(&huart2, (uint8_t *)buf, len, 1000);
    xSemaphoreGive(xPrintMutex);
}

/* ──── Command Table ──── */
typedef struct {
    const char *name;
    const char *help;
    void (*handler)(int argc, char *argv[]);
} CLI_Command_t;

void cmd_help(int argc, char *argv[]);
void cmd_status(int argc, char *argv[]);
void cmd_led(int argc, char *argv[]);
void cmd_stats(int argc, char *argv[]);
void cmd_reset(int argc, char *argv[]);

const CLI_Command_t commands[] = {
    { "help",   "Show available commands",       cmd_help   },
    { "status", "Show system status",            cmd_status },
    { "led",    "led <on|off> - Control LED",    cmd_led    },
    { "stats",  "Show FreeRTOS task statistics", cmd_stats  },
    { "reset",  "Reset the system",              cmd_reset  },
};
#define NUM_COMMANDS (sizeof(commands) / sizeof(commands[0]))

/* ──── Command Handlers ──── */
void cmd_help(int argc, char *argv[])
{
    cli_printf("\r\n=== Available Commands ===\r\n");
    for (int i = 0; i < NUM_COMMANDS; i++)
    {
        cli_printf("  %-10s %s\r\n", commands[i].name, commands[i].help);
    }
}

void cmd_status(int argc, char *argv[])
{
    cli_printf("\r\n=== System Status ===\r\n");
    cli_printf("  Uptime:     %lu ms\r\n", xTaskGetTickCount() * portTICK_PERIOD_MS);
    cli_printf("  Free Heap:  %u bytes\r\n", xPortGetFreeHeapSize());
    cli_printf("  Min Heap:   %u bytes\r\n", xPortGetMinimumEverFreeHeapSize());
    cli_printf("  Tasks:      %lu\r\n", uxTaskGetNumberOfTasks());
}

void cmd_led(int argc, char *argv[])
{
    if (argc < 2)
    {
        cli_printf("  Usage: led <on|off>\r\n");
        return;
    }

    if (strcmp(argv[1], "on") == 0)
    {
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET);
        cli_printf("  LED ON\r\n");
    }
    else if (strcmp(argv[1], "off") == 0)
    {
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_RESET);
        cli_printf("  LED OFF\r\n");
    }
    else
    {
        cli_printf("  Unknown option: %s\r\n", argv[1]);
    }
}

void cmd_stats(int argc, char *argv[])
{
    #if (configUSE_TRACE_FACILITY == 1) && (configGENERATE_RUN_TIME_STATS == 1)
    char buf[512];
    cli_printf("\r\n=== Task Statistics ===\r\n");
    cli_printf("  %-16s %8s %8s %%CPU\r\n", "Task", "State", "Priority");
    vTaskGetRunTimeStats(buf);
    cli_printf("%s\r\n", buf);
    #else
    cli_printf("  Runtime stats not enabled in FreeRTOSConfig.h\r\n");
    #endif
}

void cmd_reset(int argc, char *argv[])
{
    cli_printf("  Resetting in 1 second...\r\n");
    vTaskDelay(pdMS_TO_TICKS(1000));
    NVIC_SystemReset();
}

/* ──── CLI Engine ──── */
#define MAX_ARGS    8

void cli_parse_and_execute(char *line)
{
    char *argv[MAX_ARGS];
    int argc = 0;

    /* Tokenize */
    char *token = strtok(line, " \t");
    while (token && argc < MAX_ARGS)
    {
        argv[argc++] = token;
        token = strtok(NULL, " \t");
    }

    if (argc == 0) return;

    /* Find and execute command */
    for (int i = 0; i < NUM_COMMANDS; i++)
    {
        if (strcmp(argv[0], commands[i].name) == 0)
        {
            commands[i].handler(argc, argv);
            return;
        }
    }

    cli_printf("  Unknown command: '%s'. Type 'help'.\r\n", argv[0]);
}

/* ──── CLI Task ──── */
void Task_CLI(void *pvParameters)
{
    char line[128];
    uint16_t idx = 0;
    uint8_t byte;

    cli_printf("\r\n╔═══════════════════════════╗\r\n");
    cli_printf("║  STM32 FreeRTOS CLI v1.0  ║\r\n");
    cli_printf("╚═══════════════════════════╝\r\n");
    cli_printf("Type 'help' for commands.\r\n\r\n");
    cli_printf("stm32> ");

    for (;;)
    {
        if (HAL_UART_Receive(&huart2, &byte, 1, portMAX_DELAY) == HAL_OK)
        {
            /* Echo character */
            HAL_UART_Transmit(&huart2, &byte, 1, 10);

            if (byte == '\r' || byte == '\n')
            {
                cli_printf("\r\n");
                line[idx] = '\0';
                if (idx > 0)
                {
                    cli_parse_and_execute(line);
                }
                idx = 0;
                cli_printf("stm32> ");
            }
            else if (byte == 0x7F || byte == '\b')  /* Backspace */
            {
                if (idx > 0)
                {
                    idx--;
                    cli_printf("\b \b");
                }
            }
            else if (idx < sizeof(line) - 1)
            {
                line[idx++] = byte;
            }
        }
    }
}

void Project2_Init(void)
{
    xPrintMutex = xSemaphoreCreateMutex();
    xTaskCreate(Task_CLI, "CLI", 512, NULL, 2, NULL);
}
```

---

# Project 3: Data Logger with SD Card

## Requirements

- Read 3 analog sensors (temperature, pressure, humidity) every 1 second
- Log timestamped CSV data to SD card via FatFS
- UART status reporting every 5 seconds
- Button to start/stop logging
- LED indicator: slow blink = idle, fast blink = logging

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    DATA LOGGER                            │
│                                                           │
│  ┌──────────┐     ┌──────────────┐     ┌──────────────┐  │
│  │  Sensor  │────►│  Data Queue  │────►│  SD Logger   │  │
│  │  Task    │     │  (LogEntry)  │     │  Task        │  │
│  │  1 Hz    │     └──────────────┘     │  (Batched    │  │
│  └──────────┘                          │   writes)    │  │
│                                        └──────────────┘  │
│  ┌──────────┐                          ┌──────────────┐  │
│  │  Button  │──── Notification ──────►│  Control     │  │
│  │  ISR     │                          │  Task        │  │
│  └──────────┘                          └──────┬───────┘  │
│                                               │          │
│  ┌──────────┐                          ┌──────▼───────┐  │
│  │  Status  │                          │  LED Task    │  │
│  │  Task    │                          │  (indicates  │  │
│  │  5 Hz    │                          │   state)     │  │
│  └──────────┘                          └──────────────┘  │
└──────────────────────────────────────────────────────────┘
```

## Full Code

```c
/* ═══════════════════════════════════════════════════
 * PROJECT 3: DATA LOGGER WITH SD CARD
 * ═══════════════════════════════════════════════════ */

#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"
#include "semphr.h"
#include "event_groups.h"
#include "fatfs.h"
#include <stdio.h>

/* ──── Data Structures ──── */
typedef struct {
    uint32_t timestamp_ms;
    float    temperature;
    float    pressure;
    float    humidity;
} SensorData_t;

/* ──── Shared State ──── */
volatile uint8_t logging_active = 0;
QueueHandle_t    xDataQueue;
TaskHandle_t     xControlTask;

/* ──── Sensor Reading Task ──── */
void Task_Sensor(void *pvParameters)
{
    TickType_t xLastWake = xTaskGetTickCount();

    for (;;)
    {
        if (logging_active)
        {
            SensorData_t data;
            data.timestamp_ms = xTaskGetTickCount() * portTICK_PERIOD_MS;

            /* Read ADC channels (simulated conversion) */
            HAL_ADC_Start(&hadc1);
            HAL_ADC_PollForConversion(&hadc1, 10);
            uint16_t raw_temp = HAL_ADC_GetValue(&hadc1);

            HAL_ADC_Start(&hadc1);
            HAL_ADC_PollForConversion(&hadc1, 10);
            uint16_t raw_press = HAL_ADC_GetValue(&hadc1);

            HAL_ADC_Start(&hadc1);
            HAL_ADC_PollForConversion(&hadc1, 10);
            uint16_t raw_hum = HAL_ADC_GetValue(&hadc1);

            /* Convert to engineering units */
            data.temperature = (raw_temp / 4095.0f) * 100.0f;   /* 0-100°C */
            data.pressure    = (raw_press / 4095.0f) * 1100.0f;  /* 0-1100 hPa */
            data.humidity    = (raw_hum / 4095.0f) * 100.0f;     /* 0-100% */

            xQueueSend(xDataQueue, &data, 0);
        }

        vTaskDelayUntil(&xLastWake, pdMS_TO_TICKS(1000));
    }
}

/* ──── SD Card Logger Task ──── */
void Task_SDLogger(void *pvParameters)
{
    FATFS fs;
    FIL file;
    UINT bw;
    SensorData_t data;
    char line[128];
    uint16_t write_count = 0;
    uint8_t file_open = 0;

    f_mount(&fs, "", 1);

    for (;;)
    {
        if (xQueueReceive(xDataQueue, &data, pdMS_TO_TICKS(5000)) == pdPASS)
        {
            if (!file_open)
            {
                /* Create new file with timestamp name */
                char fname[32];
                snprintf(fname, sizeof(fname), "LOG_%05lu.csv",
                         data.timestamp_ms / 1000);
                f_open(&file, fname,
                       FA_WRITE | FA_CREATE_ALWAYS);
                f_write(&file, "Time_ms,Temp_C,Press_hPa,Hum_pct\n",
                        35, &bw);
                file_open = 1;
            }

            int len = snprintf(line, sizeof(line),
                              "%lu,%.2f,%.2f,%.2f\n",
                              data.timestamp_ms,
                              data.temperature,
                              data.pressure,
                              data.humidity);
            f_write(&file, line, len, &bw);
            write_count++;

            /* Flush every 10 entries */
            if (write_count >= 10)
            {
                f_sync(&file);
                write_count = 0;
            }
        }
        else
        {
            /* Timeout → flush and close if logging stopped */
            if (file_open && !logging_active)
            {
                f_close(&file);
                file_open = 0;
            }
            else if (file_open)
            {
                f_sync(&file);
            }
        }
    }
}

/* ──── Control Task ──── */
void Task_Control(void *pvParameters)
{
    for (;;)
    {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);

        /* Debounce */
        vTaskDelay(pdMS_TO_TICKS(50));

        /* Toggle logging */
        logging_active = !logging_active;
    }
}

/* ──── LED Indicator Task ──── */
void Task_LED_Indicator(void *pvParameters)
{
    for (;;)
    {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);

        if (logging_active)
            vTaskDelay(pdMS_TO_TICKS(100));    /* Fast blink = logging */
        else
            vTaskDelay(pdMS_TO_TICKS(1000));   /* Slow blink = idle */
    }
}

/* ──── Status Reporting Task ──── */
void Task_Status(void *pvParameters)
{
    for (;;)
    {
        printf("[STATUS] Logging: %s | Heap: %u | Queue: %lu/%lu\r\n",
               logging_active ? "ACTIVE" : "IDLE",
               xPortGetFreeHeapSize(),
               uxQueueMessagesWaiting(xDataQueue),
               (unsigned long)uxQueueSpacesAvailable(xDataQueue));

        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}

/* Button EXTI ISR */
void EXTI0_IRQHandler(void)
{
    if (__HAL_GPIO_EXTI_GET_IT(GPIO_PIN_0))
    {
        __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_0);
        BaseType_t xWoken = pdFALSE;
        vTaskNotifyGiveFromISR(xControlTask, &xWoken);
        portYIELD_FROM_ISR(xWoken);
    }
}

void Project3_Init(void)
{
    xDataQueue = xQueueCreate(32, sizeof(SensorData_t));

    xTaskCreate(Task_Sensor,        "SENSOR", 256, NULL, 3, NULL);
    xTaskCreate(Task_SDLogger,      "SDLOG",  1024, NULL, 2, NULL);
    xTaskCreate(Task_Control,       "CTRL",   128, NULL, 4, &xControlTask);
    xTaskCreate(Task_LED_Indicator, "LED",    64,  NULL, 1, NULL);
    xTaskCreate(Task_Status,        "STATUS", 256, NULL, 1, NULL);
}
```

---

# Project 4: Advanced Multi-Sensor RTOS System

## Requirements

- 3 sensor tasks (accelerometer SPI, temperature I2C, light ADC)
- Data fusion task combining all sensor data
- UART CLI for configuration and monitoring
- SD card logging with configurable rate
- Watchdog monitoring all critical tasks
- Low-power mode when idle
- Runtime statistics and self-diagnostics

## Architecture

```
 ┌─────────────────────────────────────────────────────────────────────┐
 │              ADVANCED MULTI-SENSOR RTOS SYSTEM                      │
 │                                                                     │
 │  SENSOR LAYER            PROCESSING LAYER        OUTPUT LAYER       │
 │  ┌───────────┐           ┌──────────────┐       ┌──────────────┐   │
 │  │ Accel     │──queue──►│              │      │  CLI Task    │   │
 │  │ (SPI) 50Hz│           │  Fusion      │      │  (UART)      │   │
 │  ├───────────┤           │  Task        │──►   ├──────────────┤   │
 │  │ Temp      │──queue──►│  (combines   │      │  SD Logger   │   │
 │  │ (I2C) 1Hz │           │   all data)  │      │  Task        │   │
 │  ├───────────┤           │              │      ├──────────────┤   │
 │  │ Light     │──queue──►│              │      │  Alarm Task  │   │
 │  │ (ADC) 10Hz│           └──────────────┘      └──────────────┘   │
 │  └───────────┘                                                     │
 │                                                                     │
 │  SYSTEM LAYER                                                       │
 │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌─────────────────┐ │
 │  │ Watchdog  │  │ Power Mgr │  │ Error Hndl│  │ Stats/Diag     │ │
 │  └───────────┘  └───────────┘  └───────────┘  └─────────────────┘ │
 └─────────────────────────────────────────────────────────────────────┘
```

## Priority Assignment

| Task | Priority | Rationale |
|------|----------|-----------|
| Watchdog | 5 (highest) | Must always run to feed IWDG |
| Accelerometer | 4 | Highest sample rate (50 Hz) |
| Light Sensor | 3 | Medium rate (10 Hz) |
| Fusion | 3 | Process combined data quickly |
| Temperature | 2 | Low rate (1 Hz) |
| Alarm | 2 | React to threshold breaches |
| CLI | 2 | User interaction |
| SD Logger | 1 | Background storage |
| LED/Power | 1 | Low priority housekeeping |

## Core Code

```c
/* ═══════════════════════════════════════════════════
 * PROJECT 4: ADVANCED MULTI-SENSOR RTOS SYSTEM
 * ═══════════════════════════════════════════════════ */

#include "FreeRTOS.h"
#include "task.h"
#include "queue.h"
#include "semphr.h"
#include "event_groups.h"
#include "timers.h"

/* ──── Data Types ──── */
typedef enum {
    SENSOR_ACCEL,
    SENSOR_TEMP,
    SENSOR_LIGHT,
    SENSOR_COUNT
} SensorType_t;

typedef struct {
    SensorType_t type;
    uint32_t     timestamp;
    union {
        struct { float x, y, z; } accel;
        float temperature;
        float light_lux;
    } data;
} SensorReading_t;

typedef struct {
    uint32_t timestamp;
    float    accel_magnitude;
    float    temperature;
    float    light_lux;
    uint8_t  alarm_flags;
} FusedData_t;

/* ──── Queues ──── */
QueueHandle_t xSensorQueue;      /* All sensors → Fusion */
QueueHandle_t xLogQueue;         /* Fusion → SD Logger */
QueueHandle_t xAlarmQueue;       /* Fusion → Alarm */

/* ──── Mutexes ──── */
SemaphoreHandle_t xSpiMutex;
SemaphoreHandle_t xI2cMutex;

/* ──── Watchdog Event Group ──── */
#define WD_ACCEL    (1 << 0)
#define WD_TEMP     (1 << 1)
#define WD_LIGHT    (1 << 2)
#define WD_FUSION   (1 << 3)
#define WD_ALL      (WD_ACCEL | WD_TEMP | WD_LIGHT | WD_FUSION)

EventGroupHandle_t xWatchdogFlags;

/* ──── Accelerometer Task (SPI, 50 Hz) ──── */
void Task_Accelerometer(void *pvParameters)
{
    TickType_t xLastWake = xTaskGetTickCount();

    for (;;)
    {
        SensorReading_t reading;
        reading.type = SENSOR_ACCEL;
        reading.timestamp = xTaskGetTickCount();

        /* SPI transfer (mutex-protected) */
        xSemaphoreTake(xSpiMutex, portMAX_DELAY);
        uint8_t tx[7] = { 0x80 | 0x28, 0,0,0,0,0,0 };  /* Read 6 bytes */
        uint8_t rx[7];
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_RESET);
        HAL_SPI_TransmitReceive(&hspi1, tx, rx, 7, 100);
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_SET);
        xSemaphoreGive(xSpiMutex);

        /* Convert raw to g (example: ±2g range, 16-bit) */
        int16_t raw_x = (rx[2] << 8) | rx[1];
        int16_t raw_y = (rx[4] << 8) | rx[3];
        int16_t raw_z = (rx[6] << 8) | rx[5];
        reading.data.accel.x = raw_x * 0.061f / 1000.0f;
        reading.data.accel.y = raw_y * 0.061f / 1000.0f;
        reading.data.accel.z = raw_z * 0.061f / 1000.0f;

        xQueueSend(xSensorQueue, &reading, 0);
        xEventGroupSetBits(xWatchdogFlags, WD_ACCEL);

        vTaskDelayUntil(&xLastWake, pdMS_TO_TICKS(20));  /* 50 Hz */
    }
}

/* ──── Temperature Task (I2C, 1 Hz) ──── */
void Task_Temperature(void *pvParameters)
{
    TickType_t xLastWake = xTaskGetTickCount();

    for (;;)
    {
        SensorReading_t reading;
        reading.type = SENSOR_TEMP;
        reading.timestamp = xTaskGetTickCount();

        xSemaphoreTake(xI2cMutex, portMAX_DELAY);
        uint8_t raw[2];
        HAL_I2C_Mem_Read(&hi2c1, 0x48 << 1, 0x00, 1, raw, 2, 100);
        xSemaphoreGive(xI2cMutex);

        int16_t temp_raw = (raw[0] << 4) | (raw[1] >> 4);
        if (temp_raw & 0x800) temp_raw |= 0xF000;
        reading.data.temperature = temp_raw * 0.0625f;

        xQueueSend(xSensorQueue, &reading, 0);
        xEventGroupSetBits(xWatchdogFlags, WD_TEMP);

        vTaskDelayUntil(&xLastWake, pdMS_TO_TICKS(1000));
    }
}

/* ──── Light Sensor Task (ADC, 10 Hz) ──── */
void Task_Light(void *pvParameters)
{
    TickType_t xLastWake = xTaskGetTickCount();

    for (;;)
    {
        SensorReading_t reading;
        reading.type = SENSOR_LIGHT;
        reading.timestamp = xTaskGetTickCount();

        HAL_ADC_Start(&hadc1);
        HAL_ADC_PollForConversion(&hadc1, 10);
        uint16_t raw = HAL_ADC_GetValue(&hadc1);

        reading.data.light_lux = (raw / 4095.0f) * 10000.0f;

        xQueueSend(xSensorQueue, &reading, 0);
        xEventGroupSetBits(xWatchdogFlags, WD_LIGHT);

        vTaskDelayUntil(&xLastWake, pdMS_TO_TICKS(100));
    }
}

/* ──── Fusion Task ──── */
void Task_Fusion(void *pvParameters)
{
    FusedData_t fused = { 0 };
    SensorReading_t reading;
    #include <math.h>

    for (;;)
    {
        if (xQueueReceive(xSensorQueue, &reading, pdMS_TO_TICKS(500)) == pdPASS)
        {
            fused.timestamp = reading.timestamp;

            switch (reading.type)
            {
                case SENSOR_ACCEL:
                    fused.accel_magnitude = sqrtf(
                        reading.data.accel.x * reading.data.accel.x +
                        reading.data.accel.y * reading.data.accel.y +
                        reading.data.accel.z * reading.data.accel.z
                    );
                    break;

                case SENSOR_TEMP:
                    fused.temperature = reading.data.temperature;
                    break;

                case SENSOR_LIGHT:
                    fused.light_lux = reading.data.light_lux;
                    break;

                default:
                    break;
            }

            /* Check alarm thresholds */
            fused.alarm_flags = 0;
            if (fused.temperature > 50.0f)    fused.alarm_flags |= 0x01;
            if (fused.accel_magnitude > 3.0f) fused.alarm_flags |= 0x02;
            if (fused.light_lux < 10.0f)      fused.alarm_flags |= 0x04;

            /* Send to logger */
            xQueueSend(xLogQueue, &fused, 0);

            /* Send alarms if any */
            if (fused.alarm_flags)
            {
                xQueueSend(xAlarmQueue, &fused, 0);
            }

            xEventGroupSetBits(xWatchdogFlags, WD_FUSION);
        }
    }
}

/* ──── Alarm Task ──── */
void Task_Alarm(void *pvParameters)
{
    FusedData_t fused;

    for (;;)
    {
        if (xQueueReceive(xAlarmQueue, &fused, portMAX_DELAY) == pdPASS)
        {
            if (fused.alarm_flags & 0x01)
                printf("[ALARM] Temperature: %.1f°C > 50°C!\r\n",
                       fused.temperature);
            if (fused.alarm_flags & 0x02)
                printf("[ALARM] Vibration: %.2fg > 3g!\r\n",
                       fused.accel_magnitude);
            if (fused.alarm_flags & 0x04)
                printf("[ALARM] Light: %.0f lux < 10 lux!\r\n",
                       fused.light_lux);

            /* Could trigger buzzer, send notification, etc. */
        }
    }
}

/* ──── Watchdog Task ──── */
void Task_Watchdog(void *pvParameters)
{
    IWDG_HandleTypeDef hiwdg;
    hiwdg.Instance = IWDG;
    hiwdg.Init.Prescaler = IWDG_PRESCALER_128;
    hiwdg.Init.Reload = 1500;   /* ~6 second timeout */
    HAL_IWDG_Init(&hiwdg);

    for (;;)
    {
        EventBits_t bits = xEventGroupWaitBits(
            xWatchdogFlags, WD_ALL, pdTRUE, pdTRUE,
            pdMS_TO_TICKS(5000));

        if ((bits & WD_ALL) == WD_ALL)
        {
            HAL_IWDG_Refresh(&hiwdg);
        }
        else
        {
            printf("[WATCHDOG] Task failure! Bits: 0x%02X\r\n",
                   (unsigned int)bits);
            /* Log which task failed, don't feed → reset */
        }
    }
}

/* ──── System Initialization ──── */
void Project4_Init(void)
{
    /* Create IPC */
    xSensorQueue   = xQueueCreate(64, sizeof(SensorReading_t));
    xLogQueue      = xQueueCreate(32, sizeof(FusedData_t));
    xAlarmQueue    = xQueueCreate(8,  sizeof(FusedData_t));
    xSpiMutex      = xSemaphoreCreateMutex();
    xI2cMutex      = xSemaphoreCreateMutex();
    xWatchdogFlags = xEventGroupCreate();

    /* Create tasks */
    xTaskCreate(Task_Watchdog,      "WDG",    128,  NULL, 5, NULL);
    xTaskCreate(Task_Accelerometer, "ACCEL",  256,  NULL, 4, NULL);
    xTaskCreate(Task_Light,         "LIGHT",  128,  NULL, 3, NULL);
    xTaskCreate(Task_Fusion,        "FUSE",   512,  NULL, 3, NULL);
    xTaskCreate(Task_Temperature,   "TEMP",   256,  NULL, 2, NULL);
    xTaskCreate(Task_Alarm,         "ALARM",  256,  NULL, 2, NULL);
    /* CLI and Logger tasks would also be created here */

    /* Total RAM estimate:
     * Queues:  64*28 + 32*20 + 8*20 = 2592 bytes
     * Stacks: (128+256+128+512+256+256) * 4 = 6144 bytes
     * TCBs:   6 * 92 = 552 bytes
     * Total:  ~9.3 KB
     * Fits comfortably in STM32F4 (128KB+ SRAM)
     */
}
```

---

*End of Phase 9 (Projects 1-4)*
