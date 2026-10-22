---
layout: post
title: "时区问题导致博客不在目录中"
author: "qp91mn64"
created: "2026-10-23"
---

# 时区问题导致博客不在目录中

今天（2026-10-23）凌晨写好的博客，2026-10-23-读书笔记_语言学的邀请_1.md，标题是《〈语言学的邀请〉读书笔记（1）》，推到远程仓库之后，就打开[qp91mn64.github.io](https://qp91mn64.github.io/)守着，刷新了几下都没出来，哪怕已经过去好几分钟了。

于是登录GitHub，找到个人博客对应的仓库，显示 `pages build and deployment #29` 已经完成，这就奇怪了——往常的博客都好好的，上一次遇到博客没有出现在目录的情况还是因为缺少 YAML 头，详见个人博客[FromtMatter](https://qp91mn64.github.io/2026/03/08/FrontMatter.html)。

接着找，在构建过程中找到了这个：

![Build-with-Jekyll](/assets/images/2026-10-23-014012-Build-with-Jekyll.png)

结果显示23号是未来日期，所以跳过？不会是时区问题？

于是打开 DeepSeek Harness，在默认工作区（目前没放任何文件）里面新建对话，放上截图问AI（DeepSeek-V4.1-Flash-1017）：

```
https://github.com/qp91mn64/qp91mn64.github.io/actions/runs/39823542388/job/119980746222，这上面看到的，结果最近一篇博客被跳过去了，怎么回事？与时区有关吗？
```

AI回答这个是时区问题，通过添加 `_config.yml`，写这些来处理：

```yaml
timezone: Asia/Shanghai
future: true
```

或者单篇绕开，在 fromt matter 里面写带偏移量的日期：

```yaml
date: 2026-10-23 01:35:00 +0800
```

选择前者，但是我不知道怎么写 `_config.yml`，接着问：

```
_config.yml怎么写呢？目前没这个
```

AI给出的方法是，直接在仓库根目录新建一个，放这两行即可，不需要什么主题，因为默认值有theme。

此外，AI也指出取舍是，不能定时发布。到时候如果要定时发布，再改回 `false`，用时区。

于是添加 `_config.yml`，就写AI给的两行，再提交（详见提交[9709ff2](https://github.com/qp91mn64/qp91mn64.github.io/commit/9709ff2975aa6d6554b4e090033df25ff2f82a4b)），推到远程，再刷新，总算能看到博客《〈语言学的邀请〉读书笔记（1）》了。

## 其他

AI顺便确定了目前的个人博客主题是默认的 `jekyll-theme-primer`，找到 [https://pages-themes.github.io/primer/](https://pages-themes.github.io/primer/)打开一看，还真是，最上方是GitHub上面的用户名，其余部分与经过渲染的Markdown类似，文章部分位于中部，两边是空白。
