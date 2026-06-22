---
title: "MTK 平台 GPIO-Keys 按键驱动调试实战"
date: 2026-04-20
draft: false
tags: ["Linux", "驱动", "MTK", "GPIO", "嵌入式", "Android"]
categories: ["嵌入式开发"]
summary: "基于 MTK xkt520 平台的 GPIO 按键驱动调试全流程，从 DTS 配置到 Android 层映射。"
---

## 背景

在给 xkt520（MTK 平台）适配自定义按键时，需要新增两个 GPIO 按键：Modekey（GPIO2）和 Customkey（GPIO3）。这篇记录下从 DTS 配置到 Android 层收到事件的完整调试链路。

## 驱动架构概览

Linux 内核的 `gpio-keys` 驱动（`drivers/input/keyboard/gpio_keys.c`）基于 platform driver 框架，将 GPIO 电平变化转换为标准 input event。整个事件流：

```
物理按键 → GPIO 中断 → gpio_keys 驱动 → input 子系统 → /dev/input/eventX
                                                              ↓
                                  应用层 ← Android KeyLayout 映射 ← EventHub
```

## 关键配置层

### 1. DTS 配置

在 `xkt520.dts` 中定义 gpio-keys 节点，每个按键指定 GPIO 引脚、键码和去抖时间：

```dts
&odm {
    gpio_keys: gpio-keys {
        compatible = "gpio-keys";
        pinctrl-names = "default";
        pinctrl-0 = <&modekey_pins_default>, <&customkey_pins_default>;

        modekey {
            label = "mode_key";
            linux,code = <183>;           /* KEY_F13 */
            gpios = <&pio 2 GPIO_ACTIVE_LOW>;
            debounce-interval = <30>;
        };

        customkey {
            label = "custom_key";
            linux,code = <184>;           /* KEY_F14 */
            gpios = <&pio 3 GPIO_ACTIVE_LOW>;
            debounce-interval = <30>;
        };
    };
};
```

**关键属性**：
| 属性 | 作用 |
|------|------|
| `linux,code` | 映射到 Linux 键码（如 KEY_F13 = 183） |
| `GPIO_ACTIVE_LOW` | 低电平有效，驱动自动翻转逻辑 |
| `debounce-interval` | 软件去抖，防止机械抖动误触发 |

### 2. Pinctrl 引脚复用

每个 GPIO 的电气属性通过 pinctrl 配置：

```dts
modekey_pins_default: modekey_pins_default {
    pins_cmd_dat {
        pinmux = <PINMUX_GPIO2__FUNC_GPIO2>;
        slew-rate = <0>;      /* 0 = 输入 */
        bias-pull-up;         /* 内部上拉 */
        input-enable;         /* 使能输入 */
    };
};
```

- `bias-pull-up`：保证默认高电平，按下时拉低
- `slew-rate = 0`：GPIO 方向为输入

### 3. DWS 底层配置

MTK 平台的 DWS（Driver Workbench Studio）配置 GPIO2 的中断和电气属性：

| 参数 | 值 | 说明 |
|------|-----|------|
| `eint_mode` | true | 使能外部中断 |
| `def_dir` | IN | 输入方向 |
| `inpull_en` | true | 内部上下拉使能 |
| `inpull_selhigh` | true | 选择上拉 |

### 4. Android KeyLayout 映射

内核按键码到 Android KeyCode 的映射在 `mtk-kpd.kl`：

```
key 183   F13
key 184   F14
```

| 内核键码 | kl 映射 | Android KeyCode | 用途 |
|----------|---------|-----------------|------|
| 183 | `F13` | `KEYCODE_F13` | Modekey |
| 184 | `F14` | `KEYCODE_F14` | Customkey |

## 按键事件流分析

从物理按键按下到应用收到的完整链路：

```
1. 物理按键按下 → GPIO2 电平拉低（下降沿）
         │
         ▼
2. GPIO 中断触发 → 内核 IRQ Handler
         │
         ▼
3. gpio_keys.c 驱动 → 去抖动处理（30ms）
         │
         ▼
4. input_event() → EV_KEY, code=183, value=1
         │
         ▼
5. Input 子系统 → /dev/input/eventX
         │
         ▼
6. Android EventHub → InputReader 读取
         │
         ▼
7. mtk-kpd.kl 映射 → KEYCODE_F13
         │
         ▼
8. 应用层收到 KeyEvent.KEYCODE_F13
```

事件 value 含义：

| value | 含义 |
|-------|------|
| `1` | KEY DOWN（按下） |
| `0` | KEY UP（释放） |
| `2` | KEY REPEAT（长按重复） |

排查问题时按这个链路反向追溯：应用没收到的，先 `getevent` 看内核有没有事件；内核没事件的，看中断有没有触发；中断没触发的，万用表量 GPIO 电平。

## 调试验证

```bash
# 确认驱动已加载
cat /proc/bus/input/devices | grep gpio-keys

# 实时捕获按键事件
getevent -l /dev/input/eventX

# 按下按键时能看到：
# EV_KEY  KEY_F13  DOWN
# EV_KEY  KEY_F13  UP
```

验证清单：
- [x] `/proc/bus/input/devices` 能见到 `gpio-keys`
- [x] `getevent` 能捕获按键事件
- [x] Android 应用层收到对应 KeyCode
- [x] 长按、短按、连按均正常

## 小结

GPIO 按键调试的核心链路：**DTS 定义 → Pinctrl 配置 → DWS 底层配置 → kl 文件映射 → getevent 验证**。每一步都可能出问题，但排查顺序很明确：先确认硬件电平（万用表），再确认内核中断，最后看上层映射。
