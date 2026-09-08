---
title: 我的秋招 AI 岗位每日雷达:牛客+龙哥表双源监控
description: 用 Hermes cron 每天抓取牛客校招日历和龙哥27届秋招汇总表,筛选 AI/算法岗增量推送飞书,并自动归档到 GitHub 仓库。附数据流架构和仓库链接。
pubDatetime: 2026-09-08T02:00:00
tags: [jobs, hermes, automation, cron, career]
featured: false
---

## 场景

27 届秋招,一天刷一遍牛客和各类汇总表,很容易漏掉刚开的公司——校招的窗口期往往只有十几天,漏一天就是漏一家。

与其手动刷,不如让机器每天替我盯:两条公开数据源,一个 cron,每天把**新增的 AI/算法岗位**推到我飞书上,并且自动归档成公开数据。

## 数据流

```
牛客校招日历 API ──┐
                   ├─→ Hermes cron(每天 9:00 / 9:30)
龙哥27届秋招汇总表 ─┘     ↓ 筛选: 27届 + AI/算法 + 未截止 + 过滤阿里系
                         ↓
                   飞书推送(有新增才说话,无新增发心跳)
                         ↓
                   daily-jobs 仓库(每天 9:45 自动 push)
                         data/2026-09-08.json + latest.json + README
```

## 两个数据源

**牛客校招日历**:每天 9 点抓前几页,按"新收录排前"取增量,筛选 27 届 + AI/算法方向岗位,和历史状态比对,只推新增。

**龙哥表(腾讯文档 smartsheet)**:一个每日更新的 27 届秋招汇总表,272 家公司,含内推码、截止时间。有意思的是它前端是 canvas 渲染、DOM 里抓不到数据——最后从它的网络请求里挖到了公开 JSON 接口(`/dop-api/get/sheet`),数据是 base64 + zlib 压缩的,解开就是完整表格。

## 开源与可复现

工具本身**零依赖、独立开源**,不依赖任何私有 agent 环境:纯 Python 标准库,clone 即可跑,牛客源用自己的 Cookie(环境变量传入,文档里写了获取步骤),龙哥表源无需任何凭据。

## 自动化细节

- 我自己的每日推送跑在 [Hermes](https://hermes-agent.nousresearch.com) cron 上,`no_agent` 脚本模式——不经过 LLM,零漂移、零幻觉;但任何人用 crontab 跑 `python3 run.py --push` 效果一样
- 无新增时发"心跳"消息,证明监控还活着(曾因静默被误判为挂了)
- 每天 9:45 汇总两个源,写入 `data/{date}.json`,git push 到仓库
- 一个踩过的坑:Hermes 新版本的 cron 子进程保护用了 systemd 的 `OOMPolicy=kill` 参数,而 WSL 的 systemd 249 不支持,导致定时任务全挂——删掉该参数后恢复

## 仓库

[github.com/cloud666666666/daily-jobs](https://github.com/cloud666666666/daily-jobs)

`latest.json` 保持最新数据,想接入自己工具链的人可以直接拉取。
