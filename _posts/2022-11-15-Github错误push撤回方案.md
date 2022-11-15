---
title: "Github错误push撤回方法"
categories:
  - GitHub
tags:
  - GitHub
---

当我们在使用GitHub时，有时候遇到一些commit需要做补充提交，但是已经提交上去了，这种时候怎么撤回呢？

GitHub并不支持撤回commit，但是我们有两种方式实现撤回。

# 删除分支后再重新提交
这是最简单的做法，先把GitHub的分支删除了，然后在本地回退提交，重新提交到GitHub上。
# 强制提交
只适用于自己的分支，如果是公用的分支不建议这么做
```bash
git push -f origin last_known_good_commit:branch_name
```