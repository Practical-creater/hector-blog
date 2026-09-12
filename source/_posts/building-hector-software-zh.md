---
title: 从零搭建 hector.software：域名、Cloudflare 与 Hexo 建站全记录
date: 2026-09-12 12:00:00
lang: zh-CN
categories: Meta
tags:
  - Hexo
  - NexT
  - Cloudflare
  - 建站笔记
---

这是这个博客的第一篇文章，记录一下 `hector.software` 是怎么从"一个想法"变成现在这个能打开的网站的。中间踩了几个坑，写下来给以后的自己、也给可能路过的人参考。文中涉及账号、密钥、IP 之类的地方我都换成了示例值，实际配置不是这些数字。

<!-- more -->

## 域名：教育邮箱 + GitHub Student Developer Pack

域名是通过 [GitHub Student Developer Pack](https://education.github.com/pack) 里 Name.com 的学生优惠拿到的：先用教育邮箱在 `education.github.com` 完成学生认证，登录 GitHub 授权一个叫 "Name.com Education Pack" 的第三方 OAuth 应用（只读公开信息，用来验证你确实是认证学生），再跳转到 Name.com 单独注册/登录一个账号完成域名注册——这两步是分开的，GitHub 授权只是"拿到兑换资格"，真正的域名账号还是 Name.com 自己的一套用户名密码，跟 GitHub 账号本身没有直接绑定关系。

小提醒：如果学生认证没通过或者后台报错,一般是学校邮箱域名不在 GitHub 认可的名单里,可以试试上传学生证之类的凭证走人工审核。

## 服务器怎么选：从"抢 Oracle 免费机器"到放弃

一开始想复刻一个用 Hexo + NexT 搭博客的博主的技术栈,顺带白嫖 [Oracle Cloud Always Free](https://www.oracle.com/cloud/free/) 的永久免费 ARM 实例（当前额度是 2 OCPU / 12GB 内存,注意网上很多教程写的"4核24G"是旧数据,官方已经在 2026 年年中下调过）。

但卡在了付款验证这一步——Oracle 官方明确写着**不接受虚拟卡/预付卡**，只认真实的信用卡或借记卡。我手头没有真实信用卡，试了几种"虚拟卡"方案（比如某些新型银行的关联真实余额的借记卡）成功率不稳定，风险是一旦被判定为高风险支付方式，账号可能直接被限制。权衡下来没有必要在这上面耗时间，转向了完全不需要信用卡的方案：

- **Microsoft Azure for Students**：$100 额度、完全不需要信用卡，走教育邮箱验证即可，作为"以后想要真服务器折腾"的备选。
- **Cloudflare Pages**：既然 Hexo 生成的就是纯静态文件，那压根不需要一台"服务器"，直接用静态托管更省心——这也是这个博客最终选的路线。

## DNS 迁移到 Cloudflare

把域名从 Name.com 原生 DNS 迁移到 Cloudflare 分两步：

1. 在 Cloudflare 后台"Add a domain"（新版界面这个入口不叫"Add a site"了,容易找不到,在首页中间的卡片里）添加 `hector.software`，Cloudflare 会自动扫描现有的 DNS 记录。
2. 拿到 Cloudflare 分配的两个 Nameserver（形如 `ns1.example-cf-ns.com` / `ns2.example-cf-ns.com`，实际数值因账号而异），回 Name.com 的"管理域名服务器"页面替换掉默认的 Name.com Nameserver。

扫描出来的旧 DNS 记录里有一条指向早年一台云主机的 A 记录（示例：`hector.software → 203.0.113.10`），那台机器早就不在了，IP 也已经被服务商回收，属于死记录，直接删掉，交给 Cloudflare Pages 后面自动接管。

Nameserver 生效需要等传播,官方说 1-2 小时,最长可能到 24 小时。

## Hexo + NexT 搭博客

本地环境：Node.js + npm + Git + GitHub CLI（提前用 `gh auth login` 登好）。

```bash
mkdir -p ~/Code/hector-blog && cd ~/Code/hector-blog
npm install -g hexo-cli
hexo init .
npm install
```

主题选的是 [NexT](https://theme-next.js.org/)，双栏固定侧边栏的 **Pisces** 方案（NexT 一共四种方案：Muse / Mist / Pisces / Gemini，只有 Pisces 和 Gemini 才有这种左侧固定栏的布局，之前一度搞错选成了 Mist，效果完全不对）：

```bash
npm install hexo-theme-next hexo-generator-searchdb
```

有个坑记一下：**装主题千万别用 pnpm**（或者至少默认配置下不行）。`hexo-theme-next` 有几个运行时依赖（`hexo-util`、`js-yaml`、`css`）只写在了 devDependencies 里，本质是指望 npm 那种扁平化 node_modules 帮它"顺手"就近能找到这些包。pnpm 默认的严格隔离结构不会做这种隐式兜底，直接报 `Cannot find module 'hexo-util'`。解决办法很简单：删掉 `node_modules` 和 `pnpm-lock.yaml`，老老实实 `npm install`。

主题相关的自定义配置全部写在项目根目录的 `_config.next.yml`（而不是改 `node_modules` 里主题自带的那份），这样以后 `npm update` 主题也不会冲突：

```yaml
scheme: Pisces
creative_commons:
  sidebar: true
language_switcher: true
local_search:
  enable: true
menu:
  home: / || fa fa-home
  about: /about/ || fa fa-user
  archives: /archives/ || fa fa-archive
  talks: /talks/ || fa fa-comments
```

## 部署：Cloudflare Pages

Cloudflare 控制台 → **Workers & Pages** → **Create** → **Pages** → **Connect to Git**，选中博客源码所在的 GitHub 仓库，构建配置：

- Build command: `hexo generate`
- Build output directory: `public`
- 环境变量加一条 `NODE_VERSION=20`

保存后 Cloudflare 会给一个 `*.pages.dev` 的临时地址，构建成功后再去这个 Pages 项目里 **Custom domains** 添加 `hector.software` 和 `www.hector.software`——因为域名已经是 Cloudflare 的 Zone 了，这一步会自动写好 DNS 记录，不用再手动碰 DNS 表格，HTTPS 证书也是自动签发续期。

## 小结

整个流程走下来，除了本地起项目那几步，几乎不需要一台真正的"服务器"——静态博客配合 Cloudflare Pages 是目前最省心的组合：不用管系统更新、不用开端口、不用管证书续期。以后要是想跑点别的东西（比如托管一个小模型的推理服务），再单独考虑 Azure for Students 的额度。

以后应该会陆续写一些科研、工程相关的笔记，欢迎常来看看。
