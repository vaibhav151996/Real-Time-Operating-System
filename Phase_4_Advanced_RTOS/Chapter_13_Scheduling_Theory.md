# Chapter 13: Scheduling Theory

---

## Learning Objectives

By the end of this chapter, you will be able to:

1. Explain priority inversion in depth and its solutions
2. Apply Rate Monotonic Scheduling (RMS) for priority assignment
3. Understand Earliest Deadline First (EDF) scheduling
4. Perform schedulability analysis to verify a task set meets all deadlines
5. Make informed scheduling decisions for production systems

---

## 13.1 Priority Inversion Revisited

### 13.1.1 Unbounded Priority Inversion

We introduced priority inversion in Chapter 9. Here we analyze it formally.

```
THE MARS PATHFINDER BUG (1997):
════════════════════════════════

The Mars Pathfinder rover experienced system resets on Mars due to
priority inversion between:
  - High-priority: Bus management task
  - Medium-priority: Communication task  
  - Low-priority: Data collection task

The data collection task held a mutex needed by the bus management
task. The communication task preempted the data collection task,
preventing it from releasing the mutex. The bus management task
was blocked indefinitely → watchdog timeout → system reset.

FIX: Enabled priority inheritance on the mutex.
     (VxWorks RTOS had the feature, but it was disabled.)

LESSON: Priority inheritance is not optional in production systems.
```

### 13.1.2 Solutions to Priority Inversion

| Solution | Mechanism | Pros | Cons |
|---|---|---|---|
| **Priority Inheritance** | Boost holder to waiter priority | Simple, automatic in FreeRTOS | Doesn't prevent chained inversion |
| **Priority Ceiling** | Mutex has max priority; holder gets it immediately | Prevents deadlock too | Must know all accessor priorities |
| **Disable interrupts** | No preemption during critical section | Very fast | Blocks ALL tasks + ISRs |
| **Lock-free algorithms** | No mutex needed | Best performance | Complex, hard to verify |

---

## 13.2 Rate Monotonic Scheduling (RMS)

### 13.2.1 The Theory

**Rate Monotonic Scheduling** is the optimal fixed-priority scheduling algorithm for periodic tasks. It assigns priorities based on period:

> **Shorter period → Higher priority**

This is provably optimal: if any fixed-priority assignment can meet all deadlines, RMS will too.

### 13.2.2 Schedulability Test

A set of N periodic tasks is schedulable under RMS if:

$$\sum_{i=1}^{N} \frac{C_i}{T_i} \leq N(2^{1/N} - 1)$$

Where:
- $C_i$ = Worst-case execution time of task $i$
- $T_i$ = Period of task $i$
- $U = \sum \frac{C_i}{T_i}$ = Total CPU utilization

**Utilization bounds:**

| N (tasks) | Bound $N(2^{1/N}-1)$ | Approx % |
|---|---|---|
| 1 | 1.000 | 100% |
| 2 | 0.828 | 82.8% |
| 3 | 0.780 | 78.0% |
| 4 | 0.757 | 75.7% |
| 5 | 0.743 | 74.3% |
| ∞ | 0.693 | 69.3% |

### 13.2.3 RMS Example

```
TASK SET:
═════════
                           WCET    Period   Utilization
Task_Sensor:   C=2ms,     T=10ms           2/10 = 0.20
Task_Control:  C=5ms,     T=20ms           5/20 = 0.25
Task_Display:  C=10ms,    T=50ms          10/50 = 0.20
                                    ─────────────────
                           Total U = 0.65 = 65%

RMS Bound for 3 tasks: 0.780 = 78%

0.65 < 0.78 → SCHEDULABLE ✓

Priority Assignment (shortest period = highest priority):
  Task_Sensor:  Priority 3 (period 10ms — shortest)
  Task_Control: Priority 2 (period 20ms)
  Task_Display: Priority 1 (period 50ms — longest)
```

```
TIMING DIAGRAM (RMS):
═════════════════════

Time (ms): 0    5    10   15   20   25   30   35   40   45   50
           │    │    │    │    │    │    │    │    │    │    │

Sensor:    ██        ██        ██        ██        ██        ██
(Pri 3)    2ms       2ms       2ms       2ms       2ms       2ms

Control:     █████        █████          █████         █████
(Pri 2)      5ms          5ms            5ms           5ms

Display:          ████████                     ████████████
(Pri 1)           (fills gaps)                 (fills gaps)

All deadlines met ✓
```

---

## 13.3 Earliest Deadline First (EDF)

### 13.3.1 The Theory

EDF is a **dynamic-priority** algorithm: at every scheduling point, the task with the **nearest deadline** gets the highest priority.

**Schedulability:** EDF can schedule any task set with total utilization $U \leq 1.0$ (100%).

This is better than RMS's ~69.3% bound, but EDF is harder to implement and has higher overhead.

### 13.3.2 RMS vs. EDF

| Feature | RMS | EDF |
|---|---|---|
| **Priority type** | Fixed (static) | Dynamic |
| **Utilization bound** | ~69.3% | 100% |
| **Implementation** | Simple (FreeRTOS native) | Complex (not in FreeRTOS) |
| **Overhead** | Low | Higher (recalculate priorities) |
| **Overload behavior** | Predictable (low-pri tasks miss) | Unpredictable (any task may miss) |
| **Industry usage** | ★★★★★ | ★★☆☆☆ |
| **FreeRTOS support** | Native | Must implement manually |

> **🏭 Industry Insight:** RMS is overwhelmingly used in practice. Its predictability under overload is more valuable than EDF's higher utilization. Most embedded systems run at < 60% utilization anyway.

---

## 13.4 Practical Priority Assignment

```c
/* Production system example */

/* Priority Assignment Table:
 *
 * Priority  Task              Period    WCET    Util
 * ──────────────────────────────────────────────────
 *    6      Emergency_Stop     1 ms     0.1ms   10%
 *    5      Motor_Control      5 ms     1.0ms   20%
 *    4      Sensor_Read       10 ms     1.5ms   15%
 *    3      CAN_Protocol      20 ms     2.0ms   10%
 *    2      Display           100 ms    5.0ms    5%
 *    1      Logging          1000 ms   10.0ms    1%
 *    0      IDLE               ∞        ∞       39%
 * ──────────────────────────────────────────────────
 *                              Total U = 61%  < 69.3% ✓
 */
```

---

## 13.5 Chapter Summary

| Concept | Key Takeaway |
|---|---|
| **Priority inversion** | High-priority blocked by medium via low (Mars Pathfinder) |
| **Priority inheritance** | Automatic in FreeRTOS mutexes; always enable |
| **RMS** | Shorter period → higher priority; optimal for fixed-priority |
| **RMS bound** | $U \leq N(2^{1/N}-1)$; ~69.3% for large N |
| **EDF** | Dynamic priority; 100% utilization; complex to implement |
| **Practice** | Use RMS with < 60% utilization for safety margin |

---

*End of Chapter 13*
