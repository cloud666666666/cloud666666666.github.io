---
title: 桌宠搬家:把 DeepSeek Harness 的宠物养在 CC Switch 上
description: 把 dsh-pet 从 DSH 的 Web 壳里独立出来——桌宠跟着 CC Switch 起落,头顶气泡显示今日 token 用量与余额,数据全部直接读 CC Switch 数据库。附宿主实现思路与几个工程细节。
pubDatetime: 2026-09-21T13:09:04.000Z
tags: [electron, nodejs, cc-switch, desktop-pet, windows]
featured: false
---

<p align="center">
  <img src="https://raw.githubusercontent.com/cloud666666666/dsh-pet-ccswitch/main/assets/preview.jpg" alt="桌宠头顶气泡显示: 今日 318.2 万 token / DeepSeek ¥560.11 / 95 次请求" width="360">
</p>

## 起因

[DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)(DSH)里养着一只桌宠 [dsh-pet](https://github.com/PC2005-cloud/dsh-pet)——一百多个手绘风透明动画,待机呼吸、打瞌睡、玩魔方,可爱得很。

但它住在 DSH 的 Web 界面里:不开 `dsh web`,就见不到它。而我日常反而一直在用 [CC Switch](https://github.com/farion1231/cc-switch)(Claude Code 的供应商切换器)——它本地记着全部调用数据:每日 token 用量、每家服务商的余额。

于是有了这个想法:**把这只宠物"扛"出 Harness,让它跟着 CC Switch 过日子。**

- CC Switch 一启动,桌宠自己跳出来;一退出,它跟着藏回去
- 头顶气泡显示**今日 token 用量 + 当前服务商余额**
- 右键菜单:打开 CC Switch / 查看用量与余额 / 碎碎念 / 对话 / 点播任意动画

项目地址:[cloud666666666/dsh-pet-ccswitch](https://github.com/cloud666666666/dsh-pet-ccswitch)

## 拆解:独立化只需要换三样东西

桌宠本体是 dsh-pet 的前端产物(跑在 Electron 透明小窗里),它原本会向 DSH 的 Web 服务要配置、要数据、收事件。所以独立化不需要动它本体,只需要做一个宿主,把三样东西换掉:

1. **顶替 Web 服务**——只实现桌宠需要的那几个接口(HTTP `:3080`),它几乎无感知
2. **换数据源**——「今日用量 / 余额」的来源,从 DSH 换成 CC Switch
3. **换生命周期信号**——从「DSH 会话开着吗」换成「CC Switch 进程在不在」

```
CC Switch ←── 读 cc-switch.db(用量/服务商/usage_script)──┐
     ▲                                                    │
     │ 进程在不在                                          ▼
     └──────── 宿主 host.mjs ──── HTTP :3080 ──── 桌宠(Electron)
```

## 关键选择:一切以 CC Switch 为准

这个项目的核心设计原则只有一句话——**不自己发明任何数据规则**:

- **今日 token 用量**:直接读 CC Switch 的 `cc-switch.db`(它的本地代理记着每一笔请求),按当天时间窗聚合,输入/输出/缓存四类字段全算上
- **服务商余额**:不自己写「怎么查余额」的逻辑,而是把 CC Switch 里**为每个服务商配好的 `usage_script` 拿出来直接执行**——它怎么查,我们就怎么查

好处很直接:换服务商、改解析规则、加新供应商——桌宠这边**自动跟着变**,一行适配代码都不用改。

## 几个工程细节

- **生命周期跟随**:每 2 秒探测一次 `cc-switch*` 进程,起就亮、退就藏(间隔可配置)
- **数据库被独占怎么办**:CC Switch 开着时 SQLite 可能锁着,直读失败就**复制一份再读**,读完即删
- **不写死任何路径**:找 `cc-switch.exe` 的优先级是「运行中的进程 → 上次发现的路径 → 注册表卸载项 → 常见安装路径 → 开始菜单快捷方式」,换盘符、换目录都能自己跟上
- **幂等补丁**:对桌宠本体的改造(气泡改成用量视图、菜单指向 CC Switch)都是可以重复执行的补丁——认不出原文就告警并**退化成原版行为**。宁可不改,绝不改坏
- **依赖契约而不是文件**:宿主实现的接口照着 dsh-pet 原本的契约来,它换新版本大概率能直接升级

## 已知限制

- **仅 Windows**:进程发现、开机自启、窗口置前都用了 Windows 专有手段(macOS / Linux 适配欢迎 PR)
- 气泡显示的是「CC Switch 当前在用」那一家的余额,计价单位默认按人民币校准;换成美元计价的服务商,档位刻度含义会变
- 余额查不到时**不伪造数字**,直接显示失败原因——这是 dsh-pet 原版就有的好设计,我沿用了

## 安装

```sh
npx github:cloud666666666/dsh-pet-ccswitch install
```

要求 Windows + Node.js ≥ 22.5 + 装好并运行过 CC Switch。安装脚本会准备好环境(自动拉取 dsh-pet 与 Electron)、注册开机自启并立即启动。之后 CC Switch 开开关关,桌宠自己跟。

其他命令:`start` / `stop` / `status` / `doctor`(自检)/ `patch`(升级 dsh-pet 后重打补丁)/ `uninstall`。

## 致谢

- 桌宠本体:[PC2005-cloud/dsh-pet](https://github.com/PC2005-cloud/dsh-pet)(MIT)
- 数据源与生命线:[farion1231/cc-switch](https://github.com/farion1231/cc-switch)

桌宠是薅来的,日子是它自己过的。🐾
