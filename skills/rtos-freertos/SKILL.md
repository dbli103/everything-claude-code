---
name: rtos-freertos
description: FreeRTOS 实时操作系统开发指南。涵盖任务创建、队列、信号量、互斥锁、事件组、软定时器等。
origin: ECC
---

# FreeRTOS 实时操作系统开发指南

全面的 FreeRTOS 开发指南，涵盖任务管理、任务间通信、同步机制等。

## 何时使用

- 开发基于 FreeRTOS 的多任务嵌入式应用
- 需要实时响应的系统
- 资源受限的嵌入式系统

### 何时不使用

- 极简裸机系统（无 RTOS 需求）
- 8 位 MCU（资源极度受限）
- 确定性要求极高的硬实时系统

## 核心概念

1. **任务** — 独立执行单元，有自己的栈
2. **调度器** — 决定哪个任务运行
3. **优先级** — 高优先级任务优先执行
4. **时间片** — 同优先级任务轮转

## 任务管理

### 创建任务

```c
// 任务函数
void vTaskFunction(void *pvParameters) {
    // 任务初始化
    while (1) {
        // 任务逻辑

        // 延时（让出 CPU）
        vTaskDelay(pdMS_TO_TICKS(100));
    }

    // 永远不会执行到这里
    vTaskDelete(NULL);
}

// 创建任务（方法1：静态栈）
StaticTask_t xTaskBuffer;
StackType_t xStack[configMINIMAL_STACK_SIZE * 2];

xTaskCreateStatic(
    vTaskFunction,       // 任务函数
    "TaskName",          // 任务名称
    configMINIMAL_STACK_SIZE * 2,  // 栈大小（Word）
    NULL,                // 参数
    1,                   // 优先级
    xStack,              // 栈缓冲区
    &xTaskBuffer         // 任务控制块
);

// 创建任务（方法2：动态栈，推荐）
xTaskCreate(
    vTaskFunction,
    "TaskName",
    configMINIMAL_STACK_SIZE * 2,
    NULL,
    1,
    NULL  // 任务句柄
);
```

### 任务优先级

```c
// configMAX_PRIORITIES 在 FreeRTOSConfig.h 中定义
// 优先级 0 最低，configMAX_PRIORITIES-1 最高

// 获得当前任务优先级
UBaseType_t priority = uxTaskPriorityGet(NULL);

// 设置任务优先级
vTaskPrioritySet(NULL, 2);

// 获得其他任务优先级
UBaseType_t priority = uxTaskPriorityGet(xTaskHandle);
```

### 任务延时

```c
// 相对延时（推荐）
vTaskDelay(pdMS_TO_TICKS(100));  // 延时 100ms

// 绝对延时（用于精确周期）
void vTaskFunction(void *pvParameters) {
    TickType_t xLastWakeTime = xTaskGetTickCount();
    const TickType_t xPeriod = pdMS_TO_TICKS(100);

    while (1) {
        // 执行任务

        vTaskDelayUntil(&xLastWakeTime, xPeriod);  // 精确延时 100ms
    }
}
```

### 任务删除

```c
// 删除自身
vTaskDelete(NULL);

// 删除其他任务
void otherTask(void *pvParameters) {
    TaskHandle_t xTaskToDelete = (TaskHandle_t)pvParameters;

    // ... 做一些操作

    vTaskDelete(xTaskToDelete);
}
```

## 队列（Queue）

### 队列创建

```c
// 创建队列
QueueHandle_t xQueue;
xQueue = xQueueCreate(10, sizeof(int));  // 10 个 int

if (xQueue == NULL) {
    // 队列创建失败
}

// 发送数据
int value = 42;
if (xQueueSend(xQueue, &value, pdMS_TO_TICKS(100)) != pdTRUE) {
    // 发送失败（超时）
}

// 接收数据
int received_value;
if (xQueueReceive(xQueue, &received_value, pdMS_TO_TICKS(100)) == pdTRUE) {
    // 收到数据
}
```

### 队列阻塞

```c
// 发送：队列满时阻塞
xQueueSend(xQueue, &value, portMAX_DELAY);  // 无限等待

// 接收：队列空时阻塞
xQueueReceive(xQueue, &value, portMAX_DELAY);  // 无限等待
```

### 多值队列

```c
typedef struct {
    uint8_t type;
    uint32_t data;
} Message_t;

xQueue = xQueueCreate(10, sizeof(Message_t));

Message_t msg = { .type = 1, .data = 100 };
xQueueSend(xQueue, &msg, 0);
```

## 信号量（Semaphore）

### 二值信号量

```c
// 创建二值信号量（用作互斥锁或同步）
SemaphoreHandle_t xBinarySemaphore = xSemaphoreCreateBinary();

// 释放信号量（在中断中常用）
BaseType_t xHigherPriorityTaskWoken = pdFALSE;
xSemaphoreGiveFromISR(xBinarySemaphore, &xHigherPriorityTaskWoken);
portYIELD_FROM_ISR(xHigherPriorityTaskWoken);

// 获取信号量
if (xSemaphoreTake(xBinarySemaphore, pdMS_TO_TICKS(1000)) == pdTRUE) {
    // 获取成功，执行临界代码
    xSemaphoreGive(xBinarySemaphore);  // 释放
}
```

### 计数信号量

```c
// 创建计数信号量（资源计数）
SemaphoreHandle_t xCountingSemaphore = xSemaphoreCreateCounting(5, 5);
// 最大计数值 5，初始计数值 5
```

## 互斥锁（Mutex）

### 互斥锁创建

```c
// 创建互斥锁
SemaphoreHandle_t xMutex = xSemaphoreCreateMutex();

// 使用（注意：获取和释放必须配对）
if (xSemaphoreTake(xMutex, pdMS_TO_TICKS(1000)) == pdTRUE) {
    // 临界区代码
    xSemaphoreGive(xMutex);  // 必须在同一任务中释放
}
```

### 优先级继承

```c
// 互斥锁支持优先级继承，防止优先级反转
// 自动处理，无需额外配置
```

## 事件组（Event Group）

### 事件组创建

```c
// 创建事件组
EventGroupHandle_t xEventGroup = xEventGroupCreate();

// 设置位
#define BIT_0 (1 << 0)
#define BIT_1 (1 << 1)
#define BIT_2 (1 << 2)

xEventGroupSetBits(xEventGroup, BIT_0 | BIT_1);

// 等待位
EventBits_t xBits = xEventGroupWaitBits(
    xEventGroup,           // 事件组句柄
    BIT_0 | BIT_1,         // 等待的位
    pdTRUE,                // 清除已设置的位
    pdTRUE,                // 等待所有位（AND）
    pdMS_TO_TICKS(5000)    // 超时
);

if (xBits & (BIT_0 | BIT_1)) {
    // 两个位都已设置
}
```

## 软件定时器

### 定时器创建

```c
// 定时器回调函数
void vTimerCallback(TimerHandle_t xTimer) {
    // 定时器到期处理
    uint32_t id = (uint32_t)pvTimerGetTimerID(xTimer);
}

// 创建定时器
TimerHandle_t xTimer = xTimerCreate(
    "Timer",                    // 名称
    pdMS_TO_TICKS(1000),        // 周期
    pdTRUE,                     // 自动重载
    (void *)0,                  // 定时器 ID
    vTimerCallback              // 回调函数
);

// 启动定时器
if (xTimer != NULL) {
    xTimerStart(xTimer, pdMS_TO_TICKS(0));
}

// 停止定时器
xTimerStop(xTimer, pdMS_TO_TICKS(0));

// 更改周期
xTimerChangePeriod(xTimer, pdMS_TO_TICKS(2000), pdMS_TO_TICKS(0));
```

## 任务通知（Task Notifications）

### 直接通知

```c
// 发送通知到任务
uint32_t ulValue = 100;
xTaskNotify(xTaskHandle, ulValue, eSetValueWithOverwrite);

// 在任务中接收通知
void vTaskFunction(void *pvParameters) {
    while (1) {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);  // 等待通知

        // 处理通知
    }
}
```

## 中断与 RTOS

### 中断-safe API

```c
// 在中断中使用 FromISR 函数
void vAnISR(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    // 发送数据到队列
    xQueueSendFromISR(xQueue, &data, &xHigherPriorityTaskWoken);

    // 给出信号量
    xSemaphoreGiveFromISR(xBinarySemaphore, &xHigherPriorityTaskWoken);

    // 强制上下文切换（如果需要）
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

### 延迟中断处理

```c
// 使用队列延迟中断处理到任务
void vAnISR(void) {
    // 快速读取硬件数据
    uint32_t data = hardware_read();

    // 发送到队列（不阻塞）
    xQueueSendFromISR(xQueue, &data, NULL);
}

// 任务处理中断
void vISRTask(void *pvParameters) {
    uint32_t data;
    while (1) {
        if (xQueueReceive(xQueue, &data, portMAX_DELAY) == pdTRUE) {
            // 处理数据
        }
    }
}
```

## 内存管理

### 堆内存

```c
// FreeRTOS 堆配置（FreeRTOSConfig.h）
#define configUSE_MALLOC_FAILED_HOOK 1
#define configTOTAL_HEAP_SIZE ((size_t)(20 * 1024))  // 20KB

// 内存分配失败钩子
void vApplicationMallocFailedHook(void) {
    while (1);  // 挂起
}

// 使用 pvPortMalloc / vPortFree
uint8_t *buffer = pvPortMalloc(256);
if (buffer != NULL) {
    // 使用
    vPortFree(buffer);
}
```

## 配置优化

### FreeRTOSConfig.h 关键配置

```c
#define configUSE_PREEMPTION        1           // 抢占式调度
#define configUSE_TIME_SLICING      1           // 时间片轮转
#define configCPU_CLOCK_HZ          (168000000UL)
#define configTICK_RATE_HZ          (1000)      // 1ms tick
#define configMAX_PRIORITIES        5
#define configMINIMAL_STACK_SIZE    (128)
#define configTOTAL_HEAP_SIZE       ((size_t)(20*1024))
#define configUSE_MUTEXES          1
#define configUSE_COUNTING_SEMAPHORES 1
#define configUSE_QUEUE_SETS        1
#define configUSE_TASK_NOTIFICATIONS 1
#define configUSE_IDLE_HOOK         1
#define configUSE_TICK_HOOK         1
#define configUSE_MALLOC_FAILED_HOOK 1
#define INCLUDE_vTaskDelete         1
#define INCLUDE_vTaskSuspend        1
#define INCLUDE_xTaskGetCurrentTaskHandle 1
```

## 调试技巧

### 运行时信息

```c
// 获取任务信息
TaskStatus_t xTaskDetails;
vTaskGetInfo(NULL, &xTaskDetails, pdTRUE, eRunning);

// 列出所有任务
void vListTasks(void) {
    TaskStatus_t *pxTaskStatusArray;
    volatile UBaseType_t uxArraySize, x;
    uint32_t ulTotalRunTime;

    uxArraySize = uxTaskGetNumberOfTasks();
    pxTaskStatusArray = pvPortMalloc(uxArraySize * sizeof(TaskStatus_t));

    if (pxTaskStatusArray != NULL) {
        uxArraySize = uxTaskGetSystemState(pxTaskStatusArray, uxArraySize, &ulTotalRunTime);

        for (x = 0; x < uxArraySize; x++) {
            // 打印任务信息
        }

        vPortFree(pxTaskStatusArray);
    }
}
```

### 栈溢出检测

```c
// 启用栈溢出检测
#define configCHECK_FOR_STACK_OVERFLOW 1

// 钩子函数
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName) {
    while (1);  // 挂起
}
```

## 快速参考清单

- [ ] 任务优先级设置合理
- [ ] 所有获取的信号量/互斥锁都有释放
- [ ] 中断中使用 FromISR 函数
- [ ] 使用 portMAX_DELAY 时注意超时
- [ ] 栈大小足够（避免溢出）
- [ ] 队列大小合理（避免丢失数据）
- [ ] 启用栈溢出检测（开发时）

## 相关资源

- FreeRTOS 官方文档：https://www.freertos.org/
- FreeRTOS API 参考：https://www.freertos.org/a00106.html
- FreeRTOS 书籍
