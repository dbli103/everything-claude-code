# 嵌入式 C 编码规范

## 概述

本规范适用于嵌入式系统 C 编程，涵盖 ARM Cortex-M、AVR、PIC 等微控制器。

## 文件组织

### 头文件

```c
// my_driver.h
#ifndef MY_DRIVER_H
#define MY_DRIVER_H

#include <stdint.h>
#include <stdbool.h>

// 硬件寄存器定义
#define MY_REG_BASE       ((uint32_t)0x40001000)
#define MY_REG             ((MY_REG_TypeDef *)MY_REG_BASE)

// 类型定义
typedef struct {
    volatile uint32_t CR;
    volatile uint32_t SR;
    volatile uint32_t DR;
} MY_REG_TypeDef;

// 函数声明
void my_driver_init(void);
void my_driver_write(uint8_t data);
uint8_t my_driver_read(void);

#endif // MY_DRIVER_H
```

### 源文件

```c
// my_driver.c
#include "my_driver.h"
#include "rcc.h"

// 静态变量
static bool g_initialized = false;

// 函数实现
void my_driver_init(void) {
    // 启用时钟
    RCC_ENABLE(DRIVER_CLK);

    // 配置寄存器
    MY_REG->CR = CR_ENABLE_Msk;

    g_initialized = true;
}

uint8_t my_driver_read(void) {
    if (!g_initialized) {
        return 0;
    }
    return (uint8_t)(MY_REG->DR & DR_DATA_Msk);
}
```

## 命名规范

### 常量

```c
// 枚举值：全大写下划线
typedef enum {
    STATE_IDLE = 0,
    STATE_RUNNING,
    STATE_ERROR
} driver_state_t;

// 位域定义
#define FLAG_ENABLE    (1 << 0)
#define FLAG_READY     (1 << 1)

// 常量
#define BUFFER_SIZE    256
#define TIMEOUT_MS     1000
```

### 变量

```c
// 全局变量：g_ 前缀
uint32_t g_system_ticks = 0;

// 静态变量：s_ 前缀
static bool s_initialized = false;

// 局部变量：camelCase
uint32_t timeout = 100;

// 匈牙利命名（可选，用于硬件相关）
uint8_t *pRxBuffer;  // p = pointer
uint8_t ucCounter;   // uc = unsigned char
```

### 函数

```c
// 函数：下划线分隔
void driver_init(void);
void peripheral_enable(void);
uint32_t calculate_crc(const uint8_t *data, uint32_t len);

// 内部函数：static + 下划线前缀
static void internal_process(void);
```

## 寄存器访问

### Volatile 关键字

```c
// 硬件寄存器必须使用 volatile
#define UART_SR ((volatile uint32_t *)0x40013800)

// 或使用结构体
typedef struct {
    volatile uint32_t SR;
    volatile uint32_t DR;
    volatile uint32_t BRR;
} UART_TypeDef;

#define UART1 ((UART_TypeDef *)0x40013800)

// 寄存器访问
UART1->SR = 0;  // 编译器不会优化掉
```

### 位操作

```c
// 设置位
#define BIT_SET(reg, bit)    ((reg) |= (1 << (bit)))
#define BIT_CLR(reg, bit)    ((reg) &= ~(1 << (bit)))
#define BIT_READ(reg, bit)   ((reg) & (1 << (bit)))

// 使用示例
BIT_SET(GPIOA->ODR, 5);  // 设置 PA5
```

### 寄存器结构体

```c
typedef struct {
    volatile uint32_t CR1;
    volatile uint32_t CR2;
    volatile uint32_t CNF;
    volatile uint32_t Reserved;
} GPIO_TypeDef;

#define GPIOA_BASE   0x48000000
#define GPIOA         ((GPIO_TypeDef *)GPIOA_BASE)

// 使用
GPIOA->CR1 = 0x00000001;
```

## 函数设计

### 参数验证

```c
// 入口参数验证
int driver_send(const uint8_t *data, uint32_t len) {
    if (data == NULL) {
        return -1;  // 无效参数
    }
    if (len == 0 || len > MAX_BUFFER) {
        return -2;  // 长度无效
    }
    // 处理
    return 0;
}
```

### 返回值

```c
// 统一返回类型
typedef enum {
    DRV_OK = 0,
    DRV_ERROR = -1,
    DRV_BUSY = -2,
    DRV_TIMEOUT = -3,
    DRV_INVALID_PARAM = -4
} drv_status_t;

// 使用
drv_status_t result = driver_init();
if (result != DRV_OK) {
    // 错误处理
}
```

## 内存管理

### 静态分配（推荐）

```c
// 静态缓冲区
static uint8_t tx_buffer[256];
static uint8_t rx_buffer[256];

// 对象池
typedef struct {
    bool in_use;
    uint32_t data;
} object_t;

#define POOL_SIZE 16
static object_t g_pool[POOL_SIZE];
```

### 栈使用注意

```c
// 小数组可以使用栈
void process(void) {
    uint8_t temp[32];  // 小数组，安全
}

// 大数组使用静态分配
static uint8_t large_buffer[1024];
```

### 避免动态分配

```c
// 避免：malloc/free
void critical_task(void) {
    uint8_t *buf = malloc(256);  // 危险！
    // ...
    free(buf);
}

// 推荐：静态分配
void critical_task(void) {
    static uint8_t buf[256];
    // ...
}
```

## 中断处理

### 基本原则

```c
// 保持中断处理简短
void TIM2_IRQHandler(void) {
    // 清除标志
    TIM2->SR = ~TIM_SR_UIF;

    // 最小化处理
    g_tick_count++;
    // 不执行耗时操作
}
```

### 临界区

```c
// 方法1：禁用中断
void critical_enter(void) {
    __disable_irq();
}

void critical_exit(void) {
    __enable_irq();
}

// 方法2：保存状态
void critical_safe(void) {
    uint32_t primask;
    __asm volatile ("mrs %0, primask" : "=r"(primask));
    __disable_irq();

    // 临界代码

    __set_PRIMASK(primask);
}
```

## 代码格式化

### 缩进

```c
// 使用 4 空格
void function(void) {
    if (condition) {
        do_something();
    } else {
        do_other();
    }
}
```

### 大括号

```c
// K&R 风格
void function(void)
{
    if (condition) {
        do_something();
    }
}
```

## 注释

```c
// 单行注释
// Initialize peripheral

/* 多行注释
 * 初始化外设
 * 参数: None
 * 返回: None
 */

/** Doxygen 文档
 * @brief 初始化驱动
 * @return 状态码
 */
drv_status_t driver_init(void);
```

## 预处理指令

```c
// 条件编译
#ifdef DEBUG
#define LOG(...) printf(__VA_ARGS__)
#else
#define LOG(...)
#endif

// 硬件配置
#if defined(STM32F407xx)
    #include "stm32f4xx_hal.h"
#elif defined(STM32F103xx)
    #include "stm32f1xx_hal.h"
#endif
```

## 头文件保护

```c
#ifndef PROJECT_MODULE_DRIVER_H
#define PROJECT_MODULE_DRIVER_H

// 内容

#endif // PROJECT_MODULE_DRIVER_H
```

## 断言

```c
#ifdef DEBUG
#define ASSERT(expr) \
    do { if (!(expr)) { error_handler(__FILE__, __LINE__); } } while(0)
#else
#define ASSERT(expr) ((void)0)
#endif

// 使用
ASSERT(ptr != NULL);
ASSERT(count < MAX_COUNT);
```
