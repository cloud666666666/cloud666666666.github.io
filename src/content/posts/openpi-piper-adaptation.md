---
author: Yunhao Wang
pubDatetime: 2026-08-17T12:00:00.000Z
modDatetime: 2026-08-17T12:00:00.000Z
title: openpi 适配 Piper 机械臂:VLA 训练栈的实战改造
slug: openpi-piper-adaptation
featured: true
draft: false
tags:
  - vla
  - openpi
  - fine-tuning
  - robotics
description: 把 Physical Intelligence 的 openpi(Pi0/Pi0.5)适配到 Piper 六轴机械臂:自定义 policy、LeRobot 数据转换、微调配置与 checkpoint 分析工具链。
---

## 为什么改 openpi

Physical Intelligence 的 [openpi](https://github.com/Physical-Intelligence/openpi) 是当前
最活跃的开源 VLA 训练栈之一(Pi0/Pi0.5 模型)。但它默认面向 Mobile ALOHA / Franka 等
官方硬件——**想在自己 2000 元级的 Piper 机械臂上训练 VLA,需要一层适配**。

AIIT-openpi 就是这个适配层:保留上游模型和训练栈,增加 Piper policy、LeRobot 数据
准备工具、Piper 微调配置和 checkpoint 分析。

## 适配了什么

### 1. Piper Policy

openpi 的 policy 是"模型 + 动作空间 + 归一化"的组合。Piper 六轴臂和官方硬件的
关节数、量程、单位都不同,需要:

- 定义 Piper 的动作/状态空间(六轴关节角)
- 数据归一化统计量(从自有数据计算,而不是官方默认值)
- 推理时反归一化到真实关节命令

### 2. LeRobot 数据转换

采到的 LeRobot v3.0 数据集不能直接喂给训练栈,需要转换到 v2.1 并计算归一化统计:

```bash
# Convert a LeRobot v3.0 dataset to v2.1 and compute normalization statistics.
python -m openpi.training.dataset_convert --dataset <your-dataset>
```

数据格式的版本兼容是"数据闭环"里最琐碎也最容易断的环节——上游训练栈、数据采集端、
推理端三方的格式在各自演进。

### 3. 微调配置

以 Pi0.5 为基础,微调自有机械臂数据:

```bash
# Fine-tune pi0.5 (使用缓存 checkpoint 时开启 HF_HUB_OFFLINE=1)
python -m openpi.training.train --config <piper-config>
```

关键点:**基础模型 checkpoint 提前缓存**——离线环境下用 `HF_HUB_OFFLINE=1` 训练,
避免训练中途因为下载中断。

### 4. PyTorch 路径

上游默认 JAX,新增了 PyTorch 支持(单卡/多卡/多机训练 + 推理 + policy server)。
对自有硬件更友好,也方便和更广的社区工具链对接。

## 一条完整的 VLA 数据闭环

```
采集(AIIT-Roboarm, LeRobot v3.0)
  → 转换(openpi 工具, v2.1 + 归一化)
  → 微调(Pi0.5 + Piper config)
  → 推理(Piper policy 反归一化 → 关节命令)
  → 回放部署(roboarm 推理模块)
```

四个环节全部开源、全部可复现——这是这个项目组合最有价值的地方:**不是单个模型,
是一条可以跑通的数据闭环**。

## 教训

1. **归一化统计量必须来自自有数据**——用官方硬件的统计量会在推理时输出越界动作
2. **格式版本是隐形的断点**——v3.0/v2.1 只差一点,但不转换训练直接报错
3. **checkpoint 先缓存再训练**——离线优先,网络不是训练环境的组成部分

## 代码

[github.com/cloud666666666/AIIT-openpi](https://github.com/cloud666666666/AIIT-openpi)
