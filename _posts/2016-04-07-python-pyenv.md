---
layout: post
title: Python之pyenv进行多版本管理
date: 2016-04-07 13:30:00
categories: 编程语言
tags: Python
---

## 1. 简介

使用Python的人一定会为各种Python的版本控制烦恼, 不过幸好有pyenv为我们提供了方便。Pyenv是python的版本管理工具，pyenv之于python相当于rvm之于ruby。

## 2. 配置安装

在Mac OS X下，我用homebrew安装pyenv：

```shell
brew update
brew install pyenv
```

要使改变马上生效，运行一下

```shell
$SHELL -l
```

至此，pyenv的配置也完成了。

来看看pyenv的几个命令

1. ```pyenv install -l``` 查看可以安装的版本;
2. ```pyenv versions``` 已经安装的python版本;
3. ```pyenv version``` 当前使用的版本;
4. ```pyenv install <version>``` 安装某一版本;
5. ```pyenv rehash``` 安装某一版本后需要使用该命令来刷新数据;
6. ```pyenv global/local <version>``` 运行global命令会切换全局的python版本;而local命令则会在当前目录下创建.python_version，管理当前目录及其子目录（子目录没有.python_version的情况下）的python版本。

比如我默认的启动python是2.7.2, 当我们使用```pyenv global 3.5.1```后现在指向了3.5.1

```shell
% pyenv versions  
  system
* 3.5.1(set by /Users/rcf/.pyenv/version)
```

但是当我运行python --version 却还显示的是2.7.2. 这是因为需要在.zshrc中添加一行配置(我用的是zsh)

```shell
if which pyenv > /dev/null; then eval "$(pyenv init -)"; fi
```

如此查看python版本已经正常

```shell
[2] % python --version
Python 3.5.1
```
以后需要切换版本只需要使用```pyenv global/local <version>```.

## 3. 使用中遇到的问题

### 3.1 在pycharm中设置

我们查看现在使用的python的路径

```shell
% which python
/Users/rcf/.pyenv/shims/python
```

但是pycharm默认使用的python却是```/Library/Frameworks/Python.framework/Versions```中的python.

所以需要在```Files—>default setting-->Default project—>project interpreter``` 中add local, /Users/rcf/.pyenv/shims/python.

### 3.2 #!/usr/bin/env与#!/usr/bin/python的区别

> #!/usr/bin/python是告诉操作系统执行这个脚本的时候，调用/usr/bin下的python解释器;
> #!/usr/bin/env python这种用法是为了防止操作系统用户没有将python装在默认的/usr/bin路径里。当系统看到这一行的时候，首先会到env设置里查找python的安装路径，再调用对应路径下的解释器程序完成操作。
> #!/usr/bin/python相当于写死了python路径;
> #!/usr/bin/env python会去环境设置寻找python目录,推荐这种写法。

使用了pyenv之后会, pyenv的python在/usr/bin/python前面了。

```shell
PATH=/Users/rcf/.pyenv/shims:/Users/rcf/Soft/maven/bin:/Library/Java/JavaVirtualMachines/jdk1.8.0_77.jdk/Contents/Home/bin:/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/bin:/opt/X11/bin
```

所以以后再用#!/usr/bin/python就会出现版本不匹配了。建议使用#!/usr/bin/env python。 或者
```shell
mv /usr/bin/python /usr/bin/python2.7.10
ln -s /usr/bin/python /Users/rcf/.pyenv/shims/python
```

本文完



* 原创文章，转载请注明： 转载自[Lamborryan](<http://lamborryan.github.io>)，作者：[Ruan Chengfeng](<http://lamborryan.github.io/about/>)
* 本文链接地址：http://lamborryan.github.io/python-pyenv
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
