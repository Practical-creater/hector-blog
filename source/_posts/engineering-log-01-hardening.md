---
title: 工程日志 01：六项基础加固——规范网址、SEO、自托管资源、CI、404 与清理
date: 2026-09-12 16:00:00
categories: Engineering
tags:
  - 工程日志
  - Hexo
  - Cloudflare
  - CI
  - SEO
mathjax: true
---

站点上线之后的第一轮加固。判断标准只有一个：**降低长期维护成本**——每一项都应该是做一次就不用再管的事，而不是需要定期照看的东西。六项按性价比排序，每项记录动机、做法和验证方式，方便以后回头查。

<!-- more -->

## 1. 规范网址：把 www 跳转到根域名

`www.hector.software` 和 `hector.software` 现在都能打开同一套内容。对读者无所谓，对搜索引擎是两个站：同一篇文章有两个地址，外链和权重被分散，还可能被判为重复内容。解决办法是选一个"正式地址"，把另一个 301 永久跳转过去。正式地址选根域名，因为 Hexo 配置里的 `url` 和生成的 canonical 标签都是它。

这一项在 Cloudflare 后台完成，不在仓库里：**Rules → Redirect Rules → Create rule**，模板里有现成的 "Redirect from WWW to root"，或者手写规则——条件 `Hostname equals www.hector.software`，动作 `Dynamic redirect`，表达式 `concat("https://hector.software", http.request.uri.path)`，状态码 301，勾选 preserve query string。

为什么不用 Worker 脚本做跳转：纯静态资源的 Worker 请求是免费且不计数的，一旦加了脚本就进入 Worker 的请求配额。零成本的规则能做的事，不必花配额。

验证：`curl -I https://www.hector.software/` 应返回 `301` 和 `Location: https://hector.software/`。`www` 的自定义域名绑定要保留，否则它连证书都没有，跳转规则也就轮不到执行。

## 2. SEO 基础设施：sitemap、Atom、robots.txt、描述

装两个官方生成器：

```bash
npm install hexo-generator-sitemap hexo-generator-feed
```

`_config.yml` 里加：

```yaml
sitemap:
  path: sitemap.xml
  tags: false
  categories: false
feed:
  type: atom
  path: atom.xml
  limit: 20
  content: true
```

`hexo-generator-feed` 会自动在每个页面的 `<head>` 里插入 `<link rel="alternate" type="application/atom+xml">`，RSS 阅读器打开首页就能发现订阅源——技术读者里用 RSS 的比例远高于平均水平，这是最便宜的"回访渠道"。`source/robots.txt` 声明 sitemap 地址；`description` 和 `keywords` 填上，首页的 `<meta name="description">` 就不再是空的。后续把 sitemap 提交到 Google Search Console，这一步需要账号，不在仓库里。

## 3. 自托管字体和第三方库

之前页面有三个外部依赖：字体来自一个 Google Fonts 镜像站，Font Awesome、Prism、MathJax 这些库来自 cdnjs，访客计数来自 busuanzi。任何一个抽风，页面就会变慢或变样，而且每个访客的浏览器都要去连三家陌生的服务器。

**字体**：`@fontsource/ibm-plex-sans` 和 `@fontsource/ibm-plex-mono` 这两个 npm 包里有按字重切好的 woff2 文件（OFL 1.1 许可）。取 latin 子集的 8 个文件放进 `source/fonts/`，一共 176 KB；`@font-face` 规则写在 `source/_data/styles.styl`，通过 NexT 的 `custom_file_path.style` 注入；主题配置里字体的 `external` 全部改为 `false`，NexT 就不再生成指向外部字体站的 `<link>`。中文回落到系统字体，不需要下载。

**库**：NexT 提供 `vendors.plugins: local`，配合 `@next-theme/plugins` 这个包把主题测试过的库版本锁定为依赖，构建时把库文件复制到 `public/lib/`。本地构建一切正常（15 MB、474 个文件），上线后用 `curl -I` 核对却发现每个页面多了一次 307：Cloudflare 的静态资源服务会把路径里的 `@`（`/lib/@fortawesome/...`）规范化成 `%40` 再重定向，而 NexT 生成的 HTML 引用的是带 `@` 的原始路径。Font Awesome 的样式表是阻塞渲染的，每次加载都先吃一个不可缓存的 307，比从 CDN 直接拿还慢。于是这一项回退：库继续走主题默认的 `cdnjs`——那是 Cloudflare 自家的 CDN，和本站同一张网，可靠性不是问题；真正不稳定的只有那个字体镜像站，而字体已经自托管。

**保留 busuanzi**：计数本质上就是远程服务，自托管没有意义。它挂了页脚数字空白，不影响其他内容。

验证：`public/css/main.css` 里有 8 条 `@font-face`，全部指向 `/fonts/`；首页的外部资源只剩 cdnjs（库）和 busuanzi（计数），字体镜像站的引用消失；线上对每个资源 `curl -I`，全部 200，没有重定向。

顺手用一个公式验证本地 MathJax 能不能正常渲染——世界模型里最常见的那种潜空间预测损失：

$$
\mathcal{L}(\theta) = \mathbb{E}_{t}\left[\,\lVert \hat{z}_{t+1} - z_{t+1} \rVert_2^2\right],
\qquad \hat{z}_{t+1} = f_\theta(z_t, a_t)
$$

这篇文章的 front-matter 里写了 `mathjax: true`，只有声明了的文章才会加载 MathJax，其余页面不受影响。

## 4. CI：让坏的依赖升级在合并前就红

仓库里有模板自带的 Dependabot，会定期开 PR 升级依赖。之前的问题是：合并一个 PR 才触发 Cloudflare 构建，坏了就是线上红——第一篇文章里那两次失败就是这么被发现的。

加一个 GitHub Actions 工作流 `.github/workflows/build.yml`：对所有 pull request 和非 `main` 分支的推送跑 `npm ci` → `npm run build`，再检查 `index.html`、`404.html`、`sitemap.xml`、`atom.xml` 都生成了。`main` 分支不重复跑，那是 Cloudflare 的活。Node 版本从 `.nvmrc` 读取，和 Cloudflare 构建机一致。

Dependabot 改成每月一次、最多 5 个 PR、`hexo*` 系列打包成一个 PR。每天一个 PR 对个人博客是噪音，不是安全。

## 5. 404 页面

`wrangler.jsonc` 里早就写了 `not_found_handling: 404-page`，但站里没有 `404.html`，访问不存在的地址是一片空白。加一个 `source/404.md`，front-matter 里 `permalink: 404.html` 让 Hexo 把它生成到根目录，内容是一句说明加三个回家的链接。Cloudflare 匹配不到资源时就返回它，状态码仍是 404。

## 6. 清理与可复现

- 删掉从没用过的 `hexo-theme-landscape` 依赖和空的 `_config.landscape.yml`。
- `.nvmrc` 写 `24`，`package.json` 加 `engines.node >= 20`：换电脑或换 CI 时不会悄悄用错 Node 版本。
- 写 `README.md`：日常命令、目录说明、两条不能违反的规矩（不用 pnpm、不把 wrangler 写进依赖）以及新电脑的恢复步骤。半年后的自己是这个文件的目标读者。

## 结果

构建：不到一百个文件，三百多毫秒。首页的外部请求从三家减到两家（cdnjs、busuanzi），字体全部由本站提供。`npm ci` 在本地和 CI 两种 npm 版本下都通过。仓库新增内容不到 200 KB（字体）加几个配置文件。一条经验：本地 `hexo generate` 看起来完美的改动，仍然要上线后对每个资源 `curl -I` 看一遍状态码——这次的 307 只有线上才会出现。

还没做、也不急的：`post_asset_folder` 按文章管理图片、图片压缩与懒加载、Cloudflare Web Analytics、评论区。等真正需要的时候再加，每一项都是十分钟量级。
