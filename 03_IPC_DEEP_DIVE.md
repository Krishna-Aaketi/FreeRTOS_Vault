# 🚀 STEP 2: IPC DEEP DIVE

👉 IPC = **Inter-Process (Task) Communication**

Why needed?

* Tasks run independently
* Need to **share data + coordinate safely**

---

# 📦 1. QUEUES (MOST IMPORTANT IPC)

## 🧠 Concept

👉 Queue = **FIFO buffer managed by kernel**

* One task → sends data
* Another task → receives data
* Thread-safe (no corruption)

---

## 🔧 Queue Internals (IMPORTANT)

Queue stores:

* Fixed-size elements
* Circular buffer
* Managed by kernel (no race condition)

---

## 🔹 Create Queue

```c
QueueHandle_t q;
q = xQueueCreate(5, sizeof(int));
```

👉 Meaning:

* Max 5 elements
* Each element = int

---

## 🔹 Send Data

```c
int data = 100;
xQueueSend(q, &data, portMAX_DELAY);
```

👉 If queue full:

* Task goes to **Blocked state**

---

## 🔹 Receive Data

```c
int received;
xQueueReceive(q, &received, portMAX_DELAY);
```

👉 If queue empty:

* Task blocks

---

## 🔄 Flow Visualization

```text
Sensor Task → Queue → Processing Task → UART Task
```

---

## 🔥 Important Behavior

| Condition   | Result          |
| ----------- | --------------- |
| Queue full  | Sender blocks   |
| Queue empty | Receiver blocks |

---

## ⚠️ Interview Traps

### Q: Why queue instead of global variable?

👉 Prevents:

* Race conditions
* Data corruption

---

### Q: Queue vs Buffer?

👉 Queue = **synchronized + RTOS managed**

---

# 🔐 2. SEMAPHORES (SIGNALING MECHANISM)

## 🧠 Concept

👉 Semaphore = **signal/event notification**

No data transfer—only **"something happened"**

---

## 🔹 Types

### 1. Binary Semaphore

* 0 or 1
* Used for signaling

### 2. Counting Semaphore

* Multiple events tracking

---

## 🔧 Create Semaphore

```c
SemaphoreHandle_t sem;
sem = xSemaphoreCreateBinary();
```

---

## 🔹 Give (Signal)

```c
xSemaphoreGive(sem);
```

---

## 🔹 Take (Wait)

```c
xSemaphoreTake(sem, portMAX_DELAY);
```

---

## 🔥 ISR Usage (VERY IMPORTANT)

```c
void ISR_Handler() {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    xSemaphoreGiveFromISR(sem, &xHigherPriorityTaskWoken);

    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

---

## 🧠 Flow

```text
Interrupt → Give semaphore → Task wakes → handles work
```

---

## ⚠️ Interview Questions

### Q: Why not process inside ISR?

👉 ISR must be:

* Fast
* Non-blocking

---

### Q: Semaphore vs Queue?

* Semaphore → signal
* Queue → data transfer

---

# 🔒 3. MUTEX (CRITICAL FOR REAL SYSTEMS)

## 🧠 Concept

👉 Mutex = **protect shared resource**

Example:

* UART
* Shared memory
* I2C bus

---

## 🔧 Create Mutex

```c
SemaphoreHandle_t mutex;
mutex = xSemaphoreCreateMutex();
```

---

## 🔹 Lock

```c
xSemaphoreTake(mutex, portMAX_DELAY);
```

---

## 🔹 Unlock

```c
xSemaphoreGive(mutex);
```

---

## 🔥 Why Mutex is Special?

👉 Supports **Priority Inheritance**

---

## ⚠️ PRIORITY INVERSION (VERY IMPORTANT)

### Scenario:

* Low priority task holds mutex
* High priority task waiting
* Medium priority task runs

👉 High priority task gets blocked ❌

---

## ✅ Solution

👉 Mutex enables:
**Priority Inheritance**

* Low priority task temporarily becomes high priority

---

# 🔄 COMPARISON (VERY IMPORTANT)

| Feature              | Queue | Semaphore | Mutex |
| -------------------- | ----- | --------- | ----- |
| Data transfer        | Yes   | No        | No    |
| Synchronization      | Yes   | Yes       | Yes   |
| Priority inheritance | No    | No        | Yes   |
| ISR safe             | Yes   | Yes       | No    |

---

# 🧪 REAL SYSTEM EXAMPLE

## 🎯 Design:

### Task 1: Sensor Task

* Reads temperature
* Sends via queue

### Task 2: Processing Task

* Receives data
* Uses mutex for UART

### ISR:

* Button press → semaphore

---

## 🔄 Flow

```text
Sensor → Queue → Processing → Mutex → UART
Button ISR → Semaphore → Processing Task
```

---

# 💻 MINI PROJECT CODE FLOW

## Queue + Semaphore + Mutex Combined

```c
QueueHandle_t q;
SemaphoreHandle_t sem;
SemaphoreHandle_t mutex;
```

---

## Sensor Task

```c
void SensorTask(void *p) {
    int temp = 25;

    while(1) {
        xQueueSend(q, &temp, portMAX_DELAY);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

---

## Processing Task

```c
void ProcessingTask(void *p) {
    int data;

    while(1) {

        xQueueReceive(q, &data, portMAX_DELAY);

        xSemaphoreTake(mutex, portMAX_DELAY);
        printf("Temp: %d\n", data);
        xSemaphoreGive(mutex);
    }
}
```

---

## Button ISR

```c
void Button_ISR() {
    xSemaphoreGiveFromISR(sem, NULL);
}
```

---

## Event Task

```c
void EventTask(void *p) {
    while(1) {
        xSemaphoreTake(sem, portMAX_DELAY);
        printf("Button Pressed\n");
    }
}
```

---

# ⚠️ CRITICAL MISTAKES

❌ Using mutex in ISR
❌ Not releasing mutex → deadlock
❌ Using queue for signaling only → waste

---

# 🔥 INTERVIEW QUESTIONS (HIGH VALUE)

### Q1: Queue vs Semaphore?

👉 Data vs Signal

---

### Q2: What is priority inversion?

👉 Low priority blocking high priority

---

### Q3: Why mutex has priority inheritance?

👉 Avoid inversion

---

### Q4: Can semaphore be used as mutex?

👉 Technically yes, but **no priority inheritance → unsafe**

---

### Q5: What happens if queue is full?

👉 Task blocks (or fails if timeout used)

---

# 🎯 SUMMARY

👉 Queue → data transfer
👉 Semaphore → signaling
👉 Mutex → resource protection

---
