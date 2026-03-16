---
name: c-arm-development
description: ARM Cortex-M 开发指南。涵盖 STM32、Nordic、TI 等 ARM Cortex-M MCU 的开发。包含时钟配置、GPIO、中断、外设驱动等。
origin: ECC
---

# ARM Cortex-M 开发指南

全面的 ARM Cortex-M MCU 开发指南，涵盖 STM32、Nordic、TI 等主流 ARM 芯片。

## 何时使用

- 开发基于 ARM Cortex-M 的嵌入式应用
- 移植现有代码到新的 ARM 芯片
- 配置外设和中断
- 优化 ARM 性能

### 目标芯片

- STM32 F0/F1/F2/F3/F4/F7/L0/L1/L4/L5/H7
- Nordic nRF52/nRF53
- TI Tiva/Stellaris
- NXP LPC/Kinetis
- 其他 ARM Cortex-M0/M0+/M3/M4/M7

## 时钟配置

### STM32 时钟树

```c
// STM32F4 系统时钟配置示例
void SystemClock_Config(void) {
    RCC_ClkInitTypeDef RCC_ClkInitStruct;
    RCC_OscInitTypeDef RCC_OscInitStruct;

    // 1. 启用 HSE 并配置 PLL
    RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
    RCC_OscInitStruct.HSEState = RCC_HSE_ON;
    RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
    RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
    RCC_OscInitStruct.PLL.PLLM = 8;    // 8MHz / 8 = 1MHz
    RCC_OscInitStruct.PLL.PLLN = 336;  // 1MHz * 336 = 336MHz
    RCC_OscInitStruct.PLL.PLLP = RCC_PLLP_DIV2;  // 336MHz / 2 = 168MHz
    RCC_OscInitStruct.PLL.PLLQ = 7;    // 用于 USB
    HAL_RCC_OscConfig(&RCC_OscInitStruct);

    // 2. 选择 PLL 作为系统时钟源
    RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK | RCC_CLOCKTYPE_SYSCLK
                                | RCC_CLOCKTYPE_PCLK1 | RCC_CLOCKTYPE_PCLK2;
    RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
    RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;    // 168MHz
    RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV4;      // 42MHz
    RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV2;      // 84MHz
    HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_5);
}
```

### 获取时钟频率

```c
// 获取各总线时钟频率
uint32_t HAL_RCC_GetSysClockFreq(void);
uint32_t HAL_RCC_GetHCLKFreq(void);
uint32_t HAL_RCC_GetPCLK1Freq(void);
uint32_t HAL_RCC_GetPCLK2Freq(void);

// 使用示例
uint32_t uart_clock = HAL_RCC_GetPCLK1Freq();
uint32_t brr = uart_clock / 115200;
```

## GPIO 配置

### 输入模式

```c
// 浮空输入
GPIO_InitStruct.Pin = GPIO_PIN_0;
GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
GPIO_InitStruct.Pull = GPIO_NOPULL;
HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

// 上拉输入
GPIO_InitStruct.Pull = GPIO_PULLUP;

// 下拉输入
GPIO_InitStruct.Pull = GPIO_PULLDOWN;

// 模拟输入（ADC）
GPIO_InitStruct.Mode = GPIO_MODE_ANALOG;
```

### 输出模式

```c
// 推挽输出（常用）
GPIO_InitStruct.Pin = GPIO_PIN_5;
GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
GPIO_InitStruct.Pull = GPIO_NOPULL;
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

// 开漏输出（用于 I2C、线与）
GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_OD;
```

### 复用功能

```c
// UART TX
GPIO_InitStruct.Pin = GPIO_PIN_9;
GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
GPIO_InitStruct.Pull = GPIO_PULLUP;
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
GPIO_InitStruct.Alternate = GPIO_AF7_USART1;
HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

// SPI SCK
GPIO_InitStruct.Pin = GPIO_PIN_5;
GPIO_InitStruct.Alternate = GPIO_AF5_SPI1;
HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
```

### GPIO 操作

```c
// 置位
HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET);

// 清零
HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_RESET);

// 读取
GPIO_PinState state = HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0);

// 翻转
HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
```

## NVIC 中断管理

### 中断优先级

```c
// 设置中断优先级（数值越小优先级越高）
// Cortex-M 中，优先级分组影响抢占能力

// NVIC_SetPriority 示例
NVIC_SetPriority(TIM2_IRQn, 1);  // 优先级 1
NVIC_SetPriority(USART1_IRQn, 2); // 优先级 2（更低）

// 启用中断
NVIC_EnableIRQ(TIM2_IRQn);

// 禁用中断
NVIC_DisableIRQ(TIM2_IRQn);

// 优先级分组
NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4);  // 全部用于抢占优先级
```

### 中断处理函数

```c
// STM32 中断处理函数
void TIM2_IRQHandler(void) {
    HAL_TIM_IRQHandler(&htim2);
}

// HAL 回调
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
    if (htim->Instance == TIM2) {
        // 处理定时器溢出
    }
}

// 外部中断
void EXTI0_IRQHandler(void) {
    HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_0);
}

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if (GPIO_Pin == GPIO_PIN_0) {
        // 处理按钮按下
    }
}
```

## 常用外设驱动

### UART/串口

```c
// 初始化
UART_HandleTypeDef huart1;
huart1.Instance = USART1;
huart1.Init.BaudRate = 115200;
huart1.Init.WordLength = UART_WORDLENGTH_8B;
huart1.Init.StopBits = UART_STOPBITS_1;
huart1.Init.Parity = UART_PARITY_NONE;
huart1.Init.Mode = UART_MODE_TX_RX;
huart1.Init.HwFlowCtl = UART_HWCONTROL_NONE;
huart1.Init.OverSampling = UART_OVERSAMPLING_16;
HAL_UART_Init(&huart1);

// 发送
uint8_t tx_data[] = "Hello\r\n";
HAL_UART_Transmit(&huart1, tx_data, sizeof(tx_data), HAL_MAX_DELAY);

// 接收（阻塞）
uint8_t rx_data[10];
HAL_UART_Receive(&huart1, rx_data, 10, HAL_MAX_DELAY);

// 接收（中断）
uint8_t rx_buffer[10];
HAL_UART_Receive_IT(&huart1, rx_buffer, 10);

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    // 处理接收数据
}
```

### SPI

```c
// 初始化
SPI_HandleTypeDef hspi1;
hspi1.Instance = SPI1;
hspi1.Init.Mode = SPI_MODE_MASTER;
hspi1.Init.Direction = SPI_DIRECTION_2LINES;
hspi1.Init.DataSize = SPI_DATASIZE_8BIT;
hspi1.Init.CLKPolarity = SPI_POLARITY_LOW;
hspi1.Init.CLKPhase = SPI_PHASE_1EDGE;
hspi1.Init.NSS = SPI_NSS_SOFT;
hspi1.Init.BaudRatePrescaler = SPI_BAUDRATEPRESCALER_16;
hspi1.Init.FirstBit = SPI_FIRSTBIT_MSB;
HAL_SPI_Init(&hspi1);

// 发送/接收
uint8_t tx_data[] = {0x01, 0x02, 0x03};
uint8_t rx_data[3];
HAL_SPI_TransmitReceive(&hspi1, tx_data, rx_data, 3, HAL_MAX_DELAY);
```

### I2C

```c
// 初始化
I2C_HandleTypeDef hi2c1;
hi2c1.Instance = I2C1;
hi2c1.Init.ClockSpeed = 400000;
hi2c1.Init.DutyCycle = I2C_DUTYCYCLE_2;
hi2c1.Init.AddressingMode = I2C_ADDRESSINGMODE_7BIT;
hi2c1.Init.DualAddressMode = I2C_DUALADDRESS_DISABLE;
hi2c1.Init.OwnAddress1 = 0;
hi2c1.Init.GeneralCallMode = I2C_GENERALCALL_DISABLE;
hi2c1.Init.NoStretchMode = I2C_NOSTRETCH_DISABLE;
HAL_I2C_Init(&hi2c1);

// 写入寄存器
uint8_t reg_addr = 0x10;
uint8_t reg_value = 0x55;
HAL_I2C_Mem_Write(&hi2c1, device_addr << 1, reg_addr,
                   I2C_MEMADD_SIZE_8BIT, &reg_value, 1, HAL_MAX_DELAY);

// 读取寄存器
HAL_I2C_Mem_Read(&hi2c1, device_addr << 1, reg_addr,
                 I2C_MEMADD_SIZE_8BIT, &reg_value, 1, HAL_MAX_DELAY);
```

### ADC

```c
// 初始化
ADC_HandleTypeDef hadc1;
hadc1.Instance = ADC1;
hadc1.Init.Resolution = ADC_RESOLUTION_12B;
hadc1.Init.ScanConvMode = DISABLE;
hadc1.Init.ContinuousConvMode = DISABLE;
hadc1.Init.DiscontinuousConvMode = DISABLE;
hadc1.Init.ExternalTrigConvEdge = ADC_EXTERNALTRIGCONVEDGE_NONE;
hadc1.Init.DataAlign = ADC_DATAALIGN_RIGHT;
hadc1.Init.NbrOfConversion = 1;
HAL_ADC_Init(&hadc1);

// 校准
HAL_ADCEx_Calibration_Start(&hadc1);

// 读取
HAL_ADC_Start(&hadc1);
HAL_ADC_PollForConversion(&hadc1, HAL_MAX_DELAY);
uint32_t adc_value = HAL_ADC_GetValue(&hadc1);
HAL_ADC_Stop(&hadc1);
```

### 定时器

```c
// 初始化定时器（1ms 中断）
TIM_HandleTypeDef htim2;
htim2.Instance = TIM2;
htim2.Init.Prescaler = (SystemCoreClock / 1000) - 1;  // 1ms
htim2.Init.Period = 1000 - 1;  // 1s
htim2.Init.CounterMode = TIM_COUNTERMODE_UP;
HAL_TIM_Base_Init(&htim2);

HAL_TIM_Base_Start_IT(&htim2);

// PWM 输出
TIM_HandleTypeDef htim1;
htim1.Instance = TIM1;
htim1.Init.Prescaler = 0;
htim1.Init.Period = 1000 - 1;
htim1.Init.Pulse = 500;  // 50% 占空比
htim1.Init.OCMode = TIM_OCMODE_PWM1;
HAL_TIM_PWM_Init(&htim1);

TIM_OC_InitTypeDef sConfigOC;
sConfigOC.Pulse = 500;
sConfigOC.OCMode = TIM_OCMODE_PWM1;
sConfigOC.OCPolarity = TIM_OCPOLARITY_HIGH;
HAL_TIM_PWM_ConfigChannel(&htim1, &sConfigOC, TIM_CHANNEL_1);

HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);
```

## DMA

### 内存到外设

```c
// UART DMA 发送
uint8_t tx_buffer[256];
memcpy(tx_buffer, "Hello DMA", 9);

hdma_usart1_tx.Instance = DMA2_Stream7;
hdma_usart1_tx.Init.Channel = DMA_CHANNEL_4;
hdma_usart1_tx.Init.Direction = DMA_MEMORY_TO_PERIPH;
hdma_usart1_tx.Init.PeriphInc = DMA_PINC_DISABLE;
hdma_usart1_tx.Init.MemInc = DMA_MINC_ENABLE;
hdma_usart1_tx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
hdma_usart1_tx.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
hdma_usart1_tx.Init.Mode = DMA_NORMAL;
hdma_usart1_tx.Init.Priority = DMA_PRIORITY_LOW;
HAL_DMA_Init(&hdma_usart1_tx);

huart1.hdmatx = &hdma_usart1_tx;
__HAL_LINKDMA(&huart1, hdmatx, hdma_usart1_tx);

// 启动 DMA 发送
HAL_UART_Transmit_DMA(&huart1, tx_buffer, 9);
```

## 低功耗模式

```c
// 睡眠模式
HAL_SuspendTick();
HAL_PWR_EnterSLEEPMode(PWR_MAINREGULATOR_ON, PWR_SLEEPENTRY_WFI);

// 停止模式
HAL_PWR_EnterSTOPMode(PWR_LOWPOWERREGULATOR_ON, PWR_SLEEPENTRY_WFI);

// 待机模式
HAL_PWR_EnterSTANDBYMode();

// 退出停止模式后重新配置时钟
void HAL_PWR_EnterSTOPMode(uint32_t Regulator, uint8_t STOPEntry) {
    // 进入停止模式
    HAL_PWR_EnterSTOPMode(Regulator, STOPEntry);

    // 重新配置时钟
    SystemClock_Config();
}
```

## 快速参考清单

- [ ] 时钟配置正确（HSE/PLL）
- [ ] GPIO 模式设置正确（输入/输出/复用）
- [ ] NVIC 中断优先级合理
- [ ] 外设时钟已启用
- [ ] DMA 用于高速数据传输
- [ ] 使用适当的低功耗模式
- [ ] 配置适当的看门狗

## 相关资源

- STM32 HAL 库文档
- CMSIS 标准库
- 芯片数据手册
- 参考手册（RM）
