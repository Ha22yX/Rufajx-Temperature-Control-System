<div align="center">

# Rufajx 温控系统

**用于热风机加热温度实时测量与比例调节的定制控制板。**

[English](README.md) · [硬件说明](Docs/HARDWARE.md) · [引脚定义](Docs/PINOUT.md) · [PCB 状态](Docs/PCB_DESIGN.md) · [固件指南](Docs/FIRMWARE_GUIDE.md)

![STM32C071](https://img.shields.io/badge/MCU-STM32C071-03234B?style=flat-square)
![Temperature](https://img.shields.io/badge/Target-200--600%C2%B0C-CB6C3D?style=flat-square)
![Sensor](https://img.shields.io/badge/Sensor-K--Type%20%2B%20MAX6675-2C7A68?style=flat-square)
![EDA](https://img.shields.io/badge/EDA-EasyEDA-5B63B7?style=flat-square)
![Status](https://img.shields.io/badge/Status-PCB%20validation%20required-D39B32?style=flat-square)

</div>

![项目概览图](.github/assets/readme-hero.svg)

## 项目背景

这个项目是为 Rufajx——用户父亲的公司——开发的热风机实时温控系统。K 型热电偶测量热风机的当前温度，MCU 负责快速升温、PID 调节和温度惯性处理，最后向机器的外部加热控制器输出经过滤波的 0–3.3 V 比例控制信号。

本板负责温度测量与控制，不直接开关加热器功率。实际设备还必须配合具备独立保护功能的功率驱动级。

## 已实现的硬件功能

- 使用 MAX6675 对 K 型热电偶进行冷端补偿和数字转换。
- 使用 STM32C071G8U6 执行采样、安全状态机、PID、PWM 和 USB 通信。
- MCU PWM 经过 RC 滤波后输出 0–3.3 V 比例信号。
- USB-C 支持 ROM DFU 恢复，并计划用于 USB CDC 命令和调试输出。
- 支持 USB 5 V 或经过保护的 24 V 控制电源供电。
- 带有 BOOT 和 RESET 按钮，方便首次烧录与故障恢复。
- 硬件目标覆盖约 200–600 °C 的热风加热工艺。

## 硬件组成

| 层级 | 器件 | 用途 |
|---|---|---|
| 温度输入 | K 型热电偶 + MAX6675 | 带冷端补偿的数字温度采集 |
| 控制核心 | STM32C071G8U6 | 采样、安全逻辑、PID、PWM、USB |
| 比例输出 | PA8 PWM + 1 kΩ / 10 µF RC 滤波 | 高阻负载下的 0–3.3 V 控制信号 |
| 维护接口 | USB-C + USBLC6-2SC6 | DFU 烧录和 USB CDC 调试 |
| 现场供电 | 24 V 保护 + K7805-500R3 | 将机器控制电源转换为 5 V |
| 逻辑供电 | ME6211C33M5G-N | 产生 3.3 V 电源 |
| 设计源文件 | EasyEDA `.eprj2` | 可编辑的原理图与 PCB 工程 |

## 仓库结构

```text
.
├── PCB Design/
│   └── Rufajx温控系统.eprj2       # EasyEDA 工程
├── Docs/
│   ├── README.md                 # 英文项目总览
│   ├── HARDWARE.md               # 电路模块与器件说明
│   ├── PINOUT.md                 # MCU 和接线端子引脚定义
│   ├── PCB_DESIGN.md             # 布局说明与投板检查表
│   └── FIRMWARE_GUIDE.md         # PlatformIO、DFU、PID 与调试指南
└── .github/assets/
    └── readme-hero.svg           # 仓库概览图
```

## 打开设计

1. 安装并打开 EasyEDA。
2. 打开 `PCB Design/Rufajx温控系统.eprj2`。
3. 查看[引脚定义](Docs/PINOUT.md)与[硬件说明](Docs/HARDWARE.md)。
4. 在生产前完成 [PCB 投板检查表](Docs/PCB_DESIGN.md)中的所有阻断项。

仓库目前还没有加入固件源码。[FIRMWARE_GUIDE.md](Docs/FIRMWARE_GUIDE.md)记录了计划采用的 PlatformIO + Arduino 固件架构，但不宣称当前已经存在可编译工程。

## 外部接线

| 接口 | 1 脚 | 2 脚 | 功能 |
|---|---|---|---|
| P1 | `K+` | `K-` | K 型热电偶输入 |
| P2 | `GND` | `OUT` | 经过滤波的 0–3.3 V 比例输出 |
| P3 | `GND` | `+24V` | 机器控制电源输入 |

## 当前验证状态

PCB 已经完成布局，但还没有达到可直接投板的状态。最近一次 EasyEDA DRC 发现：

- USB-C 封装中有两处焊盘到槽孔的间距为 0.171 mm，小于当前 0.18 mm 规则。
- 原理图与 PCB 网表不一致。
- 部分 BOM 项目还没有确认有效的嘉立创商城编号。

这些都是投板前必须处理的问题，已经记录在 [PCB_DESIGN.md](Docs/PCB_DESIGN.md) 中。

## 安全要求

- 200–600 °C 是被控工艺温度，不是 PCB 可以承受的环境温度。
- 热电偶断线、超温、看门狗复位和无效数据必须让输出立即回到 0 V。
- 设备必须配置独立的硬件超温保护和加热器紧急断电回路。
- 控制板和 MAX6675 冷端应远离热风与功率器件热源。
- 在无人值守运行前，必须验证整机的所有故障保护路径。

## 项目状态

- 硬件原理图：已设计
- PCB 布局：已设计，仍需处理验证问题
- 固件架构：已形成文档
- 固件实现：尚未加入仓库
- 生产验证：尚未完成

## 所有权

本项目为 **Rufajx** 定制开发，用于热风机温度实时控制。除非项目所有者另行公布许可条款，否则保留所有权利。
