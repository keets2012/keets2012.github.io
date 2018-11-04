---
title: 如何写好git commit log
categories: Utils
tags:
  - Git
  - utils
abbrlink: 48993
date: 2018-05-16 00:00:00
---
代码差异（diff）可以告知改动的内容，但只有提交信息才能正确地告诉你为什么（why）。Git仓库的贡献者知道，和后续开发者（事实上未来就是他们自己）沟通一个改动的上下文（context），最好方法就是通过一个好的 git 提交信息。

如果你对如何写好 git 提交信息没有仔细想过，那你很可能没有怎么使用过 git log 和相关工具。这里有一个恶性循环：因为提交的历史信息组织混乱而且前后矛盾，那后面的人也就不愿意花时间去使用和维护它。 又因为没有人去使用和维护它，提交的信息就会一直组织混乱和前后矛盾。

## 什么是commit log？
每次提交代码到Git时，需要描述本次提交的内容，如下：

```bash
$ git log
commit ca82a6dff817ec66f44342007202690a93763949
Author: Scott Chacon <schacon@gee-mail.com>
Date:   Mon Mar 17 21:52:11 2008 -0700

    changed the version number

commit 085bb3bcb608e1e8451d4b2432f8ecbe6306e7e7
Author: Scott Chacon <schacon@gee-mail.com>
Date:   Sat Mar 15 16:40:33 2008 -0700

    removed unnecessary test code

commit a11bef06a3f659402fe7563abf99ad00de2209e6
Author: Scott Chacon <schacon@gee-mail.com>
Date:   Sat Mar 15 10:31:28 2008 -0700

    first commit
```
## 为什么要写commit log？
一个项目的长期成功依赖于（除了其它方面）它的可维护性，一个维护者最有力的工具就是项目的日志。所以非常值得花时间学习如何正确地维护它。刚开始可能很麻烦，但很快会成为习惯，最终会成为人们自豪和生产力的源泉。

- 加快 Reviewing Code 的过程
- 帮助我们写好 release note
- 帮你快速回忆某个分支，tag 或者 commit 增加了什么功能，改变了哪些代码
- 让其他的开发者在运行 git blame 的时候想跪谢
- 总之一个好的提交信息，会帮助你提高项目的整体质量


## 如何写commit log？

一个团队提交日志的方法应该一致 。为了创建一个有用的修改历史，团队应该首先对提交信息的约定形成共识，至少明确以下三件事情：

- 风格：包含句语、自动换行间距、文法、大小写、拼写。最终的结果就是一份相当一致的日志，不仅让人读起来很爽，而且可以定期阅读；

- 内容：哪些信息应该包含在提交信息的正文中，哪些不用；

- 元数据：引用问题跟踪 ID，pull 请求编号等

综合别人总结出的五条git提交规则：

1. 用一个空行隔开标题和正文
2. 限制标题字数在 50 个字符内
4. 不要用句号结束标题行
5. 在标题行使用祈使语气
7. 使用正文解释是什么和为什么，而不是如何做

不太好的例子 `git commit -m "Fix login bug"`。而一个推荐的 commit message 应该是这样：

```
[develop]Redirect user to the requested page after login

https://trello.com/path/to/relevant/card

Users were being redirected to the home page after login, which is less
useful than redirecting to the page they had originally requested before
being redirected to the login form.

* Store requested path in a session variable
* Redirect to the stored location after successfully logging in the user
```
如果有对应的issue或者链接，可以提供上，message开头最好标注提交的类型，如修复bug、新开发某个功能、紧急修复等。

### git emoji
在知乎上看到有支持的 git emoji表情，通俗易懂，参考链接为[emoji表情](https://gitmoji.carloscuesta.me/)表情。

![](http://ovcjgn2x0.bkt.clouddn.com/git-emoji.jpg)

更多表情请参考emoji官网，示例提交一个试试：`:bug: fix a bug writtten by pig teammate`。效果如下：

![](http://ovcjgn2x0.bkt.clouddn.com/emoji-commit.jpg)
