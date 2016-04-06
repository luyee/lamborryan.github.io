---
layout: post
title: Mac Install MySQL-python
date: 2015-10-27 19:34:30
categories: 编程语言
tags: Python
---
##一.用pip安装MySQL-python会爆出以下错误

{% highlight python linenos %}
sudo pip install MySQL-python

Traceback (most recent call last):
  File "setup.py", line 15, in <module>
    metadata, options = get_config()
  File "/home/software/MySQL-python-1.2.3/setup_posix.py", line 43, in get_config
    libs = mysql_config("libs_r")
  File "/home/software/MySQL-python-1.2.3/setup_posix.py", line 24, in mysql_config
    raise EnvironmentError("%s not found" % (mysql_config.path,))
EnvironmentError: mysql_config not found
{% endhighlight python %}

##二.如何解决

###1) 从pypi上下载MySQL-python包

{% highlight bash linenos %}
wget  https://pypi.python.org/packages/source/M/MySQL-python/MySQL-python-1.2.5.zip\#md5\=654f75b302db6ed8dc5a898c625e030c
{% endhighlight bash %}

###2) 解压

{% highlight bash linenos %}
unzip MySQL-python-1.2.5.zip
cd MySQL-python-1.2.5
{% endhighlight bash %}

###3) 查找mac中mysql_config的路径

{% highlight bash linenos %}
MySQL-python-1.2.5  sudo  find / -name mysql_config
Password:
find: /dev/fd/3: Not a directory
find: /dev/fd/4: Not a directory
/usr/local/mysql-5.1.73-osx10.6-x86_64/bin/mysql_config
{% endhighlight bash %}

由此可见mac中的mysql_config路径在/usr/local/mysql-5.1.73-osx10.6-x86_64/bin/mysql_config

###4) 修改setup_posix.py文件：

{% highlight bash linenos %}
mysql_config.path = "/usr/local/mysql-5.1.73-osx10.6-x86_64/bin/mysql_config"
{% endhighlight bash %}

###5) 安装

{% highlight bash linenos %}
sudo python setup.py install
{% endhighlight bash %}

自此就安装完毕。


* 原创文章，转载请注明： 转载自[Lamborryan](<http://lamborryan.github.io>)，作者：[Ruan Chengfeng](<http://lamborryan.github.io/about/>)
* 本文链接地址：http://lamborryan.github.io/mysql-python-mac
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
