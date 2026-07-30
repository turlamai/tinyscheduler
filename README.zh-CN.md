# TinyScheduler

一个面向 STM32 的微型抢占式任务调度器，基于 Cortex-M3 PendSV 异常从零构建。不依赖任何 RTOS，除 CMSIS 外无其他依赖 — 仅用不到 200 行代码实现了最小化的轮转上下文切换。 演示工程运行在 STM32F103RC 上，通过两个独立任务分别控制两颗 LED（PC2、PC3）闪烁，验证调度器在实际硬件上的正确运行。

![C](https://img.shields.io/badge/C-A8B9CC?logo=c&logoColor=white) [![License](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

[English](README.md) · **中文**

## 工作原理

调度器利用 Cortex-M3 硬件压栈机制：

1. **SysTick** 每 10 ms 触发一次，发起 PendSV 异常
2. **PendSV 处理函数**（汇编实现，`task.s`）执行上下文切换：
   - 通过 PSP 将当前任务的 R4-R11（被调用者保存寄存器）压栈
   - 调用 `switch_current_TCB()` 按轮转顺序选择下一个任务
   - 从新任务的栈中恢复 R4-R11
   - 使用 `BX LR` 返回，EXCRETURN bit 2 置位以使用 PSP
3. **任务创建**（`create_task()`）初始化每个任务的栈帧：
   - 硬件自动保存的寄存器：xPSR、PC（入口函数）、LR、R12、R3-R0
   - 手动保存的寄存器：R4-R11（占位值）

空闲任务（`task_idle`）在没有用户任务可调度时运行。

## 特性

- **抢占式轮转调度** — 所有任务均分时间片
- **最多 8 个任务** — 通过 `CONFIG_MAX_TASK_NUM` 可配置
- **10 ms 时钟周期** — 通过 `CONFIG_OS_TICK_TIME_US` 可配置
- **纯 CMSIS** — 不依赖 HAL，仅使用寄存器级操作
- **汇编上下文切换** — PendSV 处理函数在 `task.s` 中，保证切换时序确定
- **Keil MDK 工程** — 可直接在 uVision 中打开并构建

## 快速开始

### 1. 克隆仓库

```bash
git clone https://github.com/turinglambdaai/tinyscheduler.git
cd tinyscheduler
```

### 2. 硬件

- **目标 MCU**：STM32F103RC（Cortex-M3，72 MHz）
- **LED 引脚**：PC2 和 PC3（推挽输出）

### 3. 使用 Keil MDK 构建

1. 在 Keil uVision 中打开 `tinyrtos.uvprojx`
2. 构建工程（F7）
3. 烧录到目标板
4. PC2 和 PC3 上的两颗 LED 将以不同频率闪烁

### 移植到其他 STM32

调度器核心（`OS/` 目录）在 Cortex-M3 系列内与具体 MCU 无关。移植步骤：

1. 将 `CMSIS/` 替换为目标 MCU 的 CMSIS 文件
2. 更新 `STARTUP/` 中的启动文件
3. 在 `os.c` 中调整 `CONFIG_SYSCLK` 以匹配你的时钟频率
4. 修改 `USER/led.c` 以适配你的 GPIO 引脚

## API 参考

### `os_init()`

初始化调度器并创建空闲任务。首先调用此函数。

### `os_start()`

启动 SysTick 定时器并使能 PendSV。在所有任务创建完成后调用。此函数不会返回。

### `create_task(function, arguments, stack, stack_size)`

创建新任务并加入调度队列。

| 参数 | 类型 | 说明 |
|------|------|------|
| `function` | `void*` | 任务入口函数（必须为无限循环） |
| `arguments` | `void*` | 通过 R0 传递给任务的参数（可为 `0`） |
| `stack` | `stack_t*` | 栈数组指针 |
| `stack_size` | `uint16_t` | 栈数组大小（以 `uint32_t` 字为单位） |
| **返回值** | `uint16_t` | 任务 ID（失败返回 0） |

### 使用示例

```c
#include "os.h"
#include "task.h"

#define STACK_SIZE 256
stack_t stack_task1[STACK_SIZE];

void my_task() {
    for (;;) {
        // 在此编写任务逻辑
    }
}

int main(void) {
    os_init();

    // 配置外设
    gpio_init();

    create_task(my_task, 0, stack_task1, STACK_SIZE);

    os_start();  // 永不返回
}
```

## 项目结构

```
.
├── OS/
│   ├── os.c              # 调度器核心 — SysTick、PendSV 初始化
│   ├── os.h              # 调度器 API 头文件
│   ├── task.c            # 任务管理 — 创建、切换、轮转逻辑
│   ├── task.h            # TCB 结构体和任务 API
│   └── task.s            # PendSV_Handler — 汇编上下文切换
├── USER/
│   ├── main.c            # 演示应用 — 两个 LED 闪烁任务
│   ├── led.c             # GPIO 配置（PC2 和 PC3）
│   └── led.h             # LED 头文件
├── CMSIS/                # CMSIS 内核及 STM32F10x 设备文件
├── STARTUP/              # 启动汇编（stm32f10x_hd）
├── RTE/                  # Keil 运行时环境配置
├── Objects/              # 构建输出（.o、.hex、.axf）
├── Listings/             # 列表文件（.lst、.map）
└── tinyrtos.uvprojx      # Keil MDK 工程文件
```

## 配置

所有配置位于 `os.c` 中：

| 宏 | 默认值 | 说明 |
|---|--------|------|
| `CONFIG_SYSCLK` | 72 | 系统时钟（MHz） |
| `CONFIG_OS_SYSCLK_DIV` | 8 | SysTick 时钟分频（HCLK/8） |
| `CONFIG_OS_TICK_TIME_US` | 10000 | 时钟周期（微秒，10 ms） |

在 `task.c` 中：

| 宏 | 默认值 | 说明 |
|---|--------|------|
| `CONFIG_MAX_TASK_NUM` | 8 | 最大任务数（含空闲任务） |

## 许可证

基于 [MIT 许可证](LICENSE) 授权。
