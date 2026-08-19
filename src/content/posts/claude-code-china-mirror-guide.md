---
author: Joker.Yun
pubDatetime: 2026-08-19T09:42:00.000Z
modDatetime: 2026-08-19T09:42:00.000Z
title: 国内部署 Claude Code 完全指南:nvm 与 npm 全程镜像安装(Linux/WSL/Windows)
slug: claude-code-china-mirror-guide
featured: false
draft: false
tags:
  - claude-code
  - china-mirror
  - npm
  - nvm
  - llm
description: 从零开始的国内部署实测:nvm 走 gh-proxy 装 Node、npmmirror 双镜像、Claude Code 的 postinstall 二进制坑,以及认证三条路线。WSL/Linux 全流程实测,Windows(nvm-windows)为文档核实。
---

## 目标与前置

Claude Code 是 Anthropic 官方的终端 AI 编程代理。国内部署要过三关:

1. **Node 环境**(Claude Code 要求 Node ≥ 22,实测自 v2.1.235 的 `package.json`)
2. **安装本体**(npm 包 + 平台二进制,约 300MB)
3. **认证**(OAuth / API Key / 中转,后面细说)

本文 WSL/Linux 部分是我实机跑通的;Windows 部分基于 nvm-windows 官方文档核实,没有 Windows 实机复测,已在文中标注。

## 第一关:装 Node(两个平台的 nvm 不是同一个!)

关键认知:**nvm-sh 的 nvm 只支持 POSIX(WSL/Linux/macOS);原生 Windows 要用 nvm-windows,命令和行为都不一样,别混着用。**

### 1.1 WSL / Linux:走 gh-proxy 的 nvm

#### 第 0 步:理解为什么"一行命令"装不上

网上流行的命令是:

```bash
curl -o- https://v4.gh-proxy.org/https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.2/install.sh | bash
```

实测拆解:这条命令**只把安装脚本本身走了 gh-proxy**,但脚本后续默认执行
`git clone https://github.com/nvm-sh/nvm.git`——这一步在国内大概率卡死。

好消息是 install.sh 支持 `NVM_SOURCE` 环境变量覆盖仓库地址(源码里优先级最高),
而 gh-proxy 也能代理 git clone(实测 `git ls-remote` 通过)。所以全程版是:

```bash
# 1. 下载安装脚本(经 gh-proxy)
curl -L -o /tmp/nvm-install.sh \
  https://v4.gh-proxy.org/https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.2/install.sh

# 2. 把 nvm 仓库地址指向 gh-proxy 再安装
NVM_SOURCE='https://v4.gh-proxy.org/https://github.com/nvm-sh/nvm.git' \
  bash /tmp/nvm-install.sh

# 3. 生效(重开终端,或)
source ~/.bashrc
```

两个实测细节:

- install.sh 开头的 `nvm_latest_version()` 写死返回 `v0.39.2`,所以 URL 里那个
  版本号就是脚本自身的版本,放心用
- 脚本会检查必须用 `bash` 执行,zsh 直接管道会报错退出

#### 第 1 步:让 Node 本体也走国内镜像

`nvm install` 默认从 `nodejs.org/dist` 下载,国内极慢。设置 Node 镜像:

```bash
export NVM_NODEJS_ORG_MIRROR=https://npmmirror.com/mirrors/node
echo 'export NVM_NODEJS_ORG_MIRROR=https://npmmirror.com/mirrors/node' >> ~/.bashrc

# 装 LTS 并设为默认
nvm install --lts
nvm use --lts
nvm alias default 'lts/*'
node -v    # 实测应为 v22+(Claude Code 的 engines 要求)
```

#### 第 2 步:npm 换国内源

```bash
npm config set registry https://registry.npmmirror.com
```

### 1.2 Windows:nvm-windows(管理员 + 双镜像)

**先卸载已装的 Node**(官方 README 强烈建议,避免 PATH/卸载权冲突),然后:

**下载** nvm-windows 安装包(经 gh-proxy,实测 200,约 5.6MB):

```
https://v4.gh-proxy.org/https://github.com/coreybutler/nvm-windows/releases/latest/download/nvm-setup.exe
```

**注意:以下命令必须在"管理员"PowerShell/CMD 里跑**(README 原文:
"nvm-windows runs in an Admin shell")。

```powershell
# 设置镜像 —— 官方 README 原文:"People in China can use ..."
nvm node_mirror https://npmmirror.com/mirrors/node/
nvm npm_mirror  https://npmmirror.com/mirrors/npm/

# 装 LTS(注意:Windows 版是 nvm install lts,没有 --)
nvm install lts
nvm use lts

node -v
npm config set registry https://registry.npmmirror.com
```

与 Linux 版的命令差异对照:

| 操作 | WSL/Linux(nvm-sh) | Windows(nvm-windows) |
|---|---|---|
| 装 LTS | `nvm install --lts` | `nvm install lts` |
| Node 镜像 | 环境变量 `NVM_NODEJS_ORG_MIRROR` | `nvm node_mirror <url>` |
| npm 镜像 | 环境变量(或手动) | `nvm npm_mirror <url>` |
| 权限 | 普通用户 | **必须管理员** |

## 第二关:装 Claude Code(两平台命令一致)

```bash
npm install -g @anthropic-ai/claude-code
```

### 2.1 大概率踩的坑:`claude native binary not installed`

较新的 npm(11/12+)**全局安装默认拦截安装脚本**(install scripts),
结果只装了 JS 壳,300MB 的平台二进制没就位,运行报:

```
Error: claude native binary not installed.
Either postinstall did not run (--ignore-scripts, some pnpm configs)
or the platform-native optional dependency was not downloaded (--omit=optional).
```

**修法一:允许脚本重装**

```bash
npm install -g --allow-scripts=@anthropic-ai/claude-code @anthropic-ai/claude-code
```

**修法二:手动补跑 postinstall**

```bash
npm root -g    # 拿到全局 node_modules 路径,如 /home/<user>/.local/lib/node_modules
node $(npm root -g)/@anthropic-ai/claude-code/install.cjs
```

任选其一,然后验证:

```bash
claude --version   # 2.1.235 (Claude Code) 即成功
```

### 2.2 为什么安装环节能做到"全程国内镜像"(实测验证)

我翻了 v2.1.235 的源码,证据链完整:

1. **包结构**:主包 `package.json` 的 `optionalDependencies` 里挂载了
   **8 个平台原生包**(`claude-code-linux-x64` / `-musl` / `-arm64`、
   `-win32-x64` / `-arm64`、darwin 两件套)——真正的可执行文件在这些包内
2. **install.cjs 只有 7KB,不含任何下载 URL**:它唯一作用是本地复制平台包
   里的二进制,并建好 `bin/claude` 链接。所以下载行为全部发生在 npm 安装阶段
3. **npmmirror 同步了全部包**(registry 元数据实测 200):

```bash
curl -s https://registry.npmmirror.com/@anthropic-ai%2Fclaude-code | grep -o 'claude-code-[a-z0-9-]*'
# 能看到 linux-x64、win32-x64 等全部平台包
```

结论:**`npm install` → npmmirror 拉主包+平台包 → install.cjs 本地复制 → 完成,零直连。**

## 第三关:认证(国内的真实瓶颈)

装好只是半场。三条真实路线:

### 路线 A:官方 OAuth 登录(需代理)

```bash
export https_proxy=http://<代理>:<端口> http_proxy=http://<代理>:<端口>
claude   # 浏览器 OAuth,Pro/Max 订阅用户
```

适合已有订阅的人。日常使用建议常挂代理。

### 路线 B:官方 API Key(Anthropic Console)

```bash
export ANTHROPIC_API_KEY=sk-ant-...
claude
```

坑:Anthropic Console **不支持大陆+香港卡充值**。可行办法是海外虚拟卡
(如 Depay/OneKey)或朋友代充。按 token 计费,适合轻量使用。

### 路线 C:第三方中转(国内直连)

```bash
export ANTHROPIC_BASE_URL=https://<中转站域名>
export ANTHROPIC_API_KEY=<中转站 key>
claude
```

便宜、直连、免代理,但**风险自担**:中转站能看全部请求内容(代码、密钥),
稳定性和跑路风险不可控。只喂不敏感项目。

### 对比

| 路线 | 直连 | 成本 | 隐私 | 适合 |
|---|---|---|---|---|
| A 官方订阅 OAuth | ❌ 需代理 | $20/月起 | 最安全 | 已有订阅 |
| B 官方 API Key | ❌ 需代理+海外卡 | 按 token | 安全 | 轻量 |
| C 中转站 | ✅ 直连 | 便宜 | ⚠️ 流量全过第三方 | 不敏感项目 |

## WSL 用户专属提醒

- Windows 侧 `127.0.0.1:7897` 这类代理地址在 WSL 里**不通**,必须换成
  `ip route` 查到的默认网关 IP(如 `172.18.0.1:7897`)
- Claude Code 是 TUI 应用,WSL 里配 Windows Terminal 体验最好
- `claude doctor` 自检安装健康度;`claude --version` 验证版本

## 速查总结

```
WSL/Linux:  gh-proxy 装 nvm(NVM_SOURCE)→ 镜像装 Node → npmmirror 装 Claude → 手动 install.cjs 补二进制
Windows:    nvm-windows(管理员)→ node/npm 双镜像 → nvm install lts → 同一套 npm 命令
认证:       OAuth(代理)/ 官方 API(海外卡)/ 中转(直连但隐私风险)
```

一句话:**安装墙是假墙(全程镜像可绕),认证墙才是真墙,按需选路线。**

> 诚实声明:WSL/Linux 全流程为本机实测(2026-08-19);Windows 部分命令
> 来自 nvm-windows 官方 README 核实,未经 Windows 实机复测,如踩坑欢迎评论补充。
