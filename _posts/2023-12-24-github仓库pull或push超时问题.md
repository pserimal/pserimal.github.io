---
title: git 仓库 pull 或 push 超时问题
date: 2023-12-24
categories:
    - 杂类
tags:
- git
- 代理
---

github 在国内使用经常出现超时或者各种连接问题，在使用了科学上网工具后可能依然不好用，原因是 git 可能默认还是不走代理 [开clash后Git依然超时的解决方法](https://zhuanlan.zhihu.com/p/652905080)，需要手动设置

```bash
git config --global http.proxy http://127.0.0.1:7890
​
git config --global https.proxy https://127.0.0.1:7890
```
