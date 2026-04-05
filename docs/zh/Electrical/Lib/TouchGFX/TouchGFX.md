![logo](./Picture/logo.svg)

## 前言

TouchGFX 是一款适用于 STM32 MCU的嵌入式 UI 框架，提供了便捷的 UI 设计软件，多种帧缓存策略以及详尽的开发文档。本文以 26 年战队使用的遥控器工程为例，简单讲解如何在战队模板的基础上搭建 TouchGFX  工程，方便后续队员上手使用。

[TouchGFX文档 ](https://support.touchgfx.com/zh-CN/docs/introduction/welcome)

[26遥控项目仓库](https://github.com/XJU-Hurricane-Team/26RemoteCtrl)



## 硬件选型

屏幕采用`240*320`分辨率SPI接口 St7789 主控的 LCD 屏，主控芯片为 STM32F429VET6（ROM：1024.0KB RAM：192.0KB）。





## 初始配置

cubemx 中添加 TouchGFX 组件



## 底层实现

实现UI框架所需要的函数





## UI 设置

mvp 架构