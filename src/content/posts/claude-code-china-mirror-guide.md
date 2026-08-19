---
author: Joker.Yun
pubDatetime: 2026-08-19T08:40:00.000Z
modDatetime: 2026-08-19T08:40:00.000Z
title: 国内部署 Claude Code 完全指南:npm 镜像安装 + 二进制下载坑 + 三条使用路线
slug: claude-code-china-mirror-guide
featured: false
draft: false
tags:
  - claude-code
  - china-mirror
  - npm
  - llm
description: 在国内给 WSL/Linux 装 Claude Code 的完整实测:npmmirror 一条命令装好主体,postinstall 被拦导致二进制缺失的修复,以及登录墙的三条真实可用路线(官方 API/中转 API/第三方池)。
---

## 背景

Claude Code 是 Anthropic 官方的终端 AI 编程代理,但国内装它有两道墙:
**安装**(npm 默认源慢/断)和**使用**(OAuth 登录要海外网络,API 不接受大陆卡)。
这篇文章记录我在 WSL2(Ubuntu)上的完整实测,所有命令都验证过。

## 第一步:安装(npm 走国内镜像)

### 1.1 设置 npmmirror 源

```bash
npm config set registry https://registry.npmmirror.com
npm config get registry   # 验证输出 registry.npmmirror.com
```

### 1.2 安装

```bash
npm install -g @anthropic-ai/claude-code
```

### 1.3 大概率遇到的坑:claude native binary not installed

新版 Claude Code(v2.1.x)拆成了主包 + 平台原生包两部分:
主包通过 npm 装,真正的可执行文件在平台包里
(`@anthropic-ai/claude-code-linux-x64`,约 300MB)。

npm 出于安全默认拦截了 postinstall 脚本,于是只装了主包,
二进制没就位,运行时报:

```
Error: claude native binary not installed.
Either postinstall did not run (--ignore-scripts, some pnpm configs)
or the platform-native optional dependency was not downloaded (--omit=optional).
```

**修复:手动跑 postinstall**

```bash
# 找到全局 node_modules 路径
npm root -g   # 例如 /home/<user>/.local/lib/node_modules

# 手动执行安装脚本
node $(npm root -g)/@anthropic-ai/claude-code/install.cjs

# 验证
claude --version   # 输出 2.1.235 (Claude Code) 即成功
```

### 1.4 关键结论:全程国内镜像是否成立?

成立。我翻了 `install.cjs` 的源码(整个脚本只有 7KB):
它内部**没有任何下载 URL**,唯一作用是把 npm 已装好的平台包里的
二进制复制到 `bin/` 下——而平台包本身就是 npm 装的,走的 npmmirror。

验证方式:用 npmmirror 的元数据 API 查一下就知道镜像完整同步了平台包:

```bash
curl -s https://registry.npmmirror.com/@anthropic-ai%2Fclaude-code-linux-x64 | head
# versions 列表里能看到 2.1.235,和主包同步
```

所以**安装环节 100% 国内镜像,零直连**:
`npm install` → npmmirror 拉主包+平台包 → `install.cjs` 本地复制 → 完成。

(如果想让 postinstall 自动执行,也可以:
`npm install -g --allow-scripts=@anthropic-ai/claude-code @anthropic-ai/claude-code`,
但我更倾向手动跑一次 install.cjs,显式且可控。)

## 第二步:使用(登录才是真关卡)

装好只是半场,`claude` 首次运行需要认证。国内的三条真实路线:

### 路线 A:官方 OAuth 登录(需要代理)

```bash
export https_proxy=http://<代理>:<端口> http_proxy=http://<代理>:<端口>
claude   # 浏览器 OAuth,Pro/Max 订阅用户走这条
```

适合已有 Pro/Max 订阅的人,全程代理即可。首次登录后日常使用也建议挂着代理。

### 路线 B:官方 API Key(Anthropic Console)

```bash
export ANTHROPIC_API_KEY=sk-ant-...
claude
```

坑:Anthropic Console **不支持大陆+香港的卡充值**。可行办法是
用海外卡(如 Depay/OneKey 虚拟卡)或找朋友代充。API 按 token 计费,
适合轻量使用。

### 路线 C:第三方中转/共享池(国内直连)

设置环境变量把 API 指向中转站:

```bash
export ANTHROPIC_BASE_URL=https://<中转站域名>
export ANTHROPIC_API_KEY=<中转站 key>
claude
```

便宜、直连、无需代理,但**风险自担**:中转站能看到你所有的请求内容
(代码、密钥、隐私),稳定性和跑路风险都不可控。
**只喂不敏感的项目**,别把生产代码丢进去。

### 三条路线对比

| 路线 | 直连 | 成本 | 隐私 | 适合 |
|---|---|---|---|---|
| A 官方订阅 OAuth | ❌ 需代理 | $20/月起 | 最安全 | 已有订阅 |
| B 官方 API Key | ❌ 需代理+海外卡 | 按 token | 安全 | 轻量 |
| C 中转站 | ✅ 直连 | 便宜 | ⚠️ 全部流量过第三方 | 不敏感项目 |

## 补充:WSL 用户专属提醒

- WSL2 里 Windows 侧的 `127.0.0.1:7897` 代理**不通**,必须换成
  `ip route` 查到的默认网关 IP(如 `172.18.0.1:7897`)
- Claude Code 是 TUI 应用,在 WSL 里配合 Windows Terminal 使用体验最好
- `claude doctor` 可以自检安装健康度,`claude --version` 验证版本

## 总结

```
安装: npm 镜像(npmmirror)+ 手动 install.cjs → 全程国内镜像 ✅
使用: OAuth(代理)/ 官方 API(海外卡)/ 中转(直连但隐私风险)
```

一句话:**安装墙是假墙,一行命令;认证墙才是真墙,按需选路线。**
