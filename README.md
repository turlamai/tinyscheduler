# tinyscheduler

[English](README.md) | [中文](README.zh-CN.md)

A tiny preemptive task scheduler for STM32, built from scratch on the Cortex-M3 PendSV exception. No RTOS, no dependencies beyond CMSIS — just a minimal round-robin context switcher in under 200 lines of code.

The demo project runs on an STM32F103RC and blinks two LEDs (PC2, PC3) in two independent tasks, proving cooperative scheduling works on real hardware.

## How It Works

The scheduler leverages the Cortex-M3 hardware stacking mechanism:

1. **SysTick** fires every 10 ms and triggers a PendSV exception
2. **PendSV handler** (written in assembly, `task.s`) performs the context switch:
   - Saves R4-R11 (callee-saved registers) to the current task's stack via PSP
   - Calls `switch_current_TCB()` to select the next task in round-robin order
   - Restores R4-R11 from the new task's stack
   - Returns using `BX LR` with PSP mode (EXCRETURN bit 2 set)
3. **Task creation** (`create_task()`) initializes each task's stack frame with:
   - Auto-saved registers: xPSR, PC (entry function), LR, R12, R3-R0
   - Manual-saved registers: R4-R11 (placeholder values)

The idle task (`task_idle`) runs when no user tasks are scheduled.

## Features

- **Preemptive round-robin scheduling** — equal time slices for all tasks
- **Up to 8 tasks** — configurable via `CONFIG_MAX_TASK_NUM`
- **10 ms tick period** — configurable via `CONFIG_OS_TICK_TIME_US`
- **Pure CMSIS** — no HAL dependency, uses only register-level access
- **Assembly context switch** — PendSV handler in `task.s` for deterministic switching
- **Keil MDK project** — ready to open and build in uVision

## Quick Start

### Hardware

- **Target MCU**: STM32F103RC (Cortex-M3, 72 MHz)
- **LED pins**: PC2 and PC3 (push-pull output)

### Build with Keil MDK

1. Open `tinyrtos.uvprojx` in Keil uVision
2. Build the project (F7)
3. Flash to the target board
4. Two LEDs on PC2 and PC3 will blink at different rates

### Porting to Another STM32

The scheduler core (`OS/` folder) is MCU-independent within the Cortex-M3 family. To port:

1. Replace `CMSIS/` with the CMSIS files for your target MCU
2. Update the startup file in `STARTUP/`
3. Adjust `CONFIG_SYSCLK` in `os.c` to match your clock frequency
4. Modify `USER/led.c` for your GPIO pins

## API Reference

### `os_init()`

Initializes the scheduler and creates the idle task. Call this first.

### `os_start()`

Starts the SysTick timer and enables PendSV. Call this after all tasks are created. This function does not return.

### `create_task(function, arguments, stack, stack_size)`

Creates a new task and adds it to the scheduler queue.

| Parameter | Type | Description |
|-----------|------|-------------|
| `function` | `void*` | Task entry function (must be an infinite loop) |
| `arguments` | `void*` | Argument passed to the task via R0 (can be `0`) |
| `stack` | `stack_t*` | Pointer to the stack array |
| `stack_size` | `uint16_t` | Size of the stack array in `uint32_t` words |
| **Returns** | `uint16_t` | Task ID (0 on failure) |

### Usage Example

```c
#include "os.h"
#include "task.h"

#define STACK_SIZE 256
stack_t stack_task1[STACK_SIZE];

void my_task() {
    for (;;) {
        // your task logic here
    }
}

int main(void) {
    os_init();

    // configure your peripherals
    gpio_init();

    create_task(my_task, 0, stack_task1, STACK_SIZE);

    os_start();  // never returns
}
```

## Project Structure

```
.
├── OS/
│   ├── os.c              # Scheduler core — SysTick, PendSV init
│   ├── os.h              # Scheduler API header
│   ├── task.c            # Task management — create, switch, round-robin logic
│   ├── task.h            # TCB structure and task API
│   └── task.s            # PendSV_Handler — assembly context switch
├── USER/
│   ├── main.c            # Demo application — two blinking LED tasks
│   ├── led.c             # GPIO configuration for PC2 and PC3
│   └── led.h             # LED header
├── CMSIS/                # CMSIS core and STM32F10x device files
├── STARTUP/              # Startup assembly (stm32f10x_hd)
├── RTE/                  # Keil Run-Time Environment configuration
├── Objects/              # Build output (.o, .hex, .axf)
├── Listings/             # Listing files (.lst, .map)
└── tinyrtos.uvprojx      # Keil MDK project file
```

## Configuration

All configuration is in `os.c`:

| Macro | Default | Description |
|-------|---------|-------------|
| `CONFIG_SYSCLK` | 72 | System clock in MHz |
| `CONFIG_OS_SYSCLK_DIV` | 8 | SysTick clock divider (HCLK/8) |
| `CONFIG_OS_TICK_TIME_US` | 10000 | Tick period in microseconds (10 ms) |

In `task.c`:

| Macro | Default | Description |
|-------|---------|-------------|
| `CONFIG_MAX_TASK_NUM` | 8 | Maximum number of tasks (including idle) |
