---
name: embedded-c-patterns
description: 嵌入式 C 设计模式和最佳实践。用于编写、审查或重构嵌入式 C 代码。涵盖寄存器访问、中断处理、内存管理、外设驱动等。
origin: ECC
---

# 嵌入式 C 设计模式与最佳实践

全面的嵌入式 C 设计模式，涵盖硬件寄存器访问、中断处理、内存管理、外设驱动开发等。

## 何时使用

- 编写新的嵌入式 C 代码（驱动程序、固件、中断处理）
- 审查或重构现有的嵌入式 C 代码
- 在嵌入式项目中做出架构决策
- 开发裸机或 RTOS 应用

### 何时不使用

- 上层应用开发
- Qt/图形界面开发
- 非嵌入式项目

## 核心原则

1. **硬件抽象** — 使用寄存器定义和访问函数
2. **中断安全** — 中断处理程序保持简短
3. **确定性** — 避免动态内存分配和不确定延迟
4. **资源高效** — 最小化 RAM 和 ROM 使用
5. **可移植性** — 使用标准模式，隔离平台特定代码

## 寄存器访问

### 寄存器定义

```c
// 好：使用 volatile 和有类型的定义
#define PERIPH_BASE       ((uint32_t)0x40000000)
#define APB2PERIPH_BASE   (PERIPH_BASE + 0x10000)
#define GPIOA_BASE        (APB2PERIPH_BASE + 0x0800)

// 使用结构体定义寄存器
typedef struct {
    volatile uint32_t CRL;
    volatile uint32_t CRH;
    volatile uint32_t IDR;
    volatile uint32_t ODR;
    volatile uint32_t BSRR;
    volatile uint32_t BRR;
    volatile uint32_t LCKR;
} GPIO_TypeDef;

#define GPIOA ((GPIO_TypeDef *) GPIOA_BASE)

// 使用 volatile 防止编译器优化
static volatile uint32_t * const pREG = (volatile uint32_t *)0x40021018;
```

### 位操作

```c
// 设置特定位
#define SET_BIT(reg, bit)    ((reg) |= (1 << (bit)))

// 清除特定位
#define CLR_BIT(reg, bit)    ((reg) &= ~(1 << (bit)))

// 读取特定位
#define READ_BIT(reg, bit)   ((reg) & (1 << (bit)))

// 切换特定位
#define TOG_BIT(reg, bit)    ((reg) ^= (1 << (bit)))

// 原子操作（如果架构支持）
#define SET_BIT_ATOMIC(reg, bit)   do { \
    volatile uint32_t *preg = &(reg);  \
    uint32_t mask = 1 << (bit);       \
    *preg = *preg | mask;             \
} while(0)

// 使用示例
SET_BIT(GPIOA->ODR, 5);  // 设置 GPIOA 第 5 位
```

### 位域

```c
// 谨慎使用位域（可移植性问题）
typedef struct {
    volatile uint32_t EN    : 1;
    uint32_t              : 31;  // 填充
} ControlReg_t;

#define CTRL ((ControlReg_t *)0x40001000)

// 或者使用掩码（更可移植）
typedef struct {
    volatile uint32_t REG;
} DeviceReg_t;

#define DEV_CTRL   (*(DeviceReg_t *)0x40001000)

// 位掩码定义
#define CTRL_EN_MASK    (1 << 0)
#define CTRL_IRQ_MASK   (1 << 1)
#define CTRL_MODE_MASK  (3 << 2)

// 设置
DEV_CTRL.REG = (DEV_CTRL.REG & ~CTRL_MODE_MASK) | (2 << 2);
```

## 中断处理

### 中断处理函数

```c
// 好：保持中断处理简短
void TIM2_IRQHandler(void) {
    // 清除中断标志
    TIM2->SR = ~TIM_SR_UIF;

    // 最小化处理
    g_tick_count++;

    // 不要在这里执行耗时操作
}

// 坏：在中断中执行复杂操作
void BAD_IRQHandler(void) {
    // 危险！可能丢失中断
    char buffer[100];
    sprintf(buffer, "Interrupt at %lu\n", ticks);
    log_to_uart(buffer);  // 可能阻塞
}
```

### 中断安全的数据共享

```c
// 使用 volatile 标记共享变量
volatile uint32_t g_system_ticks = 0;
volatile bool g_data_ready = false;

// 或使用原子操作（如果支持）
#include <stdatomic.h>
atomic_int g_counter = 0;

// 中断中设置标志，主循环处理
void USART1_IRQHandler(void) {
    volatile uint32_t *sr = &USART1->SR;
    if (*sr & USART_SR_RXNE) {
        g_rx_buffer[g_rx_head++] = USART1->DR;
        g_rx_head &= (BUFFER_SIZE - 1);
    }
}

// 主循环处理
void main_loop(void) {
    while (1) {
        if (g_data_ready) {
            process_data();
            g_data_ready = false;
        }
    }
}
```

### 临界区

```c
// 方法1：禁用中断（简单但影响所有中断）
void critical_section_enter(void) {
    __disable_irq();
}

void critical_section_exit(void) {
    __enable_irq();
}

// 使用
critical_section_enter();
// 关键代码
critical_section_exit();

// 方法2：保存/恢复中断状态（更好）
void safe_critical_enter(void) {
    uint32_t primask;
    __asm volatile ("mrs %0, primask" : "=r"(primask));
    __disable_irq();
    // 保存以便恢复
    __set_PRIMASK(primask);
}

// 方法3：使用 CMSIS 函数
void cmsis_critical_enter(void) {
    uint32_t pm = __get_PRIMASK();
    __disable_irq();
    // 存储旧状态
}

void cmsis_critical_exit(uint32_t old_state) {
    __set_PRIMASK(old_state);
}
```

## 外设驱动模式

### 初始化模式

```c
typedef struct {
    uint32_t baud_rate;
    uint8_t  data_bits;
    uint8_t  stop_bits;
    uint8_t  parity;
} UART_Config_t;

// 初始化函数
int uart_init(UART_TypeDef *uart, const UART_Config_t *config) {
    if (config->baud_rate == 0) return -1;

    // 启用时钟
    if (uart == UART1) {
        RCC->APB2ENR |= RCC_APB2ENR_USART1EN;
    }

    // 配置波特率
    uint32_t apb_clock = SystemCoreClock;
    uart->BRR = apb_clock / config->baud_rate;

    // 配置数据格式
    uart->CR1 = 0;
    if (config->data_bits == 8) uart->CR1 |= USART_CR1_M;
    if (config->parity != 0)    uart->CR1 |= USART_CR1_PCE;

    // 启用
    uart->CR1 |= USART_CR1_UE | USART_CR1_RE | USART_CR1_TE;

    return 0;
}

// 使用
UART_Config_t cfg = {
    .baud_rate = 115200,
    .data_bits = 8,
    .stop_bits = 1,
    .parity = 0
};
uart_init(UART1, &cfg);
```

### 状态机模式

```c
typedef enum {
    STATE_IDLE,
    STATE_TX_HEADER,
    STATE_TX_DATA,
    STATE_TX_CRC,
    STATE_RX_HEADER,
    STATE_RX_DATA,
    STATE_RX_CRC,
    STATE_COMPLETE,
    STATE_ERROR
} ProtocolState_t;

typedef struct {
    ProtocolState_t state;
    uint8_t buffer[256];
    uint16_t index;
    uint16_t expected_len;
} ProtocolContext_t;

void protocol_init(ProtocolContext_t *ctx) {
    ctx->state = STATE_IDLE;
    ctx->index = 0;
}

void protocol_byte_received(ProtocolContext_t *ctx, uint8_t byte) {
    switch (ctx->state) {
        case STATE_IDLE:
            if (byte == 0xAA) {
                ctx->buffer[ctx->index++] = byte;
                ctx->state = STATE_RX_HEADER;
            }
            break;

        case STATE_RX_HEADER:
            ctx->buffer[ctx->index++] = byte;
            if (ctx->index >= 4) {
                ctx->expected_len = (ctx->buffer[2] << 8) | ctx->buffer[3];
                ctx->state = STATE_RX_DATA;
            }
            break;

        case STATE_RX_DATA:
            ctx->buffer[ctx->index++] = byte;
            if (ctx->index >= ctx->expected_len) {
                ctx->state = STATE_COMPLETE;
            }
            break;

        default:
            ctx->state = STATE_ERROR;
            break;
    }
}
```

## 内存管理

### 静态分配（推荐用于嵌入式）

```c
// 好：静态分配缓冲区
#define BUFFER_SIZE 256

static uint8_t tx_buffer[BUFFER_SIZE];
static uint8_t rx_buffer[BUFFER_SIZE];

// 或使用结构体池
typedef struct {
    bool in_use;
    uint32_t data;
} ObjectPool_t;

#define POOL_SIZE 10
static ObjectPool_t g_object_pool[POOL_SIZE];

void* pool_alloc(void) {
    for (int i = 0; i < POOL_SIZE; i++) {
        if (!g_object_pool[i].in_use) {
            g_object_pool[i].in_use = true;
            return &g_object_pool[i];
        }
    }
    return NULL;
}

void pool_free(void *ptr) {
    if (ptr) {
        ((ObjectPool_t *)ptr)->in_use = false;
    }
}
```

### 避免动态分配

```c
// 坏：在实时系统中使用 malloc
void critical_task(void) {
    uint8_t *buffer = malloc(256);  // 危险！
    if (buffer) {
        // 使用
        free(buffer);  // 容易被遗忘
    }
}

// 好：使用静态缓冲区
void critical_task(void) {
    static uint8_t buffer[256];  // 或栈上：uint8_t buffer[256];
    // 使用
}
```

### 栈使用注意

```c
// 好：小数组在栈上
void process_data(void) {
    uint8_t temp[64];  // 小数组，安全
    // 处理
}

// 坏：大数组在栈上
void process_data(void) {
    uint8_t buffer[4096];  // 危险！可能溢出栈
}
```

## 硬件初始化顺序

```c
// 正确的初始化顺序
void system_init(void) {
    // 1. 配置系统时钟
    system_clock_config();

    // 2. 配置 NVIC（如果需要中断）
    nvic_config();

    // 3. 配置 GPIO
    gpio_init();

    // 4. 配置外设
    uart_init();
    spi_init();
    timer_init();

    // 5. 启用外设
    enable_peripherals();

    // 6. 启用全局中断
    __enable_irq();
}
```

## 调试技巧

### 断言

```c
#ifdef DEBUG
#define ASSERT(expr) \
    do { if (!(expr)) { error_handler(__FILE__, __LINE__); } } while(0)
#else
#define ASSERT(expr) ((void)0)
#endif

void error_handler(char *file, int line) {
    while(1) {
        // 闪烁 LED 或输出调试信息
    }
}

// 使用
ASSERT(ptr != NULL);
ASSERT(count < MAX_COUNT);
```

### 日志输出

```c
// 简单日志
#define LOG_DEBUG(fmt, ...) \
    do { if (DEBUG_ENABLED) { uart_send_string("[DBG] "); uart_printf(fmt, ##__VA_ARGS__); } } while(0)

#define LOG_ERROR(fmt, ...) \
    uart_send_string("[ERR] "); uart_printf(fmt, ##__VA_ARGS__)

// 使用
LOG_DEBUG("Value: %d\n", value);
LOG_ERROR("Init failed: %d\n", error_code);
```

## 快速参考清单

完成嵌入式 C 工作前检查：

- [ ] 使用 volatile 关键字访问硬件寄存器
- [ ] 中断处理程序保持简短
- [ ] 使用静态分配而非动态 malloc/free
- [ ] 验证所有指针（NULL 检查）
- [ ] 避免在中断中调用非中断安全函数
- [ ] 使用位操作而非位域（更可移植）
- [ ] 正确初始化外设时钟
- [ ] 遵循硬件初始化顺序
- [ ] 临界区使用正确的保护机制
- [ ] 避免栈溢出（大数组使用静态分配）

## 相关资源

- CMSIS 文档
- STM32 HAL/LL 库
- FreeRTOS 文档
- ARM Cortex-M 编程指南
