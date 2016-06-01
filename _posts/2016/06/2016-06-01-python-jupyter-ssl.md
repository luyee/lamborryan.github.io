---
layout: post
title: Python之给Jupyter配置ssl
date: 2016-06-01 00:30:00
categories: 编程语言
tags: Python　
---

## 1. 简介

一直觉得Jupyter是个很优雅的东西，平时用她来处理一些数据处理的工作，非常好用。 但是当我把Jupyter部署在服务器上时就遇到个安全的问题。查阅资料发现可以通过配置ssl来设置密码登入。网上的一些给Jupyter设置ssl的方法我都没成功, 于是决定自己写一篇笔记。

## 2. Jupyter配置ssl

#### 1.生成登入密码

``` python
$python
>>>from IPython.lib import passwd
>>>passwd()
>>>Enter password
>>>Verify password
'sha1:408a945027ad:fec843e6f020d6c172a16b5ad89989e3c3175d99'
```

注意这里输入的密码就是在登入jupyter时候的密码。```sha1:408a945027ad:fec843e6f020d6c172a16b5ad89989e3c3175d99```是加密后的密码。

#### 2.生成ssl的证书

```shell
cd ~/.jupyter

openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout mycert.pem -out mycert.pem
```

这样就会在.jupyter目录下面生成证书```mycert.pem```

#### 3.生成jupyter_notebook_config.py

在```~/.jupyter```下面新建```jupyter_notebook_config.py```. 很多文章说在```~/.ipython```目录下，但是我测试后没效果。

在```jupyter_notebook_config.py```填写配置:

```python
c = get_config()
c.NotebookApp.certfile = u'/path/to/mycert.pem'  # 第二步生成的ssl证书目录，这里是~/.jupyter/mycert.pem
c.NotebookApp.ip = '0.0.0.0'                     # 服务器的ip。如果这里设置localhost,那只能本机访问。需要注意这里不同网卡绑定的ip
c.NotebookApp.password = u'sha1:408a945027ad:fec843e6f020d6c172a16b5ad89989e3c3175d99' # 第一步生成的加密后的密码
c.NotebookApp.port = 9999  # 端口号
```

#### 4.启动jupyter

启动```jupyter notebook```后就可以访问```x.x.x.x:9999```了, 这里需要注意下的是如果使用http访问会出现一下错误:

```bash
SSL Error on 6 ('115.239.228.14', 19716): [SSL: WRONG_VERSION_NUMBER] wrong version number (_ssl.c:590)
```

这是因为经过ssl加密需要https访问， 但是通过https访问会导致页面响应速度慢的问题。

![img](../image/python/jupyter-ssl/jupyter.png)

这时输入未加密的登入密码即可

## 3. 总结

本文主要介绍了如何给jupyter配置ssl登入， 如果是多人使用的话建议使用jupyterhub。



本文完



* 原创文章，转载请注明： 转载自[Lamborryan](<http://www.lamborryan.com>)，作者：[Ruan Chengfeng](<http://www.lamborryan.com/about/>)
* 本文链接地址：[http://www.lamborryan.com/python-jupyter-ssl/](<http://www.lamborryan.com/python-jupyter-ssl/>)
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
