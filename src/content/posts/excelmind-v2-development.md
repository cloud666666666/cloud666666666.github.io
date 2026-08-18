---
author: Joker.Yun
pubDatetime: 2026-08-18T17:30:00.000Z
modDatetime: 2026-08-18T17:30:00.000Z
title: ExcelMind 二次开发:把一个 LLM Excel 助手改造成 v2.1
slug: excelmind-v2-development
featured: false
draft: false
tags:
  - llm
  - agent
  - python
  - open-source
description: 在 Gen-Future/ExcelMind 基础上做二次开发:重构 Skills 架构、引入双引擎写入能力、支持 CSV——40 个 star 背后的改动记录。
---

## 背景

[ExcelMind](https://github.com/cloud666666666/ExcelMind) 是一个基于 LLM 的
Excel 智能分析助手:自然语言查询、ECharts 可视化、多表联查。

我 fork 自 Gen-Future/ExcelMind 并持续二次开发,当前 40 个 star、
9 个用户 fork 了我的版本。这篇文章记录 21 个 commit 里的三个主要改动。

## 改动一:重构 Skills 架构(参考 Claude Code Skills 规范)

原版的技能是硬编码的散装函数,新增一个能力要动主流程。
参考 Claude Code 的 Skills 规范,重构为模块化架构:

- 技能按能力拆分(分析/写入/格式化),每个技能**自描述**
- 支持**动态加载**——新增技能不改主流程代码
- 技能的 description 直接参与 LLM 的工具选择,让模型自己决定调哪个

核心思路:**把"能做什么"交给技能清单,而不是写死在主逻辑里**。
这让后续加功能(CSV、工作表切换)变得只是"加一个技能文件"。

## 改动二:v2.0 双引擎架构(ExcelDocument)

原版只读不写。v2.0 引入 ExcelDocument 抽象层:

```
读取引擎(只读查询) + 写入引擎(格式化写入)
        ↓ 统一封装
   ExcelDocument(对 LLM 暴露的接口)
```

写入能力落地为格式化工具:LLM 生成的表格内容可以**直接写回 Excel**,
不再是"只能看不能改"。这对真实用户场景是质变——分析完顺手改表。

## 改动三:v2.1 CSV 支持

用户需求:很多数据源是 CSV,不是 Excel。v2.1:

- 支持 .csv 上传与拖拽
- CSV 自动转换为内部工作表处理
- 与 xlsx 走同一套查询/写入管线

## 其他小改动

- 多 Sheet 工作簿的**工作表切换**(switch_sheet 工具)
- 修复递归限制问题
- README/LICENSE 重写(补充 fork 声明,尊重上游)

## 二次开发的心得

1. **先读上游的架构,别急着改**——双引擎的改动之所以顺利,是因为先理解了原版的单引擎数据流
2. **让 LLM 做工具选择,而不是硬编码 if-else**——Skills 重构后,加功能的边际成本显著下降
3. **真实用户需求驱动版本**——CSV 支持来自用户反馈,不是自己拍脑袋
4. **诚实标注来源**——README 里明确写了 fork 声明和上游链接,这是二次开发的底线

## 数据

- 40★,9 个 child forks(用户选择 fork 我的版本而非上游)
- 21 个 commit,从 v2.0 到 v2.1.0

[项目地址](https://github.com/cloud666666666/ExcelMind)
