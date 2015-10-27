---
layout: post
title: Mac Install MySQL-python
date: 2015-10-27 19:34:30
categories: Python
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
