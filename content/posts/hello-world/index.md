+++
title = 'Hello World —— 博客开张'
date = 2026-06-21T12:00:00+08:00
# lastmod = 2026-06-21T12:00:00+08:00
draft = false
summary = '第一篇博文，记录搭建这个 Hugo 博客的过程，并演示代码高亮。'
description = '第一篇博文，记录搭建这个 Hugo 博客的过程，并演示代码高亮。'
tags = ['嵌入式 Linux', '驱动开发', 'AI 工具']
categories = ['随笔']
showTableOfContents = true
+++

折腾了几天，这个基于 **Hugo + Congo** 的博客终于开张了。以后这里会沉淀一些嵌入式 Linux 与驱动开发方面的笔记，也会聊聊怎么用 AI 工具把繁琐的工程流程自动化。

按惯例，开篇先来一个 `Hello World`。这次用两段我熟悉的代码作为演示：一段 **Linux 内核字符设备驱动**，一段 **Python 串口读取脚本**。

## 一、Linux 字符设备驱动

下面是一个基于 `miscdevice` 的极简字符设备驱动。加载后会自动创建 `/dev/hello`，用户态 `read()` 即可拿到内核里的一段字符串——非常适合作为驱动入门的最小骨架。

```c
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/miscdevice.h>
#include <linux/uaccess.h>
#include <linux/string.h>

#define BUF_LEN 32
static char msg[BUF_LEN] = "hello from kernel driver\n";

/* 用户态 read() 时回调：把内核缓冲区的内容拷给用户 */
static ssize_t hello_read(struct file *filp, char __user *buf,
                          size_t count, loff_t *off)
{
    size_t len = strlen(msg);

    if (*off >= len)
        return 0;                       /* 已读到末尾 */
    if (count > len - *off)
        count = len - *off;
    if (copy_to_user(buf, msg + *off, count))
        return -EFAULT;                 /* 用户态指针无效 */

    *off += count;
    return count;
}

static const struct file_operations hello_fops = {
    .owner = THIS_MODULE,
    .read  = hello_read,
};

static struct miscdevice hello_dev = {
    .minor = MISC_DYNAMIC_MINOR,        /* 由内核自动分配次设备号 */
    .name  = "hello",
    .fops  = &hello_fops,
};

static int __init hello_init(void)
{
    int ret = misc_register(&hello_dev);
    pr_info("hello: driver loaded, /dev/hello ready\n");
    return ret;
}

static void __exit hello_exit(void)
{
    misc_deregister(&hello_dev);
    pr_info("hello: driver unloaded\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("West");
MODULE_DESCRIPTION("A minimal Linux character driver demo");
```

加载并验证：

```bash
make                       # 编译为 hello.ko
sudo insmod hello.ko       # 加载模块
cat /dev/hello             # 应输出: hello from kernel driver
dmesg | tail               # 查看内核日志
sudo rmmod hello           # 卸载模块
```

## 二、Python 串口读取脚本

嵌入式开发里，和板子打交道最频繁的就是串口。下面这段脚本通过 `pyserial` 读取设备上报的二进制帧，用 `struct` 解析温度/湿度数据，是最常见的“上位机”逻辑。

```python
import struct
import time

import serial


FRAME_FMT = "<HHHH"   # 帧头 + 温度 + 湿度 + 校验，均为小端 16 位
FRAME_HEAD = 0xAA55


def read_sensor(port: str = "/dev/ttyS3", baud: int = 115200) -> None:
    """读取嵌入式设备通过串口上报的温湿度数据。"""
    with serial.Serial(port, baud, timeout=1) as ser:
        print(f"正在监听 {port} @ {baud}bps ...")
        while True:
            frame = ser.read(8)            # 4 个 uint16 = 8 字节
            if len(frame) != 8:
                continue

            head, temp, humi, crc = struct.unpack(FRAME_FMT, frame)
            if head != FRAME_HEAD:         # 帧头不符，丢弃
                continue

            print(f"温度={temp / 10.0:.1f}℃  湿度={humi / 10.0:.1f}%")
            time.sleep(0.5)


if __name__ == "__main__":
    try:
        read_sensor()
    except KeyboardInterrupt:
        print("\n已停止监听。")
```

## 三、接下来写什么

- BSP 与设备树调试笔记
- 一个完整的平台驱动（platform driver）示例
- 用 AI 工具批量生成驱动框架代码的实践

Stay tuned —— 且把第一篇发出去再说。🚀
