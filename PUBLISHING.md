# 博客发布规范（给任何 Claude Code session，也给未来的我）

本文件是发布博客的唯一权威。在其他项目里工作的 Claude 只需读完本文件并照做。

## 1. 站点事实

- 仓库：`~/Code/hector-blog`（Hexo 8 + NexT）。GitHub `Practical-creater/hector-blog`，本机 git/SSH 已登录，推送不需要凭证。
- 发布机制：推送到 `main` → Cloudflare 自动构建 → 约 1 分钟上线。**不要**碰 Cloudflare 后台、`wrangler`、`hexo deploy`。
- 界面英文；正文中文或英文均可。
- 文章 = `source/_posts/<slug>.md`；网址 `https://hector.software/YYYY/MM/DD/<slug>/`，日期取 front-matter 的 `date`。
- 图片放 `source/images/`，正文用 `/images/<文件名>` 引用（以 `/` 开头）。
- `source/` 同时是 Obsidian 库；Claude 直接编辑文件即可，不需要打开 Obsidian。
- 风格样本（动笔前先读）：`source/_posts/engineering-log-01-hardening.md`、`source/_posts/writing-workflow.md`。

## 2. 分类与标签

`categories` 只能从下表选**一个**：

| 分类 | 用于 |
|---|---|
| `Meta` | 博客本身：搭建、工作流、站点维护 |
| `Engineering` | 环境配置、部署、工具链、服务器、CI、工程实践 |
| `Research` | 论文阅读、方法讨论、实验记录、科研写作 |
| `Notes` | 课程、读书、学习笔记 |
| `Life` | 其他 |

- 系列文章按"工程日志 NN"编号：`ls source/_posts | grep engineering-log` 取下一个编号。
- `tags` 3–6 个，优先复用已有标签：`grep -h '^  - ' source/_posts/*.md | sort | uniq -c | sort -rn`。

## 3. 前置检查

```bash
test -d ~/Code/hector-blog || { echo "不在 Hector 的 Mac 上"; exit 1; }
cd ~/Code/hector-blog
git status --porcelain     # 必须为空，否则停下来问用户
git pull                   # 冲突或 diverged 则停下来问用户
```

不在 Mac 上时：把文章保存到 `~/blog-drafts/<slug>.md`，打印全文，告诉用户放入 Mac 的 `source/_posts/` 后执行第 6 节，然后停止。

## 4. 文件模板

slug 用小写英文和短横线。直接创建文件，不必 `hexo new`。

```yaml
---
title: <中文标题，简洁、具体>
date: <date "+%Y-%m-%d %H:%M:%S" 的输出>
categories: <第 2 节表中的一个>
tags:
  - <标签>
---
<首页摘要，一两段>

<!-- more -->

<正文>
```

只有正文含公式才加 `mathjax: true`。代码块标注语言。不写 HTML。

## 5. 内容与脱敏

内容：写给第一次做这件事的人看。详实、自然、分小标题，少用碎片化要点；每一步说清**做什么、为什么、怎么验证**；报错贴**原文**，跟上原因和修法；结尾给可复用的清单或脚本。

脱敏红线（违反任何一条不得推送）：

- 真实 IP → `203.0.113.x` / `198.51.100.x`；主机名、内网域名 → `lab-gpu-01.example.edu` 类占位；用户名 → `<user>`。
- 密码、token、API key、代理地址与端口、订阅链接 → `<PROXY_HOST>`、`<PORT>`、`<TOKEN>`。不贴 `~/.ssh/config`、私钥、`known_hosts`、代理客户端配置原文。
- 学校、实验室、导师、课题真实名称 → 泛化。

自检（两条都应无输出，或每行都是明显占位）：

```bash
F=source/_posts/<slug>.md
grep -nE '([0-9]{1,3}\.){3}[0-9]{1,3}' "$F" | grep -vE '203\.0\.113\.|198\.51\.100\.|127\.0\.0\.1|0\.0\.0\.0'
grep -niE 'password|passwd|token|secret|api[_-]?key|sk-[a-z0-9]' "$F"
```

## 6. 构建与发布

```bash
cd ~/Code/hector-blog
npm run build 2>&1 | tail -5          # 须有 "INFO  N files generated"，无 ERROR
ls public/$(date +%Y/%m/%d)/          # 应含新文章目录
git add -A && git status --short      # 只应出现新增的文章/图片
git commit -m "post: <slug>"
git push
```

禁止：pnpm；安装或升级依赖；修改 `_config.yml`、`_config.next.yml`、`wrangler.jsonc`、`package.json`；提交 `node_modules/`、`public/`、`.obsidian/`；`hexo deploy`；改动其他已有文章。

## 7. 上线验证

```bash
URL="https://hector.software/$(date +%Y/%m/%d)/<slug>/"
for i in $(seq 1 24); do c=$(curl -s -o /dev/null -w '%{http_code}' --max-time 15 "$URL"); [ "$c" = 200 ] && break; sleep 10; done; echo "$URL -> $c"
```

4 分钟后仍非 200：报告 `git log -1 --oneline` 与构建输出，不要重复推送。

## 8. 汇报

文章 URL、commit 哈希、字数、脱敏自检结果、无法确认的地方。
