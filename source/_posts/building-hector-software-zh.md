---
title: 从零搭建 hector.software：域名、Cloudflare、Hexo，以及一路踩过的坑
date: 2026-09-12 12:00:00
lang: zh-CN
categories: Meta
tags:
  - Hexo
  - NexT
  - Cloudflare
  - 建站笔记
---

这是这个博客的第一篇文章，记录 `hector.software` 是怎么从一个域名变成一个能打开的网站的。我把整个过程按实际发生的顺序写下来，包括三次失败的构建和每一次的报错原文，目的是让一个从来没有搭过网站的人也能照着做，而且知道每一步为什么要这么做。文中出现的账号、IP、密钥一类的东西都换成了示例值，真实配置不是这些数字。

<!-- more -->

## 先搞清楚：一个博客网站到底由哪几块组成

在动手之前，值得花两分钟把几个名词理顺，否则后面每一步都会觉得莫名其妙。

**域名**是网站的门牌号，比如 `hector.software`。**DNS** 是查号台，负责把这个门牌号翻译成"内容实际放在哪台机器上"。**Hexo** 是一台印刷机：你用 Markdown 写文章，它负责把文章排版成浏览器能看的 HTML 文件。**Cloudflare** 是书店，印好的 HTML 放在它那里，全世界的读者来这里取。**GitHub 仓库**是放原稿的抽屉，博客的全部源文件（文章、配置、主题设置）都存在里面。

把这几块串起来就是整个工作流：我在本地写一篇文章，推到 GitHub 的抽屉里；Cloudflare 发现抽屉里有新东西，自动把印刷机打开跑一遍，把印出来的 HTML 摆上书店的货架；读者输入域名，DNS 把他们带到书店门口。理解了这条链路，后面所有配置都只是在回答一个问题：这一环怎么接到下一环。

## 域名：教育邮箱能省下的第一笔钱

域名是通过 [GitHub Student Developer Pack](https://education.github.com/pack) 里 Name.com 的学生优惠拿到的，第一年免费。流程是先用学校邮箱在 `education.github.com` 完成学生认证，然后在礼包页面点 Name.com 的优惠，这时 GitHub 会让你授权一个叫 "Name.com Education Pack" 的第三方应用——它只读取你的公开资料，作用仅仅是向 Name.com 证明"这个人是认证过的学生"。授权完跳到 Name.com，挑一个符合优惠范围的后缀（`.software`、`.dev`、`.app` 这些都在列表里），结账时 Name.com 会让你**单独注册一个 Name.com 账号**。

这里有个容易混淆的地方，我自己就栽了：Name.com 的账号和 GitHub 账号是两套独立的东西。GitHub 授权只是拿到兑换资格，域名本身挂在 Name.com 的用户名密码下面。几个月后我回去改 DNS，一时想不起 Name.com 的密码，误以为"当时是用 GitHub 登录的"，翻了半天 GitHub 的授权记录也没用。正确的找回路径是 Name.com 登录页的 "Forgot Username"：那里要填的是**域名**而不是邮箱，系统会把用户名发到注册时留的联系邮箱；拿到用户名再走一次 "Forgot password" 拿重置链接。域名开了 WHOIS 隐私保护也不影响，那只是对外隐藏，Name.com 内部照样知道该发给谁。

一个朴素的建议：域名注册完当场把用户名、密码和当时用的邮箱存进密码管理器。域名是所有东西的根，弄丢它的成本比别的都高。

## 服务器：为什么最后一台都没买

最初的计划是照着一位博主的技术栈来——Hexo 加 NexT 主题，跑在一台自己的服务器上，顺手把 [Oracle Cloud Always Free](https://www.oracle.com/cloud/free/) 那台永久免费的 ARM 机器薅下来。它的免费额度目前是 2 OCPU、12 GB 内存（网上大量教程还写着 4 核 24 G，那是 2026 年年中之前的老数据，已经下调）。

这条路卡在了付款验证。Oracle 注册时要求绑一张卡做身份核验，而且官方明确写着**不接受虚拟卡和预付卡**，只认真实的信用卡或借记卡。我没有信用卡，试着了解了几种"虚拟卡"方案，结论是成功率看运气，而且一旦被风控标记，整个账号都可能受限，后面换真卡也未必救得回来。在这上面耗时间不划算。

退一步想，其实根本不需要服务器。Hexo 生成的是纯静态文件——一堆 HTML、CSS、JavaScript，没有数据库，没有后端逻辑。静态文件最适合的去处不是一台需要自己打补丁、开端口、续证书的虚拟机，而是静态托管服务：把文件交给它，它负责分发。Cloudflare 提供这种服务，免费，自带 HTTPS 和全球 CDN。至于以后真想有一台机器折腾，[Azure for Students](https://azure.microsoft.com/en-us/free/students) 给学生 100 美元额度，全程不需要信用卡，留作后手足够。

## 把 DNS 交给 Cloudflare

Cloudflare 既然要托管网站，就需要它来管域名的解析。做法是把域名的 DNS 服务器从 Name.com 换成 Cloudflare，域名注册商还是 Name.com，只是"查号台"换了一家。

登录 Cloudflare 后台，在首页中间那张 "Add a domain" 卡片里输入域名。老教程里这个入口叫 "Add a site"，新版界面改了名字，我第一次找了好一会儿。选免费套餐后，Cloudflare 会自动扫描域名现有的解析记录，然后让你确认。我的扫描结果里有两条：一条 A 记录把根域名指向 `203.0.113.10`（示例值），一条 CNAME 把 `www` 指回根域名。那个 IP 是几年前一台早已注销的云主机留下的，机器不在了，IP 也被服务商收回，属于指向虚空的死记录。这两条我都直接删掉了，因为后面 Cloudflare 会在绑定域名时自动写入正确的记录，留着旧的只会打架。

确认之后 Cloudflare 给出两个 nameserver，形如 `ada.ns.cloudflare.com` 和 `bob.ns.cloudflare.com`（每个账号分配的不一样）。回到 Name.com 的"管理域名服务器"页面，把默认的 `ns1.name.com` 那几条换成这两个，保存。生效需要等传播，官方说 1 到 2 小时，最长可能到 24 小时，期间 Cloudflare 的域名概览页会一直显示 "Waiting for your registrar to propagate your new nameservers"。这个页面右侧有个醒目的 "Create Worker" 按钮，跟接下来要做的事没有直接关系，可以不理会——我们的项目会从另一个入口创建。

## 本地把博客跑起来：Hexo + NexT

本地需要 Node.js、npm、Git，另外装一个 GitHub CLI（`gh`）会省很多事。项目放在 `~/Code/hector-blog`，不要放桌面，几个项目下来桌面就没法看了。

```bash
mkdir -p ~/Code/hector-blog && cd ~/Code/hector-blog
npm install -g hexo-cli
hexo init .
npm install
```

`hexo init` 会拉一个官方模板下来，里面已经有 `_config.yml`、`source/_posts/`（放文章）、`scaffolds/`（新文章的模板）这些目录。主题选 [NexT](https://theme-next.js.org/)，用 npm 装：

```bash
npm install hexo-theme-next hexo-generator-searchdb
```

NexT 有四种布局方案：Muse、Mist、Pisces、Gemini。前两种是单栏，侧边栏默认收起，点页面左下角的按钮才滑出来；后两种是双栏，侧边栏固定在旁边。我参考的那个站页脚写着 "Powered by Hexo & NexT.Mist"，深色的头像面板其实就是 **Mist** 滑出来的侧边栏。我一开始拿着一张侧边栏展开状态的截图，把它当成了 Pisces 的双栏布局，改完发现整个页面结构都不一样，又改了回来。教训很简单：想借鉴哪个站的布局，先看它页脚的方案名，别对着图猜。

主题的所有个性化设置写在项目根目录一个叫 `_config.next.yml` 的文件里，而不是去改 `node_modules` 里主题自带的配置。这样以后升级主题时你的改动不会被覆盖：

```yaml
scheme: Mist
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

根目录的 `_config.yml` 里有几项必须改：`title` 和 `author` 填自己的；`url` 填 `https://hector.software`，Hexo 用它生成站内所有链接、RSS 和站点地图，填错了以后每个链接都指向 `example.com`；`language` 我写成了 `[zh-CN, en]`，然后每篇文章的头部用 `lang: zh-CN` 或 `lang: en` 标记自己的语言，NexT 会据此切换界面文案（日期、"阅读全文"这类字样）。这是一种轻量的双语做法，代价是首页文章列表中英文混排。真正把两种语言拆成两套独立页面需要额外的生成插件，我评估后觉得那个插件维护状态不太好，先不引入。

跑 `hexo server` 后打开 `http://localhost:4000` 就能看到本地效果，改配置或文章会自动刷新。

**第一个坑在这里。** 我一开始是用 pnpm 装的主题，`hexo` 一跑就报：

```
ERROR Script load failed: node_modules/hexo-theme-next/scripts/filters/post.js
Error: Cannot find module 'hexo-util'
```

翻 `hexo-theme-next` 的 `package.json` 就明白了：它运行时要用 `hexo-util`、`js-yaml`、`css` 这几个包，但 `js-yaml` 只写在 devDependencies 里，`hexo-util` 和 `css` 干脆没写。它之所以在绝大多数人的机器上能跑，是因为 npm 的 `node_modules` 是扁平的，这些包作为别的依赖的依赖被平铺在顶层，主题"顺手"就能 `require` 到。pnpm 的 `node_modules` 是严格隔离的，没声明的依赖就是找不到。这种情况有个名字叫幽灵依赖，不是 pnpm 的错，但对使用者来说最省事的解法就是不要跟主题较劲：删掉 `node_modules` 和 `pnpm-lock.yaml`，老老实实用 `npm install`。整个 Hexo 生态的教程也都默认 npm。

## 部署：Pages 已经变成 Workers 了

本地能看到网站之后，先把源码推到 GitHub。用 `gh` 一条命令建仓库并推送：

```bash
git init
git add -A
git commit -m "Init Hexo + NexT blog"
gh repo create hector-blog --public --source=. --remote=origin --push
```

模板自带的 `.gitignore` 已经排除了 `node_modules/` 和 `public/`，这两个目录一个是依赖、一个是构建产物，都不应该进仓库——依赖可以随时用 `npm install` 装回来，产物则由部署环境重新生成。仓库里只放"原稿"。

然后回到 Cloudflare 后台，左侧菜单 **Workers & Pages**，点 Create，选择从 Git 仓库导入。这里需要解释一下为什么会牵扯到 GitHub：Cloudflare 自己并不保存你的源码，它只是被授权去读你的 GitHub 仓库。授权的方式是在 GitHub 上安装一个 Cloudflare 的应用，你选择允许它访问哪些仓库。装好以后，每次有代码推到仓库的主分支，GitHub 会通知 Cloudflare，Cloudflare 就把仓库克隆到一台干净的构建机上，安装依赖、运行构建、把产物发布出去。这就是所谓的持续部署。另一种做法是每次在自己电脑上手动运行部署命令，也能用，但意味着换台电脑或者电脑坏了就没法更新网站，而且部署过程不留记录。让 GitHub 当唯一的事实来源，让 Cloudflare 盯着它，是更省心的结构。

选好仓库后会进入一个标题为 "Set up your application — Configure your Worker project" 的页面。注意是 **Worker**。老教程里的 Cloudflare Pages 有一个专门的表单，能填"构建输出目录"和环境变量；现在新建项目一律走 Workers 的流程，那个表单不存在了，取而代之的是两个命令和一个配置文件：

- **Build command**：构建命令，填 `npm run build`。
- **Deploy command**：部署命令，保持默认的 `npx wrangler deploy`。

Wrangler 是 Cloudflare 的命令行工具，`wrangler deploy` 负责把东西发布上去。它要知道发布什么，这就靠仓库根目录的 `wrangler.jsonc`：

```jsonc
{
  "name": "hector-blog",
  "compatibility_date": "2026-09-01",
  "assets": {
    "directory": "./public",
    "not_found_handling": "404-page"
  }
}
```

`assets.directory` 指向 Hexo 的输出目录 `public`，这一行就是原来那个"构建输出目录"输入框的替代品。`name` 要和向导页里的 Project name 一致。页面上还有个 "Build variables"，是给构建过程用的环境变量，这个博客用不到。

## 三次失败的构建，以及每一次的原因

点了 Deploy 之后，构建日志分成 Initializing、Cloning、Installing、Building、Deploying 五段。我在 Installing 和 Building 各摔了一次，加起来失败了三次。把报错原文和原因写下来，因为这两个错误很典型，换个项目也可能再见到。

### 第一次：npm error Invalid Version

Installing 阶段直接失败，日志只有一行有用的：

```
Detected the following tools from environment: npm@10.9.2, nodejs@24.18.0
Installing project dependencies: npm clean-install --progress=false
npm error Invalid Version:
```

`Invalid Version` 后面是空的，什么线索都没有。我在本地用同一条命令 `npm ci` 试，一切正常。区别在于版本：我本地的 npm 是 11.6.2，Cloudflare 构建机上是 10.9.2。

背景是我之前为了"可复现"把 `wrangler` 加进了项目的 devDependencies。Wrangler 依赖一个叫 workerd 的运行时，这个运行时为每个操作系统各发一个二进制包：`@cloudflare/workerd-darwin-64`、`@cloudflare/workerd-linux-64` 等等，作为可选依赖声明，安装时只装与当前系统匹配的那个。问题在于 npm 10 和 npm 11 把这类"按平台可选"的依赖写进 `package-lock.json` 的方式不一样：npm 11 生成的锁文件 npm 10 读不懂，报 `Invalid Version`；反过来我用 npm 10.9.2 重新生成锁文件，本地的 npm 11 跑 `npm ci` 又报：

```
npm error `npm ci` can only install packages when your package.json and
package-lock.json are in sync.
npm error Missing: @cloudflare/workerd-darwin-64@ from lock file
npm error Missing: @cloudflare/workerd-linux-64@ from lock file
```

两边互不兼容，改哪一边都会让另一边坏掉。这里顺便说一下 `npm ci` 和 `npm install` 的区别：`install` 比较宽容，锁文件和 `package.json` 对不上时它会自己修正锁文件；`ci` 是给自动化环境用的，要求锁文件必须和 `package.json` 严格一致，不一致就拒绝安装，为的是保证每次构建装的东西完全一样。构建机用的是 `ci`，所以锁文件的任何格式差异都是致命的。

解法是釜底抽薪：`wrangler` 根本不需要出现在 `package.json` 里。部署命令是 `npx wrangler deploy`，`npx` 的行为就是"本地没有就临时下载一份来跑"，构建机会在部署那一刻拿到一个与它自己系统匹配的 wrangler。把它从依赖里删掉，锁文件里就不再有任何按平台分发的二进制包，两个版本的 npm 都能顺利读取。

### 第二次和第三次：/bin/sh: 1: hexo: not found

锁文件修好后 Installing 过了，Building 阶段又倒下：

```
Executing user build command: hexo generate
/bin/sh: 1: hexo: not found
Failed: error occurred while running build command
```

这个错误的意思是 shell 找不到一个叫 `hexo` 的命令。本地之所以能直接敲 `hexo`，是因为最开始 `npm install -g hexo-cli` 把它装到了全局，全局的 bin 目录在 PATH 里。构建机是一台干净的机器，只按 `package.json` 装了项目依赖，`hexo` 命令的确存在——它在项目的 `node_modules/.bin/hexo`——但 Cloudflare 执行 Build command 的方式是把那串字符原样交给 `/bin/sh`，而 `node_modules/.bin` 并不在 sh 的 PATH 里。

我第一反应是往 devDependencies 里加 `hexo-cli`，重试，结果一模一样。事后看 `node_modules/hexo/package.json` 才发现 `hexo` 核心包自己就声明了 `"bin": {"hexo": "./bin/hexo"}`，而且本来就依赖 `hexo-cli`，所以 `.bin/hexo` 从第一次构建起就一直在那里，问题从头到尾只是 PATH。

正确的修法是把 Build command 从 `hexo generate` 改成 `npm run build`。模板的 `package.json` 里早就有一条 `"build": "hexo generate"`，而 `npm run` 在执行脚本前会自动把 `node_modules/.bin` 加进 PATH——这是 npm 自己的行为，跟运行在哪台机器上无关，所以它在本地和构建机上表现完全一致。写 `npx hexo generate` 也行，`npx` 会先在本地 `node_modules` 里找。这个字段在项目的 Settings 里改，改完后对最新一次构建点 Retry build，或者随便推一个新提交触发。

### 顺带说一句那几行 allow-scripts 警告

日志里还有一段黄色的警告：

```
npm warn allow-scripts 4 packages have install scripts not yet covered by allowScripts:
npm warn allow-scripts   hexo-util@4.0.0 (postinstall: npm run build:highlight)
```

意思是 npm 现在默认不再运行第三方依赖的安装脚本，除非你明确批准。这是针对供应链攻击的安全措施，不是错误。`hexo-util` 那个 postinstall 脚本的作用是生成一个代码高亮的别名表，而这个表本来就随包发布了，脚本跑不跑没有影响，可以放心忽略。

## 绑定域名

三个问题解决后，构建的五段全部变绿，Cloudflare 会给一个 `hector-blog.<账号>.workers.dev` 形式的临时地址，打开能看到博客就说明发布成功了。然后回到项目的 Overview，右侧 "Domains and routes" 里点 Custom domains，分别添加 `hector.software` 和 `www.hector.software`。因为域名的 DNS 已经在 Cloudflare 手里，它会自动写入解析记录并签发 HTTPS 证书，几分钟内域名就能打开。之前删掉的那两条旧记录，在这一步被正确的新记录取代。

前提是 nameserver 的传播已经完成。如果域名概览页还在显示 "Waiting for your registrar"，绑定时可能会提示等待，那就过一会儿再来。

## 以后怎么写、怎么维护

日常写文章的循环很短。在项目目录里：

```bash
hexo new post "my-new-post"      # 在 source/_posts/ 下生成 my-new-post.md
hexo server                       # 本地预览，http://localhost:4000
git add -A && git commit -m "post: my new post" && git push
```

推送后一分钟左右，Cloudflare 的 Deployments 标签页会出现一条新构建，跑完网站就更新了。文章文件顶部的 front-matter 记得写 `lang: zh-CN` 或 `lang: en`，英文版和中文版是两个独立的文件。想先写着不发布，用 `hexo new draft "title"`，文件会放进 `source/_drafts/`，`hexo server --drafts` 能预览，定稿后 `hexo publish "title"` 移到正式目录。

几件事情不用再碰：Cloudflare 后台的构建设置只在命令变化时才需要改；`node_modules` 和 `public` 永远不进仓库；主题升级用 `npm update hexo-theme-next`，你的定制都在 `_config.next.yml` 里，不会被覆盖。换一台电脑只需要 `git clone` 加 `npm install`，全部环境就回来了——这也意味着仓库本身就是备份，不需要另外备份服务器。

构建失败时先看 Deployments 里的日志，定位是五段中的哪一段。Installing 失败通常是锁文件问题，Building 失败先怀疑命令和 PATH，Deploying 失败去检查 `wrangler.jsonc`。

## 小结

整个过程里我真正花时间的不是"配置"，而是弄清楚每一层在做什么：域名商和 DNS 商可以是两家，GitHub 和 Cloudflare 各管源码和发布，Hexo 只是一个把 Markdown 变成 HTML 的工具。三次构建失败也都不是什么高深的问题，一次是两个 npm 版本对同一份锁文件的理解不一致，两次是 PATH 里没有构建机需要的命令。理解了原因之后，解法都只有一两行。

后面应该会陆续写一些科研和工程相关的笔记。欢迎常来。
