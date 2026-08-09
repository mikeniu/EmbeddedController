# Framework Laptop EC Firmware Code Wiki

## 目录

- [1. 项目概述](#1-项目概述)
- [2. 整体架构](#2-整体架构)
- [3. 主要模块职责](#3-主要模块职责)
- [4. 关键数据结构与函数](#4-关键数据结构与函数)
- [5. 依赖关系图](#5-依赖关系图)
- [6. 构建系统详解](#6-构建系统详解)
- [7. 项目运行方式](#7-项目运行方式)
- [8. Framework Laptop 定制化](#8-framework-laptop-定制化)
- [9. 测试体系](#9-测试体系)
- [10. 调试与故障排查](#10-调试与故障排查)

---

## 1. 项目概述

### 1.1 项目简介

本项目是 **Framework Laptop 嵌入式控制器 (Embedded Controller, EC)** 的固件源代码，基于 Google Chromium OS EC 项目进行定制开发。EC 负责管理笔记本电脑的底层功能，包括：

- 系统电源时序控制 (Power Sequencing)
- 键盘扫描与处理
- 电池充放电管理
- 热管理与风扇控制
- 主机 (AP) 与 EC 之间的通信
- 各种外设 (I2C/SPI/UART/GPIO) 的控制
- USB-C / PD 协议处理

### 1.2 硬件背景

Framework Laptop EC 使用的芯片是 **Microchip MEC1521H-B0-I-SZ** (WFBGA144 封装)：
- CPU 内核：ARM Cortex-M4
- RAM：256KB
- 外部 SPI Flash：512KB
- 调试接口：2 线 SWD (Serial Wire Debug)

### 1.3 支持的主板

| 笔记本型号 | Board 名称 | 芯片配置 | Baseboard |
|-----------|-----------|---------|-----------|
| Intel 11th Gen | `hx20` | MEC152x | fwk |
| Intel 12th Gen | `hx30` | MEC152x_3400 | fwk |
| Intel 13th Gen | `hx30` | MEC152x_3400 | fwk |

### 1.4 代码许可

本项目基于 BSD 许可证开源，详情参见 [LICENSE](file:///workspace/LICENSE)。

---

## 2. 整体架构

### 2.1 架构分层模型

```
┌─────────────────────────────────────────────────────────┐
│                   Application Layer                     │
│  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌──────────┐  │
│  │  Tasks  │  │  Hooks   │  │  Deferred │  │ Console │  │
│  └─────────┘  └──────────┘  └──────────┘  └──────────┘  │
├─────────────────────────────────────────────────────────┤
│                   Common Services                       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐ │
│  │  Charger │ │ Thermal  │ │ Keyboard │ │ Host Comm  │ │
│  └──────────┘ └──────────┘ └──────────┘ └────────────┘ │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐ │
│  │  Sensor  │ │  USB PD  │ │  Power   │ │  Flash     │ │
│  └──────────┘ └──────────┘ └──────────┘ └────────────┘ │
├─────────────────────────────────────────────────────────┤
│                   Driver Layer                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Accel/Gyro  │  │  Temp Sensor │  │  ALS/Baro    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Charger IC  │  │  TCPC/Retimer│  │  USB Mux/PPC │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
├─────────────────────────────────────────────────────────┤
│                   Board / Baseboard                     │
│  ┌────────────────────────────────────────────────┐     │
│  │  board/hx30 + baseboard/fwk (Framework 定制)   │     │
│  └────────────────────────────────────────────────┘     │
├─────────────────────────────────────────────────────────┤
│                   Chip Abstraction                      │
│  ┌────────────────────────────────────────────────┐     │
│  │  chip/mchp (MEC152x Cortex-M4 外设驱动)         │     │
│  └────────────────────────────────────────────────┘     │
├─────────────────────────────────────────────────────────┤
│                   Core / CPU                            │
│  ┌────────────────────────────────────────────────┐     │
│  │  core/cortex-m (任务调度、中断、MPU、异常)      │     │
│  └────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────┘
```

### 2.2 目录结构总览

| 目录 | 说明 |
|-----|------|
| [board/](file:///workspace/board) | 各开发板/产品的板级配置 (约 178 个 board) |
| [baseboard/](file:///workspace/baseboard) | 共享的基础板级代码 (dedede, fwk, hatch 等) |
| [chip/](file:///workspace/chip) | 芯片级外设驱动 (mchp, stm32, npcx, it83xx 等) |
| [core/](file:///workspace/core) | CPU 内核层 (cortex-m, cortex-m0, host 等) |
| [common/](file:///workspace/common) | 跨板级共享的上层功能模块 |
| [driver/](file:///workspace/driver) | 片外器件驱动 (传感器、充电 IC、TCPC 等) |
| [power/](file:///workspace/power) | AP 芯片组电源时序管理 |
| [include/](file:///workspace/include) | 全局头文件 |
| [builtin/](file:///workspace/builtin) | 独立 libc 替代实现 (无 OS 环境) |
| [util/](file:///workspace/util) | 主机端工具 (ectool, flash_ec 等) |
| [test/](file:///workspace/test) | 单元测试源码 |
| [fuzz/](file:///workspace/fuzz) | Fuzz 测试源码 |
| [docs/](file:///workspace/docs) | 开发文档 |

### 2.3 启动流程 (Boot Flow)

```
Power On Reset
    │
    ▼
┌─────────────────┐
│  Boot-ROM (ROM) │  ── 芯片内置引导程序，校验 SPI Flash
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  LFW (Loader)   │  ── chip/mchp/lfw/ec_lfw.c，加载 RO 固件
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  RO 固件初始化  │  ── core/cortex-m/init.S → main()
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ HOOK_INIT 钩子  │  ── 按优先级顺序调用各模块 init 函数
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  任务调度启动   │  ── 进入 task_start()，最高优先级任务先运行
└─────────────────┘
```

SPI Flash 布局 (hx30)：
```
00000000:00000fff  bootsector (引导扇区)
00001000:00039fff  lfwro      (LFW + RO 区域)
00040000:00078fff  rw         (RW 区域)
```

---

## 3. 主要模块职责

### 3.1 Board 层 - [board/hx30/](file:///workspace/board/hx30)

Board 层定义了具体产品的硬件配置，是最顶层的定制层。

| 文件 | 职责 |
|-----|------|
| [build.mk](file:///workspace/board/hx30/build.mk) | 指定 CHIP=mchp, BASEBOARD=fwk, CHIP_VARIANT=mec152x_3400，以及 board 级编译目标 |
| [board.h](file:///workspace/board/hx30/board.h) | 硬件功能开关宏定义 (键盘定制、UART、eSPI、PECI 等) |
| [board.c](file:///workspace/board/hx30/board.c) | GPIO 表、I2C 端口映射、板级初始化函数 |
| [gpio.inc](file:///workspace/board/hx30/gpio.inc) | GPIO 信号定义 (输入/输出/中断/上拉等) |
| [ec.tasklist](file:///workspace/board/hx30/ec.tasklist) | 任务列表及优先级 (见 3.2 节) |
| [led.c](file:///workspace/board/hx30/led.c) | LED 状态指示实现 (电源/电池 LED) |
| [power_sequence.c](file:///workspace/board/hx30/power_sequence.c) | Framework 专用电源时序控制 |
| [ucsi.c](file:///workspace/board/hx30/ucsi.c) | USB Type-C 连接器系统信息 (UCSI) 驱动 |
| [cypress5525.c](file:///workspace/board/hx30/cypress5525.c) | Cypress 触控板 IC 驱动 |
| [keyboard_customization.c](file:///workspace/board/hx30/keyboard_customization.c) | Framework 键盘特殊按键 (Fn 组合键等) |
| [peci_over_espi.c](file:///workspace/board/hx30/peci_over_espi.c) | 通过 eSPI 总线实现 PECI (Platform Environment Control Interface) |
| [cpu_power.c](file:///workspace/board/hx30/cpu_power.c) | CPU 功耗限制动态调整 |
| [temperature.c](file:///workspace/board/hx30/temperature.c) | DTT (Dynamic Thermal Throttling) 温度传感器支持 |
| [host_command_customization.c](file:///workspace/board/hx30/host_command_customization.c) | Framework 自定义主机命令扩展 |

### 3.2 Baseboard 层 - [baseboard/fwk/](file:///workspace/baseboard/fwk)

Framework 系列共享的基础板级代码，被 hx20/hx30 复用。

| 文件 | 职责 |
|-----|------|
| [build.mk](file:///workspace/baseboard/fwk/build.mk) | baseboard 编译目标 (风扇、电池、PS/2 鼠标等) |
| [baseboard.h](file:///workspace/baseboard/fwk/baseboard.h) | Framework 系列通用配置宏 |
| [baseboard.c](file:///workspace/baseboard/fwk/baseboard.c) | 通用板级初始化、GPIO 配置 |
| [battery.c](file:///workspace/baseboard/fwk/battery.c) | 智能电池参数与 SMBus 通信 |
| [battery_extender.c](file:///workspace/baseboard/fwk/battery_extender.c) | 电池延长模式 (充电阈值控制) |
| [fan.c](file:///workspace/baseboard/fwk/fan.c) | Framework 风扇转速控制策略 |
| [thermal.c](file:///workspace/baseboard/fwk/thermal.c) | 基于虚拟温度的风扇控制 |
| [temperature_filter.c](file:///workspace/baseboard/fwk/temperature_filter.c) | 温度传感器数据滤波 (平滑处理) |
| [diagnostics.c](file:///workspace/baseboard/fwk/diagnostics.c) | 硬件诊断功能 |
| [flash_storage.c](file:///workspace/baseboard/fwk/flash_storage.c) | Flash 用户数据持久化存储 |
| [ps2mouse.c](file:///workspace/baseboard/fwk/ps2mouse.c) | PS/2 鼠标仿真 (TrackPoint) |
| [system_serial.c](file:///workspace/baseboard/fwk/system_serial.c) | 序列号读写调试接口 |
| [baseboard_host_commands.c](file:///workspace/baseboard/fwk/baseboard_host_commands.c) | Framework 共享的自定义主机命令 |

### 3.3 任务系统 (Task System)

EC 采用轻量级协作式多任务，无动态内存分配。任务定义在 [ec.tasklist](file:///workspace/board/hx30/ec.tasklist) 中，按优先级从低到高排列：

| 任务名 | 优先级 | 职责 | 核心函数 |
|-------|--------|------|---------|
| HOOKS | 最低 (0) | 钩子回调执行、延迟函数 | `hook_task()` |
| CHARGER | 1 | 电池充放电状态机 | `charger_task()` |
| CHIPSET | 2 | AP 电源状态机、G3/S0/Sx 切换 | `chipset_task()` |
| KEYPROTO | 3 | 键盘协议 (8042/MKBP) 处理 | `keyboard_protocol_task()` |
| HOSTCMD | 4 | 主机命令处理 (LPC/eSPI) | `host_command_task()` |
| CONSOLE | 5 | 串口控制台交互 | `console_task()` |
| POWERBTN | 6 | 电源按键消抖与事件 | `power_button_task()` |
| CYPD | 7 | Cypress 触控板中断处理 | `cypd_interrupt_handler_task()` |
| IICTPEMU | 8 | PS/2 鼠标中断仿真 | `mouse_interrupt_handler_task()` |
| ALS | 9 | 环境光传感器 | `als_task()` |
| HID | 10 | HID 媒体按键处理 | `hid_handler_task()` |
| KEYSCAN | 最高 (11) | 键盘矩阵扫描 | `keyboard_scan_task()` |

**关键 API (见 [include/task.h](file:///workspace/include/task.h))：**
- `task_wait_event(timeout_us)` - 阻塞等待事件或超时
- `task_set_event(tskid, event, wait)` - 向任务发送事件位
- `task_wake(tskid)` - 唤醒任务
- `task_current()` - 获取当前任务 ID

### 3.4 钩子系统 (Hooks) - [common/hooks.c](file:///workspace/common/hooks.c)

钩子系统用于在特定事件发生时按优先级顺序执行注册的回调函数，所有钩子在 HOOKS 任务中执行。

**钩子类型 (见 [include/hooks.h](file:///workspace/include/hooks.h#L61-L120))：**

| Hook 类型 | 触发时机 |
|-----------|---------|
| `HOOK_INIT` | 系统启动时 (任务调度前) |
| `HOOK_TICK` | 周期性 tick (间隔因芯片而异) |
| `HOOK_SECOND` | 每秒 1 次 |
| `HOOK_AC_CHANGE` | AC 电源插入/移除 |
| `HOOK_CHIPSET_STARTUP` | AP 上电启动 |
| `HOOK_CHIPSET_SUSPEND` | AP 进入 suspend |
| `HOOK_CHIPSET_RESUME` | AP 从 suspend 恢复 |
| `HOOK_CHIPSET_SHUTDOWN` | AP 关机 |
| `HOOK_SYSJUMP` | 跨镜像跳转 (RO→RW) |
| `HOOK_FREQ_CHANGE` | 系统时钟频率切换 |

**用法示例：**
```c
DECLARE_HOOK(HOOK_AC_CHANGE, ac_change_callback, HOOK_PRIO_DEFAULT);
```

**延迟函数 (Deferred Functions)：**
```c
DECLARE_DEFERRED(my_deferred_func);
hook_call_deferred(my_deferred_func, 30 * MSEC);  // 30ms 后执行
```

### 3.5 充电管理 - [common/charge_state_v2.c](file:///workspace/common/charge_state_v2.c)

充电状态机管理电池充放电的四种状态：

| 状态 | 说明 |
|-----|------|
| `ST_IDLE` | AC 断开，无动作 |
| `ST_DISCHARGE` | 电池放电 (系统运行) |
| `ST_PRECHARGE` | 电池电压过低时预充 (小电流) |
| `ST_CHARGE` | 正常充电 (恒流/恒压) |

**核心数据结构** (见 [include/charge_state_v2.h](file:///workspace/include/charge_state_v2.h#L33-L52))：
```c
struct charge_state_data {
    timestamp_t ts;
    int ac;                          // AC 是否连接
    int batt_is_charging;            // 电池正在充电标志
    struct charger_params chg;       // 充电器参数
    struct batt_params batt;         // 电池参数 (电压/电流/温度/SOC)
    enum charge_state_v2 state;      // 当前状态
    int requested_voltage;           // 请求充电电压 (mV)
    int requested_current;           // 请求充电电流 (mA)
    int desired_input_current;       // 期望输入电流限制
};
```

### 3.6 热管理 - [common/thermal.c](file:///workspace/common/thermal.c)

热管理模块周期性读取温度传感器，根据阈值触发温控动作：

| 温度阈值 | 动作 |
|---------|------|
| `temp_warn` | 警告温度，上报 AP |
| `temp_low` | 启动风扇最低转速 |
| `temp_high` | 风扇全速运行 |
| `temp_critical` | 触发 AP 关机，保护硬件 |
| `temp_ec_critical` | EC 自身过温复位 |

**辅助函数** (见 [include/thermal.h](file:///workspace/include/thermal.h#L20))：
- `thermal_fan_percent(low, high, cur)` - 线性计算当前温度对应的风扇转速百分比
- `board_override_fan_control(fan, tmp)` - 板子自定义风扇控制覆盖 (Framework 在 [baseboard/fwk/fan.c](file:///workspace/baseboard/fwk/fan.c) 中实现)

### 3.7 主机命令接口 - [common/host_command.c](file:///workspace/common/host_command.c)

AP (主 CPU) 通过 LPC/eSPI 总线向 EC 发送命令，命令 ID 定义在 [include/ec_commands.h](file:///workspace/include/ec_commands.h) 中。

**关键数据结构** (见 [include/host_command.h](file:///workspace/include/host_command.h#L16-L59))：
```c
struct host_cmd_handler_args {
    void (*send_response)(struct host_cmd_handler_args *args);
    uint16_t command;      // 命令号 (EC_CMD_*)
    uint8_t version;       // 命令版本 (0-31)
    const void *params;    // 输入参数
    uint16_t params_size;  // 参数大小
    void *response;        // 响应缓冲区
    uint16_t response_max; // 响应缓冲区最大大小
    uint16_t response_size;// 实际响应大小
    uint16_t result;       // EC_RES_* 状态码
};
```

**常用命令示例：**
| 命令 ID | 功能 |
|--------|------|
| `EC_CMD_PROTOCOL_VERSION` | 查询协议版本 |
| `EC_CMD_HELLO` | 连通性测试 (ping) |
| `EC_CMD_GET_VERSION` | 获取 EC RO/RW 版本字符串 |
| `EC_CMD_READ_BATTERY` | 读取电池信息 |
| `EC_CMD_CHARGE_STATE` | 获取充电状态 |
| `EC_CMD_THERMAL_GET_THRESHOLD` | 获取温度阈值配置 |
| `EC_CMD_PWM_GET_FAN_TARGET_RPM` | 获取风扇目标转速 |
| `EC_CMD_MKBP_STATE` | MKBP 事件 (键盘/按键) |
| `EC_CMD_FLASH_ERASE` / `EC_CMD_FLASH_WRITE` | 读写 SPI Flash |
| `EC_CMD_CONSOLE_SNAPSHOT` | 获取控制台日志快照 |

### 3.8 USB-C / Power Delivery 模块

USB-C 和 PD 协议栈是 EC 中最复杂的模块之一，包含多层状态机：

| 层级 | 代码路径 | 职责 |
|-----|---------|------|
| TCPC 驱动 | [driver/tcpm/](file:///workspace/driver) | Type-C Port Controller 底层通信 (FUSB302, PS8xxx 等) |
| TC 状态机 | [include/usb_tc_sm.h](file:///workspace/include/usb_tc_sm.h) | Type-C 连接状态 ( unattached → attached，Source/Sink 角色) |
| PE 状态机 | [include/usb_pe_sm.h](file:///workspace/include/usb_pe_sm.h) | Policy Engine，PD 协商过程 (GoodCRC / Accept / PS_RDY) |
| PRL 层 | [include/usb_prl_sm.h](file:///workspace/include/usb_prl_sm.h) | Protocol Layer，CC 线物理消息收发 |
| DPM 策略 | [include/usb_pd_dpm.h](file:///workspace/include/usb_pd_dpm.h) | Device Policy Manager，板级策略 (RDO/电压电流请求) |
| PPC 驱动 | [driver/ppc/](file:///workspace/driver) | Power Path Controller，过流保护/充电路径切换 |
| Retimer 驱动 | [driver/retimer/](file:///workspace/driver) | 信号重定时器 (USB3/DP Alt Mode) |
| USB Mux | [driver/usb_mux/](file:///workspace/driver) | USB 数据通路多路复用 (USB2/USB3/DP) |

### 3.9 芯片组电源时序 - [power/](file:///workspace/power)

AP 芯片组 (Intel x86 / ARM) 的上电/下电时序控制，每个芯片组系列有独立实现：

| 文件 | 芯片组 |
|-----|-------|
| [intel_x86.c](file:///workspace/power/intel_x86.c) + [cometlake.c](file:///workspace/power/cometlake.c) | Intel Comet Lake (10代) |
| [intel_x86.c](file:///workspace/power/intel_x86.c) + [cannonlake.c](file:///workspace/power/cannonlake.c) | Intel Cannon Lake |
| [intel_x86.c](file:///workspace/power/intel_x86.c) + [icelake.c](file:///workspace/power/icelake.c) | Intel Ice Lake |
| [mt8183.c](file:///workspace/power/mt8183.c) | MediaTek MT8183 (ARM) |
| [rk3399.c](file:///workspace/power/rk3399.c) | Rockchip RK3399 (ARM) |
| [sc7180.c](file:///workspace/power/sc7180.c) | Qualcomm SC7180 (ARM) |
| [board/hx30/power_sequence.c](file:///workspace/board/hx30/power_sequence.c) | **Framework 定制 x86 电源时序** |

x86 电源状态机的主要状态：
- `POWER_G3` - 机械断电 (只有 RTC 供电)
- `POWER_S5` - 软关机 (EC/待机电源运行)
- `POWER_S3` - 睡眠 (Suspend to RAM)
- `POWER_S0` - 正常运行

### 3.10 Chip 层 - [chip/mchp/](file:///workspace/chip/mchp)

Microchip MEC152x 芯片的外设寄存器级驱动：

| 模块文件 | 功能 |
|---------|------|
| [clock.c](file:///workspace/chip/mchp/clock.c) | 时钟树配置 (PLL、外设时钟门控) |
| [gpio.c](file:///workspace/chip/mchp/gpio.c) | GPIO 输入/输出/中断配置 |
| [uart.c](file:///workspace/chip/mchp/uart.c) | UART 串口收发 (console 用) |
| [i2c.c](file:///workspace/chip/mchp/i2c.c) | I2C 主机控制器 (电池、传感器用) |
| [spi.c](file:///workspace/chip/mchp/spi.c) / [qmspi.c](file:///workspace/chip/mchp/qmspi.c) | SPI 控制器 (外部 Flash) |
| [hwtimer.c](file:///workspace/chip/mchp/hwtimer.c) | 硬件定时器 (us 级高精度) |
| [lpc.c](file:///workspace/chip/mchp/lpc.c) | LPC 总线 (传统 x86 Host 命令) |
| [espi.c](file:///workspace/chip/mchp/espi.c) | eSPI 总线 (现代 x86 Host 命令，Framework 用) |
| [pwm.c](file:///workspace/chip/mchp/pwm.c) | PWM 输出 (风扇、键盘背光) |
| [fan.c](file:///workspace/chip/mchp/fan.c) | 风扇转速测量 (Tach 输入) |
| [adc.c](file:///workspace/chip/mchp/adc.c) | ADC 模数转换 (温度等) |
| [peci.c](file:///workspace/chip/mchp/peci.c) | PECI (CPU 温度读取) |
| [system.c](file:///workspace/chip/mchp/system.c) | 复位、系统时钟、唯一 ID |
| [watchdog.c](file:///workspace/chip/mchp/watchdog.c) | 看门狗定时器 (防死机) |
| [flash.c](file:///workspace/chip/mchp/flash.c) | SPI Flash 擦写 |
| [dma.c](file:///workspace/chip/mchp/dma.c) | DMA 控制器 (I2C/SPI 加速) |

### 3.11 Core 层 - [core/cortex-m/](file:///workspace/core/cortex-m)

ARM Cortex-M4 内核级代码，提供任务调度、中断、异常处理：

| 文件 | 功能 |
|-----|------|
| [init.S](file:///workspace/core/cortex-m/init.S) | 汇编入口：设置栈、BSS 清零、跳转 main |
| [task.c](file:///workspace/core/cortex-m/task.c) | **任务调度核心**：就绪表、上下文切换、栈检查 |
| [switch.S](file:///workspace/core/cortex-m/switch.S) | 汇编：任务上下文保存/恢复 (PendSV 异常) |
| [vecttable.c](file:///workspace/core/cortex-m/vecttable.c) | 中断向量表 |
| [cpu.c](file:///workspace/core/cortex-m/cpu.c) | CPU 特性初始化 (FPU、Cache、MPU) |
| [mpu.c](file:///workspace/core/cortex-m/mpu.c) | Memory Protection Unit (栈溢出保护) |
| [panic.c](file:///workspace/core/cortex-m/panic.c) | HardFault / BusFault 异常处理 (保存寄存器) |
| [ec.lds.S](file:///workspace/core/cortex-m/ec.lds.S) | 链接脚本 (定义 ROM/RAM 段布局) |
| [watchdog.c](file:///workspace/core/cortex-m/watchdog.c) | 看门狗喂狗钩子 |

---

## 4. 关键数据结构与函数

### 4.1 通用宏与工具 - [include/common.h](file:///workspace/include/common.h)

```c
/* 寄存器访问宏 */
#define REG32(addr)  (*(volatile uint32_t *)(addr))
#define REG16(addr)  (*(volatile uint16_t *)(addr))
#define REG8(addr)   (*(volatile uint8_t  *)(addr))

/* 属性宏 */
#define __aligned(n)  __attribute__((aligned(n)))
#define __packed      __attribute__((packed))
#define __unused      __attribute__((unused))
#define __maybe_unused __attribute__((unused))

/* 编译期断言 */
#define BUILD_ASSERT(cond)  _Static_assert(cond, #cond)

/* 宏拼接 */
#define CONCAT2(w, x)  CONCAT_STAGE_1(w, x, , )
```

### 4.2 时间戳与定时器 - [include/timer.h](file:///workspace/include/timer.h)

```c
typedef struct {
    uint64_t val;  // 微秒级单调时间戳
} timestamp_t;

timestamp_t get_time(void);          // 获取当前时间
void usleep(uint64_t us);            // 忙等待 (不可在任务中常用)
void udelay(uint64_t us);            // 微秒级忙等待 (中断中可用)
int timestamp_expired(timestamp_t deadline, timestamp_t *now);  // 检查超时
```

### 4.3 GPIO 系统 - [include/gpio.h](file:///workspace/include/gpio.h)

```c
/* GPIO 信号枚举 (在 gpio.inc 中用 GPIO 宏定义) */
enum gpio_signal;

int gpio_get_level(enum gpio_signal signal);        // 读电平
void gpio_set_level(enum gpio_signal signal, int value);  // 写电平
void gpio_enable_interrupt(enum gpio_signal signal);       // 使能中断
void gpio_disable_interrupt(enum gpio_signal signal);      // 禁能中断
int gpio_get_flags(enum gpio_signal signal);               // 获取配置标志

/* gpio.inc 中的定义语法 */
// GPIO(INPUT,  GPIO_SIGNAL_NAME,  PIN,  INT_WAKE_FLAGS)
// GPIO(OUTPUT, GPIO_SIGNAL_NAME,  PIN,  DEFAULT_LEVEL)
```

### 4.4 I2C 主机 - [include/i2c.h](file:///workspace/include/i2c.h)

```c
/* 8-bit 从机地址 + 寄存器读/写 */
int i2c_xfer(int port, uint16_t slave_addr,
             const uint8_t *out, int out_size,
             uint8_t *in, int in_size, int flags);

/* 便捷 API：读 8/16 位寄存器 */
int i2c_read8(int port, uint16_t slave_addr, int reg, int *data);
int i2c_read16(int port, uint16_t slave_addr, int reg, int *data);
int i2c_write8(int port, uint16_t slave_addr, int reg, int data);
int i2c_write16(int port, uint16_t slave_addr, int reg, int data);
```

### 4.5 队列 (生产者-消费者) - [include/queue.h](file:///workspace/include/queue.h)

无锁环形缓冲区，常用于任务间数据传递 (如键盘扫描码队列)：

```c
struct queue {
    uint32_t size;         // 单元数量 (2 的幂)
    uint32_t unit_bytes;   // 每单元大小
    uint32_t head;         // 写入位置
    uint32_t tail;         // 读取位置
    uint8_t *buf;
};

void queue_init(struct queue *q, void *buf, uint32_t size, uint32_t unit_bytes);
int queue_add_unit(struct queue *q, const void *unit);   // 入队
int queue_remove_unit(struct queue *q, void *unit);      // 出队
int queue_count(const struct queue *q);                  // 当前元素数
```

### 4.6 控制台命令注册 - [include/console.h](file:///workspace/include/console.h)

```c
/* 注册控制台命令，在 console 任务中解析执行 */
#define DECLARE_CONSOLE_COMMAND(name, routine, arghelp, help)

/* 范例 (在串口输入 'battery' 即调用) */
static int cmd_battery(int argc, char **argv) {
    ccprintf("Battery SOC: %d%%\n", battery_state_of_charge());
    return 0;
}
DECLARE_CONSOLE_COMMAND(battery, cmd_battery, NULL, "Display battery info");
```

### 4.7 主机命令注册 - [common/host_command.c](file:///workspace/common/host_command.c)

```c
/* 注册一个主机命令处理函数 */
#define DECLARE_HOST_COMMAND(command, routine, version_mask)

/* 处理返回值：EC_SUCCESS, EC_RES_ERROR, EC_RES_INVALID_PARAM 等 */
typedef enum ec_status (*host_command_handler_t)(struct host_cmd_handler_args *args);
```

### 4.8 电池参数 - [include/battery.h](file:///workspace/include/battery.h)

```c
struct batt_params {
    int flags;              // 状态标志 (满充/放空/支持标志)
    int state_of_charge;    // 剩余容量百分比 (0-100)
    int voltage;            // 当前电压 (mV)
    int current;            // 当前电流 (mA，负为放电)
    int desired_voltage;    // 期望充电电压 (mV)
    int desired_current;    // 期望充电电流 (mA)
    int temperature;        // 电池温度 (0.1°C 单位)
    int remaining_capacity; // 剩余容量 (mAh)
    int full_capacity;      // 满充容量 (mAh)
};

const struct batt_params *battery_get_params(void);
int battery_state_of_charge(void);
```

---

## 5. 依赖关系图

### 5.1 构建依赖链

对于 Framework hx30，构建时的 Makefile 包含顺序：

```
Makefile
  └─ include board/hx30/build.mk      # CHIP=mchp, BASEBOARD=fwk
       └─ include baseboard/fwk/build.mk   # baseboard 编译目标
       └─ include chip/mchp/build.mk       # CORE=cortex-m
            └─ include core/cortex-m/build.mk   # 内核级编译选项
  └─ include common/build.mk           # 公共模块 (按 CONFIG_*)
  └─ include driver/build.mk           # 片外驱动 (按 CONFIG_*)
  └─ include power/build.mk            # 电源时序 (按 CONFIG_CHIPSET_*)
  └─ include test/build.mk / fuzz/     # 测试目标
  └─ include util/build.mk             # 主机端工具
  └─ include Makefile.toolchain        # 交叉编译工具链配置
  └─ include Makefile.rules            # 通用构建规则与命令
```

### 5.2 模块运行时依赖

```
main() ──→ HOOK_INIT ──→ task_start()
             │
             ├─ I2C / SPI / UART / GPIO 控制器初始化
             ├─ 电池 / 充电器 IC 初始化 (I2C 通信)
             ├─ 温度传感器初始化
             ├─ 芯片组电源状态初始化
             │
任务层:
  CHARGER ──→ charger_get_status() ──→ [I2C] ──→ charger IC
           ├─ battery_get_params() ──→ [I2C] ──→ 智能电池
           └─ hook_notify(HOOK_AC_CHANGE)

  CHIPSET ──→ power_handle_state() ──→ [GPIO] RSMRST#, SLP_S0#, PCH_PWRBTN#
           └─ hook_notify(HOOK_CHIPSET_*)

  KEYSCAN ──→ keyboard_scan_matrix() ──→ [GPIO 矩阵]
           └─ queue_add(&kb_queue, scancode) ──→ KEYPROTO

  KEYPROTO ──→ [LPC/eSPI 8042] ──→ AP KBC 接口
           └─ [MKBP event] ──→ AP

  HOSTCMD  ──→ host_command_received()
           └─ 查表 → DECLARE_HOST_COMMAND 注册的 handler

  HOOKS    ──→ 执行 HOOK_TICK / HOOK_SECOND 回调
           └─ 执行 hook_call_deferred() 注册的延迟函数
```

### 5.3 Framework 定制模块依赖

```
board/hx30
  ├─ 依赖 baseboard/fwk
  │    ├─ battery_extender.o ──→ common/charge_manager.o
  │    ├─ thermal.o ──→ common/thermal.o + common/fan.o
  │    └─ flash_storage.o ──→ common/flash.o
  │
  ├─ ucsi.o            ──→ (I2C → Cypress CCG5 UCSI 控制器)
  ├─ cypress5525.o     ──→ (I2C + GPIO 中断 → Cypress 触控板)
  ├─ peci_over_espi.o  ──→ chip/mchp/espi.o → CPU PECI 通道
  └─ power_sequence.o  ──→ power/intel_x86.o → 标准 x86 状态机
```

---

## 6. 构建系统详解

### 6.1 Makefile 变量

在顶层 [Makefile](file:///workspace/Makefile) 中定义的关键变量：

| 变量 | 用途 | 默认值 |
|-----|------|--------|
| `BOARD` | 选择目标板 | `hx30` |
| `CROSS_COMPILE_arm` | ARM 交叉编译前缀 | 自动检测 `arm-none-eabi-` |
| `V` | 构建输出详细度：空/0=简洁，1=完整 | 空 |
| `out` | 构建输出目录 | `build/$(BOARD)` |
| `ARCH` | 主机架构 (影响 fuzz) | `amd64` |
| `PEM` | 签名密钥文件 | `board/$(BOARD)/dev_key.pem` |
| `TEST_ASAN/MSAN/UBSAN` | 启用对应 sanitizer (host 测试用) | 空 |

### 6.2 条件编译机制

**`*-y` 对象列表：**
```makefile
# board/hx30/build.mk
board-y = board.o led.o power_sequence.o
board-$(CONFIG_PECI) += peci_customization.o peci_over_espi.o
board-$(HAS_TASK_HOSTCMD) += host_command_customization.o
```

- `*-y` 总是编译
- `*-$(CONFIG_FOO)` 当配置项为 `y` 时编译
- `*-$(HAS_TASK_FOO)` 当任务 FOO 在 ec.tasklist 中声明时编译

**配置传播链：**
1. `board/hx30/board.h` 中 `#define CONFIG_FOO`
2. Makefile 通过 CPP 预处理器扫描生成 `.config`
3. `.config` 中 `CONFIG_FOO=y` 被 Makefile 导入
4. `board-$(CONFIG_FOO)+=bar.o` 展开为 `board-y+=bar.o`

### 6.3 构建产物

运行 `make BOARD=hx30` 后的输出目录结构：

```
build/hx30/
  ├─ ec.bin                 # ★ 最终烧录镜像 (RO+RW+LFW 打包)
  ├─ RO/
  │   ├─ ec.RO.elf          # RO 段 ELF (可反汇编)
  │   ├─ ec.RO.flat         # RO 段二进制
  │   ├─ ec.RO.map          # RO 段符号映射
  │   └─ common/ chip/ board/ ...  # RO 目标文件
  ├─ RW/
  │   ├─ ec.RW.elf
  │   ├─ ec.RW.flat
  │   ├─ ec.RW.bin
  │   └─ ...
  ├─ gen/
  │   ├─ ec_version.h       # 自动生成版本字符串
  │   ├─ ec.tasklist.h      # 预处理后的任务列表
  │   └─ board_config.h     # 汇总配置
  └─ .config                # Makefile 可导入的配置项集合
```

### 6.4 MCHP 打包流程 - pack_ec.py

[chip/mchp/util/pack_ec.py](file:///workspace/chip/mchp/util/pack_ec.py) 是 Framework EC 特有的最后一步，将：
1. LFW (Loader Firmware) 镜像
2. RO 固件镜像
3. RW 固件镜像

合并为带校验头、SPI 布局正确的 `ec.bin`，并计算 CRC32 校验和，否则 Boot-ROM 会拒绝启动。

### 6.5 常用 Make 目标

| 命令 | 说明 |
|-----|------|
| `make BOARD=hx30` | 构建默认目标，生成 `build/hx30/ec.bin` |
| `make BOARD=hx30 V=1` | 详细输出 (显示完整 gcc 命令行) |
| `make BOARD=hx30 dis` | 生成反汇编 `ec.RO.dis` / `ec.RW.dis` |
| `make BOARD=hx30 flash` | 用 OpenOCD 通过 SWD 烧录 |
| `make BOARD=hx30 clean` | 删除 build/hx30 所有目标文件 |
| `make clobber` | 删除整个 build/ 目录 |
| `make runhosttests -j` | 构建并运行全部 host 单元测试 |
| `make tests BOARD=hx30` | 全部设备端测试 (需烧录) |
| `make runtests` | host 测试 + 设备测试 + fuzz |
| `make coverage` | 生成覆盖率报告 (需 lcov/genhtml) |
| `make runfuzztests` | 运行 fuzz 测试 |
| `make buildall_only` | CI 用：构建所有 board |
| `make print-host-tests` | 列出所有 host 测试名 |

---

## 7. 项目运行方式

### 7.1 开发环境准备

**Ubuntu 系统依赖：**
```bash
sudo apt install gcc-arm-none-eabi libftdi1-dev build-essential pkg-config gawk
sudo apt install openocd          # SWD 烧录
sudo apt install lcov genhtml     # 覆盖率报告 (可选)
```

**交叉编译器验证：**
```bash
arm-none-eabi-gcc --version
# 应输出类似：arm-none-eabi-gcc (GNU Arm Embedded Toolchain ...)
```

### 7.2 构建 Framework hx30 固件

```bash
cd /workspace

# Intel 12th/13th Gen (默认 BOARD=hx30)
make BOARD=hx30 CROSS_COMPILE=arm-none-eabi-

# 产物：build/hx30/ec.bin
```

```bash
# Intel 11th Gen
make BOARD=hx20 CROSS_COMPILE=arm-none-eabi-
# 产物：build/hx20/ec.bin
```

### 7.3 通过 SWD 调试口烧录

Framework 主板右上角有 10 pin JECDB 调试接口 (SWD 2 线模式)：

| Pin | 信号 | Pin | 信号 |
|-----|------|-----|------|
| 1 | EC_VCC_3.3 | 6 | UART_TX |
| 2 | TDI / SWDIO | 7 | UART_RX |
| 3 | TMS / SWCLK | 9 | EC_RESETI |
| 4 | CLK | 10 | GND |
| 5 | TDO | | |

```bash
# 需要 OpenOCD 配置文件 + 调试器 (如 FT2232H/OLIMEX)
make BOARD=hx30 flash
# 或使用脚本
./util/flash_ec --board=hx30 --image=build/hx30/ec.bin
```

> **⚠️ 重要警告 (来自 [README.md](file:///workspace/README.md#L7-L9))：** EC 固件修改不当可能导致系统无法上电、无法启动，甚至损坏主板/电池。此类硬件损坏不在 Framework 保修范围内。特别注意 hx20 的 `0x3C000-0x3FFFF` 和 `0x79000-0x7FFFF` 扇区不要擦除或覆盖。

### 7.4 串口控制台 (EC UART)

通过调试口 Pin6(TX)/Pin7(RX) 连接 USB-TTL 适配器 (波特率通常 115200 8N1)，可获得交互式 shell：

```
> help                # 列出所有命令
> taskinfo            # 任务运行状态、栈使用
  Task Ready Name     Events      Time (s)  StkUsed
     0 R << idle >>   00000000   32.975554  196/256
     1   HOOKS        00000000    0.007835  192/488
     ...
> battery             # 电池状态
> charger             # 充电信息
> thermalget all      # 所有温度传感器
> chan 0              # 只显示 CHAN0 日志
> battfake 50         # 伪装电池 SOC=50% (调试 LED)
> fanduty 50          # 强制风扇 50%
> autofan             # 恢复自动风扇
> panicinfo           # 查看上次 panic 保存的寄存器
> version             # 版本信息
```

### 7.5 运行 Host 端单元测试

```bash
# 运行全部 host 测试 (不需要硬件，在 PC 上模拟)
make runhosttests -j

# 单个测试：先构建后运行
make host-charge_manager            # 构建
./util/run_host_test charge_manager # 运行

# 带 ASAN 的 fuzz 测试
TEST_ASAN=y make runfuzztests
```

### 7.6 使用 ectool (Host 侧访问 EC)

[util/ectool.c](file:///workspace/util/ectool.c) 编译后，在运行的设备上可通过 `/dev/cros_ec` 读取/控制 EC：

```bash
# 构建 host 工具
make util CROSS_COMPILE=
./build/util/ectool version          # 查 EC 版本
./build/util/ectool battery          # 电池信息
./build/util/ectool thermal          # 温度/风扇
./build/util/ectool console          # 读取 EC 控制台缓冲
./build/util/ectool flashinfo        # Flash 信息
./build/util/ectool gpioget <name>   # 读 GPIO
```

---

## 8. Framework Laptop 定制化

### 8.1 为什么需要定制

虽然项目基于 Chromium OS EC 上游，但 Framework Laptop 有许多 Chromebook 不需要的功能，因此维护了 fork：

| 定制功能 | 代码位置 |
|---------|---------|
| UCSI (USB Type-C Connector System Information) 驱动 (非 Chromebook 标准) | [board/hx30/ucsi.c](file:///workspace/board/hx30/ucsi.c) |
| Framework 专用电源时序 (非 Intel RVP 参考设计) | [board/hx30/power_sequence.c](file:///workspace/board/hx30/power_sequence.c) |
| PECI 走 eSPI 通道 (非传统 LPC) | [board/hx30/peci_over_espi.c](file:///workspace/board/hx30/peci_over_espi.c) |
| Cypress 触控板定制 | [board/hx30/cypress5525.c](file:///workspace/board/hx30/cypress5525.c) |
| 键盘 Fn 组合键、媒体键、I2C-HID 媒体按键 | [board/hx30/keyboard_customization.c](file:///workspace/board/hx30/keyboard_customization.c) [board/hx30/i2c_hid_mediakeys.c](file:///workspace/board/hx30/i2c_hid_mediakeys.c) |
| PS/2 鼠标 (TrackPoint 仿真) | [baseboard/fwk/ps2mouse.c](file:///workspace/baseboard/fwk/ps2mouse.c) |
| 电池延长模式 (不充满，保护寿命) | [baseboard/fwk/battery_extender.c](file:///workspace/baseboard/fwk/battery_extender.c) |
| 用户数据 Flash 持久化存储 | [baseboard/fwk/flash_storage.c](file:///workspace/baseboard/fwk/flash_storage.c) |
| 温度滤波 + 虚拟温度热管理 | [baseboard/fwk/temperature_filter.c](file:///workspace/baseboard/fwk/temperature_filter.c) [baseboard/fwk/thermal.c](file:///workspace/baseboard/fwk/thermal.c) |
| Framework 自定义 Host 命令 | [baseboard/fwk/baseboard_host_commands.c](file:///workspace/baseboard/fwk/baseboard_host_commands.c) [board/hx30/host_command_customization.c](file:///workspace/board/hx30/host_command_customization.c) |
| 动态 CPU 功率限制 (PL1/PL2) | [board/hx30/cpu_power.c](file:///workspace/board/hx30/cpu_power.c) |

### 8.2 hx30 配置宏速查 (来自 [board/hx30/board.h](file:///workspace/board/hx30/board.h))

```c
#define CONFIG_KEYBOARD_CUSTOMIZATION          // 启用 Framework 键盘定制
#define CONFIG_8042_AUX                        // 启用 PS/2 AUX (鼠标)
#define CONFIG_KEYBOARD_BACKLIGHT              // 启用键盘背光
#define CONFIG_KEYBOARD_CUSTOMIZATION_COMBINATION_KEY  // Fn 组合键
#define CONFIG_KEYBOARD_SCANCODE_CALLBACK      // 按键扫描写回调
#define CONFIG_BOARD_PRE_INIT                  // JTAG 提前初始化
#define CONFIG_MCHP_JTAG_MODE 0x03             // 2-pin SWD + SWV
#define CONFIG_HOSTCMD_ESPI                    // 走 eSPI 而不是 LPC
#define CONFIG_HOSTCMD_ESPI_EC_MAX_FREQ 20     // eSPI 最大 20MHz
#define CONFIG_HOSTCMD_ESPI_EC_CHAN_BITMAP 0x0F // 四通道全开
#define CONFIG_DTT_SUPPORT                     // 动态温度节流
#define CONFIG_I2C_HID_MEDIAKEYS               // I2C-HID 媒体键
```

---

## 9. 测试体系

### 9.1 Host 模拟器测试 (无需硬件)

测试文件位于 [test/](file:///workspace/test)，每个测试由两部分组成：
- `foo.c` - 测试用 C 代码 (定义 TEST_ASSERT 等断言)
- `foo.tasklist` - 运行该测试时的任务列表 (可包含最小任务集)

**Host 测试构建过程：**
1. 指定 `BOARD=host`，使用 `chip/host/` 模拟芯片层
2. 用 `core/host/` 模拟任务调度 (pthread 或简单轮询)
3. 编译为 Linux x86 可执行文件 `build/host/$test/$test.exe`
4. `util/run_host_test` 执行并检查返回码

**典型 host 测试：**

| 测试名 | 验证内容 |
|-------|---------|
| charge_manager | 充电管理器逻辑 |
| charge_ramp | 输入电流爬升算法 |
| hooks | 钩子/延迟函数机制 |
| kb_8042 | 8042 键盘协议 |
| usb_pd | Type-C PD 协议状态机 |
| sha256 / rsa / aes | 加密算法正确性 |
| vboot | Verified Boot 哈希校验 |
| motion_angle | 运动感应角度计算 |
| stillness_detector | 静止检测算法 |
| compile_time_macros / static_if | 编译期宏正确性 |

### 9.2 设备端测试 (Device Tests)

```bash
# 编译 + 烧录到目标板，然后在板子上手动运行
make test-<test_name> BOARD=hx30
make flash BOARD=hx30
# 在 EC console 输入:
> runtest <test_name>
```

### 9.3 Fuzz 测试 - [fuzz/](file:///workspace/fuzz)

| Fuzzer | 测试对象 |
|--------|---------|
| [host_command_fuzz.c](file:///workspace/fuzz/host_command_fuzz.c) | 主机命令解析 (防畸形输入 crash) |
| [usb_pd_fuzz.c](file:///workspace/fuzz/usb_pd_fuzz.c) | USB PD 消息解析 |
| [usb_tcpm_v2_rev20_fuzz.c](file:///workspace/fuzz/usb_tcpm_v2_rev20_fuzz.c) | TCPCI 寄存器级 PD v2.0 解析 |

运行：
```bash
TEST_ASAN=y make runfuzztests -j
```

### 9.4 CI / Presubmit

本地运行预提交检查：
```bash
# 1. 代码风格 (Linux kernel checkpatch)
./util/presubmit_check.sh

# 2. 配置选项完整性
python3 util/config_option_check.py

# 3. 主机命令一致性
./util/host_command_check.sh

# 4. 全部构建 + 测试
make buildall_only
make runtests
```

---

## 10. 调试与故障排查

### 10.1 Panic 信息分析

当 EC 遇到 HardFault/BusFault，核心会将寄存器写入 persist 区域 (跨复位保留)：

```
Saved panic data: (NEW)
=== HANDLER EXCEPTION: 05 ====== xPSR: 6100001e ===
r0 :00000001 r1 :00000f15 r2 :4003800c r3 :000000ff
r4 :ffffffed r5 :00000799 r6 :0000f370 r7 :00000000
r8 :00000001 r9 :00000003 r10:20002fe0 r11:00000000
r12:00000008 sp :20000fd8 lr :000012e1 pc :0000105e   ← 关键
```

**定位步骤：**
1. 记录 `pc` (程序计数器) 和 `lr` (链接寄存器)
2. 生成反汇编：`make BOARD=hx30 dis`
3. 打开 `build/hx30/RO/ec.RO.dis`，搜索地址 `0000105e` 附近指令
4. 如果是 "Imprecise data bus error"，在 board.h 中临时加：
   ```c
   #define CONFIG_DEBUG_DISABLE_WRITE_BUFFER
   ```
   可获得 Precise 错误 (精确 PC 值，性能会下降)

### 10.2 栈溢出排查

每个任务有固定大小栈 (在 ec.tasklist 中定义)，无 MMU。栈溢出会触发 MPU 保护异常或静默数据损坏。

**检查栈使用：**
```
> taskinfo
  StkUsed 列显示历史最大使用量 / 总大小
  例: 392/488 表示最大用了 392 字节，还剩 96 字节 (余量偏少)
```

如果余量不足，将 ec.tasklist 中的 `TASK_STACK_SIZE` 升级为 `LARGER_TASK_STACK_SIZE` 或 `VENTI_TASK_STACK_SIZE`。

### 10.3 GDB 调试 (SWD)

[util/gdbinit](file:///workspace/util/gdbinit) 提供了 GDB 脚本：
```bash
# 启动 OpenOCD 服务器
openocd -f interface/ftdi/olimex-arm-usb-tiny-h.cfg -f target/at91sam3XXX.cfg

# 另开窗口
arm-none-eabi-gdb build/hx30/RW/ec.RW.elf
(gdb) source util/gdbinit
(gdb) target remote :3333
(gdb) monitor reset halt
(gdb) load
(gdb) b charger_task        # 下断点
(gdb) c                     # 继续运行
(gdb) bt                    # 调用栈
(gdb) p current_task        # 查看当前任务
```

### 10.4 I2C 调试

启用 I2C 追踪宏 (board.h 中)：
```c
#define CONFIG_I2C_DEBUG
```

然后 console 中：
```
> i2c trace <port> on      # 开启该端口跟踪
> i2c xfer <port> <addr> w <data>...     # 手动写
> i2c xfer <port> <addr> r <count>       # 手动读
```

### 10.5 代码风格

遵循 **Linux kernel 风格**：
- 8 字符宽 Tab (非空格)
- 单行 80 列限制
- 内核式命名 (`snake_case`，全局加前缀)
- `.clang-format` 提供 clang-format 配置

自动检查：
```bash
./util/presubmit_check.sh    # 调用 checkpatch.pl
```

---

## 附录 A：核心 Makefile 与配置文件索引

| 文件 | 作用 |
|-----|------|
| [Makefile](file:///workspace/Makefile) | 顶层入口，包含 board/chip/core 子 makefile |
| [Makefile.rules](file:///workspace/Makefile.rules) | 通用编译/链接/打包规则 (`cmd_c_to_o`, `cmd_elf` 等) |
| [Makefile.toolchain](file:///workspace/Makefile.toolchain) | 工具链选择、CFLAGS/LDFLAGS 基础选项 |
| [include/config.h](file:///workspace/include/config.h) | 全局 `CONFIG_*` 宏默认值与注释 |
| [include/compile_time_macros.h](file:///workspace/include/compile_time_macros.h) | `IS_ENABLED()`、`BUILD_ASSERT()` 等编译期宏 |
| [include/ec_commands.h](file:///workspace/include/ec_commands.h) | Host 命令号、请求/响应结构体规范 (与内核共享) |
| [include/task_id.h](file:///workspace/include/task_id.h) | 任务 ID 枚举自动生成 (由 ec.tasklist 派生) |
| [firmware_builder.py](file:///workspace/firmware_builder.py) | CI 构建脚本入口 (Chromium OS buildbot) |

## 附录 B：文档资源 (docs/)

| 文档 | 内容 |
|-----|------|
| [docs/getting_started_quickly.md](file:///workspace/docs/getting_started_quickly.md) | 新手上手指南 |
| [docs/ap-ec-comm.md](file:///workspace/docs/ap-ec-comm.md) | AP-EC 主机通信协议详解 |
| [docs/unit_tests.md](file:///workspace/docs/unit_tests.md) | 单元测试编写说明 |
| [docs/usb-c.md](file:///workspace/docs/usb-c.md) / [docs/usb_power.md](file:///workspace/docs/usb_power.md) | USB-C/PD 开发指南 |
| [docs/case_closed_debugging.md](file:///workspace/docs/case_closed_debugging.md) | CCD 调试 (Servo) |
| [docs/write_protection.md](file:///workspace/docs/write_protection.md) | 固件写保护机制 |
| [docs/code_coverage.md](file:///workspace/docs/code_coverage.md) | 覆盖率报告生成 |
| [docs/new_board_checklist.md](file:///workspace/docs/new_board_checklist.md) | 新增 Board 检查清单 |

---

*本 Code Wiki 基于 `/workspace` 代码库的结构分析自动生成，最后更新: 2026-08-10。*
