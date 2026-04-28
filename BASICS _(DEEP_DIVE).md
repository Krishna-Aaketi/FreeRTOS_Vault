# 🚀 STEP 1: BASICS (DEEP DIVE)

We focus on 3 pillars:

👉 **Tasks + Scheduler + Delay**

---

# 🧠 1. TASK (Core Execution Unit)

## 🔹 What is a Task?

👉 A **task is an independent execution unit** (like a thread)

* Runs its own function
* Has its own stack
* Managed by scheduler

---

## 🔹 Internal Structure of a Task

Each task has:

* Stack (local variables, function calls)
* Task Control Block (**TCB**)

---

## 🔥 What is TCB (VERY IMPORTANT)

👉 Kernel maintains this structure for each task

Contains:

* Task state (Running / Ready / Blocked)
* Stack pointer
* Priority
* Task name

---

## 🔹 Task Function Rule (IMPORTANT)

```c
void Task(void *pvParameters)
{
    while(1) {
        // Your logic
    }
}
```

👉 Why infinite loop?

* RTOS expects task to **never exit**
* If it exits → undefined behavior

---

## 🔹 Task Creation

```c
xTaskCreate(
    TaskFunction,     // Task code
    "TaskName",       // Debug name
    1024,             // Stack size (words!)
    NULL,             // Parameters
    2,                // Priority
    NULL              // Handle
);
```

---

## ⚠️ Important Points (Interview Level)

* Stack size is **in words**, not bytes
* Wrong stack → **stack overflow → crash**
* Too many tasks → memory exhaustion

---

## 🔹 Task States (VERY IMPORTANT)

```text
Running → Ready → Blocked → Suspended
```

### Explanation:

* **Running** → currently executing
* **Ready** → waiting for CPU
* **Blocked** → waiting (delay / queue / semaphore)
* **Suspended** → manually paused

---

## 🔄 Task State Transition Example

```text
Task runs → calls vTaskDelay → goes to Blocked
Time completes → moves to Ready
Scheduler → Running
```

---

# ⚙️ 2. SCHEDULER (THE BRAIN)

## 🔹 What is Scheduler?

👉 Decides **which task runs at any moment**

---

## 🔥 Types of Scheduling

### 1. Preemptive (DEFAULT)

👉 Higher priority task can **interrupt** lower priority

Example:

* Task A (priority 3)
* Task B (priority 1)

👉 If A becomes ready → B is stopped immediately

---

### 2. Cooperative

👉 Tasks must **voluntarily yield**

(Not commonly used)

---

## 🔹 Scheduling Algorithm

👉 FreeRTOS uses:

* Priority-based scheduling
* Round-robin within same priority

---

## 🔹 Round Robin Example

If 2 tasks same priority:

```text
Task1 → Task2 → Task1 → Task2
```

---

## 🔹 When Scheduler Runs?

Triggered by:

* Tick interrupt
* Task yield (`taskYIELD()`)
* Blocking call (delay, queue, semaphore)

---

## 🔥 Context Switching (VERY IMPORTANT)

👉 Switching CPU from one task to another

### What happens internally:

1. Save current task registers (PC, SP)
2. Save stack pointer
3. Load next task stack
4. Restore registers

---

## ⚠️ Interview Question

👉 What is context switch overhead?

* Time spent switching tasks
* Too many switches → performance drop

---

# ⏱️ 3. DELAY (CRITICAL CONCEPT)

## 🔹 Why Delay is Needed?

👉 Prevents CPU hogging

Without delay:

```c
while(1) {
    // infinite loop
}
```

👉 This blocks all other tasks ❌

---

## 🔹 vTaskDelay()

```c
vTaskDelay(pdMS_TO_TICKS(1000));
```

👉 Puts task into **Blocked state**

---

## 🔹 What happens internally?

1. Task moves → Blocked state
2. Scheduler picks next task
3. After time expires → task moves → Ready

---

## 🔹 pdMS_TO_TICKS()

👉 Converts milliseconds → RTOS ticks

Example:

* Tick rate = 1000 Hz → 1 tick = 1 ms

---

## 🔥 vTaskDelay vs vTaskDelayUntil (IMPORTANT)

### vTaskDelay()

* Relative delay

### vTaskDelayUntil()

* Absolute periodic delay (real-time accurate)

```c
TickType_t lastWakeTime = xTaskGetTickCount();

vTaskDelayUntil(&lastWakeTime, pdMS_TO_TICKS(1000));
```

👉 Used in:

* Sensor sampling
* Real-time control systems

---

# 🧪 FULL WORKING EXAMPLE (UNDERSTAND FLOW)

```c
void Task1(void *pvParameters)
{
    while(1) {
        printf("Task 1\n");
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

void Task2(void *pvParameters)
{
    while(1) {
        printf("Task 2\n");
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

int main()
{
    xTaskCreate(Task1, "T1", 1000, NULL, 1, NULL);
    xTaskCreate(Task2, "T2", 1000, NULL, 2, NULL);

    vTaskStartScheduler();

    while(1);
}
```

---

# 🧠 EXPECTED OUTPUT BEHAVIOR

* Task2 runs more frequently (500 ms)
* Task2 has higher priority → executes first

---

# ⚠️ CRITICAL MISTAKES (VERY IMPORTANT)

## ❌ 1. No Delay

👉 Causes CPU starvation

---

## ❌ 2. Same Priority + No Blocking

👉 Round-robin won't happen → one task dominates

---

## ❌ 3. Stack Overflow

👉 Leads to hard fault

---

## ❌ 4. Long Blocking Operations

👉 Breaks real-time behavior

---

# 🔥 INTERVIEW QUESTIONS (YOU MUST KNOW)

### Q1: What is a task?

👉 Independent execution unit with its own stack

---

### Q2: What is scheduler?

👉 Decides which task runs based on priority

---

### Q3: What is preemption?

👉 Higher priority task interrupts lower priority

---

### Q4: What is blocking?

👉 Task waits for event/time → moved to Blocked state

---

### Q5: Difference: Ready vs Running?

* Ready → waiting
* Running → executing

---

# 🎯 SUMMARY (IMPORTANT)

👉 Task = execution unit
👉 Scheduler = decision maker
👉 Delay = CPU sharing mechanism

---
