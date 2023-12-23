---
title: 使用 Jekyll NexT 主题搭建 github page 博客
date: 2023-12-24
categories:
- Linux
- blogs
---

[Jekyll](https://jekyllrb.com/) 可以很方便地用来搭建 github page 静态博客，也是 github 官方推荐的工具 [About GitHub Pages and Jekyll](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/about-github-pages-and-jekyll) ，之前偶然看见有人使用的 next 主题十分简洁，但 NexT 主题是 Hexo 下的主题，好在几年前有人将其移植到了 Jekyll 上 -> [jekyll-theme-next](https://github.com/Simpleyyt/jekyll-theme-next)

## 环境准备

1. 首先需要准备好 ruby 环境，经过多次尝试和失败，推荐使用 rvm 或者其他的 ruby 版本管理工具安装 ruby 2.3.8 版本，我使用的 ubuntu 系统可以参考 [ubuntu_rvm](https://github.com/rvm/ubuntu_rvm) 进行安装
2. git clone [jekyll-theme-next](https://github.com/Simpleyyt/jekyll-theme-next) 到本地

```bash
git clone https://github.com/Simpleyyt/jekyll-theme-next.git
```
3. 按照 [jekyll-theme-next 文档](http://theme-next.simpleyyt.com/getting-started.html) 进行操作，可能会遇到很多依赖之类的问题，按照前文使用 ruby 2.3.8 会较为顺利
4. 操作完成后建议使用 `bundle exec jekyll server -w --host=0.0.0.0` 命令进行启动进行本地预览，直接使用 `bundle exec jekyll server` 会只有本机能预览（默认运行在 127.0.0.1），局域网内其他主机无法访问

## 添加文章

直接在 `_post` 目录下添加文章即可，注意文件名前需要带日期，格式为 `yyyy-MM-dd`，如 `2023-12-24-使用Jekyll搭配NexT主题搭建githubpage.md`

markdown 文件需要添加头信息

```yaml
---
title: 使用 Jekyll NexT 主题搭建 github page 博客
date: 2023-12-24
categories:
    - linux
    - blogs
tags:
---
```

分别表示文章标题、日期、分类、标签
