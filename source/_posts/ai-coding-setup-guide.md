---
title: 给零基础同学的 AI 编程助手安装指南（macOS / Windows）
date: 2026-09-14 21:06:00
categories: Engineering
tags:
  - Claude Code
  - Codex
  - CC Switch
  - Stata
  - 教程
---

两份面向零基础用户的安装配置指南，覆盖 Claude Code、Codex、CC Switch、Skills 与 Stata 联动，macOS 与 Windows 各一份。做成了带侧栏标签与一键复制的固定画幅网页，可以一边翻一边照着敲。下文记录内容编排、几个设计决定，以及发布时踩到的坑。

<!-- more -->

## 指南本体

- **[macOS 版](/guides/setup-macos)**（55 页）
- **[Windows 版](/guides/setup-windows)**（53 页）

操作方式：`←` `→` 翻页，按 `T` 打开目录跳转，代码块右上角有「复制」按钮，`⌘P` 可打印。页面会记住上次看到第几页——装到一半被打断是常态。

## 为什么写

起因是几个非计算机专业的同学要配这套工具，Windows 和 Mac 都有，且完全没有命令行经验。现成的教程要么假设读者知道什么是 PATH，要么只覆盖单一平台。

内容按"一页一个可执行动作"重排，不是把 Markdown 直接倒进幻灯片：

| 章节 | 页数 | 要点 |
|---|---:|---|
| 准备工作 | 6 | 网络、终端、全局关系图 |
| 装工具 | 8 | Homebrew / winget、Git、Node |
| 配模型 | 8 | CC Switch 与 DeepSeek 接入 |
| 装技能 | 7 | 插件、个人技能、Nature Skills |
| 跑 Stata | 9 | 批处理闭环、引力模型 Demo |
| 排错速查 | 5 | 按现象查表 |

## 几个设计决定

**把 CC Switch 讲成"换 SIM 卡"。** 这是全篇最需要解释的概念。最后用的比喻是：Claude Code 和 Codex 是两部手机，API Key 是 SIM 卡，模型厂商是运营商套餐，CC Switch 是能插多张卡的卡槽。这个比喻还顺带解释了最容易填错的地方——Claude Code 的接口地址要写 `/anthropic` 后缀，Codex 要写 `/v1`，因为「同一张卡插不同手机，信号制式不一样」。

**Stata 联动讲的是闭环，不是"AI 操作鼠标"。** 实际流程是 AI 写 `.do`、批处理运行、读 `.log`、报错就自己改了重跑。把这个闭环画成图，比讲十句都管用。配了一个引力模型的完整 Demo，数据是模拟生成的，不依赖外部文件——跑通它就说明整条链路没问题，能把环境问题和数据问题分开。

**复制按钮只剥提示符，不剥注释。** 最初的实现会把行尾注释一并去掉，对 shell 命令合理，但 `.do` 文件的注释是正文，剥掉就毁了它的教学价值。改成只去掉 `$` 和 `PS>`。

## 发布时踩到的两个坑

**Google Fonts 对目标读者不可达。** 这份教程的读者恰恰是还没配好代理的人——他们来读就是为了学怎么配。页面原本依赖 Google Fonts，直连会全部回退、排版垮掉。解决办法不是自托管（思源宋体几 MB，不划算），而是写足回退栈：

```css
--serif: 'Noto Serif SC','Songti SC','STSong','Source Han Serif SC','SimSun',serif;
--display: 'Bodoni Moda','Didot','Times New Roman','Songti SC',serif;
--sans: 'DM Sans','PingFang SC','Microsoft YaHei','Helvetica Neue',sans-serif;
--mono: 'JetBrains Mono','SF Mono',Menlo,Consolas,'Courier New',monospace;
```

拦掉 `fonts.googleapis.com` 实测过：落到系统宋体后结构、配色、框线全部保留，只是字形变化。这套设计的识别度本来就主要在结构上。

**HTML 要放在 Hexo 不渲染的地方。** 独立页面自己控制整个视口，套上主题的页眉页脚就废了。放 `source/guides/`，并在 `_config.yml` 里声明：

```yaml
skip_render:
  - 'guides/**'
```

这样 Hexo 原样拷贝到 `public/guides/`，不经 Markdown 处理、不套 layout。放 `_posts/` 会被当文章渲染，放 `source/images/` 则与资源目录的语义不符。

## 已知限制

- 固定 16:9 画幅，手机上会缩得比较小。这是为了保证每页排版可控做的取舍，横屏看没问题。
- 机场订阅需要读者自己解决，指南里只说明它是前提。
- Stata 的批处理调用在部分学校授权版本下可能被限制，指南里给了"只写 `.do`、自己在 Stata 里跑"的备用路径。
