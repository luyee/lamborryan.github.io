---
layout: post
title: Hive数据仓库之权限操作
date: 2015-10-29 00:29:30
categories: 大数据
tags: Hive
---
# Hive数据仓库之权限操作

#一. 简介

* 像MySQL一样, Hive也可以实现较细颗粒度的权限管理, 比如可以赋予某个用户CREATE, SELECT, ALTER, DELETE等不同的权限, 也可以对不同的用户和组分配权限, 但是由于Hive是NoSQL, 所以他有自身的一些特点和缺陷。
* Hive的安全机制是为了防止用户误删, 而不是为了防止数据泄漏, 所以只要通过hive验证的用户都能够修改自己的权限。 假设root和user1都没有select ＊from table1 的权限, user1经过验证后就可以通过grant和revoke来修改root和user1自己的权限, 同样user1也可以修改root的权限。
* 不像Linux中root(管理员)和user1(普通用户), 在Hive中默认是没有管理员的, 也就是说root用户和user1的用户的地位是等同的, 所以root和user1可以互相修改对方的权限。在下一篇文章中, 将介绍如何通过代码来实现Hive的管理员。

#二. Hive的权限操作

上一节简单介绍了Hive的优点和不足, 本节开始将介绍Hive如何来控制权限.

##1. 配置

要实现权限的相关操作, 需要在hive-site.xml修改配置
{% highlight bash linenos %}
<property>
    <name>hive.security.authorization.enabled</name>
    <value>true</value>
    <description>enable or disable the hive client authorization</description>
</property>
<property>
    <name>hive.security.authorization.createtable.owner.grants</name>
    <value>ALL</value>
    <!--value>admin1,edward:select;user1:create</value-->
    <description>the privileges automatically granted to the owner whenever a table gets created. An example like "select,drop" will grant select and drop privilege to the owner of the table</description>
</property>
{% endhighlight bash %}
也可以在hive命令行中开启权限
{% highlight bash linenos %}
hive> set hive.security.authorization.enabled=true;
hive> CREATE TABLE auth_test (key int, value string);    
Authorization failed:No privilege 'Create' found for outputs { database:default}.    
Use show grant to get more details.
{% endhighlight bash %}

##2. 权限的细分

权限控制主要分为以下几个:

* ALTER         更改表结构，创建分区
* CREATE        创建表
* DROP          删除表，或分区
* INDEX         创建和删除索引
* LOCK          锁定表，保证并发
* SELECT        查询表权限
* SHOW_DATABASE 查看数据库权限
* UPDATE        为表加载本地数据的权限

Hive支持不同层次的权限控制, 从全局－> 数据库－> 数据表 -> 列 （-> 分区)。 注意有些权限比如drop等在列上不起作用。

语法:
{% highlight bash linenos %}
GRANT
    priv_type [(column_list)]
      [, priv_type [(column_list)]] ...
    [ON object_type]
    TO principal_specification [, principal_specification] ...
    [WITH GRANT OPTION]

REVOKE
    priv_type [(column_list)]
      [, priv_type [(column_list)]] ...
    [ON object_type priv_level]
    FROM principal_specification [, principal_specification] ...

REVOKE ALL PRIVILEGES, GRANT OPTION
    FROM USER [, USER] ...

object_type:
    TABLE
  ¦ DATABASE

priv_level:
    db_name
  ¦ tbl_name
{% endhighlight bash %}

##3. 操作举例

查看权限

* SHOW GRANT USER hadoop ON DATABASE default;    查看用户hadoop在数据库default的权限
* SHOW GRANT ON USER hadoop;   查看用户hadoop的所有权限
* SHOW GRANT;   查看所有用户的所有权限       

赋予权限

* grant create to user root;  赋予root所有database和table的create权限
* grant all to user root;  赋予root所有database和table的所有权限
* grant select on database default to user root; 赋予root用户default数据库的select权限
* grant drop on table default.test to user root; 赋予root用户删除default的test表权限
* grant select (id), select (name) on default.test to user root; 赋予root用户查看test的id和name列

删除权限          

* revoke create from user root;
* revoke all from user root;
* revoke select on database default from user root;
* revoke drop on table default.test from user root;
* revoke select (id), select (name) on default.test from user root;

##4. 用户,组,角色

当Hive里面用于N多用户和N多张表的时候，管理员给每个用户授权每张表会让他崩溃的。所以，这个时候就可以进行组(GROUP)授权。
{% highlight bash linenos %}
hive> CREATE TABLE auth_test_group(a int,b int);  
hive> SELECT * FROM auth_test_group;  
Authorization failed:No privilege 'Select' found for inputs  
{ database:default, table:auth_test_group, columnName:a}.  
Use show grant to get more details.  
hive> GRANT SELECT on table auth_test_group to group hadoop;
hive> SELECT * FROM auth_test_group;  
OK  
Time taken: 0.119 seconds  
{% endhighlight bash %}
当给用户组授权变得不够灵活的时候，角色(ROLES)就派上用途了。用户可以被放在某个角色之中，然后角色可以被授权。角色不同于用户组，是由Hadoop控制的，它是由Hive内部进行管理的。
{% highlight bash linenos %}
hive> CREATE TABLE auth_test_role (a int , b int);  
hive> SELECT * FROM auth_test_role;  
Authorization failed:No privilege 'Select' found for inputs  
{ database:default, table:auth_test_role, columnName:a}.  
Use show grant to get more details.  
hive> CREATE ROLE users_who_can_select_auth_test_role;  
hive> GRANT ROLE users_who_can_select_auth_test_role TO USER hadoop;  
hive> GRANT SELECT ON TABLE auth_test_role  TO ROLE users_who_can_select_auth_test_role;  
hive> SELECT * FROM auth_test_role;  
OK  
Time taken: 0.103 seconds  
{% endhighlight bash %}
group和role的权限操作只需将前面的user换成group或者role即可。

##5. 分区表的授权

默认情况下，分区表的授权将会跟随表的授权，也可以给每一个分区建立一个授权机制，只需要设置表的属性PARTITION_LEVEL_PRIVILEGE设置成TRUE:
{% highlight bash linenos %}
hive> CREATE TABLE authorize_part (key INT, value STRING) > PARTITIONED BY (ds STRING);
hive> ALTER TABLE authorization_part SET TBLPROPERTIES ("PARTITION_LEVEL_PRIVILEGE"="TRUE");
Authorization failed:No privilege 'Alter' found for inputs {database:default, table:authorization_part}.
Use show grant to get more details.
hive> GRANT ALTER ON table authorization_part to user edward; hive> ALTER TABLE authorization_part SET TBLPROPERTIES ("PARTITION_LEVEL_PRIVILEGE"="TRUE");
hive> GRANT SELECT ON TABLE authorization_part TO USER edward;
hive> ALTER TABLE authorization_part ADD PARTITION (ds='3');
hive> ALTER TABLE authorization_part ADD PARTITION (ds='4');
hive> SELECT * FROM authorization_part WHERE ds='3';
hive> REVOKE SELECT ON TABLE authorization_part partition (ds='3') FROM USER edward;
hive> SELECT * FROM authorization_part WHERE ds='3';
Authorization failed:No privilege 'Select' found for inputs
{ database:default, table:authorization_part, partitionName:ds=3, columnName:key}. Use show grant to get more details.
hive> SELECT * FROM authorization_part WHERE ds='4'; OK
Time taken: 0.146 seconds
{% endhighlight bash %}

##6. metastore

在hive metastore database里存放跟权限有关的是以下几张表, 这些表对应相应层次存储。

1. GLOBAL_PRIVS, 存放全局的权限, 比如grant select to user userA 对所有数据库都有
2. DB_PRIVS, 存放数据库级的权限
3. TBL_PRIVS, 存放表级的权限
4. TBL_COL_PRIVS, 存放表列级的权限
5. PART_PRIVS, 存放表分区的权限
6. PART_COL_PRIVS, 存放表分区的列的权限

#三. 总结

最后再重点强调下Hive的权限的缺陷。

1. 假设我有用户userA, 没有table default的select权限, 那么我可以通过jdbc客户端或者hive客户端等方式用grant命令赋予自己select权限
2. 用户userA 可以通过各种客户端来修改userB的权限, 反过来userB也可以修改userA
3. Hive内的user地位都是等同, 人人都是管理员。
4. 因此Hive的权限更多的是防止用户的误操作, 而不是真正权限管理

那么如何解决以上问题呢:

1. 在上一篇文章中提到使用custom认真, 防止其他用户通过客户端登入Hive
2. 将在下一篇文章中介绍, 实现AbstractSemanticAnalyzerHook接口, 当用户要进行相应权限操作时候, 在该接口中通过hook过滤掉我们设定的不具有相应权限的用户, 从而实现超级管理员。

本文完。
