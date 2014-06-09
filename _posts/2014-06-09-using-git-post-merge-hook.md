---
layout: post
title: git post-merge hook 的使用
---

人懒，有些 git 仓库希望一有更新就自动执行某个操作，试下写个脚本什么的好了。

之前用过的都是 server-side 的 hooks，这次稍有点不同，我希望在 client-side 完成这些操作。

我希望是在检测到某文件更新之后就自动执行某个操作，搜索一圈之后，终于找到[这个脚本](https://gist.github.com/sindresorhus/7996717)，照猫画虎一下就解决问题了。

有个点在于 ORIG_HEAD 的使用，于是 man gitrevisions 找到这么一句：

> ORIG_HEAD is created by commands that move your HEAD in a drastic way, to record the position of the HEAD before their operation, so that you can easily change the tip of the branch back to the state before you ran them.o

于是一切就都明了了～这么一来 merge 前后的内容都可以直接列出来了，同样对 pull 也有效果，因为 pull 中也 call 了一次 git merge。

然后我又不是很喜欢用 git diff-tree 这些看起来比较底层的命令，所以还是想个办法用了 git log：

```
git log --name-only  --pretty="format:" ORIG_HEAD..HEAD|sed '/^$/d'|sort -u
```

判断下更改的文件里面有没有我关心的，执行后续的步骤就是了～
