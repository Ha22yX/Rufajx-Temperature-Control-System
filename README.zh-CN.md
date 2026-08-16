<div align="center">

# Rufajx 热风机温控系统

**面向工业热风机的温度采集、预测控制与可回读 0–5 V 指令输出控制板。**

[English](README.md) · [系统总览](Docs/README.md) · [硬件说明](Docs/HARDWARE.md) · [引脚定义](Docs/PINOUT.md) · [PCB 说明](Docs/PCB_DESIGN.md) · [固件指南](Docs/FIRMWARE_GUIDE.md)

![MCU](https://img.shields.io/badge/MCU-STM32G0B1CBT6N-03234B?style=flat-square)
![Temperature](https://img.shields.io/badge/Process-200--600%C2%B0C-CB6C3D?style=flat-square)
![Output](https://img.shields.io/badge/Output-0--5V%20DAC-2C7A68?style=flat-square)
![Interfaces](https://img.shields.io/badge/Interfaces-USB%20%2B%20RS--485-5B63B7?style=flat-square)
![Status](https://img.shields.io/badge/Status-Engineering%20validation-D39B32?style=flat-square)

</div>

![Rufajx 温控系统概览](.github/assets/readme-hero.svg)

## 项目用途

本项目为 Rufajx 热风机开发实时闭环温控控制板。绝缘型 K 型热电偶测量当前温度，STM32 根据热惯性、升降温不对称和温度变化趋势执行状态机、预测控制与分段 PI/PID，最终向机器控制器输出经校准的 0–5 V 加热指令。

P2 只输出控制电压，不直接驱动加热器。P4 是另一条独立的常闭继电器触点回路，用于接入危险的 220 VAC 外部线路。

## 当前硬件基线

| 功能 | 当前实现 |
|---|---|
| MCU | STM32G0B1CBT6N，LQFP-48 |
| 温度输入 | MAX6675＋绝缘型/非接地结 K 型热电偶 |
| 控制输出 | PA4 DAC1_OUT1、2.5 V VREFBUF、TLV9351，P2 输出 0–5 V |
| 输出回读 | PA0 ADC 在 R11 后采集 P2 实际端口电压 |
| USB | USB 2.0 Type-C，用于 ROM DFU 与计划中的 USB CDC |
| 现场通信 | THVD1400DR 非隔离半双工 RS-485 |
| 下载调试 | 5 针 SWD、BOOT、RESET |
| 外部切断 | 24 V 线圈、220 VAC 常闭继电器触点 |
| 供电 | 受保护的 24 V 输入；USB/24 V 低压电源防倒灌汇合 |
| 工艺温度 | 约 200–600 °C |

RS-485 的 P5 为 1 脚 B、2 脚 A。本接口没有隔离，依赖机器系统中已经确认的公共地。R22 120 Ω 只能在本板被选为总线两个电气终端之一时装配；被动多分支网络不能在每个分支末端都并联 120 Ω。

## 外部接口

| 接口 | 1 脚 | 2 脚 | 用途 |
|---|---|---|---|
| P1 | K+ | K- | 绝缘型 K 型热电偶 |
| P2 | GND | 0–5V OUT | 可回读的加热控制电压 |
| P3 | GND | +24V | 机器控制电源 |
| P4 | NC (L) | COM (L) | 危险 220 VAC 常闭触点 |
| P5 | B | A | 两线半双工 RS-485 |

H1 依次提供 GND、3V3/VTref、NRST、SWCLK/BOOT0、SWDIO。

## 文档

- [项目与系统总览](Docs/README.md)
- [硬件说明](Docs/HARDWARE.md)
- [MCU 与接口引脚](Docs/PINOUT.md)
- [PCB 与生产说明](Docs/PCB_DESIGN.md)
- [固件开发指南](Docs/FIRMWARE_GUIDE.md)
- [量产前必须修复与验证清单](Docs/PCB_REQUIRED_FIXES.md)

## 仓库结构

- `PCB Design/`：可编辑的 EasyEDA 工程
- `Docs/`：硬件、引脚、PCB、固件和量产放行文档
- `.github/assets/`：仓库展示资源

仓库目前尚未加入固件源码。已确定的生产开发基线为 VS Code＋PlatformIO＋STM32Cube HAL/LL；开发和救砖使用 SWD，系统 ROM USB DFU 用于恢复烧录。

## 当前状态

最新 PCB 在当前 EasyEDA 规则下严格 DRC 为 0。这只能证明符合现有 CAD 规则，不能证明市电隔离、EMC、模拟精度、继电器寿命或整机安全已经合格。

**当前版本尚不能直接批准用于 220 VAC 批量生产。** 已测得的部分市电到 SELV 距离小于项目暂定的保守 8 mm 目标，而且当前规则未完整强制市电隔离走廊。RS-485 端接与拓扑、ESD 泄放回路、0–5 V 校准、真实负载、首板测试、EMC 和独立超温保护也必须形成验证记录。

批量采购前必须完成 [PCB_REQUIRED_FIXES.md](Docs/PCB_REQUIRED_FIXES.md)。

## 安全边界

- 200–600 °C 是被控工艺温度，不是 PCB 的允许环境温度。
- 传感器、参考源、DAC、ADC 回读、看门狗或控制周期异常时，模拟指令必须立即归零。
- 本板掉电会使 P4 常闭触点保持闭合，而不是自动切断 220 VAC。
- 整机必须配置独立超温保护和额定值合适的加热电源切断路径。
- 普通台架调试期间不得接入市电。

## 所有权

本项目为 **Rufajx** 定制开发，用于工业热风机实时温度控制。除非项目所有者另行公布许可条款，否则保留所有权利。
