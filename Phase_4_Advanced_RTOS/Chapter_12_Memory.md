# Chapter 12: Memory Management

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Explain FreeRTOS heap implementations (heap_1 through heap_5)
2. Choose the right heap for your application
3. Understand and prevent heap fragmentation
4. Implement static allocation for safety-critical systems
5. Monitor and debug memory issues

---

## 12.1 FreeRTOS Heap Implementations

FreeRTOS provides five heap implementations. You include **exactly one** in your project.

### 12.1.1 Comparison Table

| Feature | heap_1 | heap_2 | heap_3 | heap_4 | heap_5 |
|---|---|---|---|---|---|
| **Allocate** | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Free** | ✗ | ✓ | ✓ | ✓ | ✓ |
| **Merge free blocks** | N/A | ✗ | N/A | ✓ | ✓ |
| **Deterministic** | ✓ | ✗ | ✗ | ~✓ | ~✓ |
| **Thread-safe** | ✓ | ✓ | Depends | ✓ | ✓ |
| **Fragmentation** | None | Yes | libc dep. | Minimal | Minimal |
| **Multiple regions** | ✗ | ✗ | ✗ | ✗ | ✓ |
| **Best for** | Static | Similar-size allocs | Legacy | **General use** | Non-contiguous RAM |

### 12.1.2 heap_1: Allocate Only

```
heap_1: Simple bump allocator. Never frees memory.
═══════════════════════════════════════════════════

  configTOTAL_HEAP_SIZE = 8192 bytes
  
  ┌──────────────────────────────────────────────────────────┐
  │████████│████│████████████│                                │
  │ TCB 1  │TCB2│  Stack 1   │  ← FREE (never reclaimed)    │
  └──────────────────────────────────────────────────────────┘
           ▲
           Next allocation starts here (never goes back)

Use when: All tasks/queues/semaphores are created at startup
          and never deleted. Simplest, most deterministic.
```

### 12.1.3 heap_4: General Purpose (Recommended)

```
heap_4: First-fit allocator with adjacent block merging.
════════════════════════════════════════════════════════

  After several alloc/free cycles:
  
  ┌────────┬──────┬────────┬──────┬────────┬──────────────┐
  │ USED   │ FREE │ USED   │ FREE │ USED   │    FREE      │
  │ 200B   │ 64B  │ 512B   │ 128B │ 100B   │   remaining  │
  └────────┴──────┴────────┴──────┴────────┴──────────────┘
  
  If middle USED block (512B) is freed:
  ┌────────┬──────────────────────┬────────┬──────────────┐
  │ USED   │     MERGED FREE      │ USED   │    FREE      │
  │ 200B   │    64+512+128 = 704B │ 100B   │   remaining  │
  └────────┴──────────────────────┴────────┴──────────────┘
  
  Adjacent free blocks merged → reduces fragmentation.
```

### 12.1.4 heap_5: Multiple Memory Regions

```c
/* heap_5: Use when RAM is split across multiple regions */
/* Example: STM32H7 has DTCM, SRAM1, SRAM2, SRAM3 */

const HeapRegion_t xHeapRegions[] = {
    { (uint8_t *)0x20000000, 0x10000 },  /* 64KB SRAM1 */
    { (uint8_t *)0x24000000, 0x80000 },  /* 512KB AXI SRAM */
    { NULL, 0 }                           /* Terminator */
};

/* Call BEFORE any FreeRTOS API */
void main(void)
{
    vPortDefineHeapRegions(xHeapRegions);
    /* Now xTaskCreate, xQueueCreate, etc. work across both regions */
}
```

---

## 12.2 Fragmentation

### 12.2.1 What Is Fragmentation?

```
EXTERNAL FRAGMENTATION:
═══════════════════════

Total free: 512 bytes. Need to allocate 256 bytes. FAILS!

┌──────┬────┬──────┬────┬──────┬────┬──────┐
│USED  │free│USED  │free│USED  │free│USED  │
│      │64B │      │128B│      │64B │      │
└──────┴────┴──────┴────┴──────┴────┴──────┘

Free space exists (64+128+64+256 = 512 bytes total)
but no SINGLE contiguous block of 256 bytes!

Prevention:
1. Use heap_4 (merges adjacent blocks)
2. Allocate/free blocks of SAME size (heap_2)
3. Avoid runtime malloc/free — allocate everything at startup
4. Use static allocation (xTaskCreateStatic)
```

### 12.2.2 Memory Monitoring

```c
/* Available heap */
size_t free = xPortGetFreeHeapSize();

/* Minimum heap ever available (high-water mark) */
size_t min_free = xPortGetMinimumEverFreeHeapSize();

/* Malloc failed hook */
void vApplicationMallocFailedHook(void)
{
    /* Log the failure, halt, or take corrective action */
    taskDISABLE_INTERRUPTS();
    for (;;);
}
```

---

## 12.3 Static vs. Dynamic Allocation

```c
/* DYNAMIC: Memory from FreeRTOS heap */
xTaskCreate(Task_A, "A", 256, NULL, 1, NULL);
xQueueCreate(10, sizeof(uint32_t));
xSemaphoreCreateMutex();

/* STATIC: Memory from your arrays (compile-time known) */
StaticTask_t xTaskBuf;
StackType_t  xStack[256];
xTaskCreateStatic(Task_A, "A", 256, NULL, 1, xStack, &xTaskBuf);

StaticQueue_t xQueueBuf;
uint8_t       xQueueStorage[10 * sizeof(uint32_t)];
xQueueCreateStatic(10, sizeof(uint32_t), xQueueStorage, &xQueueBuf);

StaticSemaphore_t xMutexBuf;
xSemaphoreCreateMutexStatic(&xMutexBuf);
```

> **🏭 Industry Insight:** Safety-critical systems (IEC 61508, ISO 26262) strongly prefer static allocation. All memory is visible in the linker map. No runtime allocation failures possible.

---

## 12.4 Chapter Summary

| Concept | Key Takeaway |
|---|---|
| **heap_1** | Allocate only, never free; simplest, deterministic |
| **heap_4** | General purpose with merging; recommended default |
| **heap_5** | Multiple non-contiguous RAM regions |
| **Fragmentation** | Free space exists but not contiguous; prevented by heap_4 |
| **Static allocation** | No heap needed; all memory known at compile time |
| **Monitoring** | `xPortGetFreeHeapSize()`, `xPortGetMinimumEverFreeHeapSize()` |

---

*End of Chapter 12*
