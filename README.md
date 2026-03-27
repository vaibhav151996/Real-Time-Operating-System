# Real-Time Operating System — Complete Textbook

**A professional-grade RTOS textbook: from absolute beginner to expert level using STM32 + FreeRTOS.**

Covers theory, hands-on code, debugging, system design, full projects, and interview preparation.

---

## Table of Contents

### Phase 1: RTOS Foundations
| Chapter | Topic | File |
|---------|-------|------|
| 1 | RTOS Basics — What, Why, When | [Chapter\_1\_RTOS\_Basics.md](Phase_1_RTOS_Foundations/Chapter_1_RTOS_Basics.md) |
| 2 | RTOS Architecture — Kernel, Scheduler, Tick | [Chapter\_2\_RTOS\_Architecture.md](Phase_1_RTOS_Foundations/Chapter_2_RTOS_Architecture.md) |
| 3 | Task Management — States, TCB, Stack | [Chapter\_3\_Task\_Management.md](Phase_1_RTOS_Foundations/Chapter_3_Task_Management.md) |

### Phase 2: FreeRTOS on STM32
| Chapter | Topic | File |
|---------|-------|------|
| 4 | Development Setup — CubeIDE, FreeRTOS Config | [Chapter\_4\_Setup.md](Phase_2_FreeRTOS_on_STM32/Chapter_4_Setup.md) |
| 5 | Tasks Deep Dive — Creation, Priorities, Patterns | [Chapter\_5\_Tasks.md](Phase_2_FreeRTOS_on_STM32/Chapter_5_Tasks.md) |
| 6 | Timing — Delays, Software Timers, Jitter | [Chapter\_6\_Timing.md](Phase_2_FreeRTOS_on_STM32/Chapter_6_Timing.md) |

### Phase 3: Inter-Task Communication
| Chapter | Topic | File |
|---------|-------|------|
| 7 | Queues — FIFO, Blocking, ISR Patterns | [Chapter\_7\_Queues.md](Phase_3_Inter_Task_Communication/Chapter_7_Queues.md) |
| 8 | Semaphores — Binary, Counting, ISR Deferral | [Chapter\_8\_Semaphores.md](Phase_3_Inter_Task_Communication/Chapter_8_Semaphores.md) |
| 9 | Mutex — Priority Inheritance, Deadlock Prevention | [Chapter\_9\_Mutex.md](Phase_3_Inter_Task_Communication/Chapter_9_Mutex.md) |
| 10 | Event Groups — Multi-Condition Sync, Rendezvous | [Chapter\_10\_Event\_Groups.md](Phase_3_Inter_Task_Communication/Chapter_10_Event_Groups.md) |

### Phase 4: Advanced RTOS Concepts
| Chapter | Topic | File |
|---------|-------|------|
| 11 | Interrupts + FreeRTOS — FromISR, Priority Config | [Chapter\_11\_Interrupts.md](Phase_4_Advanced_RTOS/Chapter_11_Interrupts.md) |
| 12 | Memory Management — heap\_1 to heap\_5 | [Chapter\_12\_Memory.md](Phase_4_Advanced_RTOS/Chapter_12_Memory.md) |
| 13 | Scheduling Theory — RMS, EDF, Priority Inversion | [Chapter\_13\_Scheduling\_Theory.md](Phase_4_Advanced_RTOS/Chapter_13_Scheduling_Theory.md) |
| 14 | Power Management — Tickless Idle, Low-Power Modes | [Chapter\_14\_Power\_Management.md](Phase_4_Advanced_RTOS/Chapter_14_Power_Management.md) |

### Phase 5: Debugging & Optimization
| Chapter | Topic | File |
|---------|-------|------|
| 15 | Debugging — HardFault, Stack Overflow, CFSR | [Chapter\_15\_Debugging.md](Phase_5_Debugging_Optimization/Chapter_15_Debugging.md) |
| 16–17 | Performance & Optimization | [Chapter\_16\_17\_Performance\_Optimization.md](Phase_5_Debugging_Optimization/Chapter_16_17_Performance_Optimization.md) |

### Phase 6: System Design
| Chapter | Topic | File |
|---------|-------|------|
| 18–20 | Design Patterns, Architecture, Real-Time Guarantees | [Chapter\_18\_19\_20\_Design\_Patterns\_Architecture\_Guarantees.md](Phase_6_System_Design/Chapter_18_19_20_Design_Patterns_Architecture_Guarantees.md) |

### Phase 7: Peripherals with RTOS
| Chapter | Topic | File |
|---------|-------|------|
| 21 | UART — Queue + DMA Design | [Chapter\_21\_22\_23\_24\_UART\_SPI\_ADC\_DMA.md](Phase_7_Peripherals/Chapter_21_22_23_24_UART_SPI_ADC_DMA.md) |
| 22 | SPI / I2C — Mutex Protection | [Chapter\_21\_22\_23\_24\_UART\_SPI\_ADC\_DMA.md](Phase_7_Peripherals/Chapter_21_22_23_24_UART_SPI_ADC_DMA.md) |
| 23 | ADC — Timer-Triggered DMA Sampling | [Chapter\_21\_22\_23\_24\_UART\_SPI\_ADC\_DMA.md](Phase_7_Peripherals/Chapter_21_22_23_24_UART_SPI_ADC_DMA.md) |
| 24 | DMA — Double Buffering | [Chapter\_21\_22\_23\_24\_UART\_SPI\_ADC\_DMA.md](Phase_7_Peripherals/Chapter_21_22_23_24_UART_SPI_ADC_DMA.md) |

### Phase 8: Advanced Systems
| Chapter | Topic | File |
|---------|-------|------|
| 25 | FATFS + FreeRTOS — SD Card Data Logger | [Chapter\_25\_26\_27\_28\_FATFS\_Network\_Bootloader\_Safety.md](Phase_8_Advanced_Systems/Chapter_25_26_27_28_FATFS_Network_Bootloader_Safety.md) |
| 26 | Networking — LWIP + MQTT | [Chapter\_25\_26\_27\_28\_FATFS\_Network\_Bootloader\_Safety.md](Phase_8_Advanced_Systems/Chapter_25_26_27_28_FATFS_Network_Bootloader_Safety.md) |
| 27 | Bootloader + OTA Updates | [Chapter\_25\_26\_27\_28\_FATFS\_Network\_Bootloader\_Safety.md](Phase_8_Advanced_Systems/Chapter_25_26_27_28_FATFS_Network_Bootloader_Safety.md) |
| 28 | Safety — Watchdog + Fault Handling | [Chapter\_25\_26\_27\_28\_FATFS\_Network\_Bootloader\_Safety.md](Phase_8_Advanced_Systems/Chapter_25_26_27_28_FATFS_Network_Bootloader_Safety.md) |

### Phase 9: Complete Projects
| Project | Description | File |
|---------|-------------|------|
| 1 | Multitasking LED Controller | [Complete\_Projects.md](Phase_9_Projects/Complete_Projects.md) |
| 2 | UART Command-Line Interface | [Complete\_Projects.md](Phase_9_Projects/Complete_Projects.md) |
| 3 | Data Logger with SD Card | [Complete\_Projects.md](Phase_9_Projects/Complete_Projects.md) |
| 4 | Advanced Multi-Sensor RTOS System | [Complete\_Projects.md](Phase_9_Projects/Complete_Projects.md) |

### Interview Preparation
| Section | Topics | File |
|---------|--------|------|
| A–E | Core Concepts, IPC, System Design, Quick-Fire, Design Exercises | [RTOS\_Interview\_Guide.md](Interview_Preparation/RTOS_Interview_Guide.md) |

---

## How to Use This Book

1. **Beginners**: Start at Phase 1, Chapter 1. Read sequentially.
2. **Intermediate**: Jump to Phase 3 (IPC) or Phase 4 (Advanced) based on gaps.
3. **Interview Prep**: Go directly to the Interview Preparation guide.
4. **Project Reference**: Phase 9 has complete, annotated projects to study.

## Prerequisites

- Basic C programming
- STM32CubeIDE installed
- An STM32 development board (e.g., NUCLEO-F446RE)
- Curiosity and patience

## Tech Stack

- **MCU**: STM32 (ARM Cortex-M4)
- **RTOS**: FreeRTOS
- **IDE**: STM32CubeIDE
- **Language**: C
