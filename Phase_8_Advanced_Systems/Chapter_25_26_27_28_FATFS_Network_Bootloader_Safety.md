# Phase 8: Advanced Systems

---

# Chapter 25: FATFS + FreeRTOS (SD Card File System)

## 25.1 Learning Objectives

- Integrate FatFS middleware with FreeRTOS
- Implement thread-safe file I/O
- Design a data logger with buffered writes

## 25.2 Architecture

```
DATA LOGGER ARCHITECTURE:
═════════════════════════

  Sensor Tasks ──► Data Queue ──► Logger Task ──► FatFS ──► SD Card (SPI/SDIO)
                    (buffered)        │
                                      ├── Mutex protects FatFS
                                      ├── Writes in batches (reduce wear)
                                      └── Flush on timer or buffer full
```

## 25.3 Thread-Safe FatFS Implementation

```c
/* FatFS provides ff_mutex hooks for FreeRTOS */
/* In ffconf.h: */
#define FF_FS_REENTRANT    1
#define FF_FS_TIMEOUT      1000
#define FF_SYNC_t          SemaphoreHandle_t

/* CubeMX generates the sync functions automatically.
   Manual implementation if needed: */

int ff_cre_syncobj(BYTE vol, FF_SYNC_t *sobj)
{
    *sobj = xSemaphoreCreateMutex();
    return (*sobj != NULL);
}

int ff_del_syncobj(FF_SYNC_t sobj)
{
    vSemaphoreDelete(sobj);
    return 1;
}

int ff_req_grant(FF_SYNC_t sobj)
{
    return (xSemaphoreTake(sobj, pdMS_TO_TICKS(FF_FS_TIMEOUT)) == pdTRUE);
}

void ff_rel_grant(FF_SYNC_t sobj)
{
    xSemaphoreGive(sobj);
}
```

## 25.4 Buffered Data Logger

```c
#define LOG_QUEUE_LEN    64
#define LOG_ENTRY_SIZE   64

typedef struct {
    uint32_t timestamp;
    float    temperature;
    float    pressure;
    float    humidity;
} LogEntry_t;

QueueHandle_t xLogQueue;

void Task_DataLogger(void *pvParameters)
{
    FIL file;
    LogEntry_t entry;
    UINT bw;
    char line[128];
    uint32_t write_count = 0;

    /* Mount filesystem */
    FATFS fs;
    f_mount(&fs, "", 1);

    /* Open file (append mode) */
    f_open(&file, "log.csv", FA_WRITE | FA_OPEN_APPEND);

    /* Write header if new file */
    if (f_size(&file) == 0)
    {
        f_write(&file, "Time,Temp,Press,Hum\n", 20, &bw);
    }

    for (;;)
    {
        if (xQueueReceive(xLogQueue, &entry, pdMS_TO_TICKS(5000)) == pdPASS)
        {
            int len = snprintf(line, sizeof(line),
                              "%lu,%.2f,%.2f,%.2f\n",
                              entry.timestamp,
                              entry.temperature,
                              entry.pressure,
                              entry.humidity);

            f_write(&file, line, len, &bw);
            write_count++;

            /* Flush every 10 writes to reduce SD wear */
            if (write_count >= 10)
            {
                f_sync(&file);
                write_count = 0;
            }
        }
        else
        {
            /* Timeout: flush pending data */
            f_sync(&file);
            write_count = 0;
        }
    }
}
```

### 25.5 Common Mistakes

| Mistake | Fix |
|---------|-----|
| Calling `f_open` from multiple tasks | Use a single gateway task |
| Not calling `f_sync` | Data lost on power failure |
| Stack too small for FatFS | FatFS needs ≥1KB stack |
| SPI clock too fast for SD | Start at 400kHz, then speed up |

---

# Chapter 26: Networking — LWIP + MQTT

## 26.1 Learning Objectives

- Integrate lwIP TCP/IP stack with FreeRTOS
- Implement MQTT publish/subscribe pattern
- Handle network events with RTOS primitives

## 26.2 Architecture

```
MQTT OVER LWIP + FREERTOS:
══════════════════════════

  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │ App Tasks │────►│  MQTT    │────►│  lwIP    │────► ETH / WiFi
  │ (Publish) │     │  Client  │     │  Stack   │
  └──────────┘     │  Task    │     │  (tcpip   │
                   │          │◄────│   thread) │◄─── Incoming
  ┌──────────┐     │          │     └──────────┘     packets
  │ Handler  │◄────│ Callback │
  │ Tasks    │     └──────────┘
  │(Subscribe)│
  └──────────┘
```

## 26.3 lwIP + FreeRTOS Integration

```c
/* lwipopts.h key settings for FreeRTOS */
#define NO_SYS                  0       /* Use OS */
#define LWIP_NETCONN            1       /* Enable netconn API */
#define LWIP_SOCKET             0       /* Disable sockets (save RAM) */
#define TCPIP_THREAD_STACKSIZE  1024
#define TCPIP_THREAD_PRIO       (configMAX_PRIORITIES - 2)
#define DEFAULT_THREAD_STACKSIZE 512
#define TCPIP_MBOX_SIZE         16
#define DEFAULT_UDP_RECVMBOX_SIZE 16
#define DEFAULT_TCP_RECVMBOX_SIZE 16

/* FreeRTOS-specific */
#define LWIP_FREERTOS_CHECK_CORE_LOCKING  1
#define LWIP_ASSERT_CORE_LOCKED()         /* Define if needed */
```

## 26.4 MQTT Client Task

```c
#include "lwip/apps/mqtt.h"

static mqtt_client_t *mqtt_client;
static TaskHandle_t   xMqttTask;

/* Connection callback */
static void mqtt_connection_cb(mqtt_client_t *client,
                                void *arg,
                                mqtt_connection_status_t status)
{
    if (status == MQTT_CONNECT_ACCEPTED)
    {
        /* Subscribe to topics */
        mqtt_subscribe(client, "device/command", 1, NULL, NULL);
    }
}

/* Incoming publish callback */
static void mqtt_incoming_publish_cb(void *arg,
                                      const char *topic,
                                      u32_t tot_len)
{
    /* Topic received, data follows in data callback */
}

static void mqtt_incoming_data_cb(void *arg,
                                   const u8_t *data,
                                   u16_t len,
                                   u8_t flags)
{
    /* Process incoming MQTT message */
    char buf[128];
    memcpy(buf, data, (len < 127) ? len : 127);
    buf[len < 127 ? len : 127] = '\0';

    /* Send to command queue for processing by another task */
    xQueueSend(xCommandQueue, buf, 0);
}

/* Publish sensor data */
void MQTT_PublishSensor(float temperature)
{
    char payload[32];
    snprintf(payload, sizeof(payload), "{\"temp\":%.2f}", temperature);

    mqtt_publish(mqtt_client, "device/sensor",
                 payload, strlen(payload),
                 1,     /* QoS 1 */
                 0,     /* Not retained */
                 NULL, NULL);
}

void Task_MQTT(void *pvParameters)
{
    ip_addr_t broker_ip;
    IP4_ADDR(&broker_ip, 192, 168, 1, 100);

    struct mqtt_connect_client_info_t ci = {
        .client_id   = "stm32_device",
        .keep_alive  = 60,
    };

    mqtt_client = mqtt_client_new();
    mqtt_set_inpub_callback(mqtt_client,
                            mqtt_incoming_publish_cb,
                            mqtt_incoming_data_cb, NULL);

    mqtt_client_connect(mqtt_client, &broker_ip, 1883,
                        mqtt_connection_cb, NULL, &ci);

    for (;;)
    {
        /* Periodic telemetry publishing */
        MQTT_PublishSensor(Read_Temperature());
        vTaskDelay(pdMS_TO_TICKS(10000));
    }
}
```

---

# Chapter 27: Bootloader + OTA Updates

## 27.1 Learning Objectives

- Understand STM32 dual-bank flash memory layout
- Implement a simple bootloader that validates and jumps to application
- Design OTA update mechanism with RTOS

## 27.2 Flash Memory Layout

```
STM32 FLASH LAYOUT FOR BOOTLOADER + APP:
═════════════════════════════════════════

  0x0800_0000 ┌─────────────────────┐
              │    BOOTLOADER       │  16KB
              │    (Validates app,  │
              │     jumps to app)   │
  0x0800_4000 ├─────────────────────┤
              │    APPLICATION      │  224KB
              │    (FreeRTOS app)   │
              │                     │
  0x0803_C000 ├─────────────────────┤
              │    UPDATE SLOT      │  224KB
              │    (Received OTA    │
              │     image stored    │
              │     here first)     │
  0x0807_8000 ├─────────────────────┤
              │    CONFIG / FLAGS   │  4KB
              │    (Update pending  │
              │     flag, CRC, ver) │
  0x0807_9000 └─────────────────────┘
```

## 27.3 Bootloader Core

```c
/* Bootloader code (runs WITHOUT FreeRTOS) */

#define APP_ADDRESS       0x08004000
#define UPDATE_ADDRESS    0x0803C000
#define CONFIG_ADDRESS    0x08078000

typedef struct {
    uint32_t update_pending;
    uint32_t app_crc;
    uint32_t app_size;
    uint32_t version;
} BootConfig_t;

uint32_t calculate_crc(uint32_t addr, uint32_t size)
{
    /* Use STM32 hardware CRC unit */
    __HAL_RCC_CRC_CLK_ENABLE();
    CRC->CR = CRC_CR_RESET;
    for (uint32_t i = 0; i < size / 4; i++)
    {
        CRC->DR = *(volatile uint32_t *)(addr + i * 4);
    }
    return CRC->DR;
}

void jump_to_app(uint32_t addr)
{
    /* Validate stack pointer (must be in RAM range) */
    uint32_t sp = *(volatile uint32_t *)addr;
    if (sp < 0x20000000 || sp > 0x20020000)
        return;  /* Invalid image */

    /* Get reset handler address */
    uint32_t reset = *(volatile uint32_t *)(addr + 4);

    /* Disable all interrupts */
    __disable_irq();

    /* Relocate vector table */
    SCB->VTOR = addr;

    /* Set stack pointer and jump */
    __set_MSP(sp);
    void (*app_entry)(void) = (void (*)(void))reset;
    app_entry();

    /* Never reaches here */
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();

    BootConfig_t *cfg = (BootConfig_t *)CONFIG_ADDRESS;

    if (cfg->update_pending == 0xDEADBEEF)
    {
        /* Validate update image CRC */
        uint32_t crc = calculate_crc(UPDATE_ADDRESS, cfg->app_size);
        if (crc == cfg->app_crc)
        {
            /* Erase application area */
            Flash_Erase(APP_ADDRESS, cfg->app_size);
            /* Copy update to application area */
            Flash_Copy(APP_ADDRESS, UPDATE_ADDRESS, cfg->app_size);
            /* Clear update flag */
            Flash_Erase(CONFIG_ADDRESS, sizeof(BootConfig_t));
        }
    }

    jump_to_app(APP_ADDRESS);
}
```

## 27.4 OTA Receive Task (Application Side)

```c
/* This runs inside the FreeRTOS application */
void Task_OTA_Receive(void *pvParameters)
{
    /* Receive firmware image over UART/MQTT/HTTP */
    uint32_t offset = 0;
    uint8_t chunk[256];
    uint32_t total_size;

    /* Wait for OTA start command */
    ulTaskNotifyTake(pdTRUE, portMAX_DELAY);

    /* Erase update slot */
    Flash_Erase(UPDATE_ADDRESS, 224 * 1024);

    /* Receive chunks */
    while (offset < total_size)
    {
        uint16_t len = OTA_ReceiveChunk(chunk, sizeof(chunk));
        HAL_FLASH_Unlock();
        for (int i = 0; i < len; i += 8)
        {
            HAL_FLASH_Program(FLASH_TYPEPROGRAM_DOUBLEWORD,
                              UPDATE_ADDRESS + offset + i,
                              *(uint64_t *)&chunk[i]);
        }
        HAL_FLASH_Lock();
        offset += len;
    }

    /* Write config flags */
    BootConfig_t cfg = {
        .update_pending = 0xDEADBEEF,
        .app_crc        = calculate_crc(UPDATE_ADDRESS, total_size),
        .app_size       = total_size,
        .version        = NEW_VERSION,
    };
    Flash_Write(CONFIG_ADDRESS, &cfg, sizeof(cfg));

    /* Reboot into bootloader */
    NVIC_SystemReset();
}
```

---

# Chapter 28: Safety — Watchdog + Fault Handling

## 28.1 Learning Objectives

- Implement Independent Watchdog (IWDG) with FreeRTOS
- Design task health monitoring
- Handle safety-critical scenarios

## 28.2 Watchdog Architecture

```
WATCHDOG MONITOR ARCHITECTURE:
══════════════════════════════

  Task A ──► Sets bit 0 ──┐
  Task B ──► Sets bit 1 ──┼──► Event Group ──► Watchdog Task ──► Feeds IWDG
  Task C ──► Sets bit 2 ──┘
                                            │
                                            ▼
                                    If ANY task misses
                                    its check-in:
                                    ┌──────────────────┐
                                    │ DON'T feed IWDG  │
                                    │ Log error        │
                                    │ System resets    │
                                    └──────────────────┘
```

## 28.3 Implementation

```c
#define TASK_A_ALIVE    (1 << 0)
#define TASK_B_ALIVE    (1 << 1)
#define TASK_C_ALIVE    (1 << 2)
#define ALL_TASKS_ALIVE (TASK_A_ALIVE | TASK_B_ALIVE | TASK_C_ALIVE)

EventGroupHandle_t xAliveFlags;
IWDG_HandleTypeDef hiwdg;

void Task_Watchdog(void *pvParameters)
{
    /* Configure IWDG: ~2 second timeout */
    hiwdg.Instance = IWDG;
    hiwdg.Init.Prescaler = IWDG_PRESCALER_64;
    hiwdg.Init.Reload = 1000;  /* ~2s at 32kHz/64 */
    HAL_IWDG_Init(&hiwdg);

    for (;;)
    {
        /* Wait for ALL tasks to check in within 1 second */
        EventBits_t bits = xEventGroupWaitBits(
            xAliveFlags,
            ALL_TASKS_ALIVE,
            pdTRUE,             /* Clear bits on exit */
            pdTRUE,             /* Wait for ALL bits */
            pdMS_TO_TICKS(1000)
        );

        if ((bits & ALL_TASKS_ALIVE) == ALL_TASKS_ALIVE)
        {
            /* All tasks alive — feed the dog */
            HAL_IWDG_Refresh(&hiwdg);
        }
        else
        {
            /* A task missed its deadline! */
            Error_Log("Task watchdog failure: 0x%02X", bits);
            /* Don't feed IWDG → system will reset */
        }
    }
}

/* Each monitored task periodically checks in: */
void Task_A(void *pvParameters)
{
    for (;;)
    {
        /* ... do work ... */
        xEventGroupSetBits(xAliveFlags, TASK_A_ALIVE);
        vTaskDelay(pdMS_TO_TICKS(200));
    }
}
```

## 28.4 Stack Overflow Detection

```c
/* In FreeRTOSConfig.h */
#define configCHECK_FOR_STACK_OVERFLOW  2

void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName)
{
    /* Save task name for post-mortem */
    volatile char *name = pcTaskName;

    /* Log to non-volatile memory */
    Error_LogNV("STACK OVERFLOW: %s", pcTaskName);

    /* Halt in debug, reset in production */
    #ifdef DEBUG
        __BKPT(0);
    #else
        NVIC_SystemReset();
    #endif
}
```

## 28.5 HardFault Post-Mortem

```c
/* Store crash info in a section that survives reset */
__attribute__((section(".noinit")))
volatile struct {
    uint32_t magic;
    uint32_t r0, r1, r2, r3;
    uint32_t r12, lr, pc, psr;
    uint32_t cfsr, hfsr, mmfar, bfar;
} crash_info;

void HardFault_Handler_C(uint32_t *stack)
{
    crash_info.magic = 0xCRASH01;
    crash_info.r0  = stack[0];
    crash_info.r1  = stack[1];
    crash_info.r2  = stack[2];
    crash_info.r3  = stack[3];
    crash_info.r12 = stack[4];
    crash_info.lr  = stack[5];
    crash_info.pc  = stack[6];
    crash_info.psr = stack[7];

    crash_info.cfsr  = SCB->CFSR;
    crash_info.hfsr  = SCB->HFSR;
    crash_info.mmfar = SCB->MMFAR;
    crash_info.bfar  = SCB->BFAR;

    NVIC_SystemReset();
}

/* On startup, check for crash info */
void Check_Crash_Report(void)
{
    if (crash_info.magic == 0xCRASH01)
    {
        UART_Printf("=== CRASH REPORT ===\r\n");
        UART_Printf("PC:   0x%08X\r\n", crash_info.pc);
        UART_Printf("LR:   0x%08X\r\n", crash_info.lr);
        UART_Printf("CFSR: 0x%08X\r\n", crash_info.cfsr);
        crash_info.magic = 0;  /* Clear */
    }
}
```

---

*End of Phase 8 (Chapters 25-28)*
