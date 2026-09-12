---
title: 从 Markdown 到上线：本站的写作与发布工作流
date: 2026-09-12 17:00:00
categories: Meta
tags:
  - 写作流程
  - Hexo
  - Git
  - Obsidian
---

这篇文章回答一个问题：写完一篇 Markdown 之后，它是怎么变成网页的，中间每一步在做什么。它假设读者没有用过 git，每个命令都附带原因。文末记录了这篇文章本身是用哪种方式发布的。

<!-- more -->

## 一句话模型

网站的全部源文件放在 GitHub 的一个仓库里。Cloudflare 盯着这个仓库，仓库一有新提交，它就把仓库克隆到一台干净的机器上，运行 Hexo 把 Markdown 排版成 HTML，再把 HTML 发布出去。所以**发布 = 让仓库多一个提交**。提交是从你电脑推上去的、在 GitHub 网页上直接改的、还是编辑器插件推的，Cloudflare 不关心，也不需要你写任何 HTML。

参与的四个地方：

| 位置 | 角色 |
|---|---|
| 电脑上的 `~/Code/hector-blog` | 工作副本，在这里写 |
| GitHub 仓库 | 正本，唯一被信任的版本 |
| Cloudflare | 监听仓库，构建并发布 |
| `https://hector.software` | 构建结果 |

git 只需要认识三个动作，顺序固定：`add`（选中改动）→ `commit`（把选中的改动打成一个带说明的快照，仍在本地）→ `push`（把快照上传到 GitHub）。所有 git 命令都必须在项目文件夹里执行，因为"仓库"就是那个文件夹本身——里面有个隐藏的 `.git` 目录记着全部历史，命令只对当前所在的文件夹生效。

## 方式 A：本地写，命令行发布

这是日常主力。

**1. 打开终端，进入项目。** Spotlight 搜 Terminal，或在 VS Code 里按 ⌃` 打开内置终端。

```bash
cd ~/Code/hector-blog
pwd            # 确认位置是 /Users/hector/Code/hector-blog
```

**2. 先同步云端。**

```bash
git pull
```

如果你在别处（GitHub 网页、另一台电脑）改过东西，这一步把那些改动拉下来。没改过就是 `Already up to date.`，无害。先 pull 再动手，是避免后面 push 被拒绝的唯一习惯。

**3. 新建文章。**

```bash
hexo new post "world-models-notes"
```

引号里的是文件名，也是文章网址的一部分（`/2026/09/13/world-models-notes/`），所以用英文和短横线。它生成 `source/_posts/world-models-notes.md`，顶部带好了 front-matter。文章的显示标题在文件里改，可以是中文。

**4. 写。** 用任何编辑器打开那个文件，例如 `code source/_posts/world-models-notes.md`。文件结构：

```markdown
---
title: 世界模型读书笔记
date: 2026-09-13 10:00:00
categories: AI
tags: [world model, JEPA]
---

这一段会出现在首页摘要里。

<!-- more -->

正文从这里开始。
```

两条 `---` 之间是 front-matter，给 Hexo 看的元数据；下面是正文。`<!-- more -->` 以上的部分作为首页摘要。图片放 `source/images/`，正文写 `![说明](/images/xxx.png)`——路径以 `/images/` 开头，因为文章的网址在 `/2026/09/13/...` 这样的子路径下，相对路径会指错，`/` 开头表示从网站根目录算。

**5. 本地预览。**

```bash
hexo server
```

浏览器打开 `http://localhost:4000`。`localhost` 指你这台电脑，预览服务器只在本机运行，别人看不到；改文件会自动刷新。看完在终端按 Ctrl+C 停掉。

**6. 看改了什么。**

```bash
git status
```

红色列出的就是新建或修改过、但还没被选中的文件。

**7. 选中、打快照、上传。**

```bash
git add -A
git commit -m "post: world models notes"
git push
```

`-A` 是全部改动；`-m` 后面的说明写给未来的自己，简短即可；`git push` 输出末尾出现 `main -> main` 就是成功。

**8. 等一分钟，刷新网站。** 想看过程：Cloudflare 后台 → Workers & Pages → hector-blog → Deployments，一条新构建从 Initializing 走到 Deploying。

熟练之后就是三行：

```bash
cd ~/Code/hector-blog && git pull && hexo new post "slug"
hexo server                       # 写完看一眼
git add -A && git commit -m "post: slug" && git push
```

**会遇到的报错：**

| 提示 | 原因 | 处理 |
|---|---|---|
| `! [rejected] ... fetch first` | 云端有你本地没有的提交 | `git pull` 再 `git push` |
| `nothing to commit, working tree clean` | 没有改动 | 多半是编辑器忘了保存 |
| `hexo: command not found` | 不在项目文件夹，或这台电脑没装全局 hexo-cli | 用 `npx hexo server` |
| Cloudflare 构建红了 | 看 Deployments 日志是哪一段失败 | Installing 看锁文件，Building 看命令，Deploying 看 `wrangler.jsonc` |

**不想敲命令：** VS Code 左侧"源代码管理"面板做的是同一件事——改动旁点 **+**（add），上方写说明，点 **✓ Commit**，再点 **Sync Changes**（pull + push）。

## 方式 B：在 GitHub 网页上改

不需要电脑上有任何东西，手机也行，适合改错字和临时短文。

- **改现有文章**：`github.com` → 仓库 → `source/_posts` → 点文章 → 右上角铅笔 **Edit** → 改 → **Commit changes...** → 写说明 → 保持 **Commit directly to the main branch** → **Commit changes**。这一下在云端直接完成了 add + commit，没有 push，因为它本来就在 GitHub 上。
- **新文章**：`source/_posts` 页面 → **Add file** → **Create new file** → 文件名 `my-post.md` → 内容先粘 front-matter 再写正文 → Commit。
- **传图**：`source/images` → **Add file** → **Upload files** → 拖入 → Commit，正文用 `/images/文件名` 引用。

提交后 Cloudflare 照常构建。**唯一要记住的**：回到电脑上先 `git pull`，因为这次提交直接进了 GitHub，本地副本并不知道。GitHub 手机 App 的流程相同。

## 方式 C：用 Obsidian 写

Obsidian 把一个文件夹当作"库"，笔记就是文件夹里的 `.md`。把博客的 `source` 文件夹当库打开，`_posts` 里的文件就是文章——不存在"同步"，编辑的本来就是同一批文件。

1. Obsidian → **Open folder as vault** → 选 `~/Code/hector-blog/source`。
2. Settings → Files & Links：关掉 **Use [[Wikilinks]]**（Hexo 不认双方括号链接）；**Default location for new attachments** 选指定文件夹并填 `images`；**New link format** 选 Absolute path in vault。插图后它会写 `![](images/x.png)`，发布前在前面补一个 `/`。
3. 新文章仍推荐终端 `hexo new post "slug"` 生成，再回 Obsidian 写；front-matter 会显示为顶部的属性面板，分类标签直接在里面填。也可以用 Templates 核心插件，把模板放在 `source/_templates/post.md`——以 `_` 开头的文件夹 Hexo 会忽略。
4. 没写完的放 `_drafts/`，Hexo 不发布；写完拖到 `_posts/`。
5. 发布仍走方式 A 的三条命令或 VS Code 按钮。想在 Obsidian 内一键发布，装社区插件 Obsidian Git，命令面板里 "Commit-and-sync"；它要求库是仓库根目录，若报错就把整个 `~/Code/hector-blog` 当库打开，并在 Excluded files 里排除 `node_modules` 和 `public`。

`.obsidian/` 已在 `.gitignore` 里，那是界面状态文件，不该进仓库。

## 文章文件规范

| 字段 | 说明 |
|---|---|
| `title` | 显示标题，任何语言 |
| `date` | 决定排序和网址里的日期 |
| `categories` | 少而稳定，如 `AI`；写 `[AI, 世界模型]` 表示两级 |
| `tags` | 细粒度，可多个 |
| `mathjax: true` | 需要公式时加，默认不加载 MathJax |
| `permalink` | 一般不写；特殊页面（如 404）才指定 |

正文用标准 Markdown；代码块用三个反引号加语言名；`<!-- more -->` 控制摘要长度；草稿用 `hexo new draft "slug"`，预览加 `--drafts`，定稿 `hexo publish "slug"`。

## 本文是怎么发布的

方式 A。在项目文件夹里执行的命令，按顺序：

```bash
cd ~/Code/hector-blog
git pull
# 直接创建了 source/_posts/writing-workflow.md 并写入内容
npm run build            # 等价于 hexo generate，确认能生成、无报错
git add -A
git commit -m "Posts: engineering log 01 and writing workflow guide"
git push
```

推送后 Cloudflare 自动构建，约一分钟后这篇文章出现在首页。没有打开过 Cloudflare 后台，没有写过一行 HTML。
