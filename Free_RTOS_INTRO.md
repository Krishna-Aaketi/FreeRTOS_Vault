# 🚀 What is FreeRTOS?

**FreeRTOS** is a **real-time operating system (RTOS)** designed for **microcontrollers and embedded systems**.

👉 Key idea:

> It allows multiple tasks to run *“concurrently”* with precise timing control.

---

# 🧠 Why FreeRTOS is Needed?

Without RTOS:

* You write a **super loop (while(1))**
* Tasks run sequentially → slow + messy

With FreeRTOS:

* System runs **multiple tasks**
* Each task has **priority**
* Scheduler decides execution

---

# ⚙️ FreeRTOS Architecture (Core Components)

## 1. Scheduler (Brain of RTOS)

* Decides which task runs
* Based on **priority + state**

Types:

* Preemptive (most used)
* Cooperative

---

## 2. Tasks (Like Threads)

Each task:

* Has its own stack
* Runs independently

Example:

```c
void Task1(void *params) {
    while(1) {
        printf("Task1 running\n");
    }
}
```

---

## 3. Task States

```
Running → Ready → Blocked → Suspended
```

* **Running** → currently executing
* **Ready** → waiting for CPU
* **Blocked** → waiting (delay / event)
* **Suspended** → manually stopped

---

## 4. Context Switching (VERY IMPORTANT)

👉 OS saves current task state and switches to another

* Saves registers
* Loads next task context

💡 Happens:

* Timer interrupt
* Higher priority task arrives

---

## 5. Tick Interrupt

* Periodic timer (e.g., every 1 ms)
* Drives scheduler

---

# 🔄 Task Scheduling Concept

Example:

| Task   | Priority |
| ------ | -------- |
| Task A | 3        |
| Task B | 2        |

👉 Task A always runs first (if ready)

---

# 🔗 Inter-Task Communication (IPC)

## 1. Queues

* Send data between tasks

```c
xQueueSend(queue, &data, portMAX_DELAY);
xQueueReceive(queue, &data, portMAX_DELAY);
```

---

## 2. Semaphores

Used for:

* Synchronization
* Resource protection

Types:

* Binary semaphore
* Counting semaphore

---

## 3. Mutex

* Prevent race conditions
* Has **priority inheritance**

---

## 4. Event Groups

* Multiple signals combined

---

# ⏱️ Timing APIs

```c
vTaskDelay(pdMS_TO_TICKS(1000));
```

* Task sleeps for 1 second

---

# 🧩 Memory Management

FreeRTOS provides:

* heap_1 → simple
* heap_2 → best-fit
* heap_4 → most used (coalescing)
* heap_5 → multiple regions

---

# 🛠️ Basic FreeRTOS Flow (VERY IMPORTANT)

```c
int main() {

    xTaskCreate(Task1, "Task1", 1000, NULL, 1, NULL);
    xTaskCreate(Task2, "Task2", 1000, NULL, 2, NULL);

    vTaskStartScheduler();

    while(1);
}
```

---

# 🔥 Real Embedded Example

Imagine:

| Task               | Work               |
| ------------------ | ------------------ |
| Sensor Task        | Read temperature   |
| Communication Task | Send data via UART |
| Control Task       | Adjust fan         |

👉 FreeRTOS runs all efficiently.

---

# 🧠 Interview-Level Points (VERY IMPORTANT)

### Q: Difference RTOS vs Linux?

* RTOS → deterministic timing
* Linux → not real-time (unless patched)

---

### Q: What is deterministic?

👉 Same operation takes **predictable time**

---

### Q: What is priority inversion?

👉 Low priority task blocks high priority task

Solution:

* **Mutex with priority inheritance**

---

### Q: What is context switch overhead?

👉 Time taken to switch tasks

---

# 🧪 Where FreeRTOS is Used

* IoT devices
* Automotive ECUs
* Qualcomm firmware layers
* Wearables
* Sensors systems

---

# 🧭 How to Learn FreeRTOS (STEP-BY-STEP PLAN)

### Step 1: Basics

* Tasks
* Scheduler
* Delay

### Step 2: IPC

* Queues
* Semaphores
* Mutex

### Step 3: Advanced

* Interrupt handling
* Event groups
* Timers

### Step 4: System Design

* Multi-task projects

---

# 💻 Practice Project (Start THIS)

👉 Build:

* LED blink task
* Button interrupt task
* UART print task

---

# ⚡ Next Level (Important for YOU)

Since you're targeting **Qualcomm / Embedded Linux roles**, next you should learn:

* FreeRTOS + Drivers (UART, SPI, I2C)
* Interrupt handling in RTOS
* RTOS vs Linux kernel comparison
* Real-time scheduling

---
