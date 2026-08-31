---
title: 批量 Star 某个 GitHub 用户的所有仓库:一行命令搞定
description: 用 Python + GitHub API 写的批量 Star 工具,支持跳过 fork、dry-run 预览、自动翻页。附完整代码和使用说明。
pubDatetime: 2026-08-31T09:11:00
tags: [github, tools, python, api]
featured: false
---

## 场景

遇到一个很欣赏的开发者(比如刚接触 Robonix 社区时关注了 nbsheep、Ciliphen),想把他名下的所有仓库都 Star 一遍——手动一个个点,几十个仓库点到手酸,还会漏掉 fork 的仓库。

于是写了个小工具:**一条命令,Star 某用户的所有公开仓库**。

## 功能特性

- ✅ 自动翻页(per_page=100),几百个仓库也全覆盖
- ✅ 默认跳过 fork,`--include-forks` 参数可包含
- ✅ `--dry-run` 先预览,不实际执行
- ✅ 幂等:已 Star 的重复执行无害
- ✅ 显示每个仓库的星标数
- ✅ 错误处理:404 / 限速会明确提示

## 使用方法

```bash
# 预览(不实际执行)
GITHUB_TOKEN=你的token python3 star_all_repos.py 目标用户名 --dry-run

# 实际执行
GITHUB_TOKEN=你的token python3 star_all_repos.py 目标用户名

# 包含 fork 仓库
GITHUB_TOKEN=你的token python3 star_all_repos.py 目标用户名 --include-forks
```

Token 权限要求:

- classic token:`repo` 或 `public_repo` scope
- fine-grained token:Stars(星标)仓库权限

## 原理

三步走,都是标准 REST API:

1. **列仓库**:`GET /users/{username}/repos?per_page=100&page=N`,根据响应头的 `Link` 字段判断是否还有下一页
2. **Star**:`PUT /user/starred/{owner}/{repo}`(PUT 幂等,重复 Star 返回 204)
3. **认证**:请求头带 `Authorization: token <token>` 和 `X-GitHub-Api-Version: 2022-11-28`

完整代码在仓库里:

- GitHub:[github-tools](https://github.com/cloud666666666/github-tools)

## 注意事项

- GitHub API 速率限制:Star 操作 5000 次/小时。一般用户的仓库数量(几十到几百)完全够用;如果目标是几千仓库的大 V,脚本会收到 HTTP 403 提示限速
- 只 Star 公开仓库(私人仓库 token 也没权限看)
- Star 只是收藏标记,不会产生任何通知打扰对方——放心批量操作

## 实测

对 Robonix 社区的两位开发者跑了一遍:

```text
✅ 认证成功,当前账号: cloud666666666
🔍 获取 nbsheep 的仓库列表...
   共 5 个仓库(跳过fork: True)
✅ nbsheep/dataStructure
✅ nbsheep/github-auto-commits
✅ nbsheep/nbsheep
✅ nbsheep/nbsheep.github.io
✅ nbsheep/robonix
完成: 成功 5,失败 0,共 5
```

Ciliphen 含 fork 共 14 个,同样 14/14 全部成功。

---

一个小工具,解决一个小痛点。代码开放在 [github-tools](https://github.com/cloud666666666/github-tools) 仓库,后续会陆续加其他 GitHub 自动化脚本,欢迎 Star。
