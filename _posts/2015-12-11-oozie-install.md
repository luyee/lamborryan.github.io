---
layout: post
title: Oozie系列(1)之Oozie的安装
date: 2015-12-11 10:30:35
categories: 大数据
tags: Oozie
---

## 1.简介

Oozie是Hadoop生态自带的工作流调度工具, 它是基于MapReduce来进行任务启动的, 相比Linkin的Azkaban无疑笨重点, 本系列文章主要描述我调研Oozie的笔记记录。本文是系列1, 描述如何进行安装.

## 2.安装

### 2.1 安装环境

Linux Ubuntu 12.04.5 LTS (GNU/Linux 3.13.0-43-generic x86_64)

Hadoop 2.6.0

Oozie 4.1.0

### 2.2 JDK MAVEN安装

### 2.3 Oozie Server安装

1.下载oozie安装包，这里假设安装在/home/hadoop. 并解压

```shell
wget http://mirrors.cnnic.cn/apache/oozie/4.2.0/oozie-4.2.0.tar.gz
tar xzvf oozie-4.2.0.tar.gz
```

2.下载ExtJS工具包

```shell
wget http://extjs.com/deploy/ext-2.2.zip
```

3.构建oozie

```shell
cd oozie-4.2.0
bin/mkdistro.sh -DskipTests
```

构建完oozie后,可以在oozie-4.2.0/distro/target目录下看到构建后的文件，例如我的路径
/home/hadoop/oozie/distro/target/oozie-4.1.0-distro/oozie-4.1.0, 内容如下:

```shell
hadoop@hadoop-master:~/oozie/distro/target/oozie-4.1.0-distro/oozie-4.1.0$ pwd
/home/hadoop/oozie/distro/target/oozie-4.1.0-distro/oozie-4.1.0
hadoop@hadoop-master:~/oozie/distro/target/oozie-4.1.0-distro/oozie-4.1.0$ ls

bin   docs.zip  ext-2.2.zip  libext    logs                       oozie-core             oozie-server                 oozie.war

conf  examples  lib          libtools  oozie-client-4.1.0.tar.gz  oozie-examples.tar.gz  oozie-sharelib-4.1.0.tar.gz  release-log.txt
```

4.配置环境变量，修改~/.profile

```shell
export OOZIE_HOME=/home/hadoop/oozie/distro/target/oozie-4.1.0-distro/oozie-4.1.0
export PATH="$PATH:$OOZIE_HOME/bin"
```

5.在$OOZIE_HOME下新建libext目录,将ext-2.2.zip copy到libext下，将```mysql-connector-*.jar``` copy到libext下，将$HADOOP_HOME目录下的jar包移到libext中

```shell
cp -r $HADOOP_HOME/share/hadoop/common/*.jar  $OOZIE_HOME/libext
cp -r $HADOOP_HOME/share/hadoop/hdfs/*.jar  $OOZIE_HOME/libext
cp -r $HADOOP_HOME/share/hadoop/mapreduce/*.jar  $OOZIE_HOME/libext
cp -r $HADOOP_HOME/share/hadoop/tools/lib/*.jar  $OOZIE_HOME/libext
cp -r $HADOOP_HOME/share/hadoop/common/lib/*.jar  $OOZIE_HOME/libext
```

需要注意的是$OOZIE_HOME/oozie-server/lib 存放着tomcat启动时候的依赖包，所以需要除去$OOZIE_HOME/libext中重复的jar包，否则会出现包冲突导致页面显示不起来。

6.构建war包```oozie-setup.sh prepare-war```, 上述命令会在$OOZIE_HOME/oozie-server/webapps生成war包

7.配置oozie

修改oozie-site.xml配置文件

```xml
<property>
    <value>abc</value>
    <description>DB user name.</description>
</property>

<property>
    <name>oozie.service.JPAService.jdbc.password</name>
    <value>efg</value>
    <description>
        DB user password.
        IMPORTANT: if password is emtpy leave a 1 space string, the service trims the value,if empty Configuration assumes it is NULL.
    </description>
</property>
```

如果设置oozie.service.JPAService.create.db.schema为true，那么在启动oozie会自动创建oozie数据库，如果设置false则支持手动创建，还可以增加权限。

```xml
<property>
    <name>oozie.service.JPAService.create.db.schema</name>
    <value>true</value>
    <description>
        Creates Oozie DB.
        If set to true, it creates the DB schema if it does not exist. If the DB schema exists is a NOP.
        If set to false, it does not create the DB schema. If the DB schema does not exist it fails start up.
    </description>
</property>
```

修改Hadoop的配置文件core-site.xml,因为我用的账号是hadoop

```xml
<!-- OOZIE -->

<property>
     <name>hadoop.proxyuser.hadoop.hosts</name>
     <value>*</value>
</property>

<property>
     <name>hadoop.proxyuser.hadoop.groups</name>
     <value>*</value>
</property>
```

还需要配置share lib， 否则会一直报错, 运行该命令后会在/user/hadoop/share/lib下生成一个

```shell
oozie-setup sharelib create -fs /user/hadoop/share/lib
hadoop@hadoop-master:~/ruancf$ hadoop fs -ls /user/hadoop/share/lib

Found 1 items

drwxr-xr-x   - hadoop supergroup          0 2015-07-27 12:05 /user/hadoop/share/lib/lib_20150727120533

hadoop@hadoop-master:~/ruancf$ hadoop fs -ls /user/hadoop/share/lib/lib_20150727120533

Found 8 items

drwxr-xr-x   - hadoop supergroup          0 2015-07-27 12:05 /user/hadoop/share/lib/lib_20150727120533/distcp

drwxr-xr-x   - hadoop supergroup          0 2015-07-27 12:05 /user/hadoop/share/lib/lib_20150727120533/hcatalog

drwxr-xr-x   - hadoop supergroup          0 2015-07-27 12:05 /user/hadoop/share/lib/lib_20150727120533/hive

drwxr-xr-x   - hadoop supergroup          0 2015-07-27 12:05 /user/hadoop/share/lib/lib_20150727120533/mapreduce-streaming

drwxr-xr-x   - hadoop supergroup          0 2015-07-27 12:05 /user/hadoop/share/lib/lib_20150727120533/oozie

drwxr-xr-x   - hadoop supergroup          0 2015-07-27 12:05 /user/hadoop/share/lib/lib_20150727120533/pig

-rw-r--r--   3 hadoop supergroup       1354 2015-07-27 12:05 /user/hadoop/share/lib/lib_20150727120533/sharelib.properties

drwxr-xr-x   - hadoop supergroup          0 2015-07-27 12:05 /user/hadoop/share/lib/lib_20150727120533/sqoop
```

修改oozie-site.xml，见http://hadooptutorial.info/oozie-share-lib-does-not-exist-error/

```shell
<property>
    <name>oozie.service.HadoopAccessorService.hadoop.configurations</name>
    <value>*=/home/hadoop/hadoop/etc/hadoop</value>
    <description>
        Comma separated AUTHORITY=HADOOP_CONF_DIR, where AUTHORITY is the HOST:PORT of
        the Hadoop service (JobTracker, HDFS). The wildcard '*' configuration is
        used when there is no exact match for an authority. The HADOOP_CONF_DIR contains
        the relevant Hadoop *-site.xml files. If the path is relative is looked within
        the Oozie configuration directory; though the path can be absolute (i.e. to point
        to Hadoop client conf/ directories in the local filesystem.
    </description>
</property>

<property>
    <name>oozie.service.WorkflowAppService.system.libpath</name>
    <value>hdfs:///user/${user.name}/share/lib</value>
    <description>
        System library path to use for workflow applications.
        This path is added to workflow application if their job properties sets
        the property 'oozie.use.system.libpath' to true.
    </description>
</property>
```

>If we do not specify the oozie.libpath as shown above, by default oozie will refer to the local file system directories and it will not be able to find the share lib in LFS and it will throw this error message. And even hadoop configurations will not be setup properly in oozie-site.xml by default.

>We have to change these two properties to point to correct hadoop configurations directory and hdfs share lib path as shown below. Here i have to provide HADOOP_CONF_DIR value to first property as shown below. (do not skip ```*=```) and provide hdfs file system qualifier as hdfs:// for the second property.

8.启动oozie, ```./oozie-start.sh```

9.如果可以浏览器打开```http://hadoop-master:11000/oozie/``` 则说明安装成功。

10.oozie server是自带oozie client的，如果其他服务器要使用oozie client，只需要解$OOZIE_HOME的oozie client的tar.gz包并配置path即可。

### 2.4 HUE集成OOZIE

1.修改HUE的配置文件hue.ini

```shell
###########################################################################
# Settings to configure liboozie
###########################################################################
[liboozie]
  # The URL where the Oozie service runs on. This is required in order for
  # users to submit jobs. Empty value disables the config check.
  oozie_url=http://localhost:11000/oozie
  # Requires FQDN in oozie_url if enabled
  ## security_enabled=false
  # Location on HDFS where the workflows/coordinator are deployed when submitted.
  ## remote_deployement_dir=/user/hue/oozie/deployments
```

2.由于HUE的账户名是HUE，所以还需要在OOIZE设置代理, 修改oozie-site.xml。否则会报错```JA006: Call From hadoop-master/10.1.10.204 to hadoop-master:10020 failed on connection exception: :java.net.ConnectException: Connection refused; For more details see: http://wiki.apache.org/hadoop/ConnectionRefused```

```xml
<property>
    <name>oozie.service.ProxyUserService.proxyuser.hue.hosts</name>
    <value>*</value>
    <description>
        List of hosts the '#USER#' user is allowed to perform 'doAs'
        operations.
        The '#USER#' must be replaced with the username o the user who is
        allowed to perform 'doAs' operations.
        The value can be the '*' wildcard or a list of hostnames.
        For multiple users copy this property and replace the user name
        in the property name.
    </description>
</property>

<property>
    <name>oozie.service.ProxyUserService.proxyuser.hue.groups</name>
    <value>*</value>
    <description>
        List of groups the '#USER#' user is allowed to impersonate users
        from to perform 'doAs' operations.
        The '#USER#' must be replaced with the username o the user who is
        allowed to perform 'doAs' operations.
        The value can be the '*' wildcard or a list of groups.
        For multiple users copy this property and replace the user name
        in the property name.
    </description>
</property>
```

## 3. 总结

Oozie确实太笨重了, 看看安装配置过程就知道了。


本文完



* 原创文章，转载请注明： 转载自[Lamborryan](<http://www.lamborryan.com>)，作者：[Ruan Chengfeng](<http://www.lamborryan.com/about/>)
* 本文链接地址：http://www.lamborryan.com/oozie-install
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
