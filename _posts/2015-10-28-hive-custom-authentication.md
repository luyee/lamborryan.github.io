---
layout: post
title: Hive如何使用Custom方式进行认证
date: 2015-10-28 13:34:30
categories: 大数据
tags: Hive
---
#Hive如何使用Custom方式进行认证

##一.简介
Hive默认情况下是不需任何认证就可以访问HiveServer2的，这种情况下显然不适合生产环节的。

* Hive具有三种用户登入认证方式:
*    1.LDAP Authentication using OpenLDAP
*    2.Setting up Authentication with Pluggable Access Modules
*    3.Configuring Custom Authentication

本文主要介绍第三种Custom方式。

* To implement custom authentication for HiveServer2, create a custom Authenticator class derived from the following interface:

要实现Custom方式的认证，需要实现一下接口:

{% highlight java linenos %}
public interface PasswdAuthenticationProvider {
  /**
   * The Authenticate method is called by the HiveServer2 authentication layer
   * to authenticate users for their requests.
   * If a user is to be granted, return nothing/throw nothing.
   * When a user is to be disallowed, throw an appropriate {@link AuthenticationException}.
   *
   * For an example implementation, see {@link LdapAuthenticationProviderImpl}.
   *
   * @param user - The username received over the connection request
   * @param password - The password received over the connection request
   * @throws AuthenticationException - When a user is found to be
   * invalid by the implementation
   */
  void Authenticate(String user, String password) throws AuthenticationException;
}
{% endhighlight java %}

##二.接口实现
以下是完整的代码实例：

{% highlight java linenos %}
package com.lamborryan.authentication;
import javax.security.sasl.AuthenticationException;
import org.apache.commons.configuration.ConfigurationException;
import org.apache.commons.configuration.PropertiesConfiguration;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.apache.hadoop.conf.Configurable;
import org.apache.hadoop.conf.Configuration;
import org.apache.hive.service.auth.PasswdAuthenticationProvider;
public class CustomPasswdAuthenticator implements PasswdAuthenticationProvider,Configurable {
    private static final Log LOG=LogFactory.getLog(CustomPasswdAuthenticator.class);
    private Configuration conf=null;
    private static final String HIVE_JDBC_AUTH_CONFIG="hive.jdbc.auth.config";
    private static final String HIVE_JDBC_PASSWD_AUTH_PREFIX="hive.jdbc_passwd.auth.%s";
    public CustomPasswdAuthenticator() {
        init();
    }
    /**
     *
     */
    public void init(){
    }
    @Override
    public void Authenticate(String userName, String passwd)
            throws AuthenticationException {
        LOG.info("user: "+userName+" try login.");
        String confPath = getConf().get(HIVE_JDBC_AUTH_CONFIG);
        PropertiesConfiguration authConfig;
        try {
            authConfig = new PropertiesConfiguration(confPath);
        } catch (ConfigurationException e) {
            String message = "Error load auth config " + confPath ;
            throw new AuthenticationException(message);
        }
        LOG.info("load success conf " + confPath);
        String passwdConf = authConfig.getString(String.format(HIVE_JDBC_PASSWD_AUTH_PREFIX, userName));
        if(passwdConf==null){
            String message = "user's ACL configration is not found. user:"+userName;
            LOG.info(message);
            throw new AuthenticationException(message);
        }
        if(!passwd.equals(passwdConf)){
            String message = "user name and password is mismatch. user:"+userName;
            throw new AuthenticationException(message);
        }
        LOG.info("user "+userName+" login system successfully.");
    }
    @Override
    public Configuration getConf() {
        if(conf==null){
            this.conf=new Configuration();
        }
        return conf;
    }
    @Override
    public void setConf(Configuration arg0) {
        this.conf=arg0;
    }
}
{% endhighlight java %}

* 以上代码实现了一个hook:
* 1.在进行认证的时候从配置文件中读取用户名和密码，并判断是否当前的用户名密码一致。
* 2.hive-site.xml里面hive.jdbc.auth.config项的值存的是存放用户名和密码的文件路径，如此就可以不用在线更新用户名密码。

##三.配置

###1.maven依赖配置

{% highlight bash linenos %}
<dependencies>
     <dependency>
         <groupId>org.apache.hive</groupId>
         <artifactId>hive-service</artifactId>
         <version>0.14.0</version>
     </dependency>
     <dependency>
         <groupId>org.apache.hadoop</groupId>
         <artifactId>hadoop-common</artifactId>
         <version>2.6.0</version>
     </dependency>
 </dependencies>
{% endhighlight bash %}

将编译好的包存放入 ${HIVE_HOME}/lib下面

###2.修改hive-site.xml

{% highlight bash linenos %}
 <property>
  <name>hive.security.authorization.enabled</name>
  <value>true</value>
  <description>enable or disable the hive client authorization</description>
</property>
<property>
  <name>hive.security.authorization.createtable.owner.grants</name>
  <value>ALL</value>
  <description>the privileges automatically granted to the owner whenever a table gets created. An example like "select,drop" will grant select and drop privilege to the owner of the table</description>
</property>
<property>
  <name>hive.server2.authentication</name>
  <value>CUSTOM</value>
</property>
<property>
  <name>hive.server2.custom.authentication.class</name>
  <value>com.lamborryan.authentication.CustomPasswdAuthenticator</value>
</property>
<property>
    <name>hive.jdbc.auth.config</name>
    <value>/kiss/configs/hive/auth.properties</value>
</property>
<property>
  <name>hive.server2.enable.doAs</name>
  <value>true</value>
</property>
{% endhighlight bash %}

###3.设置用户名和密码

每当用户进行认证，都会读取hive.jdbc.auth.config的值，比如这里的/kiss/configs/hive/auth.properties，该配置文件存放了用户名和对应的密码

{% highlight bash linenos %}
hive.jdbc_passwd.auth.admin=AAAAAAAAAAAA
hive.jdbc_passwd.auth.lamboray=BBBBBBBBBB
{% endhighlight bash %}

需要注意以下配置项

{% highlight bash linenos %}
<property>
  <name>hive.server2.enable.doAs</name>
  <value>true</value>
</property>
{% endhighlight bash %}

该配置使得hive server会以提交用户的身份去执行语句，如果设置为false，则会以起hive server daemon的admin user来执行语句。

##四.验证

使用python pyhs2客户端来进行验证

{% highlight bash linenos %}
import pyhs2
with pyhs2.connect(host='127.0.0.1',
                   port=10000,
                   authMechanism="PLAIN",
                   user='lamborryan',
                   password='BBBBBBBBBB',
                   database='default',
                   timeout=30000) as conn:
    with conn.cursor() as cur:
         cur.execute("select * from default.test")
         print cur.fetchmany(100)
{% endhighlight bash %}

如果user和password错误就登入失败了。

##五.总结

该方法是个比较简单且有效的方法。本文到此结束
