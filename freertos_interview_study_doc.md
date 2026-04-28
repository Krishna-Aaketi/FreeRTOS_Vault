# FreeRTOS — Interview Study Document

A single-sitting read that takes you from absolute basics to advanced internals, written for an embedded engineer who knows C and bare-metal but has not used an RTOS before. Every section explains *what*, *why*, *how internally*, *when to use*, *common pitfalls*, and shows runnable FreeRTOS code. Each major section ends with what an interviewer is really probing, likely questions, model answers, and traps.

---

## Table of Contents

1. [Foundations](#1-foundations)
2. [Tasks](#2-tasks)
3. [The Scheduler](#3-the-scheduler)
4. [Inter-task Communication and Synchronization](#4-inter-task-communication-and-synchronization)
5. [Interrupts and the FromISR API](#5-interrupts-and-the-fromisr-api)
6. [Memory Management](#6-memory-management)
7. [Timers](#7-timers)
8. [Advanced and Production Topics](#8-advanced-and-production-topics)
9. [Design Patterns and Anti-Patterns](#9-design-patterns-and-anti-patterns)
10. [Rapid-Fire Round — 30 Q&A](#10-rapid-fire-round)
11. [Scenario Questions](#11-scenario-questions)
12. [One-Page Cheat Sheet](#12-one-page-cheat-sheet)

---

## 1. Foundations

### 1.1 What an RTOS is, and how it differs from a super-loop and from Linux

A **real-time operating system** is a small piece of software whose main job is to decide *which piece of code runs on the CPU right now* and to give that code predictable timing. In FreeRTOS this software is roughly 6–15 KB of C, linked into your firmware. There is no shell, no filesystem by default, no users, no processes — just **tasks**, a **scheduler**, and primitives for tasks to talk to each other and to interrupts.

A **bare-metal super-loop** looks like this:

```c
int main(void) {
    hw_init();
    for (;;) {
        read_sensors();
        update_control();
        handle_uart();
        update_display();
    }
}
```

It works until two things become true: (a) one of those functions starts taking long enough that another one misses its deadline, and (b) you start needing to *block* — wait for a byte, wait for a timer, wait for a button — without freezing the rest of the system. At that point you reach for state machines inside every function, and the code rots. An RTOS lets you write each activity as if it were the only thing the CPU does, and then stitches them together by time-slicing.

A **general-purpose OS like Linux** also runs many things "at once", but it optimizes for *throughput and fairness* across unknown workloads. It uses MMUs, virtual memory, processes, demand paging — all of which introduce non-determinism. An interrupt on Linux can be delayed by tens of milliseconds. An interrupt on a Cortex-M running FreeRTOS is serviced in microseconds, every time. **An RTOS trades features for predictability.**

### 1.2 Hard vs soft real-time, determinism, latency vs throughput

A **hard real-time** system is one where missing a deadline is a system failure — an airbag controller, a motor commutation loop, a brake-by-wire ECU. A **soft real-time** system is one where missing a deadline degrades quality but does not break the system — a UI redraw, an audio buffer underrun you can mask. FreeRTOS itself is suitable for hard real-time *if* the rest of your design is, but FreeRTOS does not guarantee your deadlines for you; it only guarantees that *the scheduler itself* is deterministic.

**Determinism** means: the same input produces the same timing, every time. The worst-case execution time is bounded and known.

**Latency** is the time between a stimulus and the start of its response. **Throughput** is how much work you complete per unit time. An RTOS optimizes latency. Linux optimizes throughput. A super-loop accidentally optimizes neither.

### 1.3 Why FreeRTOS specifically

- **License**: MIT since 2017 (it was a modified GPL before). You can ship it in closed-source commercial firmware with no royalty and no copyleft on your code.
- **Footprint**: a usable kernel fits in well under 10 KB of flash and a few hundred bytes of RAM per task.
- **Portability**: official ports for Cortex-M0/M3/M4/M7/M33, RISC-V, Xtensa (ESP32), ARM7, AVR, MSP430, PIC32, x86 — over 40 architectures.
- **Ecosystem**: AWS owns it now, so FreeRTOS+TCP, FreeRTOS+FAT, the LTS releases, OTA, MQTT-agent, and the AWS IoT integrations are first-party. SafeRTOS is the IEC 61508 / ISO 26262 certified sibling for safety-critical work.

### 1.4 The source tree and what FreeRTOSConfig.h actually controls

A FreeRTOS source tree has, at the top:

```
FreeRTOS/Source/
    tasks.c          ← the scheduler and all task APIs
    queue.c          ← queues, semaphores, mutexes (all built on queues internally)
    list.c           ← the doubly-linked lists the scheduler uses for ready/blocked/delayed
    timers.c         ← software timer service
    event_groups.c   ← event flags
    stream_buffer.c  ← stream and message buffers
    portable/<compiler>/<arch>/
        port.c       ← context switch, tick ISR, critical sections — the per-MCU bits
        portmacro.h
    include/         ← the public API headers (FreeRTOS.h, task.h, queue.h, ...)
```

You add **one file of your own**: `FreeRTOSConfig.h`. This is not a kernel file; it is *your* configuration, and it must be on the include path. Every behavior of the kernel is gated on a `#define` in this file. The ones that actually matter in interviews:

**Bold the ones you must know:**

- **`configUSE_PREEMPTION`** — 1 for preemptive, 0 for cooperative.
- **`configCPU_CLOCK_HZ`** — the CPU clock the port uses to set up the SysTick.
- **`configTICK_RATE_HZ`** — usually 1000 (1 ms tick). Ticks are the kernel's clock.
- **`configMAX_PRIORITIES`** — number of distinct priority levels. Cortex-M ports use a 1-bit-per-priority bitmap, so going from 5 to 32 has a small constant cost, not a per-priority cost.
- **`configMINIMAL_STACK_SIZE`** — stack for the idle task, in *words*, not bytes.
- **`configTOTAL_HEAP_SIZE`** — bytes given to the FreeRTOS heap (heap_1..heap_5).
- **`configMAX_SYSCALL_INTERRUPT_PRIORITY`** — the priority threshold for FreeRTOS-aware ISRs. Critically misunderstood; section 5 covers it.
- **`configUSE_TIMERS`**, **`configUSE_MUTEXES`**, **`configUSE_COUNTING_SEMAPHORES`**, **`configSUPPORT_STATIC_ALLOCATION`**, **`configSUPPORT_DYNAMIC_ALLOCATION`** — feature gates.
- **`configCHECK_FOR_STACK_OVERFLOW`** — 0, 1, or 2. Section 6 explains the difference.
- **`configGENERATE_RUN_TIME_STATS`** — enables `vTaskGetRunTimeStats()`.
- **`INCLUDE_xxx`** macros — finer-grain gates on individual API functions.

**Pitfall**: `configMINIMAL_STACK_SIZE` is in *words*. On a 32-bit Cortex-M, `configMINIMAL_STACK_SIZE = 128` means **512 bytes**. This trips people up routinely.

#### 1.4.1 What an interviewer is really testing

Whether you understand that an RTOS is a *scheduling and synchronization* layer, not a magic "make everything fast" library, and whether you can articulate when reaching for it is justified versus over-engineering. Senior interviews probe whether you treat it as a tool with tradeoffs.

#### 1.4.2 Likely interview questions

1. When would you *not* use an RTOS, even if you have one available?
2. What is the difference between hard and soft real-time, and which is FreeRTOS?
3. Walk me through what happens at `configCPU_CLOCK_HZ / configTICK_RATE_HZ` — what does that number become?
4. If I changed `configMAX_PRIORITIES` from 5 to 32 on a Cortex-M, what is the cost?
5. What does `configMINIMAL_STACK_SIZE = 128` mean in bytes, on a 32-bit MCU?
6. Why is FreeRTOS suitable for hard real-time but not by itself sufficient for it?
7. What is in `FreeRTOSConfig.h` versus in the kernel source?

#### 1.4.3 Model answers

1. *"If the workload is genuinely sequential and the timing budget fits in a super-loop, an RTOS adds context-switch overhead, RAM cost per task, and a debugging surface for race conditions I would not otherwise have. I'd skip it for a sub-1KB-RAM target, a single-control-loop motor controller where the loop *is* the system, or anything where the worst-case super-loop iteration is comfortably under the tightest deadline."*
2. *"Hard real-time means missing a deadline is a system failure — like airbag deployment. Soft means it degrades quality, like a dropped audio frame. FreeRTOS is deterministic enough for hard real-time, but real-time-ness is a property of the whole system, including ISR design, driver code, and the application — not just the kernel."*
3. *"That's the SysTick reload value. It's the number of CPU cycles between kernel ticks. At 168 MHz with a 1 kHz tick, SysTick fires every 168,000 cycles."*
4. *"On the Cortex-M port the ready-list lookup uses CLZ on a priority bitmap, so it's O(1) regardless. The cost is one extra word of RAM per priority for the ready list head, and slightly more bitmap manipulation. Negligible."*
5. *"512 bytes — it's in stack words, and a word is 4 bytes on 32-bit ARM."*
6. *"The kernel guarantees its own determinism, but your deadlines depend on ISR latency, the longest critical section in your drivers, priority assignment, and worst-case execution time of your tasks. The kernel doesn't fix any of those for you."*
7. *"`FreeRTOSConfig.h` is application-owned and on your include path. The kernel reads it via `#include "FreeRTOS.h"`. The kernel source is untouched between projects; the config is what changes."*

#### 1.4.4 Traps and gotchas

- Saying "FreeRTOS is hard real-time" without qualifying that real-time is a system property.
- Confusing words and bytes for stack sizes — extremely common, and a fast way to look junior.
- Thinking you "install" FreeRTOS like a library; you compile its `.c` files into your firmware.
- Forgetting that FreeRTOS does not provide drivers, a network stack, or a filesystem in the base distribution — those are separate (FreeRTOS+TCP, FreeRTOS+FAT).

---

## 2. Tasks

### 2.1 What a task is

A task is an independent thread of execution with its own stack and its own priority. Each task is implemented as a C function that **never returns**; it runs an infinite loop. The kernel saves and restores the CPU registers (the *context*) when switching between tasks, so each task sees the CPU as if it were the only thing running.

A task at runtime is described by a **TCB** — Task Control Block — a small struct in the kernel containing:

- A pointer to the top of the task's stack (this is what gets swapped on context switch)
- The base of the stack (for overflow checking)
- The task's priority and base priority (the latter matters for priority inheritance)
- A `ListItem_t` linking the task into whichever list it currently belongs to (Ready, Delayed, Suspended, or a queue's waiter list)
- An event list item for waiting on multiple things
- The task name (for debugging) and a handle

The TCB itself can live in the FreeRTOS heap (dynamic) or in memory you provide (static).

### 2.2 Task creation: `xTaskCreate` and `xTaskCreateStatic`

```c
void vSensorTask(void *pvParameters);

TaskHandle_t hSensor = NULL;

BaseType_t ok = xTaskCreate(
    vSensorTask,        // function
    "Sensor",           // name (for debug only, copied)
    256,                // stack depth in WORDS
    NULL,               // parameter passed to vSensorTask
    3,                  // priority (0 = idle priority)
    &hSensor);          // out: handle, can be NULL if you don't need one

if (ok != pdPASS) {
    // heap exhausted
}
```

**Static creation** is identical except you provide the storage:

```c
static StaticTask_t xSensorTCB;
static StackType_t  xSensorStack[256];

hSensor = xTaskCreateStatic(
    vSensorTask, "Sensor", 256, NULL, 3,
    xSensorStack,   // your stack buffer
    &xSensorTCB);   // your TCB buffer
```

Static creation has three advantages: no heap, deterministic placement (you can put the stack in a specific RAM region with linker attributes), and the failure mode is a link error instead of a runtime `NULL`. Safety-critical projects use it almost exclusively. It requires `configSUPPORT_STATIC_ALLOCATION = 1`.

### 2.3 The shape of a task function

```c
void vSensorTask(void *pvParameters) {
    sensor_init();
    TickType_t xLastWake = xTaskGetTickCount();
    const TickType_t xPeriod = pdMS_TO_TICKS(100);

    for (;;) {
        int16_t raw = sensor_read();
        publish(raw);
        vTaskDelayUntil(&xLastWake, xPeriod);
    }
}
```

Three rules: there is **always** an outer `for(;;)`, there is **always** a blocking call somewhere in the loop (otherwise the task hogs the CPU at its priority level), and the function **never returns**. If a task function does return, FreeRTOS calls `configASSERT` or — depending on configuration — undefined behavior. The standard hardening is to call `vTaskDelete(NULL)` at the end if a task could ever fall through, but in practice every task is an infinite loop.

### 2.4 Priority

Priority is an integer from 0 (lowest, the idle task's priority) up to `configMAX_PRIORITIES - 1` (highest). Multiple tasks can share a priority; among equal-priority tasks the scheduler round-robins them on tick boundaries (if `configUSE_TIME_SLICING` is 1).

A common starting scheme:

| Priority | Use |
|---|---|
| 0 | Idle task (kernel-owned) |
| 1 | Background work — logging, telemetry, housekeeping |
| 2 | Normal application tasks |
| 3 | Higher-rate periodic tasks |
| 4 | Communications — UART, SPI, network |
| 5 | Time-critical control loops |
| (highest) | Reserved for emergency shutdown / deferred ISR handlers |

Do not assign priority "by feel". Section 9 covers rate-monotonic assignment.

### 2.5 Task states and transitions

A task is in exactly one of four states:

- **Running** — currently executing on the CPU. On a single core there is exactly one running task at a time.
- **Ready** — runnable, waiting only because a higher- or equal-priority task is using the CPU.
- **Blocked** — waiting for something: a time delay, a queue item, a semaphore, an event bit, a notification. Has a timeout (which may be `portMAX_DELAY` for forever).
- **Suspended** — explicitly removed from scheduling by `vTaskSuspend`. Does not time out. Resumed by `vTaskResume`.

Transitions, described as a state diagram (imagine four nodes arranged as a diamond):

```
                 +-----------+
                 |  Running  |
                 +-----------+
                 ^     |   |  ^
   higher-pri   |     |   |  | preempted /
   becomes ready |     |   |  | time slice
                 |     v   v  |
+-----------+   +-----------+   +-----------+
|  Blocked  |<->|   Ready   |<->| Suspended |
+-----------+   +-----------+   +-----------+
   timeout         (vTaskResume from any state)
   or event
   occurred
```

In words:

- **Ready → Running**: scheduler picks it as the highest-priority ready task.
- **Running → Ready**: a higher-priority task became ready (preempted), or the time slice expired and an equal-priority sibling exists.
- **Running → Blocked**: the task called `vTaskDelay`, a blocking queue/semaphore receive, etc.
- **Blocked → Ready**: timeout fired, or the event the task was waiting for occurred.
- **Any → Suspended**: `vTaskSuspend` (rarely the right answer in production code).
- **Suspended → Ready**: `vTaskResume`.

### 2.6 The idle task and the idle hook

When *no* application task is ready, somebody has to run, because the CPU has to fetch instructions. FreeRTOS solves this by automatically creating an **idle task** at priority 0 when you call `vTaskStartScheduler`. Its job is to do nothing useful: clean up tasks that have deleted themselves (their stack and TCB cannot be freed inside `vTaskDelete` because you might be deleting yourself), and optionally call into your **idle hook**.

```c
// FreeRTOSConfig.h: configUSE_IDLE_HOOK = 1
void vApplicationIdleHook(void) {
    // runs in idle task context, at priority 0
    __WFI();   // sleep until next interrupt — basic low-power
}
```

**Three rules for the idle hook**: it must never block, it must be quick (an idle hook that runs for 50 ms means deleted tasks' memory is not freed for 50 ms), and `__WFI` here is the simplest sleep — for real low-power use tickless idle (section 8).

### 2.7 `vTaskDelay` vs `vTaskDelayUntil` — why the difference matters

```c
// Drift-prone:
for (;;) {
    do_work();                          // takes 3 ms, but variable
    vTaskDelay(pdMS_TO_TICKS(100));     // delays 100 ms FROM NOW
}
// Period = 103 ms, drifting

// Drift-free:
TickType_t xLast = xTaskGetTickCount();
for (;;) {
    do_work();
    vTaskDelayUntil(&xLast, pdMS_TO_TICKS(100));  // wakes at xLast + 100ms
}
// Period = exactly 100 ms (assuming work fits)
```

`vTaskDelay` is "sleep for at least N ticks, starting now". Any time you spend before calling it adds to the period. `vTaskDelayUntil` is "wake me at this absolute tick", and it updates the reference for you. Use `vTaskDelayUntil` for **periodic** work — sensor sampling, control loops, watchdog kicks. Use `vTaskDelay` for **one-shot** waits — debouncing, retry backoff.

In modern FreeRTOS (V10.4+), `vTaskDelayUntil` was renamed `xTaskDelayUntil` and now returns a `BaseType_t` indicating whether the task actually delayed (it doesn't, if the deadline is already past — that's a sign your work overran).

### 2.8 Deleting and suspending — and the gotchas

`vTaskDelete(NULL)` deletes the calling task. `vTaskDelete(handle)` deletes another task. The actual freeing of the deleted task's stack and TCB happens in the **idle task**, so if the idle task never runs (because you have a higher-priority task that never blocks — itself a bug), deleted tasks accumulate in the heap.

Deleting a task that owns a mutex, holds a queue receiver, or is in the middle of a critical section is a **disaster**. The kernel will not unwind these for you. Treat `vTaskDelete` as "I am positive this task holds nothing." In practice, well-architected firmware never deletes tasks at all — tasks live for the lifetime of the system, and short-lived work is dispatched to long-lived worker tasks via queues.

`vTaskSuspend` and `vTaskResume` have similar hazards: a suspended task that holds a mutex blocks every other task that needs it, indefinitely.

#### 2.8.1 What an interviewer is really testing

Whether you understand a task's life cycle and can pick the right delay/blocking primitive for the job — and whether you have the discipline to avoid the lazy answers (`vTaskSuspend`, `vTaskDelay` for synchronization, dynamic task creation as control flow).

#### 2.8.2 Likely interview questions

1. Walk me through what happens when `xTaskCreate` returns.
2. Difference between `vTaskDelay(100)` and `vTaskDelayUntil` — when do I use which?
3. Why must a task function never return?
4. What runs when no task is ready?
5. A task is in the Blocked state. What are *all* the things that can move it to Ready?
6. Static vs dynamic task creation — which would you pick for safety-critical, and why?
7. I see a task at priority 5 with no `vTaskDelay` and no blocking calls. What happens?
8. What's the cost in RAM of one task?

#### 2.8.3 Model answers

1. *"Kernel allocates a TCB and stack — from the heap or from buffers I supply — initializes the stack so a context restore would launch the task function with its parameter, then puts the TCB on the ready list at the requested priority. If the new task is higher priority than the current running task and the scheduler is already running, a context switch happens before `xTaskCreate` returns to the caller."*
2. *"`vTaskDelay` is relative to *now*, so any work done before it adds to the period — useless for periodic loops. `vTaskDelayUntil` wakes at an absolute tick and updates the reference, so the period is exactly what I asked for. Periodic control: `DelayUntil`. One-shot wait or backoff: `Delay`."*
3. *"There's no caller to return to. The kernel started this function as a thread of execution; if it returns, you're popping into garbage memory. FreeRTOS will assert if `configASSERT` is enabled, but the safe pattern is `for(;;)` — a task is by construction infinite."*
4. *"The idle task. It runs at priority 0, frees memory of any tasks that deleted themselves, and optionally calls the idle hook where you'd typically `WFI` or enter tickless idle."*
5. *"The block timeout expires; the event it was waiting on (queue item, semaphore give, notification, event bit, mutex unlock) occurs; or another task or ISR explicitly aborts its block via `xTaskAbortDelay`."*
6. *"Static — for safety-critical I want all memory accounted for at link time, deterministic addresses for analysis, and no possibility of a heap-allocation failure at runtime. The cost is having to declare buffers explicitly, which is fine because the task set is known."*
7. *"It hogs the CPU. Anything at priority < 5 starves; anything at priority 5 round-robins with it on tick boundaries; anything at priority > 5 preempts normally. The idle task never runs, so deleted tasks leak and the idle hook never fires. This is one of the top causes of mysterious system hangs."*
8. *"TCB is around 80–120 bytes depending on configuration, plus the stack. A modest task with a 256-word stack on Cortex-M is roughly 1.1 KB."*

#### 2.8.4 Traps and gotchas

- Sizing stacks "by gut" — always use `uxTaskGetStackHighWaterMark` to measure (section 6).
- Calling `vTaskDelay(0)` thinking it yields — it doesn't reliably; use `taskYIELD()`.
- Using `vTaskSuspend` as a synchronization primitive instead of a semaphore.
- Creating tasks dynamically as a way to "do work later" — that's what worker tasks and queues are for.
- Forgetting that `xTaskCreate` can fail, and not checking the return.

---

## 3. The Scheduler

### 3.1 Preemptive vs cooperative vs time-sliced

FreeRTOS supports three modes via two config bits:

| `configUSE_PREEMPTION` | `configUSE_TIME_SLICING` | Behavior |
|---|---|---|
| 1 | 1 | **Preemptive with time slicing** (the default). Higher-priority ready tasks immediately preempt; equal-priority tasks round-robin on each tick. |
| 1 | 0 | **Preemptive, no time slicing**. Higher-priority preempts, but equal-priority tasks run until they block — no round-robin. Slightly lower overhead. |
| 0 | — | **Cooperative**. Tasks run until they explicitly yield (`taskYIELD`) or block. No tick-driven switches. Used in tiny systems where determinism by inspection is desired. |

In a preemptive system, a higher-priority task that becomes ready *immediately* takes the CPU — even mid-instruction-sequence in the running task — at the next instruction boundary. That immediacy is the point of an RTOS.

### 3.2 How the tick interrupt drives scheduling

The kernel's heartbeat is the **tick interrupt**, normally driven by SysTick on Cortex-M, configured at `configTICK_RATE_HZ` (typically 1000 Hz → 1 ms tick). On every tick, the kernel:

1. Increments the tick count.
2. Walks the **delayed list** to see if any blocked-with-timeout tasks should now be moved to the ready list.
3. If time slicing is on, checks whether to round-robin among equal-priority ready tasks.
4. If a context switch is now warranted, sets the PendSV exception pending; the actual switch happens in PendSV (the lowest-priority exception), so the tick ISR exits quickly.

The tick rate is a tradeoff. Higher tick rate = finer-grained delays but more overhead (each tick is non-zero work). 1 kHz is the sweet spot for most systems. For low-power, 100 Hz or tickless idle is better.

A 1 ms tick means **`vTaskDelay(1)` may sleep anywhere from ~0 to 1 ms** — if you call it 0.1 ms before the next tick, the task wakes 0.1 ms later. To guarantee at least N ms, use `pdMS_TO_TICKS(N) + 1`. Most production code accepts the one-tick fuzz and uses `pdMS_TO_TICKS(N)` directly.

### 3.3 How the scheduler picks the next task

FreeRTOS keeps **one ready list per priority level** — `pxReadyTasksLists[configMAX_PRIORITIES]`. The scheduler's job, on every potential switch point, is: *find the highest priority that has any ready tasks, and pick the next task from that list (round-robin if multiple).*

There are two implementations:

- **Generic, portable**: a linear scan from the highest priority down. O(`configMAX_PRIORITIES`).
- **Architecture-optimized (Cortex-M, etc.)**: a 32-bit bitmap where bit *N* is set if priority *N* has any ready task. The scheduler does `__CLZ` (count-leading-zeros) on the bitmap — one instruction — to find the highest set bit. **O(1) regardless of how many priorities you configured**, capped at 32 priorities.

This is why on Cortex-M you can have up to 32 priorities for free, and why the official guidance says "use as many priorities as you need; don't try to be economical with them."

### 3.4 Context switching — what is saved, where, at what cost

A **context** is the set of CPU registers that define a thread of execution. On Cortex-M4F:

- 17 core registers (`r0`–`r12`, `sp`, `lr`, `pc`, `xPSR`)
- Optionally 33 FPU registers (`s0`–`s31`, `FPSCR`) — only saved if the task has used the FPU since the last switch (lazy FP stacking)

The hardware **automatically** stacks half the core context on exception entry: `r0`–`r3`, `r12`, `lr`, `pc`, `xPSR` (the "exception frame"). FreeRTOS's PendSV handler **manually** stacks the rest: `r4`–`r11`. So the context is split between hardware-stacked and software-stacked, and lives entirely on **the task's own stack**. Each task therefore needs enough stack for: (a) its worst-case function call depth, plus (b) its worst-case ISR nesting depth, plus (c) the saved context (~64 bytes without FPU, ~200 with).

**Cost on a typical Cortex-M4 at 168 MHz**: a context switch is roughly 80–150 cycles, ~1 microsecond. At a 1 kHz tick this is negligible (0.1% overhead). It scales with how often you switch, which is why you don't make every function its own task.

### 3.5 Critical sections — `taskENTER_CRITICAL` / `taskEXIT_CRITICAL`

A **critical section** is a region of code where the scheduler will not preempt and (importantly) interrupts up to `configMAX_SYSCALL_INTERRUPT_PRIORITY` are masked.

```c
taskENTER_CRITICAL();
// non-atomic update of shared state
shared_buffer.head = (shared_buffer.head + 1) % N;
shared_buffer.count++;
taskEXIT_CRITICAL();
```

These macros are **nestable** — entering twice and exiting twice leaves you in the same state you started. The implementation is a counter, not a single bit.

Three iron rules:

1. **Keep them short.** Every cycle spent in a critical section is a cycle of ISR latency added to *every* interrupt below `configMAX_SYSCALL_INTERRUPT_PRIORITY`.
2. **Never block inside one.** Calling `vTaskDelay`, a blocking queue receive, or anything similar from inside a critical section is undefined behavior — the scheduler is locked out, so the task cannot actually be unblocked.
3. **Never call `printf`, the C library, or anything that might block, allocate, or take a long lock from inside one.**

If your only contention is between two tasks (no ISR involved), `vTaskSuspendAll()` / `xTaskResumeAll()` is gentler — it locks the scheduler but leaves interrupts fully enabled. For ISR-vs-task contention, use a critical section. For ISR-vs-ISR contention on the same priority, you don't need anything (ARM ISRs of the same priority don't preempt each other). For ISR-vs-ISR at different priorities, you need the higher one to be `< configMAX_SYSCALL_INTERRUPT_PRIORITY` or you need to manage priorities by hand (rare and risky).

### 3.6 When critical sections are dangerous

```c
// WRONG — blocks inside a critical section
taskENTER_CRITICAL();
xQueueSend(q, &item, portMAX_DELAY);  // will deadlock
taskEXIT_CRITICAL();

// WRONG — a 50 ms critical section means 50 ms of added ISR latency
taskENTER_CRITICAL();
for (int i = 0; i < 10000; i++) shared[i] = compute(i);
taskEXIT_CRITICAL();

// RIGHT — minimal, non-blocking
taskENTER_CRITICAL();
local_head = ring.head;
ring.head = (ring.head + 1) % N;
taskEXIT_CRITICAL();
ring.buf[local_head] = value;  // outside the critical section
```

#### 3.6.1 What an interviewer is really testing

Whether you understand the *mechanism* of how the kernel chooses the next task and effects a switch, and whether you treat critical sections as the surgical tool they are rather than a "make it safe" hammer.

#### 3.6.2 Likely questions

1. What exactly does the SysTick ISR do?
2. On a Cortex-M, how does the scheduler find the highest-priority ready task in O(1)?
3. What is saved on a context switch on Cortex-M4F, and where does it live?
4. Why is there a separate PendSV exception for the actual switch?
5. Difference between `taskENTER_CRITICAL` and `vTaskSuspendAll`?
6. If `configMAX_SYSCALL_INTERRUPT_PRIORITY` is 5, what does a critical section mask?
7. Can I call `vTaskDelay` from inside `taskENTER_CRITICAL`? Why or why not?

#### 3.6.3 Model answers

1. *"Increments the tick count, walks the delayed list to wake any tasks whose timeouts expired, handles round-robin if time slicing, and if a context switch is now needed, pends PendSV so the switch happens at the lowest exception priority. SysTick itself stays short."*
2. *"There's a 32-bit ready-priorities bitmap; bit N is set if any task at priority N is ready. `__CLZ` gives the highest set bit in one cycle. Then it picks the next TCB from that priority's ready list, round-robin."*
3. *"Hardware stacks `r0`–`r3`, `r12`, `lr`, `pc`, `xPSR` on exception entry; PendSV stacks `r4`–`r11` manually, plus FPU registers lazily if the FPU was used. All of it goes on the *task's* stack, not a separate kernel stack — that's why each task's stack must accommodate one full context."*
4. *"PendSV runs at the lowest exception priority, so it doesn't add latency to higher-priority ISRs. The tick ISR just *requests* a switch; the switch itself is deferred until no higher-priority work is pending. Without PendSV, switching from inside the tick handler would inflate latency for everything."*
5. *"`taskENTER_CRITICAL` masks interrupts up to the syscall priority *and* prevents context switches; it's heavy. `vTaskSuspendAll` only stops the scheduler but leaves interrupts fully on; it's lighter and is the right choice when the contention is between two tasks and not with an ISR."*
6. *"It raises `BASEPRI` to 5 — so any interrupt with a numerical priority value of 5 or higher (which on Cortex-M is *lower* urgency, since lower numbers are more urgent) is masked. Interrupts with priority numbers 0–4 are still serviced normally; those are non-FreeRTOS-aware ISRs that must not call any FreeRTOS API."*
7. *"No. The scheduler is locked out, so any task that would unblock me also can't run. Best case, my timeout expires and I get an error; worst case, deadlock. Critical sections are for short, non-blocking, register-or-RAM-level updates."*

#### 3.6.4 Traps and gotchas

- Saying "the SysTick ISR switches tasks" — it pends PendSV, which switches.
- Forgetting that the FPU context is stacked lazily — a task that *might* use the FPU but currently isn't doesn't pay the full cost.
- Confusing `taskENTER_CRITICAL` (used in tasks) with `taskENTER_CRITICAL_FROM_ISR` (used in ISRs, returns a saved interrupt-mask value to pass back to the matching exit).
- Putting `printf` in a critical section "to make logs atomic" — guaranteed pain.

---

## 4. Inter-task Communication and Synchronization

This is where most interview time is spent, because the difference between a candidate who has used FreeRTOS in production and one who has only read about it shows up here. The primitives all do something subtly different, and picking the wrong one gives code that *appears* to work in testing.

### 4.1 Queues — the fundamental primitive

A **queue** is a fixed-length FIFO that copies items by value. You set the item size and the queue length at creation; thereafter every send/receive copies `item_size` bytes in or out.

```c
typedef struct {
    uint16_t adc_value;
    uint32_t timestamp_ms;
} sample_t;

QueueHandle_t qSamples = xQueueCreate(16, sizeof(sample_t));
if (qSamples == NULL) { /* heap exhausted */ }

// Producer — typically a sensor task
sample_t s = { .adc_value = adc_read(), .timestamp_ms = now_ms() };
xQueueSend(qSamples, &s, pdMS_TO_TICKS(10));   // block up to 10ms if queue full

// Consumer
sample_t got;
if (xQueueReceive(qSamples, &got, portMAX_DELAY) == pdTRUE) {
    process(&got);
}
```

**Key properties:**

- **Copy semantics** — `s` on the producer's stack is *copied* into the queue's storage. The producer can reuse `s` immediately. This is great because there's no aliasing, no lifetime question, and no "who frees this?" problem. It's expensive only if the item is large (over ~32 bytes), in which case you queue *pointers* to a pool-allocated buffer instead.
- **Blocking** — sender blocks if full (or returns immediately if timeout is 0); receiver blocks if empty. Multiple tasks can wait on the same queue; the highest-priority waiter is unblocked first.
- **ISR-safe variants** — `xQueueSendFromISR`, `xQueueReceiveFromISR`. Section 5.

A queue with item size `sizeof(void*)` is the standard pattern for sending pointers to large objects:

```c
QueueHandle_t qFrames = xQueueCreate(4, sizeof(frame_t *));
frame_t *f = pool_alloc();
fill(f);
xQueueSend(qFrames, &f, 0);   // sends the POINTER, not the frame
```

### 4.2 Binary vs counting semaphores — when each is the right tool

A **semaphore** in FreeRTOS is internally a queue of length 1 (binary) or N (counting) with zero-byte items — what's queued is just the act of giving.

A **binary semaphore** has two states: available or not. It's the right tool for **signaling** — "an event happened" or "this resource is free." Use cases:

- ISR signals a task (the *original* deferred-interrupt pattern, though task notifications are now preferred — see 4.6).
- A driver task signals "transfer complete" to a waiter.

```c
SemaphoreHandle_t semDone = xSemaphoreCreateBinary();   // starts EMPTY

// Waiter
xSemaphoreTake(semDone, portMAX_DELAY);
process_completed_buffer();

// Signaler — could be ISR or another task
xSemaphoreGive(semDone);
```

**Pitfall**: a binary semaphore created with `xSemaphoreCreateBinary` starts *empty*. The first `Take` blocks. With `xSemaphoreCreateMutex`, it starts *available*. Easy to confuse.

A **counting semaphore** has a count from 0 to a max. It's the right tool for **resource counting** — you have N identical resources and tasks need to claim and release them — or for **deferred event counting** — an ISR happens many times and a task processes each one.

```c
// Resource pool of 4 DMA buffers
SemaphoreHandle_t semPool = xSemaphoreCreateCounting(4, 4);

// Acquire one
xSemaphoreTake(semPool, portMAX_DELAY);
buf_t *b = pool_get();
// ... use b ...
pool_put(b);
xSemaphoreGive(semPool);

// Event counting from ISR
SemaphoreHandle_t semBytes = xSemaphoreCreateCounting(64, 0);
// In ISR per byte:
xSemaphoreGiveFromISR(semBytes, &xHigherPri);
// In task:
xSemaphoreTake(semBytes, portMAX_DELAY);   // wakes once per byte
```

If a binary semaphore could lose events (ISR fires twice before task consumes once → second event lost), a counting semaphore is the fix. If you don't care about *count* and only need "did anything happen at all," a binary semaphore is right.

### 4.3 Mutexes — and why they are not just binary semaphores

Functionally a mutex looks like a binary semaphore — `Take` to lock, `Give` to unlock. The differences matter enormously:

| | Mutex | Binary Semaphore |
|---|---|---|
| Conceptual purpose | Protect shared resource | Signal an event |
| Owner tracking | Yes (the task that took it) | No |
| Priority inheritance | **Yes** | No |
| Must be given by the same task that took it? | Yes | No (anyone can give) |
| ISR-safe to give? | **No** — never give a mutex from an ISR | Yes |
| Initial state | Available | Empty |

The most important practical difference: **priority inheritance**. Without it, you get the **unbounded priority inversion** problem.

### 4.4 Priority inversion — the canonical RTOS bug

Three tasks: `H` (high), `M` (medium), `L` (low). `H` and `L` share a resource protected by lock `R`. `M` does not use `R`.

```
Time:  ──────────────────────────────────────────►
L:     [running][holds R    ][          ][holds R][gives R]
H:                       [waits on R............][takes R]
M:                          [running                ]
```

1. `L` is running and acquires `R`.
2. `H` becomes ready, preempts `L`. `H` tries to take `R` — blocks, because `L` has it.
3. While `L` is blocked-waiting-to-resume, `M` becomes ready. `M` is higher priority than `L`, so `M` runs. `L` is now starved.
4. As long as `M` is running, `L` cannot finish, so `R` cannot be released, so `H` cannot run.
5. `H` is effectively waiting on `M` — a task it has never heard of and that is lower priority than itself.

**This is unbounded** because the delay depends on how long `M` runs, not on how long the critical section is. The Mars Pathfinder bug (1997) was this exact problem.

**Priority inheritance** fixes it: when `H` blocks on `R` held by `L`, the kernel temporarily boosts `L`'s priority to `H`'s. Now `M` cannot preempt `L` (because boosted-`L` is at `H`'s priority). `L` finishes its critical section, releases `R`, and drops back to its own priority. `H` takes `R` and runs.

```c
SemaphoreHandle_t mutex = xSemaphoreCreateMutex();   // priority inheritance is automatic

void shared_state_writer(void *p) {
    for (;;) {
        xSemaphoreTake(mutex, portMAX_DELAY);
        // ... modify shared state ...
        xSemaphoreGive(mutex);
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

Priority inheritance is *not* a complete solution — it bounds the delay to the length of the critical section but doesn't eliminate it, and it does not solve deadlock. For deadlock you still need lock ordering.

### 4.5 Recursive mutexes

A normal mutex deadlocks if the holding task tries to take it again — it would block waiting for itself. A **recursive mutex** lets the same task take it multiple times; it must be given the same number of times to actually release.

```c
SemaphoreHandle_t rmutex = xSemaphoreCreateRecursiveMutex();

void outer(void) {
    xSemaphoreTakeRecursive(rmutex, portMAX_DELAY);
    inner();
    xSemaphoreGiveRecursive(rmutex);
}
void inner(void) {
    xSemaphoreTakeRecursive(rmutex, portMAX_DELAY);
    // ...
    xSemaphoreGiveRecursive(rmutex);
}
```

Use them when you have a public API where any function may be called from any other, and they all need the same lock. They have slightly higher overhead than non-recursive mutexes. If you can refactor to lock at exactly one entry point, prefer the non-recursive version.

### 4.6 Event groups — bit flags, AND/OR conditions

An **event group** is a 24-bit (or 8-bit, depending on `configUSE_16_BIT_TICKS`) bitmask. Tasks can wait for *any* (OR) or *all* (AND) of a set of bits to be set, optionally clearing the bits on exit.

This shines when a single task needs to wait for **multiple independent events** — semaphores can't express "wait for either A or B" without polling.

```c
#define EVT_BUTTON_PRESS  (1 << 0)
#define EVT_TIMEOUT       (1 << 1)
#define EVT_DATA_READY    (1 << 2)
#define EVT_ALL           (EVT_BUTTON_PRESS | EVT_TIMEOUT | EVT_DATA_READY)

EventGroupHandle_t evt = xEventGroupCreate();

// Task waits for ANY of three events
EventBits_t bits = xEventGroupWaitBits(
    evt,
    EVT_ALL,
    pdTRUE,       // clear on exit
    pdFALSE,      // wait for any (not all)
    portMAX_DELAY);

if (bits & EVT_BUTTON_PRESS) handle_button();
if (bits & EVT_TIMEOUT)      handle_timeout();
if (bits & EVT_DATA_READY)   handle_data();

// Setters
xEventGroupSetBits(evt, EVT_BUTTON_PRESS);
xEventGroupSetBitsFromISR(evt, EVT_DATA_READY, &xHigherPri);
```

When event groups replace multiple semaphores: any time you would otherwise create three binary semaphores and a polling loop to check them, that's an event group. Also useful for **task synchronization barriers** — N tasks each set their bit, and a coordinator waits for all bits set with the AND flag.

### 4.7 Task notifications — the lightweight, fast alternative

Task notifications were added in V8.2 (2015) and are now the recommended primitive for most one-task-to-one-task signaling. Each task has, built into its TCB, a 32-bit notification value and a notification state. No separate kernel object is created — the storage is already in the TCB you're paying for.

**Why modern code prefers them**:

- **45% faster** than the equivalent semaphore operation in benchmarks (no separate object lookup, no waiter list — there is exactly one waiter, the task itself).
- **Lower RAM** — no semaphore handle to allocate.
- **More expressive** — the 32-bit value can carry data, be set as a bitmask (replacing event groups for one-waiter cases), incremented (replacing counting semaphores), or overwritten.

The five notify modes:

| Action | Replaces |
|---|---|
| `eNoAction` | Just unblock |
| `eSetBits` | Event group |
| `eIncrement` | Counting semaphore |
| `eSetValueWithOverwrite` | Queue of length 1 |
| `eSetValueWithoutOverwrite` | Queue of length 1, fail if full |

```c
// ISR-to-task signaling — the modern deferred-interrupt pattern
void UART_IRQHandler(void) {
    BaseType_t xHigherPri = pdFALSE;
    if (UART->SR & RXNE) {
        rx_byte = UART->DR;     // grab byte before clearing interrupt
        vTaskNotifyGiveFromISR(hUartTask, &xHigherPri);
    }
    portYIELD_FROM_ISR(xHigherPri);
}

void vUartTask(void *p) {
    for (;;) {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);   // pdTRUE = clear-on-exit
        process_byte(rx_byte);
    }
}
```

The drawback: notifications are **point-to-point**. You can't have multiple tasks waiting on the same notification. For one-to-one signaling, use notifications. For one-to-many or many-to-many, use queues, semaphores, or event groups.

### 4.8 Stream buffers and message buffers

Stream and message buffers (added in V10) are **single-reader, single-writer** primitives optimized for byte streams.

- **Stream buffer** — a byte ring buffer. Variable amounts of data in, variable out. Right for UART RX/TX, audio sample streams.
- **Message buffer** — like a stream buffer, but each `Send` writes a length-prefixed message and each `Receive` returns one whole message. Right for delimited protocol frames.

They are **lock-free** between one writer and one reader (using memory barriers and atomic operations on head/tail), so they're cheaper than queues for the byte-stream case. The single-reader, single-writer constraint is the price of that speed: if you need multiple producers or consumers, use a queue.

```c
StreamBufferHandle_t sbUartRx = xStreamBufferCreate(
    256,    // buffer size in bytes
    1);     // trigger level — receiver unblocks when at least 1 byte available

// ISR
xStreamBufferSendFromISR(sbUartRx, &byte, 1, &xHigherPri);

// Task
uint8_t buf[64];
size_t n = xStreamBufferReceive(sbUartRx, buf, sizeof(buf), portMAX_DELAY);
```

### 4.9 Putting it together — when to use what

| You need to... | Use |
|---|---|
| Pass structured data, possibly many producers/consumers | **Queue** |
| Signal "an event happened" (one-shot) | **Task notification** (or binary semaphore if multi-waiter) |
| Count discrete events you might process later | Counting semaphore (or notification with `eIncrement`) |
| Protect a shared resource against concurrent access | **Mutex** (always — for the priority inheritance) |
| Wait for any/all of multiple independent events | **Event group** |
| Pass a byte stream from one writer to one reader (like UART) | **Stream buffer** |
| Pass discrete framed messages from one to one | **Message buffer** |

**Default choice for new ISR-to-task signaling**: task notification.
**Default choice for new task-to-task data passing**: queue.
**Default choice for new shared-resource protection**: mutex.
**Default choice for new "wait on multiple things"**: event group.

#### 4.9.1 What an interviewer is really testing

Whether you can pick the *right* primitive — not just any primitive — for a problem, and whether you understand priority inversion deeply enough to debug it. This is the section that separates senior from mid-level.

#### 4.9.2 Likely questions

1. Walk me through a priority inversion scenario, and how priority inheritance resolves it.
2. When would you use a counting semaphore over a binary one?
3. Why are mutexes not just binary semaphores with a different name?
4. Why can't I give a mutex from an ISR?
5. What's the advantage of task notifications over semaphores? When would I *not* use them?
6. Difference between queue and stream buffer?
7. How would you let one task wait for *either* a button press or a 5-second timeout, whichever comes first?
8. I have a queue with 32-byte items, and I'm getting performance issues. What would you change?

#### 4.9.3 Model answers

1. *"Three tasks, H, M, L. L holds a mutex. H tries to take it, blocks. M wakes up — M is higher than L — and runs, starving L. H is now blocked indefinitely on M, even though M doesn't even use the mutex. Priority inheritance means: when H blocks on the mutex L holds, L's priority is temporarily boosted to H's, so M can no longer preempt L. L finishes the critical section, releases the mutex, drops back to its own priority. H takes the mutex and runs. The Mars Pathfinder bug was this."*
2. *"When events can happen faster than the consumer processes them, and I care about each one. Bytes from a UART ISR — if the task is mid-processing one byte and two more arrive, a binary semaphore stays at 1 and I lose count. A counting semaphore increments to 3 and the task processes all three."*
3. *"Mutexes track ownership and provide priority inheritance — neither of which a binary semaphore does. The compiler doesn't enforce the difference, but using a binary semaphore for resource protection means you don't get inheritance, and unbounded priority inversion is back on the table."*
4. *"Mutexes track the owner task, and ISRs are not tasks. There's no task to attribute ownership to, and priority inheritance has no meaning in an ISR context. The kernel will assert if you try."*
5. *"Notifications are stored in the TCB, so there's no separate kernel object — they're faster (~45%) and use less RAM. The 32-bit value lets one notification carry the data a queue or semaphore wouldn't. The catch: only one task can wait on a given task's notification — point-to-point only. For multi-waiter, I still need a semaphore or queue."*
6. *"A queue copies fixed-size items and supports any number of senders and receivers, with full mutual exclusion. A stream buffer is a single-writer single-reader byte ring — much cheaper, but breaks if you have more than one of either side."*
7. *"Event group with two bits — one set by the button ISR, one set by a timer callback. Task waits with `xEventGroupWaitBits` on the OR, with the bits-clear-on-exit flag. Whichever fires first wakes the task."*
8. *"I'd switch to queueing pointers — `xQueueCreate(N, sizeof(payload_t*))` — and pass pointers into a buffer pool managed by a counting semaphore. The 32-byte copy on every send and every receive is wasted work; with a pointer it's 4 bytes."*

#### 4.9.4 Traps and gotchas

- Using a binary semaphore for shared-resource protection — no priority inheritance, eventual silent inversion bug.
- Forgetting that `xSemaphoreCreateBinary` starts empty — the first `Take` blocks forever if no one ever gives.
- Giving a mutex from an ISR — kernel assert or undefined behavior.
- Sending large structures by value through a queue when you could send a pointer.
- Using `vTaskDelay` to wait for an event ("it should be ready in 50 ms") instead of blocking on the actual event.
- Forgetting to use the FromISR variants in ISRs — non-FromISR APIs cannot be called from interrupt context.

---

## 5. Interrupts and the FromISR API

This section is where interview candidates either show fluency or get politely shown the door. Get it right, and the rest of the conversation is downhill.

### 5.1 Why "FromISR" variants exist

A regular FreeRTOS API call may **block** — it may walk linked lists, manipulate the ready list, and ultimately yield to another task. None of that is legal from an interrupt handler: ISRs cannot block, must return quickly, and cannot trigger context switches by the same mechanism a task does.

So FreeRTOS provides parallel `FromISR` variants for every API that can be called from interrupt context: `xQueueSendFromISR`, `xSemaphoreGiveFromISR`, `xEventGroupSetBitsFromISR`, `xTaskNotifyFromISR`, `vTaskNotifyGiveFromISR`. They:

1. Are non-blocking (they fail rather than wait if a queue is full, etc.).
2. Use the FromISR-safe critical sections (a `BASEPRI` save/restore pattern, not the task-level `taskENTER_CRITICAL`).
3. Do **not** trigger a context switch directly. Instead, they accept a `BaseType_t *pxHigherPriorityTaskWoken` out-parameter, set it to `pdTRUE` if the operation made a higher-priority task ready, and leave it to the caller to actually request the switch.

### 5.2 `portYIELD_FROM_ISR` and the higher-priority-task-woken pattern

The canonical structure of a FreeRTOS-aware ISR:

```c
void DMA1_Stream0_IRQHandler(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    if (DMA1->LISR & DMA_LISR_TCIF0) {
        DMA1->LIFCR = DMA_LIFCR_CTCIF0;     // clear interrupt flag
        xQueueSendFromISR(qDmaDone, &dma_status, &xHigherPriorityTaskWoken);
    }

    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

**Why this dance**: if the ISR's action made a higher-priority task ready (e.g., the task that was blocked on `qDmaDone`), the ISR should return *into that task*, not back into whatever was running before. `portYIELD_FROM_ISR` checks the flag and, if set, pends PendSV — the actual switch happens at PendSV exit, after this ISR (and any nested ISRs) finish.

If you forget the yield, the higher-priority task does not run until the next "natural" scheduling point — the next tick — which destroys the latency benefit of having an RTOS in the first place.

### 5.3 `configMAX_SYSCALL_INTERRUPT_PRIORITY` — the most-misunderstood macro

On Cortex-M, interrupt priorities are confusing because **lower numerical values are higher logical priority**. Priority 0 is the most urgent; priority 15 is the least urgent (on a chip with 4-bit priority encoding).

FreeRTOS divides ISRs into two tiers based on `configMAX_SYSCALL_INTERRUPT_PRIORITY`:

- **ISRs with priority < `configMAX_SYSCALL_INTERRUPT_PRIORITY`** (more urgent than the threshold) — these ISRs are **never masked** by FreeRTOS, run with the lowest possible latency, and **must not call any FreeRTOS API**, not even FromISR variants. They are "above" the kernel.
- **ISRs with priority >= `configMAX_SYSCALL_INTERRUPT_PRIORITY`** (less urgent than or equal to the threshold) — these are FreeRTOS-aware. They are masked during critical sections. They can use `xxx_FromISR` APIs.

**Picture it as a number line** (lower = more urgent on Cortex-M):

```
Priority 0 ──────────────────────► Priority 15
[MOST URGENT]                       [LEAST URGENT]

  Non-kernel ISRs  │  Kernel-aware ISRs
  (no FreeRTOS API)│  (must use FromISR APIs)
                   │
       configMAX_SYSCALL_INTERRUPT_PRIORITY (the line)
```

**The trap**: there is also a config macro `configKERNEL_INTERRUPT_PRIORITY` which is the priority *of the SysTick and PendSV exceptions themselves* — and it must be the lowest priority (highest numerical value). These two macros are different and frequently confused.

**Worse trap**: priority values must be written in the *shifted form* the chip actually uses. On STM32 with 4 priority bits, priority 5 is `5 << (8 - 4) = 0x50`. Setting `configMAX_SYSCALL_INTERRUPT_PRIORITY = 5` instead of `(5 << 4)` is a silent bug — the kernel masks at priority `5/16` which is "almost everything" or "nothing" depending on alignment. The `configASSERT`-based debug checks in modern FreeRTOS catch this if you enable them.

### 5.4 Deferred interrupt handling — why ISRs should be short

The pattern above — ISR does the absolute minimum (clear flag, capture data) and signals a task to do the actual work — is called **deferred interrupt handling**.

Why defer? Three reasons:

1. **ISR latency** — an ISR running blocks every other ISR at the same or lower priority. A 200-microsecond ISR is 200 microseconds of latency added to the next interrupt that needs to fire. Keep ISRs measured in *microseconds*, not milliseconds.
2. **Composition** — code that runs in an ISR cannot use most FreeRTOS APIs, cannot block, cannot use most C library calls, cannot allocate. Code that runs in a task can do all of those. By deferring, you escape the ISR-context restrictions.
3. **Prioritization** — once the work is in a task, the scheduler arbitrates it against other work. In an ISR, it preempts everything below its priority no matter what.

The split:

```c
// ISR — keep this short
void TIM2_IRQHandler(void) {
    BaseType_t xWoken = pdFALSE;
    TIM2->SR = ~TIM_SR_UIF;        // clear flag

    captured_t c = { TIM2->CCR1, ticks_now() };
    xQueueSendFromISR(qCapture, &c, &xWoken);

    portYIELD_FROM_ISR(xWoken);
}

// Task — does the heavy lifting
void vCaptureProcessor(void *p) {
    captured_t c;
    for (;;) {
        if (xQueueReceive(qCapture, &c, portMAX_DELAY)) {
            // expensive: filtering, FFT, logging, network publish, whatever
            process_capture(&c);
        }
    }
}
```

WRONG version, for contrast — actual ISRs that look like this exist in every codebase that hasn't been reviewed:

```c
// WRONG — long, blocks, uses C library
void TIM2_IRQHandler(void) {
    TIM2->SR = ~TIM_SR_UIF;
    char buf[64];
    snprintf(buf, sizeof(buf), "capture %lu\r\n", TIM2->CCR1);   // floats, locks
    HAL_UART_Transmit(&huart1, (uint8_t*)buf, strlen(buf), 100); // BLOCKS 100ms
    log_to_flash(buf);                                           // can take ms
}
```

#### 5.4.1 What an interviewer is really testing

Whether you understand the kernel/non-kernel ISR split, whether you can write a correct ISR pattern from memory, and whether you know to defer work. This is *the* most-asked area in embedded RTOS interviews.

#### 5.4.2 Likely questions

1. Walk me through writing a UART-RX ISR that hands bytes to a processing task.
2. What is `configMAX_SYSCALL_INTERRUPT_PRIORITY` and why does it matter?
3. Can I call `xQueueSend` from an ISR? Why or why not?
4. What is `portYIELD_FROM_ISR` doing under the hood?
5. I have an ISR at priority 2 (higher urgency than the kernel's 5). What can I call inside it?
6. My ISR sets a flag and a task polls it in a loop with `vTaskDelay(1)`. What's wrong?
7. My ISR does a `printf`. Walk me through what could go wrong.
8. What happens if I forget to pass `&xHigherPriorityTaskWoken` to a FromISR call?

#### 5.4.3 Model answers

1. *"The ISR clears the RXNE flag, reads the data register, and either pushes the byte into a stream buffer with `xStreamBufferSendFromISR` or notifies a task with `vTaskNotifyGiveFromISR`. It tracks the higher-priority-task-woken flag and ends with `portYIELD_FROM_ISR(xWoken)`. The processing task blocks on the stream buffer and decodes bytes into frames."*
2. *"It's the priority threshold that splits ISRs into kernel-aware and not. ISRs at priorities numerically below it are more urgent than the kernel can mask — they have minimum latency but can't call any FreeRTOS API. ISRs at or above it are masked during kernel critical sections and can call FromISR APIs. On Cortex-M with 4 priority bits, the value must be pre-shifted into the upper nibble."*
3. *"No — the non-FromISR variant assumes task context. It uses task-level critical sections, can yield, and can mutate the running task's state. From an ISR, use `xQueueSendFromISR`."*
4. *"On Cortex-M, it sets the PENDSVSET bit if the woken-flag is true, which schedules a context switch to run at PendSV's priority. The actual switch is deferred to PendSV, so it doesn't add latency to higher-priority pending interrupts. If the flag is false, it's a no-op."*
5. *"Nothing FreeRTOS-related, period. Plain register access, my own data structures with their own protection, my own logic. If I need to signal a task, I'd have to defer to a lower-priority kernel-aware ISR or use a hardware mechanism. This tier is reserved for hard-real-time work like motor commutation."*
6. *"Polling defeats the purpose of having an RTOS. The task wastes the 1 ms tick waiting after the flag is set, ISR latency-to-processing is up to 1 ms instead of microseconds, and the polling task is hot when it should be blocked. Replace it with a binary semaphore or task notification — `xSemaphoreGiveFromISR` in the ISR, `xSemaphoreTake(..., portMAX_DELAY)` in the task."*
7. *"`printf` is not reentrant in most C libraries — it can deadlock if interrupted mid-call by another `printf`. It uses heap. It can take milliseconds. It can call `malloc`. Doing this in an ISR causes ISR latency to balloon, can corrupt the stdout state of an interrupted task, and can deadlock the heap. The pattern is: ISR enqueues a small struct describing what should be logged, a logger task formats and prints."*
8. *"Compile error if the prototype is correct — it's a required parameter. If somehow it compiled, the kernel never knows a higher-priority task is now ready, so the ISR returns to the previously running task and the woken task waits for the next tick or scheduling point — destroying real-time response."*

#### 5.4.4 Traps and gotchas

- Setting `configMAX_SYSCALL_INTERRUPT_PRIORITY` to a small unshifted value like `5` instead of `(5 << (8 - configPRIO_BITS))`.
- Mixing up `configKERNEL_INTERRUPT_PRIORITY` (priority of SysTick/PendSV — should be lowest) with `configMAX_SYSCALL_INTERRUPT_PRIORITY` (the threshold — should be high enough to leave room for "above kernel" ISRs).
- Calling FreeRTOS API from an ISR running above `configMAX_SYSCALL_INTERRUPT_PRIORITY` — silent corruption.
- Forgetting `portYIELD_FROM_ISR` — system "works" but is laggy.
- Long work in the ISR: floats, library calls, blocking driver calls, retries with delays.
- Letting an ISR allocate from the FreeRTOS heap — `pvPortMalloc` is not ISR-safe.

---

## 6. Memory Management

FreeRTOS does not use the C library's `malloc`/`free`. It uses its own `pvPortMalloc`/`vPortFree`, implemented by one of five heap files you choose at compile time. Pick the wrong one for your system and you'll suffer.

### 6.1 The five heap implementations — what each does, when to use which

| Heap | Allocate | Free | Fragmentation | Determinism | Use when |
|---|---|---|---|---|---|
| **heap_1** | Yes | **No** | None (you can't free) | Perfect | Safety-critical, all allocation up-front, never freed. Static-task-like behavior with the convenience of `xTaskCreate`. |
| **heap_2** | Yes | Yes | Yes (no coalescing) | Good | Legacy. Don't use in new code. heap_4 is strictly better. |
| **heap_3** | Yes | Yes | Whatever your `malloc` does | Whatever your libc does | You already have a working libc heap and want FreeRTOS to use it. Mostly: you don't want this. |
| **heap_4** | Yes | Yes | Yes, but **coalesces** adjacent free blocks | Good | **Default choice for most projects.** General-purpose; best fragmentation behavior of the freeable heaps. |
| **heap_5** | Yes | Yes | Like heap_4 | Like heap_4 | Same algorithm as heap_4 but supports **multiple non-contiguous memory regions**. For chips where RAM is split (e.g., DTCM + SRAM1 + SRAM2 on STM32H7). |

**Decision flow:**

- Allocate everything in `main()` before `vTaskStartScheduler` and never free? → **heap_1**, or skip the heap entirely with static allocation.
- Allocate and free during normal operation, single contiguous RAM region? → **heap_4**.
- Multiple RAM regions you want pooled together? → **heap_5**.
- Already have a tested libc heap and want one allocator? → **heap_3** (rare, slightly slower than heap_4, less deterministic).

### 6.2 Static vs dynamic allocation, `configSUPPORT_STATIC_ALLOCATION`

Every FreeRTOS object — task, queue, semaphore, mutex, event group, software timer, stream buffer — has both a `xQueueCreate(...)` (dynamic) and an `xQueueCreateStatic(..., buffer, struct)` (static) constructor. Set `configSUPPORT_STATIC_ALLOCATION = 1` to enable the static variants.

**Static benefits**:

- No heap needed for kernel objects (heap_1 with no allocations means you can leave heap effectively empty).
- Deterministic placement — buffer goes wherever your linker puts the `static` variable, including specific RAM regions, DTCM, etc.
- Failure mode is link-time, not run-time.

**Static cost**: more verbose — every object needs storage declared.

**The hybrid pattern**: enable both, allocate kernel objects (TCBs, queues) statically, allocate application data (buffers in pools) dynamically. This is what production safety-critical firmware looks like.

If you set `configSUPPORT_STATIC_ALLOCATION = 1`, you must also provide two callbacks for the kernel to ask you for the idle task's and timer task's storage:

```c
void vApplicationGetIdleTaskMemory(StaticTask_t **ppxTcb,
                                   StackType_t **ppxStack,
                                   uint32_t *pulStackSize) {
    static StaticTask_t xIdleTcb;
    static StackType_t  xIdleStack[configMINIMAL_STACK_SIZE];
    *ppxTcb = &xIdleTcb;
    *ppxStack = xIdleStack;
    *pulStackSize = configMINIMAL_STACK_SIZE;
}
// And vApplicationGetTimerTaskMemory analogously
```

Forgetting this gives a link error — which is the right way to fail.

### 6.3 Stack overflow detection — method 1 vs method 2

`configCHECK_FOR_STACK_OVERFLOW` enables runtime overflow detection in the context-switch path:

- **Method 1** (= 1) — On every context switch, the kernel checks whether the outgoing task's saved SP is within its allocated stack range. **Cheap, but only catches the overflow if it persists at switch time.** A transient deep-stack-call that comes back before the next switch is missed.
- **Method 2** (= 2) — On task creation, the kernel fills the stack with a known pattern (`0xA5A5A5A5`). On every context switch, it checks the **last 16 bytes** of the stack to see if any of the pattern was overwritten. **Catches more overflows, including transient ones near the limit.** Slightly more expensive.

**Always use method 2 in development.** The cost is a few percent of context-switch time; the value is catching the bug that otherwise corrupts random TCBs and gives you a "system mysteriously hangs once a week" support ticket.

You also provide a hook:

```c
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcName) {
    // Disable interrupts and stop here. Print pcName if you have a way to do so safely.
    taskDISABLE_INTERRUPTS();
    for (;;);
}
```

In production, this hook should log to non-volatile memory and reset, so you can diagnose post-mortem.

### 6.4 `uxTaskGetStackHighWaterMark` — measuring how close you are to overflow

This API returns the **smallest amount of stack space that has remained unused** since the task started — measured in stack words. Combine it with method 2 (which fills the stack with a pattern) to get an honest estimate of stack usage.

```c
void vMonitorTask(void *p) {
    for (;;) {
        UBaseType_t freeWords = uxTaskGetStackHighWaterMark(NULL);   // for self
        if (freeWords < 32) {
            log_warn("Stack low: %u words free", (unsigned)freeWords);
        }
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

**The right workflow**:

1. Size each task's stack generously during development (start with 512 words for a non-trivial task, 128 for a trivial one).
2. Run the system through every code path you can — error paths matter most.
3. Inspect high-water marks. If a task has 200 words free out of 512, you can shrink to ~300 (leave headroom for the worst case you didn't exercise — for ISR nesting and for compiler optimizations changing).
4. Tighten and re-run.

Don't tighten without measuring. Don't measure once on a happy path and ship. The deepest stack usage is almost always in error paths or rare combinations of ISR nesting.

### 6.5 The malloc-failed hook

When `pvPortMalloc` fails:

```c
void vApplicationMallocFailedHook(void) {
    taskDISABLE_INTERRUPTS();
    for (;;);
}
```

In production, log and reset. In development, sit here in a debugger so you can find what was being allocated.

#### 6.5.1 What an interviewer is really testing

Whether you treat memory as a finite, predictable resource or as something that "usually works", and whether you instrument for stack overflow before it bites you in the field.

#### 6.5.2 Likely questions

1. Walk me through the five heap options and which you'd pick by default.
2. Why is heap_4 better than heap_2?
3. What does "stack overflow detection method 2" do that method 1 doesn't?
4. How do you size a task stack?
5. Static vs dynamic allocation — when would you mix them?
6. Is `pvPortMalloc` thread-safe? Is it ISR-safe?
7. What's `uxTaskGetStackHighWaterMark` measuring?

#### 6.5.3 Model answers

1. *"heap_1: alloc only, no free, perfect determinism, for systems that allocate up-front. heap_2: legacy, don't use. heap_3: thunk to libc malloc. heap_4: malloc-and-free with coalescing — my default. heap_5: heap_4 across multiple regions, for chips with split RAM. I default to heap_4 unless I have a specific reason."*
2. *"heap_2 doesn't coalesce free blocks, so the heap fragments under any non-trivial allocation pattern and you eventually fail to allocate even though there's plenty of total free memory. heap_4 merges adjacent free blocks on free, which keeps fragmentation bounded for typical workloads."*
3. *"Method 1 just checks the stack pointer at context switch — only catches overflows that persist at the switch point. Method 2 fills the stack with a sentinel pattern at task creation and checks whether the last 16 bytes still hold the pattern at every switch — catches transient deep-stack incursions that recovered before the switch. Always method 2 in development."*
4. *"Start oversize, run all paths I can think of, read `uxTaskGetStackHighWaterMark` to see how much was unused, leave a margin for the paths I missed and for ISR nesting, then tighten. I never tighten without measuring."*
5. *"Production firmware: kernel objects (tasks, queues, mutexes) static — link-time accounting, no heap dependency for kernel state. Application data buffers dynamic from a heap, ideally fixed-size pools rather than arbitrary allocations. Mix gives me determinism where it matters and flexibility where I need it."*
6. *"Thread-safe — yes, it takes a critical section internally. ISR-safe — no. Never call `pvPortMalloc` from an ISR. Pre-allocate buffers and use a pool."*
7. *"The smallest free-stack-words count the task has reached since it started. Combined with method 2's stack-fill pattern, it tells me the actual high-water of stack usage — what I need to size against."*

#### 6.5.4 Traps and gotchas

- Specifying stack sizes in bytes when the API wants words (or vice-versa).
- Using heap_2 in new code.
- Disabling stack overflow detection "for performance" without measuring the cost.
- Calling `pvPortMalloc` from an ISR.
- Freeing a buffer that was passed through a queue — both sides need a clear ownership rule.
- Sizing stacks once on the happy path and shipping.

---

## 7. Timers

### 7.1 Software timers vs hardware timers

A **hardware timer** is a peripheral on the MCU. You configure a counter, an autoreload, and an interrupt; when it fires, your ISR runs at hardware-defined priority. Resolution is whatever the timer clock supports — sub-microsecond on most modern MCUs.

A **FreeRTOS software timer** is a kernel object. It does not have its own hardware. It runs entirely in software, on top of the system tick. Its resolution is therefore *one tick* — typically 1 ms. Its callback runs in the **timer service task**, not in an ISR.

**Use a hardware timer when**: you need sub-millisecond resolution; you need the callback to be deterministic regardless of system load; you're driving something time-critical (PWM updates, motor commutation, audio sample tick).

**Use a software timer when**: you have many independent timeouts (LED blink, "session inactivity," "reconnect retry," "watchdog kick"); millisecond resolution is fine; you want to avoid burning a hardware timer per timeout.

A typical embedded system uses 2–3 hardware timers and 10+ software timers concurrently.

### 7.2 The timer service task and the timer command queue

Software timers are managed by a kernel task — the **timer service task** — created by `vTaskStartScheduler` if `configUSE_TIMERS = 1`. Tasks (including ISRs) don't talk to timers directly; they send commands (start, stop, reset, change period) through the **timer command queue**, and the timer service task picks them up.

This is why there are config knobs:

- `configTIMER_TASK_PRIORITY` — make this *high* if you want timer callbacks to be punctual under load (a common choice is `configMAX_PRIORITIES - 1`).
- `configTIMER_QUEUE_LENGTH` — how many pending timer commands can be queued.
- `configTIMER_TASK_STACK_DEPTH` — stack for the timer task; size it for the worst callback you'll write.

If your timer callbacks are blocked by higher-priority tasks, the callbacks fire late. **The timer service task being lower priority than your other tasks is a bug pattern**, not a "tradeoff".

### 7.3 One-shot vs auto-reload

```c
// Auto-reload — fires repeatedly every 1000 ms
TimerHandle_t hHeartbeat = xTimerCreate(
    "Heartbeat",
    pdMS_TO_TICKS(1000),
    pdTRUE,                  // pdTRUE = auto-reload, pdFALSE = one-shot
    NULL,                    // timer ID — opaque pointer for your use
    vHeartbeatCallback);
xTimerStart(hHeartbeat, 0);

// One-shot — fires once 5 seconds after start
TimerHandle_t hReconnect = xTimerCreate(
    "Reconnect", pdMS_TO_TICKS(5000), pdFALSE, NULL, vReconnectCallback);
// Start it from inside a connection-failed handler:
xTimerStart(hReconnect, 0);

void vReconnectCallback(TimerHandle_t t) {
    try_reconnect();   // runs in timer task
}
```

**Reset** is the most useful timer operation people forget about: `xTimerReset(hTimer, 0)` restarts a timer's countdown from zero. The classic use is **debouncing**:

```c
// On every button-edge interrupt, reset a 50 ms one-shot timer
// Button considered "pressed" when the timer actually fires (edge has been quiet 50 ms)
void EXTI0_IRQHandler(void) {
    BaseType_t xWoken = pdFALSE;
    EXTI->PR = EXTI_PR_PR0;
    xTimerResetFromISR(hDebounce, &xWoken);
    portYIELD_FROM_ISR(xWoken);
}
void vDebounceCallback(TimerHandle_t t) {
    // 50 ms after the last edge — read pin and dispatch
    bool pressed = !(GPIOA->IDR & (1 << 0));
    if (pressed) xQueueSend(qButton, &(button_evt_t){...}, 0);
}
```

### 7.4 Why timer callbacks must never block

Callbacks run **in the timer service task**. The timer task processes them sequentially. If one callback blocks for 100 ms, every other software timer with a callback queued behind it fires 100 ms late. If one callback blocks forever, *every* software timer in the system stops.

The rule is: a timer callback must do small, non-blocking work — set a flag, signal a task via notification or queue, kick a hardware register. If real work needs doing, defer it to a worker task.

```c
// WRONG — block in callback
void vBadCallback(TimerHandle_t t) {
    xSemaphoreTake(mutex, portMAX_DELAY);   // could block forever
    do_long_work();
    xSemaphoreGive(mutex);
}

// RIGHT — signal a task
void vGoodCallback(TimerHandle_t t) {
    xTaskNotifyGive(hWorker);   // returns immediately
}
```

#### 7.4.1 What an interviewer is really testing

Whether you understand that software timers run in a task — not in an ISR, not "magically" — and what that implies for priority assignment and callback discipline.

#### 7.4.2 Likely questions

1. Where does a software timer callback execute?
2. Software vs hardware timers — how do you choose?
3. What happens if a software timer callback blocks?
4. How would you debounce a button using a software timer?
5. What does `xTimerReset` do that `xTimerStart` doesn't?
6. I see software timer callbacks firing late. Where do I look?

#### 7.4.3 Model answers

1. *"In the timer service task, which is a regular FreeRTOS task created by the kernel. So callbacks are subject to scheduling — they're delayed by anything higher priority — and they run in task context, meaning they can call regular task APIs but must not block, because they share the timer task with every other software timer."*
2. *"Hardware timer when I need sub-millisecond resolution, deterministic callback timing, or I'm driving a peripheral like PWM. Software timer for everything else — LED blinks, reconnect timeouts, watchdog kicks. Hardware timers are scarce; software timers are essentially free in count."*
3. *"Every other software timer waits behind it. If it blocks indefinitely, all software timers in the system stop. Callbacks must be short and non-blocking — signal a task to do work, don't do work in the callback."*
4. *"In the GPIO ISR, on every edge, call `xTimerResetFromISR` on a 50 ms one-shot timer. The timer only fires when 50 ms passes without another edge. The callback then samples the line and dispatches a button event. The bouncing edges keep resetting the timer; the stable state lets it fire."*
5. *"`xTimerStart` starts a timer that may already be running and updates its expiry to (now + period). `xTimerReset` restarts the countdown from zero — if it's already running, it pushes the expiry back. For debounce-style 'extend the timeout on every event' patterns, reset is the right one."*
6. *"First check the timer task priority — if it's lower than busy application tasks, the timer task is starved and callbacks queue up. Then check whether some callback is blocking or running long. Then check the timer command queue depth — if it's full, commands are being silently dropped. `vTaskGetRunTimeStats` shows the timer task's CPU share."*

#### 7.4.4 Traps and gotchas

- Treating timer callbacks like ISRs and trying to use FreeRTOS-from-ISR APIs in them.
- Setting `configTIMER_TASK_PRIORITY` low and wondering why callbacks are imprecise.
- Blocking in a callback — even briefly, with `xSemaphoreTake` and a small timeout.
- Using a software timer for sub-millisecond timing — that's a hardware timer's job.
- Creating one-shot timers without a plan for their lifecycle (do you delete them? reuse them?).

---

## 8. Advanced and Production Topics

### 8.1 Tickless idle and low-power design

The default kernel tick fires every millisecond regardless of whether anything needs to happen. On a battery-powered device, that's 1000 wakeups per second from sleep — most of them pointless.

**Tickless idle** disables the periodic tick when the system goes idle. The kernel computes how long until the next task is due to run (the next delayed task's wake time), reprograms a hardware timer to wake the CPU at that point, and enters deep sleep. When the chip wakes (either from the programmed timer or from any interrupt), the kernel adjusts its tick count to compensate for the time spent sleeping and resumes normally.

```c
// FreeRTOSConfig.h
#define configUSE_TICKLESS_IDLE             1
#define configEXPECTED_IDLE_TIME_BEFORE_SLEEP 2   // ticks; under this, don't bother sleeping
```

Two important caveats:

1. **Peripherals that need the tick must be considered**. UART RX timeout based on tick won't fire while sleeping; a hardware timer-based timeout is needed.
2. **Errata**. Some MCUs have wake-up latency or oscillator restart times that eat into the savings. Measure actual current with a scope-grade ammeter, not just trust the math.

For aggressive low-power: `configPRE_SLEEP_PROCESSING` and `configPOST_SLEEP_PROCESSING` macros let you hook in your own code to power down peripherals before sleep and bring them back up after.

### 8.2 SMP FreeRTOS — multi-core

Since V11 (2024), FreeRTOS officially supports symmetric multiprocessing (SMP) on multi-core MCUs (RP2040, RP2350, ESP32 dual-core, some Cortex-R series).

The differences from single-core:

- **Cores share the ready list**. The scheduler picks the highest N ready tasks to run on N cores.
- **Core affinity** — you can pin a task to a specific core (`vTaskCoreAffinitySet`) or let it migrate. Pinning is essential when a task touches hardware that's per-core.
- **Critical sections are global** — `taskENTER_CRITICAL` now also acquires a spin-lock, since the "stop the scheduler" semantics aren't enough on multi-core. Critical sections cost more.
- **Interrupts can fire on either core** — affinity for interrupts is a hardware concern.
- **Same-priority round-robin** behaves differently — you can have N tasks at the same priority all running concurrently.

Most concepts (queues, semaphores, mutexes) work the same way; the kernel handles cross-core coordination internally. Single-core code generally compiles and runs on SMP, but you should review every shared resource for whether the cross-core path was considered.

### 8.3 Trace and runtime statistics

Two built-in mechanisms, plus third-party tooling:

#### `vTaskGetRunTimeStats`

Requires `configGENERATE_RUN_TIME_STATS = 1` and two macros that hook a high-resolution counter (a hardware timer running at 10–20× the tick rate is typical):

```c
// FreeRTOSConfig.h
#define configGENERATE_RUN_TIME_STATS  1
#define portCONFIGURE_TIMER_FOR_RUN_TIME_STATS()  start_dwt_cyccnt()
#define portGET_RUN_TIME_COUNTER_VALUE()          (DWT->CYCCNT)
```

Then:

```c
char buf[512];
vTaskGetRunTimeStats(buf);
// Outputs:
// Task            Abs Time         % Time
// ----------      --------         ------
// IDLE            0001AC73D8         92%
// Sensor          0000031E2A          2%
// Comm            00000ABF14          5%
// ...
```

`IDLE` percentage is your CPU headroom. If it's under 20%, the system is hot and may not survive a worst-case event.

#### Tracealyzer / Percepio Tracealyzer

The industry-standard FreeRTOS visualizer. You compile in lightweight trace recorders that log every kernel event (task switch, queue send, ISR, semaphore take/give) into a ring buffer; export the buffer over UART, RTT, or USB; load it in Tracealyzer's GUI. You see the actual timeline — *which* task ran, when it preempted, when an ISR fired, when a queue send unblocked which receiver. This is *the* tool for diagnosing real-time bugs.

### 8.4 Common debugging patterns

**Hard fault analysis on Cortex-M.** When a hard fault occurs:

1. The exception stack frame holds `r0`–`r3`, `r12`, `lr`, `pc`, `xPSR` of the offending instruction.
2. Read the fault status registers (`CFSR`, `HFSR`, `MMFAR`, `BFAR`) to know *what* faulted (bus fault, usage fault, mem-manage fault) and *where*.
3. The `pc` in the stack frame points to the faulting instruction. Cross-reference with the map file or `addr2line`.
4. The fault might not be in the offending task's code — it might be in a callee. Walk the call stack via the saved `lr` and via stack scanning.

A standard hard-fault handler that captures and dumps this is essential — most CMSIS HAL handlers just spin in a tight loop, which is useless.

**Stack corruption.** Symptoms: random crashes that move with code changes, garbage in supposedly-stable variables, fault PCs that point to nowhere sensible. Tools: stack overflow detection method 2, `uxTaskGetStackHighWaterMark`, fill stacks with a known pattern at create time.

**Deadlock.** Two tasks each hold a mutex the other needs. Symptoms: two specific tasks both stuck Blocked; everything else fine; system "hangs" in a way the watchdog catches. Diagnosis: a debugger break shows which task is blocked on which mutex; trace tools show the lock-acquisition order. Fix: a global lock-ordering rule. If task A ever needs both mutex X and mutex Y, *every* task takes X before Y.

**Starvation.** A low-priority task never makes progress because higher-priority tasks always have something to do. Symptoms: telemetry stops updating, log buffer fills up, housekeeping tasks fall behind. Diagnosis: `vTaskGetRunTimeStats` shows the starved task at 0%. Fix: lower the busy task's priority, or split its work, or use mutex priority inheritance, or restructure with a worker pool.

### 8.5 MPU support and memory protection

Cortex-M3/M4/M7/M33 have an MPU (Memory Protection Unit). FreeRTOS-MPU is a variant that runs each task in unprivileged mode and uses the MPU to restrict each task's accessible memory regions.

Without MPU: any task can stomp on any memory. A pointer bug in task A corrupts task B's data.

With MPU: a task that touches memory outside its declared regions takes a memory-management fault. The kernel itself runs privileged; tasks can be configured per-region with read/write/execute attributes.

Cost: setup is more verbose (you declare regions per task), context switches reload MPU regions, and not all FreeRTOS APIs are unprivileged-callable (you go through SVC for the ones that aren't). Use it for safety-critical or for separating untrusted code (e.g., a third-party plugin) from trusted code.

### 8.6 Integrating with C++, HAL drivers, TCP/IP stacks, file systems

**C++**: FreeRTOS is C, but it integrates fine with C++. A common pattern is a thin RAII wrapper around mutex and queue:

```cpp
class ScopedLock {
    SemaphoreHandle_t m_;
public:
    ScopedLock(SemaphoreHandle_t m) : m_(m) { xSemaphoreTake(m_, portMAX_DELAY); }
    ~ScopedLock() { xSemaphoreGive(m_); }
};

void some_function() {
    ScopedLock lock(g_mutex);
    // ... critical section ...
}   // lock released here, even on exception (if exceptions are enabled)
```

Avoid running task functions as non-static class methods — task entry must be a plain C-callable function. Use a static trampoline that casts the parameter back to the object pointer.

**HAL drivers (STM32 HAL, NXP MCUXpresso)**: HAL functions often busy-wait. In FreeRTOS, replace with interrupt-or-DMA versions and signal task wakeup with a notification. `HAL_UART_Transmit` (blocking) becomes `HAL_UART_Transmit_DMA` plus a notification from the transfer-complete callback.

**TCP/IP stacks**: FreeRTOS+TCP is the first-party option, designed for FreeRTOS. lwIP is the de-facto standard third-party stack with FreeRTOS sys-arch glue. Both create a "TCP/IP task" that owns the stack state; application tasks call into it via socket-like APIs.

**File systems**: FreeRTOS+FAT (FAT12/16/32, exFAT) and FatFs (the older de-facto choice). Both need a block-device driver that's FreeRTOS-aware (i.e., uses task notification rather than busy-wait for SD card transactions).

#### 8.6.1 What an interviewer is really testing

Whether you've actually shipped FreeRTOS in production. Tickless idle, runtime stats, and trace tooling are things you only know from having had to debug a real system.

#### 8.6.2 Likely questions

1. How would you make a battery-powered FreeRTOS device sleep most of the time?
2. What does Tracealyzer give you that `vTaskGetRunTimeStats` doesn't?
3. Walk me through diagnosing a hard fault on Cortex-M.
4. I see two tasks both Blocked and the system hung. What do I check first?
5. How do you detect stack overflow in a release build?
6. SMP FreeRTOS — what changes about critical sections?
7. We have a HAL driver that does `HAL_UART_Transmit(...)`. What's wrong with using that as-is in a FreeRTOS task, and how would you change it?

#### 8.6.3 Model answers

1. *"Enable tickless idle. Set `configUSE_TICKLESS_IDLE = 1` and use `pre_sleep` / `post_sleep` hooks to power down peripherals. Replace tick-based timeouts in driver code with hardware-timer-based ones, since the tick stops during sleep. Profile with a current meter to verify sleep current matches datasheet — and watch for peripherals that prevent the deepest sleep modes."*
2. *"Runtime stats give per-task CPU share — useful for headroom checks. Tracealyzer shows the timeline: which task ran when, when ISRs fired, when one task signaled another, the latency from ISR to woken task. For diagnosing missed deadlines or inversions, the timeline view is what you need; aggregate percentages can't show ordering bugs."*
3. *"Read the fault status registers — CFSR tells me whether it's bus, usage, or mem-manage; HFSR tells me if it escalated. The exception stack frame has the PC of the faulting instruction; I cross-reference with the map file. If it's a bus fault, BFAR has the address. I also examine the LR for FPU lazy-stacking flags and to walk back into the calling function. The handler should capture all of this to non-volatile memory before resetting."*
4. *"Deadlock first — likely each holds a mutex the other needs. I look at what each task is blocked on with the debugger, then check whether they're acquiring locks in different orders. The fix is a global lock-ordering rule."*
5. *"Method 2 stack overflow detection runs in release with negligible cost — it just compares the last 16 bytes of stack against the fill pattern on every context switch. The hook function should log to non-volatile memory, set a fault flag, and reset, so the device recovers and we can post-mortem. Don't ship without this."*
6. *"On single-core, a critical section is mostly free — it just raises BASEPRI and bumps a counter. On SMP, it has to also acquire a spin-lock to prevent the other core(s) from racing. Critical sections become genuinely expensive, and the rule 'keep them short' graduates from good practice to load-bearing."*
7. *"`HAL_UART_Transmit` busy-waits on TXE — it burns CPU during transmission, blocks higher-priority tasks from running, and gives no chance for the scheduler to switch. I'd replace it with `HAL_UART_Transmit_DMA` (or `_IT`), have the transmit-complete callback `vTaskNotifyGiveFromISR` to a waiting task, and `ulTaskNotifyTake` in the API wrapper. Same logical interface, doesn't waste cycles."*

#### 8.6.4 Traps and gotchas

- Enabling tickless idle without auditing the rest of the code for tick-dependent timing.
- Treating `vTaskGetRunTimeStats` as proof of correctness — high CPU headroom doesn't catch ordering bugs or deadlocks.
- Shipping with the default CMSIS hard-fault handler that just spins.
- Using HAL blocking functions in tasks because "they work" — they work *until* something else needs the CPU.
- Disabling stack overflow detection in release "for size".
- Moving from single-core to SMP without re-auditing every shared resource.

---

## 9. Design Patterns and Anti-Patterns

### 9.1 Producer-consumer with a queue

The base pattern of any RTOS application: one task produces work items, another consumes them, decoupled by a queue.

```c
// Sensor task produces
void vSensorTask(void *p) {
    sample_t s;
    TickType_t xLast = xTaskGetTickCount();
    for (;;) {
        s.value = sensor_read();
        s.ts = xLast;
        // Drop if queue full — better than blocking the sensor at the cost of one stale sample
        xQueueSend(qSamples, &s, 0);
        vTaskDelayUntil(&xLast, pdMS_TO_TICKS(10));
    }
}

// Filter task consumes, processes, forwards
void vFilterTask(void *p) {
    sample_t s, filtered;
    for (;;) {
        if (xQueueReceive(qSamples, &s, portMAX_DELAY)) {
            filtered = apply_filter(&s);
            xQueueSend(qFiltered, &filtered, pdMS_TO_TICKS(50));
        }
    }
}
```

Why this works: the sensor task's timing is decoupled from the filter task's timing. If the filter is briefly busy, samples buffer in the queue. If the queue overflows, *the sensor* decides what to do (drop, log, alert) — not the filter task or the kernel.

### 9.2 Deferred ISR handling with a task notification

The modern pattern for ISR-to-task signaling — what every UART, SPI, I2C, ADC-DMA driver should look like:

```c
TaskHandle_t hUart = NULL;

void USART2_IRQHandler(void) {
    BaseType_t xWoken = pdFALSE;
    if (USART2->ISR & USART_ISR_RXNE) {
        rx_byte = USART2->RDR;          // read clears the flag
        vTaskNotifyGiveFromISR(hUart, &xWoken);
    }
    portYIELD_FROM_ISR(xWoken);
}

void vUartTask(void *p) {
    for (;;) {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
        process(rx_byte);
    }
}
```

For multi-byte: replace `rx_byte` with a stream buffer.

### 9.3 Resource manager task

Pattern: a single task owns a hardware resource (LCD, SPI bus, flash chip, file system). Other tasks send it commands via a queue. The resource is never touched directly by anyone else.

```c
typedef struct { lcd_op_t op; uint16_t x, y; const char *str; } lcd_cmd_t;
QueueHandle_t qLcd;

void vLcdTask(void *p) {
    lcd_cmd_t cmd;
    for (;;) {
        if (xQueueReceive(qLcd, &cmd, portMAX_DELAY)) {
            switch (cmd.op) {
                case LCD_PUTS:  lcd_puts(cmd.x, cmd.y, cmd.str); break;
                case LCD_CLEAR: lcd_clear();                     break;
                // ...
            }
        }
    }
}

// Anyone can post commands; the resource is implicitly serialized by the queue
void lcd_print(uint16_t x, uint16_t y, const char *s) {
    lcd_cmd_t cmd = { LCD_PUTS, x, y, s };
    xQueueSend(qLcd, &cmd, pdMS_TO_TICKS(100));
}
```

Compared to "everyone takes a mutex around the LCD": no lock contention, no inversion possible, much easier to extend (commands can be batched, ordered, prioritized via multiple queues), failure mode is queue-full instead of mutex-deadlock.

### 9.4 State machine inside a task

A task that processes events from a queue and acts according to a state variable.

```c
typedef enum { IDLE, CONNECTING, CONNECTED, ERROR } state_t;

void vNetTask(void *p) {
    state_t state = IDLE;
    net_evt_t evt;
    for (;;) {
        if (xQueueReceive(qNet, &evt, portMAX_DELAY)) {
            switch (state) {
                case IDLE:
                    if (evt.type == EVT_CONNECT_REQ) { start_connect(); state = CONNECTING; }
                    break;
                case CONNECTING:
                    if (evt.type == EVT_CONNECTED)    { state = CONNECTED; }
                    else if (evt.type == EVT_TIMEOUT) { state = ERROR; }
                    break;
                // ...
            }
        }
    }
}
```

This is the FreeRTOS analog of a top-level dispatch loop in any event-driven system. Easy to test (you can drive it by feeding events), easy to add states.

### 9.5 Anti-patterns

#### Anti-pattern: busy waiting

```c
// WRONG
while (!data_ready) {}                    // hogs CPU at task priority

// WRONG, slightly less so
while (!data_ready) { vTaskDelay(1); }    // wakes every tick to check; wasteful
```

```c
// RIGHT
xSemaphoreTake(semDataReady, portMAX_DELAY);   // block until the signal
```

#### Anti-pattern: `vTaskDelay` for synchronization

```c
// WRONG — assumes hardware will be ready in 50 ms
start_adc_conversion();
vTaskDelay(pdMS_TO_TICKS(50));      // will it be done? maybe?
read_adc_result();
```

```c
// RIGHT — signal driven
start_adc_conversion();              // ISR will set semDone on completion
xSemaphoreTake(semDone, pdMS_TO_TICKS(100));   // block until done OR timeout
read_adc_result();
```

#### Anti-pattern: shared data without protection

```c
// WRONG — two tasks updating shared_state non-atomically
// Task A:
shared_state.x = 1; shared_state.y = 2;
// Task B (preempts mid-update):
process(shared_state);   // sees torn state — x=1, y=stale
```

```c
// RIGHT — mutex (or, if it's small and platform-atomic, atomic ops)
xSemaphoreTake(stateMutex, portMAX_DELAY);
shared_state.x = 1; shared_state.y = 2;
xSemaphoreGive(stateMutex);
```

#### Anti-pattern: blocking APIs from ISRs

```c
// WRONG — calling task-context API from ISR
void EXTI0_IRQHandler(void) {
    xQueueSend(qButton, &evt, 0);   // undefined behavior in FromISR context
}
```

```c
// RIGHT
void EXTI0_IRQHandler(void) {
    BaseType_t xWoken = pdFALSE;
    xQueueSendFromISR(qButton, &evt, &xWoken);
    portYIELD_FROM_ISR(xWoken);
}
```

#### Anti-pattern: priority assignment by gut feel

The temptation: "this task is important, give it priority 9. This other one is also important, priority 8. This one too, priority 7." Suddenly there are 12 tasks at priorities 5–10 and you have no model for whether any of them meets its deadline.

The right approach is **rate-monotonic** priority assignment: tasks with shorter periods get higher priorities. A 1 ms task is higher priority than a 10 ms task is higher priority than a 100 ms task. Rate-monotonic is provably optimal for fixed-priority preemptive scheduling of independent periodic tasks (Liu & Layland, 1973), and assigns priorities by a rule, not opinion.

When you can't apply rate-monotonic literally (mixed periodic and event-driven, dependencies between tasks), use **deadline-monotonic** as a generalization: assign priority inversely to *deadline*. Closer deadline → higher priority.

The schedulability check: for *N* periodic tasks with periods *T<sub>i</sub>* and worst-case execution times *C<sub>i</sub>*, rate-monotonic guarantees all deadlines met if total CPU utilization ∑(C<sub>i</sub>/T<sub>i</sub>) ≤ N(2<sup>1/N</sup> − 1). For large N this is ~69%. Above that, you may still be schedulable but it requires response-time analysis.

Whether or not you do the math, the *priorities-by-period* rule is the right starting point and gives you something to defend in code review.

#### 9.5.1 What an interviewer is really testing

Whether your design instincts are sound — whether you reach for the right pattern when given a problem, and whether you can spot the wrong pattern in someone else's code.

#### 9.5.2 Likely questions

1. Design the task structure for a sensor node that reads three sensors at different rates and uploads over BLE.
2. How would you protect a shared SPI bus used by three drivers in three tasks?
3. I see a task doing `while (!flag) vTaskDelay(1)`. What's wrong, and how would you fix it?
4. Why is "the most important task gets the highest priority" wrong?
5. Walk me through implementing a UART driver that doesn't busy-wait.
6. How would you implement a watchdog that checks all tasks are alive?

#### 9.5.3 Model answers

(Section 11 has the full sensor-node design as a worked scenario. Briefer answers here:)

1. *"Three sensor tasks at priorities chosen by rate (rate-monotonic — fastest sensor highest priority). They sample on `vTaskDelayUntil` and push readings to a single queue. A processor task reads the queue, batches and runs whatever filtering, and pushes batched payloads to a transmit queue. A BLE task reads the transmit queue and handles the radio. BLE state changes drive an event group that the BLE task waits on. Everything decoupled by queues so a slow uplink doesn't stall sampling."*
2. *"Resource-manager task — one task owns the SPI bus and a queue of SPI requests. Drivers don't touch the bus; they post requests with their own per-request notification or callback. No lock, no inversion possible, naturally serialized."*
3. *"Two things wrong: it's polling instead of blocking — wakes the task every tick to check a flag — and it's racing on `flag` (not declared `volatile` or atomic). Replace with a binary semaphore or task notification: producer gives, this task takes with `portMAX_DELAY`. Zero CPU until signaled."*
4. *"'Important' is not a scheduling property. The right property is *deadline*: a task with a short deadline must preempt one with a long deadline, regardless of which is 'more important.' Rate-monotonic — shorter period → higher priority — is provably optimal and removes the subjectivity."*
5. *"Driver API blocks the calling task on a notification. Transmit: kick off DMA or interrupt-driven send, return immediately. The transmit-complete ISR notifies the waiting task. Receive: ISR puts incoming bytes into a stream buffer; API reads from the stream buffer. Caller blocks on the stream buffer or the notification — never on a hardware flag in a loop."*
6. *"Each task periodically sets its own bit in an event group (or increments a counter in a shared array under a mutex). A monitor task wakes every N seconds and checks that every expected bit is set or every counter has changed since last check. If any task has missed, log which one and reset (or reset the watchdog only if all tasks are alive). The hardware watchdog itself is kicked only by the monitor."*

#### 9.5.4 Traps and gotchas

- "Make it work, then refactor" — easy to ship before you refactor.
- Mutex around a long-running operation, when a resource-manager task would have been simpler.
- Priority assigned in `xTaskCreate` calls scattered across the codebase, with no central priority document.
- Signaling that goes "through" a global flag instead of a kernel primitive.

---

## 10. Rapid-Fire Round

Thirty short Q&A pairs covering the document. Read them the morning of an interview.

**1. Q:** What is a TCB?
**A:** Task Control Block — the kernel's per-task struct holding stack pointer, priority, state, name, and list links.

**2. Q:** Where does each task's stack live?
**A:** In RAM, separately per task — either heap-allocated (dynamic) or in a buffer you supplied (static).

**3. Q:** Default heap implementation to pick today?
**A:** heap_4. heap_5 if you have multiple non-contiguous RAM regions.

**4. Q:** `configMINIMAL_STACK_SIZE` units?
**A:** Stack words. On 32-bit, 1 word = 4 bytes.

**5. Q:** What does `pdMS_TO_TICKS(N)` evaluate to?
**A:** Roughly `N * configTICK_RATE_HZ / 1000` — the number of ticks corresponding to N milliseconds.

**6. Q:** Idle task's priority?
**A:** 0, the lowest.

**7. Q:** Can multiple tasks share the same priority?
**A:** Yes, and they round-robin on tick boundaries if `configUSE_TIME_SLICING = 1`.

**8. Q:** Difference between `vTaskDelay` and `vTaskDelayUntil`?
**A:** Delay is "from now"; DelayUntil wakes at an absolute tick. DelayUntil for periodic, Delay for one-shot.

**9. Q:** What does `vTaskSuspend(NULL)` do?
**A:** Suspends the calling task indefinitely until another task or ISR resumes it.

**10. Q:** Mutex vs binary semaphore — one-line difference?
**A:** Mutex tracks owner and provides priority inheritance; binary semaphore does neither.

**11. Q:** Can you give a mutex from an ISR?
**A:** No — never. Use a binary semaphore or task notification.

**12. Q:** Counting semaphore use case?
**A:** Resource pool of N items, or counting events (one give per event, multiple gives possible before take catches up).

**13. Q:** What does `xSemaphoreCreateBinary` initial state default to?
**A:** Empty. First take blocks. (Mutex defaults to available.)

**14. Q:** How do task notifications differ from semaphores?
**A:** Notification storage is in the TCB itself — no separate object, faster, lower RAM, but only one waiter (the owning task).

**15. Q:** Three things an event group can do that semaphores can't easily?
**A:** Wait for AND of multiple events, wait for OR of multiple events, sync N tasks at a barrier.

**16. Q:** Stream buffer constraint?
**A:** Single writer, single reader.

**17. Q:** Why must ISRs use FromISR API variants?
**A:** Non-FromISR APIs may take task-level critical sections and yield in ways illegal from interrupt context.

**18. Q:** `portYIELD_FROM_ISR(xWoken)` — what does the parameter do?
**A:** If `pdTRUE`, requests a context switch to the woken higher-priority task; if `pdFALSE`, no-op.

**19. Q:** Two interrupt-priority macros, what are they?
**A:** `configKERNEL_INTERRUPT_PRIORITY` (priority of SysTick/PendSV — lowest) and `configMAX_SYSCALL_INTERRUPT_PRIORITY` (threshold above which ISRs cannot use FreeRTOS API).

**20. Q:** What can an ISR running above `configMAX_SYSCALL_INTERRUPT_PRIORITY` call?
**A:** No FreeRTOS API at all, not even FromISR variants.

**21. Q:** Cortex-M priority numerical convention?
**A:** Lower number = higher priority (more urgent).

**22. Q:** What runs first after the kernel ticks: the tick handler or PendSV?
**A:** The tick handler runs (at higher exception priority), pends PendSV; PendSV does the actual switch when no higher-priority work remains.

**23. Q:** Priority inversion — one-line definition?
**A:** A high-priority task waiting on a resource held by a low-priority task that is itself preempted by a medium-priority task.

**24. Q:** How does priority inheritance fix it?
**A:** The kernel temporarily boosts the lock-holder's priority to that of the highest waiter, so medium tasks can no longer preempt.

**25. Q:** Does priority inheritance prevent deadlock?
**A:** No. Deadlock requires lock ordering discipline.

**26. Q:** Stack overflow detection method 1 vs 2?
**A:** 1: SP-range check at switch — cheap, misses transient overflows. 2: pattern check at switch — catches transient too.

**27. Q:** API to measure stack high-water mark?
**A:** `uxTaskGetStackHighWaterMark(handle)` — returns minimum free words ever observed.

**28. Q:** Where do software timer callbacks run?
**A:** In the timer service task — *not* in an ISR, *not* in the calling task.

**29. Q:** Can a software timer callback block?
**A:** No. It will hold up every other software timer.

**30. Q:** How would you debounce a button using a software timer?
**A:** GPIO ISR calls `xTimerResetFromISR` on a 50 ms one-shot timer; callback fires only when 50 ms passes without another edge, then samples and dispatches.

---

## 11. Scenario Questions

Eight open-ended design problems with worked solutions. These are what senior interviews look like.

### Scenario 1: Sensor node that reads three sensors at different rates and uploads over BLE

**Stated requirements**: A node samples temperature at 1 Hz, accelerometer at 100 Hz, microphone at 16 kHz. It batches data and uploads over BLE every 5 seconds. The MCU has 256 KB flash, 64 KB RAM, runs at 80 MHz.

**Design**:

```
                 ┌────────────────────┐
                 │    BLE Stack Task  │   priority 5
                 │   (vendor stack)   │
                 └─────────▲──────────┘
                           │ Tx queue (batched packets)
                           │
                 ┌─────────┴──────────┐
                 │  Uploader Task     │   priority 2
                 │ (batch + xmit)     │
                 └─────────▲──────────┘
                           │ samples queue (mixed)
              ┌────────────┼─────────────┐
              │            │             │
   ┌──────────┴──┐ ┌───────┴────┐ ┌──────┴──────┐
   │ Temp Task   │ │ Accel Task │ │ Audio Task  │
   │ priority 2  │ │ priority 4 │ │ priority 5  │
   │ 1 Hz        │ │ 100 Hz     │ │ DMA-driven  │
   └──────────┬──┘ └────────────┘ └─────────────┘
              │
   I2C bus owned by a Resource-Manager task
```

**Priority assignment** (rate-monotonic):

- Audio at 16 kHz is the fastest periodic — but we don't sample at 16 kHz from a task; we use I2S DMA. The audio task wakes per DMA-half-complete (every 256 samples = 16 ms). Priority 5.
- Accelerometer at 100 Hz → priority 4.
- Temperature at 1 Hz → priority 2.
- Uploader at 0.2 Hz → priority 2 (lower deadline-criticality).
- BLE stack task at priority 5 (vendor's recommendation usually).

**Flow**:

- Each sensor task on `vTaskDelayUntil` for its period (audio uses DMA-half-complete notification rather than a delay).
- Each pushes a `sample_t {sensor_id, ts, payload}` into a shared queue.
- Queue depth sized for 5 seconds of accumulated samples plus margin; overflow logs and drops.
- Uploader task wakes on a 5-second `vTaskDelayUntil`, drains the queue into a contiguous buffer, frames it, and posts to a transmit queue read by the BLE stack task.
- An event group holds BLE state — `BLE_CONNECTED`, `BLE_LINK_LOSS` — and the uploader checks it before transmitting; if not connected, it buffers (or drops, depending on policy).

**Memory plan**: heap_4 or heap_5 if RAM is split. Static allocation for kernel objects. Audio buffers allocated as a static pool (DMA-friendly alignment). High-water-mark monitoring on every task.

**Power**: tickless idle on. Audio DMA is the wake source between intervals.

### Scenario 2: Mysterious 50 ms latency added to UART responses, only when the network is busy

**Symptom**: UART command/response normally turns around in 5 ms. When the device is also handling network traffic, response time spikes to 50+ ms, and command responses occasionally come out garbled.

**Diagnosis path**:

1. Run the system with Tracealyzer or even just enable `vTaskGetRunTimeStats`. Look at what the UART task is doing in the bad case.
2. If the UART task is `Ready` for tens of ms before running, you have a priority-inversion or starvation issue. Check whether the UART task is taking a mutex held by a lower-priority task that's being preempted by the network task.
3. If the UART task is `Running` but slow, check whether it's busy-waiting on hardware or making a blocking HAL call.
4. The garble points to a *race*, not a latency issue alone — some shared state (output buffer, format buffer) is being interleaved.

**Likely root cause**: the UART driver and the network logger both call the same `printf` / `snprintf` with a shared buffer, protected by no lock. Under network load, the network task interrupts the UART task mid-format. Fix: per-task format buffers, plus a logger task with its own queue for any cross-task log output.

### Scenario 3: System resets randomly, never reproducibly. Watchdog isn't catching anything specific

**Diagnosis path**:

1. Are the resets watchdog resets, hard fault resets, or brown-outs? Read the reset cause register on every boot and log it. (This is essential infrastructure.)
2. If hard fault: install a real fault handler that captures CFSR/HFSR/MMFAR/BFAR and the stack frame to non-volatile memory before resetting. Read them after the next event.
3. If watchdog: instrument every task to set a bit in an event group every loop; watchdog kicker checks all bits set and which one was missing if not.
4. If brown-out: not a software problem — power supply or load issue.
5. Enable stack overflow detection method 2 unconditionally, and `configASSERT`. Use a meaningful assert hook that captures file and line.
6. Enable malloc-failed hook with logging.

**Most common findings**, in order of frequency: stack overflow on a rare error path; null pointer in a callback; unprotected shared variable causing a torn write that fails an assert later; ISR running too long and watchdog timing out a task that *was* fine.

### Scenario 4: Tasks running, but one specific task makes no progress under load

**Diagnosis**: priority inversion or starvation. Run `vTaskGetRunTimeStats` — does the task have ~0% CPU even though it's supposed to be doing work? Does its priority seem reasonable?

If it's blocked on a mutex held by a lower-priority task being preempted by a third, that's classic inversion. Switch from binary semaphore to mutex if you accidentally used the wrong primitive. Confirm `configUSE_MUTEXES = 1` and that you're using `xSemaphoreCreateMutex`, not `xSemaphoreCreateBinary`.

If it's just lower-priority than a busy task, you may need to lower the busy task's priority or split its work.

### Scenario 5: I2C driver hangs forever when the bus glitches

**Likely cause**: the driver was written with `HAL_I2C_Master_Transmit(..., portMAX_DELAY)` — busy-wait with no timeout. Bus glitch leaves the I2C peripheral in a state where the BUSY flag never clears.

**Fix**: rewrite the driver to be interrupt- or DMA-driven. Initiate the transfer, block on a notification with a *finite* timeout, recover on timeout (reset the I2C peripheral, return error). Never use `portMAX_DELAY` on hardware operations — there is no such thing as a hardware operation that *cannot* fail.

### Scenario 6: Need to add OTA firmware update to an existing system

**Considerations**:

- OTA is naturally a *long-running, low-priority* task. Make sure it can be preempted by everything important.
- Network reception during OTA can starve the rest of the system if not rate-limited. Use a counting semaphore between RX and the OTA writer to bound buffering.
- Writes to flash typically *block* (and disable interrupts on some MCUs during a write). Plan for the worst-case write time and ensure no real-time deadline overlaps.
- Power loss during OTA must leave a working firmware. Implement A/B partitions and a verified-write protocol.
- Static allocation for OTA buffers — you don't want to discover you need 32 KB after you've already loaded 200 KB of partial firmware.

### Scenario 7: Migrating a 50,000-line super-loop firmware to FreeRTOS

**Approach**:

1. Don't rewrite — *wrap*. Identify natural concurrency boundaries (each peripheral, each protocol stack, each control loop) and turn each into a task at first.
2. Replace busy-wait drivers first; that's where you'll get the biggest immediate win and the most pain if you skip.
3. Identify shared state. Wrap with mutexes (or refactor to message-passing). Don't try to clean up everything at once.
4. Move things gradually: start with one task running the old super-loop verbatim, plus a few side tasks for new work. Then peel off pieces of the loop into their own tasks.
5. Watch out for global flags used as poor man's signaling — convert each one to a kernel primitive deliberately.

The mistake to avoid: declaring 30 tasks on day one and trying to make them all work together. You'll lose weeks chasing race conditions.

### Scenario 8: System works in development, fails in the field after running for 6 days

**Diagnosis**:

- 6 days suggests a counter or timer rolling over. `TickType_t` is 32 bits on most ports. At 1 ms tick, it rolls every 49.7 days — not 6, but worth checking. At 10 kHz tick, it rolls every 4.97 days — close to 6.
- Memory leak: heap usage growing slowly. Log free-heap (`xPortGetFreeHeapSize`) periodically and watch.
- Stack creeping: a rare path is consuming stack and never exiting. Method 2 detection plus high-water-mark logging.
- Hardware: a counter in a peripheral overflowing, a sensor accumulating drift, an SD card filling up.

The framework: anything that grows monotonically over time is a candidate. Add monotonic-resource logging.

---

## 12. One-Page Cheat Sheet

The absolute must-remembers, on a single screen.

### Critical config in `FreeRTOSConfig.h`

```
configUSE_PREEMPTION                    1
configCPU_CLOCK_HZ                      <your CPU Hz>
configTICK_RATE_HZ                      1000
configMAX_PRIORITIES                    5..32
configMINIMAL_STACK_SIZE                128       // WORDS, not bytes
configTOTAL_HEAP_SIZE                   <bytes>
configMAX_SYSCALL_INTERRUPT_PRIORITY    (5 << (8-PRIO_BITS))   // PRE-SHIFTED
configKERNEL_INTERRUPT_PRIORITY         (15 << (8-PRIO_BITS))  // LOWEST priority
configCHECK_FOR_STACK_OVERFLOW          2
configUSE_MUTEXES                       1
configSUPPORT_STATIC_ALLOCATION         1
configGENERATE_RUN_TIME_STATS           1   // in development
```

### Task API

```
xTaskCreate(fn, name, stackWords, param, prio, &handle)   // dynamic
xTaskCreateStatic(fn, name, stackWords, param, prio, stackBuf, tcbBuf)
vTaskDelay(ticks)                       // relative; use for one-shot
vTaskDelayUntil(&lastWake, period)      // absolute; use for periodic
vTaskDelete(handle)                     // freeing happens in idle task
uxTaskGetStackHighWaterMark(handle)     // measure stack usage
xTaskGetTickCount()
```

### Queues, semaphores, notifications

```
xQueueCreate(n, item_size)
xQueueSend(q, &item, ticks_to_wait)              xQueueSendFromISR(...)
xQueueReceive(q, &item, ticks_to_wait)           xQueueReceiveFromISR(...)

xSemaphoreCreateBinary()                         // starts EMPTY
xSemaphoreCreateCounting(max, initial)
xSemaphoreCreateMutex()                          // starts AVAILABLE, has PI
xSemaphoreCreateRecursiveMutex()
xSemaphoreTake(s, ticks)                         xSemaphoreTakeFromISR(...)
xSemaphoreGive(s)                                xSemaphoreGiveFromISR(...)
                                                 // NEVER xSemaphoreGive on a mutex from ISR

vTaskNotifyGive(handle)                          vTaskNotifyGiveFromISR(...)
ulTaskNotifyTake(clearOnExit, ticks)             // returns count
xTaskNotify(handle, value, action)               xTaskNotifyFromISR(...)
xTaskNotifyWait(clearOnEntry, clearOnExit, &val, ticks)
```

### Event groups, stream buffers, timers

```
xEventGroupCreate()
xEventGroupSetBits(g, bits)                      xEventGroupSetBitsFromISR(...)
xEventGroupWaitBits(g, bits, clearOnExit, waitForAll, ticks)

xStreamBufferCreate(size, trigger_level)
xStreamBufferSend(b, data, len, ticks)           xStreamBufferSendFromISR(...)
xStreamBufferReceive(b, data, max_len, ticks)

xTimerCreate("name", period, autoReload, id, callback)
xTimerStart(t, ticks_to_wait)                    xTimerStartFromISR(...)
xTimerReset(t, ticks_to_wait)                    xTimerResetFromISR(...)
```

### ISR pattern (memorize)

```c
void XXX_IRQHandler(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    /* clear hardware flag */
    /* short, non-blocking: capture data, signal a task */
    xQueueSendFromISR(q, &data, &xHigherPriorityTaskWoken);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

### Rules of thumb

- Stack size in **words**.
- `configMAX_SYSCALL_INTERRUPT_PRIORITY` is **pre-shifted**.
- Mutex from ISR: **never**.
- ISR work: **micro**seconds, not milliseconds. Defer the rest.
- Software timer callback: **never block**.
- `vTaskDelay` for one-shot, `vTaskDelayUntil` for periodic.
- Task notification > semaphore for one-to-one signaling.
- Mutex (not binary semaphore) for shared resources — for the priority inheritance.
- Critical sections: **short**, **non-blocking**, never around `printf`.
- Stack overflow detection method **2**, always in development.
- Static allocation for kernel objects in safety-critical.
- Priority by **period** (rate-monotonic), not by feeling.
- Lock ordering rule, globally consistent, to avoid deadlock.

### When to use what — at a glance

| Need | Use |
|---|---|
| ISR → task signal (one waiter) | Task notification |
| ISR → task signal (multi waiter) | Binary semaphore |
| ISR → task with byte stream | Stream buffer |
| Producer → consumer with data | Queue |
| Multiple events, wait for any/all | Event group |
| Protect shared resource | Mutex (not binary semaphore) |
| Resource pool of N items | Counting semaphore |
| Periodic tick at ms resolution | Software timer |
| Periodic tick at sub-ms resolution | Hardware timer |
| Many tasks needing one peripheral | Resource-manager task |
