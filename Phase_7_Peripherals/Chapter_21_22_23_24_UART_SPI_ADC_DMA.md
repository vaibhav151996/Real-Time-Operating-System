# Phase 7: Peripherals with RTOS

---

# Chapter 21: UART with RTOS

## 21.1 Queue + DMA Design

### 21.1.1 Architecture

```
UART RTOS ARCHITECTURE:
═══════════════════════

TRANSMIT PATH:
  Task A ──► TX Queue ──► UART Gateway Task ──► DMA ──► UART TX Pin
  Task B ──►    ↑
  Task C ──►    │
            Thread-safe

RECEIVE PATH:
  UART RX Pin ──► DMA ──► Circular Buffer ──► ISR ──► RX Queue ──► Parser Task
                                              (half/full complete callback)
```

### 21.1.2 Full Implementation

```c
/* ──── Configuration ──── */
#define UART_TX_QUEUE_SIZE    256
#define UART_RX_QUEUE_SIZE    256
#define UART_DMA_BUF_SIZE     64

QueueHandle_t    xUartTxQueue;
QueueHandle_t    xUartRxQueue;
TaskHandle_t     xUartTxTask;

uint8_t          uart_rx_dma_buf[UART_DMA_BUF_SIZE];
volatile uint16_t uart_rx_dma_old_pos = 0;

/* ──── Transmit: Queue → DMA ──── */
void Task_UartTransmit(void *pvParameters)
{
    uint8_t tx_buf[64];
    uint16_t count;

    for (;;)
    {
        count = 0;

        /* Collect bytes from queue into a batch buffer */
        while (count < sizeof(tx_buf))
        {
            TickType_t timeout = (count == 0) ? portMAX_DELAY : 0;
            if (xQueueReceive(xUartTxQueue, &tx_buf[count], timeout) == pdPASS)
            {
                count++;
            }
            else
            {
                break;    /* No more bytes waiting */
            }
        }

        if (count > 0)
        {
            /* Send batch via DMA */
            HAL_UART_Transmit_DMA(&huart2, tx_buf, count);

            /* Wait for DMA completion */
            ulTaskNotifyTake(pdTRUE, pdMS_TO_TICKS(1000));
        }
    }
}

/* DMA TX complete callback */
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)
{
    BaseType_t xWoken = pdFALSE;
    vTaskNotifyGiveFromISR(xUartTxTask, &xWoken);
    portYIELD_FROM_ISR(xWoken);
}

/* ──── Receive: DMA Circular Buffer → Queue ──── */
void HAL_UARTEx_RxEventCallback(UART_HandleTypeDef *huart, uint16_t Size)
{
    BaseType_t xWoken = pdFALSE;
    uint16_t old_pos = uart_rx_dma_old_pos;

    if (Size > old_pos)
    {
        /* Linear region: old_pos to Size */
        for (uint16_t i = old_pos; i < Size; i++)
        {
            xQueueSendFromISR(xUartRxQueue, &uart_rx_dma_buf[i], &xWoken);
        }
    }
    else
    {
        /* Wrapped: old_pos to end, then 0 to Size */
        for (uint16_t i = old_pos; i < UART_DMA_BUF_SIZE; i++)
        {
            xQueueSendFromISR(xUartRxQueue, &uart_rx_dma_buf[i], &xWoken);
        }
        for (uint16_t i = 0; i < Size; i++)
        {
            xQueueSendFromISR(xUartRxQueue, &uart_rx_dma_buf[i], &xWoken);
        }
    }

    uart_rx_dma_old_pos = Size;
    portYIELD_FROM_ISR(xWoken);
}

/* ──── Parser Task ──── */
void Task_UartParser(void *pvParameters)
{
    uint8_t byte;
    char line_buf[128];
    uint16_t idx = 0;

    for (;;)
    {
        if (xQueueReceive(xUartRxQueue, &byte, portMAX_DELAY) == pdPASS)
        {
            if (byte == '\n' || idx >= sizeof(line_buf) - 1)
            {
                line_buf[idx] = '\0';
                Process_Command(line_buf);
                idx = 0;
            }
            else if (byte != '\r')
            {
                line_buf[idx++] = byte;
            }
        }
    }
}

/* ──── Initialization ──── */
void UART_RTOS_Init(void)
{
    xUartTxQueue = xQueueCreate(UART_TX_QUEUE_SIZE, sizeof(uint8_t));
    xUartRxQueue = xQueueCreate(UART_RX_QUEUE_SIZE, sizeof(uint8_t));

    xTaskCreate(Task_UartTransmit, "UART_TX", 256, NULL, 3, &xUartTxTask);
    xTaskCreate(Task_UartParser,   "UART_RX", 512, NULL, 3, NULL);

    /* Start DMA receive in circular mode */
    HAL_UARTEx_ReceiveToIdle_DMA(&huart2, uart_rx_dma_buf, UART_DMA_BUF_SIZE);
    __HAL_DMA_DISABLE_IT(&hdma_usart2_rx, DMA_IT_HT);  /* Disable half-transfer if not needed */
}

/* ──── Helper: Thread-safe print ──── */
void UART_Printf(const char *fmt, ...)
{
    char buf[128];
    va_list args;
    va_start(args, fmt);
    int len = vsnprintf(buf, sizeof(buf), fmt, args);
    va_end(args);

    for (int i = 0; i < len; i++)
    {
        xQueueSend(xUartTxQueue, &buf[i], pdMS_TO_TICKS(10));
    }
}
```

---

# Chapter 22: SPI / I2C with RTOS

## 22.1 Mutex Protection

### 22.1.1 The Problem

SPI and I2C are **shared buses**. Multiple tasks accessing the same bus simultaneously corrupts communication.

### 22.1.2 Solution: Mutex-Protected Bus Access

```c
SemaphoreHandle_t xSpiMutex;

typedef struct {
    SPI_HandleTypeDef *hspi;
    GPIO_TypeDef      *cs_port;
    uint16_t           cs_pin;
} SPI_Device_t;

/* Thread-safe SPI transaction */
HAL_StatusTypeDef SPI_TransferSafe(SPI_Device_t *dev,
                                    uint8_t *tx, uint8_t *rx,
                                    uint16_t len)
{
    HAL_StatusTypeDef result;

    /* Take mutex (blocks if another device is using SPI) */
    xSemaphoreTake(xSpiMutex, portMAX_DELAY);

    /* Assert chip select */
    HAL_GPIO_WritePin(dev->cs_port, dev->cs_pin, GPIO_PIN_RESET);

    /* Transfer */
    result = HAL_SPI_TransmitReceive(dev->hspi, tx, rx, len, 1000);

    /* Deassert chip select */
    HAL_GPIO_WritePin(dev->cs_port, dev->cs_pin, GPIO_PIN_SET);

    /* Release mutex */
    xSemaphoreGive(xSpiMutex);

    return result;
}

/* Task A and Task B can safely share the SPI bus */
void Task_Accelerometer(void *pv)
{
    SPI_Device_t accel = { &hspi1, GPIOA, GPIO_PIN_4 };
    for (;;)
    {
        uint8_t tx[] = { 0x80 | REG_ACCEL_X, 0x00, 0x00 };
        uint8_t rx[3];
        SPI_TransferSafe(&accel, tx, rx, 3);
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}

void Task_Pressure(void *pv)
{
    SPI_Device_t press = { &hspi1, GPIOB, GPIO_PIN_0 };
    for (;;)
    {
        uint8_t tx[] = { 0x80 | REG_PRESSURE, 0x00, 0x00, 0x00 };
        uint8_t rx[4];
        SPI_TransferSafe(&press, tx, rx, 4);
        vTaskDelay(pdMS_TO_TICKS(50));
    }
}
```

---

# Chapter 23: ADC with RTOS

## 23.1 Sampling Strategies

```c
/* STRATEGY 1: Timer-triggered ADC with DMA + Task Notification */
/* ADC triggers automatically via TIM3 at exact intervals.
   DMA fills buffer. Half/Complete interrupt notifies task. */

#define ADC_BUF_SIZE    256
uint16_t adc_buffer[ADC_BUF_SIZE];
TaskHandle_t xAdcTask;

void HAL_ADC_ConvHalfCpltCallback(ADC_HandleTypeDef *hadc)
{
    BaseType_t xWoken = pdFALSE;
    /* First half of buffer ready */
    xTaskNotifyFromISR(xAdcTask, 0, eSetValueWithOverwrite, &xWoken);
    portYIELD_FROM_ISR(xWoken);
}

void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    BaseType_t xWoken = pdFALSE;
    /* Second half of buffer ready */
    xTaskNotifyFromISR(xAdcTask, 1, eSetValueWithOverwrite, &xWoken);
    portYIELD_FROM_ISR(xWoken);
}

void Task_AdcProcess(void *pvParameters)
{
    uint32_t half;
    for (;;)
    {
        xTaskNotifyWait(0, 0xFFFFFFFF, &half, portMAX_DELAY);

        uint16_t *buf = (half == 0) ? &adc_buffer[0]
                                     : &adc_buffer[ADC_BUF_SIZE / 2];
        uint16_t count = ADC_BUF_SIZE / 2;

        float avg = 0;
        for (int i = 0; i < count; i++)
            avg += buf[i];
        avg /= count;

        /* Process average... */
    }
}

/* Setup: Start ADC with DMA in circular mode */
void ADC_RTOS_Init(void)
{
    xTaskCreate(Task_AdcProcess, "ADC", 256, NULL, 4, &xAdcTask);
    HAL_TIM_Base_Start(&htim3);
    HAL_ADC_Start_DMA(&hadc1, (uint32_t *)adc_buffer, ADC_BUF_SIZE);
}
```

---

# Chapter 24: DMA with RTOS

## 24.1 Double Buffering

```
DOUBLE BUFFERING:
═════════════════

  DMA fills Buffer A    │    Task processes Buffer A
  Task processes Buffer B│    DMA fills Buffer B

  ┌──────────┐          ┌──────────┐
  │ Buffer A │ ◄── DMA  │ Buffer A │ ◄── Task (reading)
  │ (filling) │          │ (reading) │
  └──────────┘          └──────────┘
  ┌──────────┐          ┌──────────┐
  │ Buffer B │ ◄── Task │ Buffer B │ ◄── DMA (filling)
  │ (reading) │          │ (filling) │
  └──────────┘          └──────────┘

  DMA circular mode achieves this automatically:
  - Half-transfer interrupt → process first half
  - Transfer complete interrupt → process second half
  
  NO data is EVER lost or corrupted because the task
  processes ONE half while DMA fills the OTHER half.
```

```c
/* DMA circular mode double-buffering is exactly what
   HAL_ADC_Start_DMA with HAL_ADC_ConvHalfCpltCallback
   and HAL_ADC_ConvCpltCallback implements (Chapter 23 above). */
```

---

*End of Phase 7 (Chapters 21-24)*
