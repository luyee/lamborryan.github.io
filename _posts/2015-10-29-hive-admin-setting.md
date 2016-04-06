---
layout: post
title: Hive数据仓库之超级管理员实现
date: 2015-10-31 00:29:30
categories: 大数据
tags: Hive
---
# Hive数据仓库之超级管理员实现

##一. 简介
Hive默认情况下是没有超级管理员的, hive的user和group其实对应的是linux的user和group, 也就是说任何一个用户都可以修改自身的权限以及别的用户的权限。本文将介绍如何通过hive的hook来实现超级管理员权限，主要的思路就是除了管理员可以进行grant, revoke这些操作外，其他用户不能使用这些命令。

##二. 实现

要实现超级管理员需要实现AbstractSemanticAnalyzerHook接口, 以下代码实现功能:只有hive-hite.xml里配置的hive.admin.username用户才能使用权限相关的操作。
{% highlight java linenos %}
package com.lamboryan.authentication;
import org.apache.hadoop.hive.ql.parse.ASTNode;
import org.apache.hadoop.hive.ql.parse.AbstractSemanticAnalyzerHook;
import org.apache.hadoop.hive.ql.parse.HiveParser;
import org.apache.hadoop.hive.ql.parse.HiveSemanticAnalyzerHookContext;
import org.apache.hadoop.hive.ql.parse.SemanticException;
import org.apache.hadoop.hive.ql.session.SessionState;
/**
 * 设置超级管理员
 * Created by ruanchengfeng on 15/10/23.
 */
public class AuthHook extends AbstractSemanticAnalyzerHook {
    private static String admin = "root";
    private static String HIVE_ADMIN_USERNAME = "hive.admin.username";
    @Override
    public ASTNode preAnalyze(HiveSemanticAnalyzerHookContext context, ASTNode ast) throws SemanticException {
        String adminName = context.getConf().get(HIVE_ADMIN_USERNAME);
        if (adminName == null){
            adminName = admin;
        }
        switch (ast.getToken().getType()) {
            case HiveParser.TOK_CREATEDATABASE:
            case HiveParser.TOK_DROPDATABASE:
            case HiveParser.TOK_CREATEROLE:
            case HiveParser.TOK_DROPROLE:
            case HiveParser.TOK_GRANT:
            case HiveParser.TOK_REVOKE:
            case HiveParser.TOK_GRANT_ROLE:
            case HiveParser.TOK_REVOKE_ROLE:
                String userName = null;
                if (SessionState.get() != null
                        && SessionState.get().getAuthenticator() != null) {
                    userName = SessionState.get().getAuthenticator().getUserName();
                }
                if (!adminName.equalsIgnoreCase(userName)) {
                   throw new SemanticException(userName + " can't use ADMIN options, except " + adminName + ".");
                }
                break;
            default:
                break;
        }
        return ast;
    }
}
{% endhighlight java %}

需要在hive-site.xml配置以下内容
{% highlight bash linenos %}
<property>
    <name>hive.semantic.analyzer.hook</name>
    <value>com.paitao.xmlife.authentication.AuthHook</value>  
</property>
<property>
    <name>hive.admin.username</name>
    <value>root</value>
</property>
{% endhighlight bash %}

查看效果, 当我尝试用bmw账户去修改权限时候提示不行
{% highlight bash linenos %}
0: jdbc:hive2://localhost:10000> show tables;
+---------------------+--+
|      tab_name       |
+---------------------+--+
| individuals         |
| mongo_hadoop_test   |
| mongo_hadoop_test2  |
| src                 |
| test                |
+---------------------+--+
5 rows selected (0.953 seconds)
0: jdbc:hive2://localhost:10000> grant select on table test to user bmw;
Error: Error while compiling statement: FAILED: SemanticException bmw can't use ADMIN options, except root. (state=42000,code=40000)
{% endhighlight bash %}

##三. 总结

本文简单的介绍了如何用hive接口实现超级管理员。

本文完


* 原创文章，转载请注明： 转载自[Lamborryan](<lamborryan.github.io>)，作者：[Ruan Chengfeng](<http://lamborryan.github.io/about/>)
* 本文链接地址：http://lamborryan.github.io/hive-admin-setting
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
