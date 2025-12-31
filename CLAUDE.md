# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 提供在此代码库中工作的指导。

## 语言偏好

**在此仓库中工作时，请始终使用中文回答。**

## 项目概述

基于 STM32U575xx 微控制器的时钟应用，使用 STM32CubeMX 生成的代码配合 FreeRTOS。
- **MCU**: STM32U575xx (ARM Cortex-M33 带 FPU)
- **RTOS**: FreeRTOS v11.2.0 with CMSIS-RTOS v2 API
- **构建系统**: CMake + Ninja
- **工具链**: arm-none-eabi-gcc

## 构建命令

```bash
# 构建项目
./build.sh

# 或手动构建:
cmake -DCMAKE_TOOLCHAIN_FILE=cmake/gcc-arm-none-eabi.cmake -B build -G Ninja
ninja -C build

# 清理后重新构建
rm -rf build && ./build.sh
```

**构建输出**: `build/U5_Clock.elf`

## 项目结构

```
Core/
├── Inc/          # 应用头文件（包含 main.h 和 GPIO 引脚定义）
├── Src/          # 应用源文件（main.c, app_freertos.c, 外设初始化）
Drivers/          # STM32 HAL 和 CMSIS（请勿修改）
Middlewares/
└── Third_Party/
    ├── CMSIS/    # CMSIS-RTOS2 API（请勿修改）
    └── FreeRTOS/ # FreeRTOS 内核（请勿修改）
cmake/            # 构建配置
```

## STM32CubeMX 代码生成

本项目使用 STM32CubeMX 进行外设配置。`.ioc` 文件（`U5_Clock.ioc`）是配置的单一真实来源。

**USER CODE 代码段**：所有用户代码必须放在 `/* USER CODE BEGIN */` 和 `/* USER CODE END */` 标记之间。此标记之外的代码在 STM32CubeMX 重新生成代码时会被覆盖。

常用的 USER CODE 代码段：
- `/* USER CODE BEGIN Includes */` / `/* USER CODE END Includes */` - 添加自定义头文件
- `/* USER CODE BEGIN 2 */` / `/* USER CODE END 2 */` - 在 main.c 中，外设初始化之后、RTOS 启动之前
- `/* USER CODE BEGIN RTOS_THREADS */` - 在 app_freertos.c 中添加新的 FreeRTOS 任务
- `/* USER CODE BEGIN 4 */` / `/* USER CODE END 4 */` - 额外的函数实现

## 应用架构

**启动流程**：
1. `Reset_Handler()` → `SystemInit()` → `main()`
2. HAL 初始化和外设 `MX_*_Init()` 调用
3. `osKernelInitialize()` → `MX_FREERTOS_Init()`（创建任务）
4. `osKernelStart()`（永不返回）

**FreeRTOS 配置** (`Core/Inc/FreeRTOSConfig.h`)：
- CPU: Cortex-M33，启用 FPU，无 TrustZone
- 堆: 100,240 字节
- 时钟节拍: 1000 Hz
- 最大优先级: 56
- 栈溢出检查: 已启用（configCHECK_FOR_STACK_OVERFLOW = 2）

**添加任务**：编辑 `Core/Src/app_freertos.c` 中的 `MX_FREERTOS_Init()` 函数，在 `/* USER CODE BEGIN RTOS_THREADS */` 代码段内添加。

## GPIO 引脚定义 (Core/Inc/main.h)

| 引脚  | 端口  | 功能           |
|-------|-------|----------------|
| PD3   | GPIOD | LED1           |
| PD2   | GPIOD | LED2           |
| PE1   | GPIOE | KEY1 (按键)    |
| PE0   | GPIOE | KEY2 (按键)    |
| PB9   | GPIOB | KEY3 (按键)    |
| PB4   | GPIOB | LCD_RST        |
| PB10  | GPIOB | MAX_MODE       |
| PD6   | GPIOD | FSM_INT        |

## 已配置的外设

- **I2C1** - 可能用于 RTC/显示通信
- **USART1/USART3** - 通信接口
- **SAI2** - 音频接口
- **TIM15** - FreeRTOS 的 HAL 时基
- **GPDMA1** - DMA 控制器
- **ICACHE** - 指令缓存
- **FMC** - 外部存储控制器

## 链接器配置

- **FLASH**: 2048K @ 0x08000000
- **RAM**: 768K @ 0x20000000
- **SRAM4**: 16K @ 0x28000000
- 最小堆: 0x200, 最小栈: 0x400

文件：`STM32U575xx_FLASH.ld`（主要），`STM32U575xx_RAM.ld`

## 工具链要求

- arm-none-eabi-gcc (ARM GCC)
- cmake >= 3.22
- ninja
- arm-none-eabi-newlib (C 库 - nano 版本)
